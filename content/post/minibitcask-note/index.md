---
title: 从零撸一个迷你 Bitcask：日志结构存储与索引的一点直觉
description: 用 Go 实现单文件追加型 KV；顺带理清 Bitcask、LSM 与 B+ 树分别在优化什么、牺牲什么。
slug: minibitcask-note
date: 2026-04-28T12:00:00+08:00
categories:
    - 技术笔记
tags:
    - Go
    - 存储引擎
    - Bitcask
    - LSM
weight: 2
---

最近在补存储相关的直觉，顺手用 Go 写了一个 **极简 Bitcask**：单数据文件顺序追加 + 内存里一张 `map[key]offset`。代码量不大，却把「日志结构存储」里最核心的几件小事串起来了：**写只在尾部、读靠索引定位、删要靠墓碑、脏数据靠 merge 回收**。这篇既当实现笔记，也当给自己用的对照表：**Bitcask、LSM、B+ 树到底在各自的甜蜜区里做什么取舍**。

---

## Bitcask 在干什么（一句话）

把所有写入都当成 **只会增长的日志**：每次 `Put` 都在文件末尾追加一整条记录，内存索引记住「这个 key 最新一条在哪」。旧版本留在磁盘上不予理会，直到你做 **Merge** 把仍然活着的数据压实到新文件里。

这和「传统页式存储」最大的反差是：**写路径几乎不做原地更新**，工程上非常好懂；代价是磁盘里会有大量历史版本，必须周期性压缩。

---

## 实现里最关键的三段代码

下面片段来自个人练习项目（与生产库相比省略了校验、hint file、多文件等），只保留骨架，便于对照论文或 [mini-bitcask](https://github.com/rosedblabs/mini-bitcask) 这类参考实现去理解。

### 记录布局与编码

定长头部（10 字节）+ 变长 `key` / `value`，大端整数，便于顺序扫描和按 offset 随机读：

```go
// 一条记录在磁盘上的长度：10 字节头 + key + value
func (e *Entry) Size() int64 {
    return int64(10 + e.keySize + e.valueSize)
}

func (e *Entry) Encode() ([]byte, error) {
    buf := make([]byte, e.Size())
    binary.BigEndian.PutUint32(buf[0:4], e.keySize)
    binary.BigEndian.PutUint32(buf[4:8], e.valueSize)
    binary.BigEndian.PutUint16(buf[8:10], e.mark)
    copy(buf[10:10+e.keySize], e.key)
    copy(buf[10+e.keySize:], e.value)
    return buf, nil
}
```

### 启动时回放：重建内存索引

顺序读文件，遇到 `PUT` 就更新索引指向当前 offset；遇到 **DEL 墓碑** 就把 key 从索引里删掉——这样重启后删除不会「复活」：

```go
func (db *BitCask) parseAndLoad() error {
    var offset int64
    for {
        e, err := db.dbFile.read(offset)
        if err != nil {
            if err == io.EOF {
                break
            }
            return err
        }
        db.indexes[string(e.key)] = offset
        if e.mark == DEL {
            delete(db.indexes, string(e.key))
        }
        offset += e.Size()
    }
    db.dbFile.Offset = offset
    return nil
}
```

### 读路径：索引 + 一次定位读

单点查询在「索引已全内存」的前提下，理想情况是 **O(1) 哈希 + 一次磁盘读**（value 不大时体验接近纯内存）：

```go
func (db *BitCask) Get(key string) (string, error) {
    db.mu.RLock()
    defer db.mu.RUnlock()
    off, ok := db.indexes[key]
    if !ok {
        return "", ErrKeyNotFound
    }
    entry, err := db.dbFile.read(off)
    if err != nil {
        return "", err
    }
    return string(entry.value), nil
}
```

Merge 的实现思路无外乎两种：**按索引里存的 offset 把存活记录搬到新文件**（实现直接，读模式偏随机），或 **顺序扫旧文件、用「当前 offset 是否仍是该 key 的最新一条」过滤**（顺序 I/O 更友好）。工程上会在二者之间权衡，这里没有展开。

---

## Bitcask、LSM、B+ 树：它们在优化什么？

这三类并不是「谁绝对更强」，而是 **默认负载假设不同**。下面这张表适合面试前扫一眼，也适合对照自己的业务问一句：**热点是写还是读？要不要范围扫描？数据集能不能塞进内存索引？**

| 维度 | Bitcask（单文件日志型） | LSM-Tree（Level / Tiered…） | B+ 树（页式，典型 InnoDB） |
|------|-------------------------|------------------------------|-----------------------------|
| **写模型** | 顺序追加，写放大极低（不计 merge） | 分层合并，写放大与压缩策略强相关 | 原地/分裂更新页，随机写相对多 |
| **点查** | 全内存索引则极快；索引放不下就崩模型 | 可能跨 memtable + 多层 SST，读放大可控需设计 | 树高通常很低，单次查询稳定 |
| **范围扫描 / 顺序读** | 并非强项（哈希索引为主） | SST 有序时可高效范围查询（需迭代器） | **强项**：叶子链表顺序扫 |
| **后台维护** | Merge 回收空洞 | Compaction 决定读写曲线 | 分裂合并、缓冲刷盘 |
| **实现与运维心智** | 最简单，单机教学友好 | 组件多（WAL、flush、压缩策略…） | 缓冲池、锁、页缓存复杂度高 |
| **典型甜蜜区** | **全索引进内存**的嵌入式 KV、中小数据集 | **写密集**、愿意用后台合并换写入吞吐 | **通用 OLTP**、范围查询与事务页 |

几句个人理解，避免教条：

- **Bitcask** 最漂亮的一点是「写入语义简单」：**追加即真理**。但一旦数据量大到 **哈希索引无法常驻内存**，优势会断崖式缩减——这也是为何经典 Bitcask 论文场景往往是嵌入式、可控体量。
- **LSM** 不是在「打败 B+ 树」，而是在 **高写入吞吐与后台合并** 上做文章；代价是 compaction 峰值、读写放大和空间放大都要算账，调参空间大。
- **B+ 树** 赢在 **通用**：点查、范围、排序扫描都自然；代价是缓冲池与页级并发复杂，写入路径上的随机写与分裂合并让纯写入场景未必最优。

一句话串起来：**日志类（Bitcask / LSM）擅长把写变成顺序；页式树（B+）擅长把读变成局部性很好的页遍历。** 选型不是背名词，而是把你的访问模式画出来，看谁默认假设和你一致。

---

## 小结

实现迷你 Bitcask 的价值不在于造轮子本身，而在于亲手摸到：**追加日志 + 索引指针 + 墓碑删除 + _merge 压实** 这四件事如何咬合。写完再去看 LSM、B+ 树或工业级 KV，你会更容易分辨——每一层设计到底是在付哪种税、换哪种红利。

### 参考与延伸

- LSM 叙事清晰的一篇中文笔记：[博客园 · LSM 相关](https://www.cnblogs.com/johnnyzen/p/19235030)
- 参考实现：[rosedblabs/mini-bitcask](https://github.com/rosedblabs/mini-bitcask)


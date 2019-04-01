---
id: version-0.17-snapshot
title: Snapshot
original_id: snapshot
---
## 快照功能概述

状态、区块快照，将必需的区块链数据经高度压缩优化后存档，可在较短时间内同步恢复链数据。

将重要的状态、区块数据存放在数据库，分别优化压缩到单个文件。

下载该文件，解压后恢复，即可得到所需链数据。

注：当前快照功能还未经充分测试，暂不建议用于生产环境。

## 使用快照功能

使用快照功能包含：创建状态、区块快照文件；依据快照文件恢复状态、区块数据。

### 创建快照

假设当前工作目录为`../cita/target/install/`，执行如下快照命令：

```bash
$ cd test-chain/0
$ ../../bin/snapshot_tool -m snapshot
$ ls snap*
snap-chain.rlp  snap.rlp
```

快照命令行工具接受快照参数后，在节点当前目录下会生成两个快照文件：

`snap.rlp`、`snap-chain.rlp`，分别表示状态快照、区块快照。

### 快照恢复

使用快照文件恢复数据。

1. 将上述生成的快照文件拷贝到需恢复链数据的节点目录下
    
    ```bash
    $ cd test-chain/1
    $ cp ../0/snap.rlp ../0/snap-chain.rlp ./
    ```

2. 依快照文件快照恢复
    
    快照命令行工具接受恢复参数后，依据快照文件恢复数据。
    
    ```bash
    $ ../../bin/snapshot_tool -m restore
    ```
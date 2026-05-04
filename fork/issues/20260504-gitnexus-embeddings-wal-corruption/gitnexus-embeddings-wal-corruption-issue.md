# GitNexus Embeddings WAL Corruption Issue

## 问题概述

在 Windows 环境下使用 GitNexus 的 `--embeddings` 选项时，会导致 SQLite 数据库的 WAL (Write-Ahead Log) 文件损坏，使得所有后续的数据库操作失败。

## 环境信息

- **操作系统**: Windows 11 Pro 10.0.26200
- **Shell**: PowerShell
- **Node.js**: v22.19.0
- **GitNexus 版本**: 1.6.4-rc.48
- **项目路径**: `E:\Projects\WorkDataHubPro`
- **代码库规模**: 
  - 6,897 个符号
  - 10,118 条关系
  - 199 个集群
  - 105 个执行流

## 复现步骤

### 1. 初始状态
```bash
npx gitnexus status
```
输出显示索引正常：
```
Repository: E:\Projects\WorkDataHubPro
Indexed: 2026/5/4 12:17:22
Indexed commit: 5073632
Current commit: 5073632
Status: ✅ up-to-date
```

### 2. 尝试使用 embeddings
```bash
npx gitnexus analyze --embeddings --skills --verbose
```

索引过程看似成功完成（耗时 151.7s）：
```
Repository indexed successfully (151.7s)
6,898 nodes | 10,118 edges | 200 clusters | 105 flows
```

但会显示警告：
```
Semantic embeddings were generated without a VECTOR index; 
queries will use exact-scan fallback within the configured limit.
```

### 3. 尝试查询触发错误
```bash
npx gitnexus query "orchestrator" -r WorkDataHubPro --limit 5
```

返回空结果（异常）：
```json
{
  "processes": [],
  "process_symbols": [],
  "definitions": [],
  "timing": {...}
}
```

### 4. 尝试直接访问数据库
```bash
npx gitnexus cypher "MATCH (n:Symbol) RETURN count(n) as total" -r WorkDataHubPro
```

触发 WAL 损坏错误：
```json
{
  "error": "Runtime exception: Corrupted wal file. Read out invalid WAL record type."
}
```

### 5. 清理并重建（不使用 embeddings）
```bash
npx gitnexus clean --force
npx gitnexus analyze --skills
```

索引成功（耗时 10.1s），所有功能恢复正常。

## 错误表现

### 主要错误信息
```
Runtime exception: Corrupted wal file. Read out invalid WAL record type.
```

### 次要警告
```
GitNexus: FTS extension unavailable; continuing without FTS features. 
load-only policy: extension not pre-installed
```

### 影响范围
- ❌ `gitnexus query` - 返回空结果
- ❌ `gitnexus context` - 无法访问符号信息
- ❌ `gitnexus cypher` - 数据库查询失败
- ❌ 所有 MCP 工具（`mcp__gitnexus__*`）- 无法使用

## 对比测试结果

### 使用 `--embeddings` 的结果
| 指标 | 值 | 状态 |
|------|-----|------|
| 索引时间 | 151.7s | ⚠️ 慢 15 倍 |
| 符号数量 | 6,898 | ✅ |
| 关系数量 | 10,118 | ✅ |
| 集群数量 | 200 | ✅ |
| 查询功能 | 失败 | ❌ WAL 损坏 |
| Context 工具 | 失败 | ❌ WAL 损坏 |

### 不使用 `--embeddings` 的结果
| 指标 | 值 | 状态 |
|------|-----|------|
| 索引时间 | 10.1s | ✅ |
| 符号数量 | 6,897 | ✅ |
| 关系数量 | 10,118 | ✅ |
| 集群数量 | 199 | ✅ |
| 查询功能 | 正常 | ✅ |
| Context 工具 | 正常 | ✅ |

## 技术分析

### 可能的原因

1. **SQLite WAL 模式与 embeddings 写入冲突**
   - Embeddings 生成过程可能涉及大量并发写入
   - Windows 文件系统锁定机制可能与 WAL 模式不兼容

2. **VECTOR 索引缺失导致的回退逻辑问题**
   - 警告信息提示 "without a VECTOR index"
   - 可能在 exact-scan fallback 路径中存在 bug

3. **FTS 扩展缺失的连锁反应**
   - FTS (Full-Text Search) 扩展不可用
   - 可能影响 embeddings 的存储或检索逻辑

4. **Worker 线程超时或资源竞争**
   - Embeddings 生成使用多线程（ONNX）
   - 可能存在线程安全问题或死锁

### 相关配置选项

```bash
# Embeddings 相关选项
--embeddings                    # 启用 embedding 生成
--drop-embeddings               # 删除现有 embeddings
--embedding-threads <n>         # 限制 ONNX CPU 线程数
--embedding-batch-size <n>      # 每批次节点数
--embedding-sub-batch-size <n>  # 每次模型调用的块数
--embedding-device <device>     # 设备：auto, cpu, dml, cuda, wasm

# 环境变量
GITNEXUS_EMBEDDING_THREADS=N
GITNEXUS_SEMANTIC_EXACT_SCAN_LIMIT=N  # 默认 10000
```

## 临时解决方案

### 当前推荐配置
```bash
# 仅使用 BM25 文本搜索 + 技能生成
npx gitnexus analyze --skills
```

### 如果需要尝试 embeddings
```bash
# 尝试限制线程数和批次大小
npx gitnexus analyze --embeddings --skills \
  --embedding-threads 1 \
  --embedding-batch-size 10 \
  --embedding-sub-batch-size 5
```

## 需要的信息

为了进一步诊断和修复此问题，需要收集：

1. **SQLite 版本信息**
   - GitNexus 使用的 SQLite 版本
   - 是否启用了 WAL 模式
   - VECTOR 扩展的可用性

2. **Embeddings 实现细节**
   - 使用的 ONNX 模型
   - 写入 embeddings 的具体代码路径
   - 是否有事务管理

3. **Windows 特定问题**
   - 是否在 Linux/macOS 上也能复现
   - 文件锁定机制的差异

4. **日志和调试信息**
   - `--verbose` 模式下的完整输出
   - SQLite 的 WAL 文件状态
   - 崩溃时的堆栈跟踪

## 影响评估

### 功能影响
- **BM25 文本搜索**: ✅ 完全可用，性能良好
- **语义搜索**: ❌ 不可用（embeddings 损坏）
- **符号关系图**: ✅ 完全可用
- **执行流追踪**: ✅ 完全可用
- **技能生成**: ✅ 完全可用

### 用户影响
- 对于大多数代码搜索场景，BM25 已经足够
- 语义搜索的缺失主要影响概念性查询（如 "how to load data"）
- 精确符号查询和关系分析不受影响

## 相关链接

- GitNexus 仓库: https://github.com/gitnexus/gitnexus
- SQLite WAL 模式文档: https://www.sqlite.org/wal.html
- ONNX Runtime 文档: https://onnxruntime.ai/

## 下一步行动

1. ~~在 GitNexus 仓库提交 issue~~
2. ~~尝试在 Linux 环境复现~~
3. ~~收集详细的调试日志~~
4. ~~检查 SQLite WAL 文件的完整性~~
5. ~~审查 embeddings 写入代码的事务处理~~

---

## ✅ 问题已解决

**根本原因**: `lbug-adapter.ts` 中的 `closeLbug()` 函数在关闭数据库连接前没有执行 `CHECKPOINT` 命令，导致 LadybugDB 0.16.0 的非阻塞 checkpoint 线程在 close() 调用后仍在写入 WAL 文件，造成 WAL 损坏。

**解决方案**: 在 `lbug-adapter.ts` 的三个数据库关闭路径中添加了 `CHECKPOINT` 调用：
1. `closeLbug()` - 主清理函数
2. `doInitLbug()` - 数据库切换路径
3. `withLbugDb()` - 重试清理路径

**详细说明**: 参见 `fork/issues/gitnexus-embeddings-wal-corruption-fix.md`

**修复文件**: `gitnexus/src/core/lbug/lbug-adapter.ts`

---

**文档创建时间**: 2026-05-04  
**问题解决时间**: 2026-05-04  
**GitNexus 版本**: 1.6.4-rc.48  
**问题状态**: ✅ 已修复，等待测试验证

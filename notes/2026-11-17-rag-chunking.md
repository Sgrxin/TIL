# 2026-11-17 · RAG 切分策略：chunk_size 与 overlap 怎么取舍

做 rag-backend 时第一个要拍板的参数就是切分，今天把思路整理清楚。

## 为什么切分是 RAG 质量的地基

检索的最小单位是 chunk（块），不是整篇文档：

- **块太大** → 一块里混进多个主题，embedding 向量被"稀释"，检索时每块都不够像查询；命中后塞给 LLM 的上下文也变贵。
- **块太小** → 单块语义不完整，召回到一堆碎片，答案拼不起来。
- **边界切断** → 关键句正好被切在两块交界，哪块都不完整。

## overlap（重叠）是干嘛的

相邻块之间留一段重叠文本，专门兜底"边界切断"：被切断的句子至少会在其中一块里完整出现。代价是存储和索引体积变大，所以不能无限叠。

## 我项目里的取值

- chunk_size = 400 字，overlap = 80 字（20%），中文文档实测召回质量比较稳。
- 经验区间：中文 300~500 字/块，overlap 10%~20%，最终要拿自己的评测集调，没有万能值。

## 核心代码（TypeScript 滑窗切分）

```ts
export function slidingWindowSplit(
  text: string,
  size = 400,
  overlap = 80
): string[] {
  const chunks: string[] = [];
  const step = size - overlap; // 每次前进的步长
  for (let i = 0; i < text.length; i += step) {
    chunks.push(text.slice(i, i + size));
    if (i + size >= text.length) break; // 剩余不足一块时结束
  }
  return chunks;
}
```

## 面试一句话

> 切分不是越细越好：chunk_size 管语义完整性和成本，overlap 管边界切断，两者要在自己的评测集上权衡，而不是抄参数。

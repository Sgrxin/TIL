# 2026-11-28 · 混合检索：向量 + 关键词双路召回

RAG 系列第六篇。前天说"检索不准四板斧"，今天展开最有工程味的一板：混合检索。

## 向量检索的盲区

余弦相似度管"语义"，但有软肋：**精确词不敏感**。用户搜"OAPAC-2026-B1 议程"，型号/编号/人名这类**字面 token**，embedding 容易糊掉；反过来，传统关键词检索（BM25）恰好是字面精确匹配的强项，但"猫咖收费 ≈ 门店价格"这种同义改写它就抓瞎。

## 双路各取所长

```
              ┌→ 向量路（语义）→ top50 ┐
用户 query ──┤                        ├→ 融合（RRF）→ 最终 top5
              └→ BM25 路（字面）→ top50 ┘
```

**RRF（Reciprocal Rank Fusion）**是最常用的融合法：不比分数（两路分数量纲不同），只比排名——每条文档的得分 = Σ 1/(k + rank)，k 常取 60。实现只要几行，稳健又无需调参。

```ts
function rrf(rankings: string[][], k = 60) {
  const score = new Map<string, number>();
  for (const list of rankings)
    list.forEach((doc, i) =>
      score.set(doc, (score.get(doc) ?? 0) + 1 / (k + i + 1)));
  return [...score.entries()].sort((a, b) => b[1] - a[1]);
}
```

## 什么时候值得上

- 知识库含**编号、型号、专有名词、人名**多 → 必上
- 纯自然语义问答 → 向量单路够用，别过度设计

我的 rag-backend 升级路线图：混合检索 → rerank → 评估集护航，三步走。

## 面试一句话

> 向量管语义、BM25 管字面，两路召回用 RRF 按排名融合——混合检索用最小成本补齐了语义检索对精确词不敏感的盲区。

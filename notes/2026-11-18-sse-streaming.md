# 2026-11-18 · SSE 流式输出：RAG 问答为什么都爱用它

做智能问答小程序时，"回答像 ChatGPT 一样逐字打出"是我负责的小程序端核心交互。今天把 SSE 讲透。

## SSE 是什么

Server-Sent Events：基于普通 HTTP 长连接的**服务器单向推送**协议。客户端发一次请求，服务器不定时往回吐数据块，直到结束——正好匹配大模型"边生成边输出"的特性。

## 为什么不用 WebSocket / 轮询

- **轮询**：反复问"好了没"，延迟高、请求浪费
- **WebSocket**：双向独立协议，要维护连接和心跳——可流式回答本质是**单向**的，杀鸡用牛刀
- **SSE**：走普通 HTTP 生态、自带断线重连语义、实现最轻

## 小程序端实现三步

1. `wx.request` 开 `enableChunked: true`
2. 监听 `onChunkReceived`，逐块收到文本增量
3. 拼接 → `setData` 渲染 → scroll-view 滚到底

```js
const task = wx.request({
  url: '/api/ask', method: 'POST',
  enableChunked: true, // 关键开关
  data: { question },
  success: () => {},
});
task.onChunkReceived((res) => {
  const delta = decodeChunk(res.data); // ArrayBuffer -> 文本
  currentAnswer += delta;
  throttledSetData(); // 高频 setData 会卡，节流合并 ~100ms 一次
});
```

## 踩坑提醒

高频 `setData` 是小程序性能大坑：每个 chunk 都刷一次界面必卡。解法是节流合并，只在时间片到达时统一渲染。

## 面试一句话

> 流式回答是单向推送场景，SSE 用普通 HTTP 就能实现打字机效果，比 WebSocket 轻；小程序端的关键工程点是 enableChunked + onChunkReceived + setData 节流。

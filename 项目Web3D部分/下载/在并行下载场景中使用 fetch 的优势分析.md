# 在并行下载场景中使用 fetch 的优势分析

在这个并行下载实现中，选择使用 fetch 而不是 axios 或 XMLHttpRequest，主要有以下几个原因：

## 1. 流式处理能力

```typescript
const reader = response?.body?.getReader();
return new ReadableStream({
  async start(controller) {
    while (true) {
      const { done, value } = await reader?.read() || {}
      // ... 处理数据流
    }
  }
});
```

fetch API 提供了原生的流式处理能力：
- 可以通过 `response.body.getReader()` 获取 ReadableStream
- 支持逐块读取大文件，更好地控制内存使用
- 可以实时获取下载进度

## 2. Range 请求的原生支持

```typescript
fetch(url, {
  method: 'GET',
  headers: {
    Range: `bytes=${cursor}-${Math.min(cursor + chunkSize - 1, contentLength)}`,
  },
})
```

fetch 对 HTTP Range 请求有很好的原生支持：
- 可以直接在 headers 中设置 Range
- 便于实现文件分片下载
- 响应处理更直接

## 3. 取消请求的控制

fetch 提供了 `AbortController` 用于取消请求：
```typescript
const controller = new AbortController();
fetch(url, { signal: controller.signal });
```

## 4. 性能考虑

- fetch 是浏览器原生 API，不需要额外的库
- 直接操作底层数据流，减少中间转换
- 内存使用更高效

## 与其他方案对比

### 1. axios

```typescript
// axios 实现分片下载需要额外配置
axios.get(url, {
  headers: { Range: 'bytes=0-100' },
  responseType: 'arraybuffer'
});
```

缺点：
- 需要引入额外依赖
- 流式处理需要特殊配置
- 对大文件处理不如 fetch 原生支持好

### 2. XMLHttpRequest

```typescript
const xhr = new XMLHttpRequest();
xhr.open('GET', url);
xhr.setRequestHeader('Range', 'bytes=0-100');
```

缺点：
- API 较老，使用回调方式
- 代码冗长，不易维护
- 流式处理实现复杂

## 总结

在这个并行下载场景中，fetch 是最佳选择因为：
1. 原生支持流式处理
2. Range 请求实现简单
3. 内存使用更高效
4. 代码更简洁易维护
5. 不需要额外依赖

虽然项目中其他地方可能使用 axios，但在这种特殊的大文件并行下载场景，fetch 的优势更明显。

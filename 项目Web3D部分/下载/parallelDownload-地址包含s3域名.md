- 实现了并行下载文件的功能，大文件的分片下载，
- 服务器需要根据实际情况调整，服务器需要支持range请求

```js
export const parallelDownload = async (
  url: string,
  options?: { splitNumber?: number, onProcess?: (v: number, total: number) => void, }
) => {
  // 设置分片数量，默认4
  const SPLIT_NUM = options?.splitNumber || 4
  
  // 初始化一个数组用于记录每个分片的下载字节数
  const downloadedBytes = new Array<number>(SPLIT_NUM).fill(0)
  
  // 获取文件总大小
  const contentLength = await fetch(url, {
    method: 'GET',
    credentials: 'omit',
    // 发送一个Range请求（只请求0-0字节）来获取文件总大小
    headers: { Range: 'bytes=0-0' },
  }).then(resp => {
    // 从响应头Content-Range中解析文件总大小
    const contentLength = resp.headers.get('Content-Range')?.match(/\/(\d+)/)?.[1] || ''
    return parseInt(contentLength) || 0
  }).catch((err) => {
    // 如果获取文件大小失败，记录错误并返回0
    ConsoleUtil.error(err)
    return 0
  })
  
  // 如果成功获取文件大小
  if (contentLength > 0) {
    // 计算每个分片的大小
    const chunkSize = Math.ceil(contentLength / 4)
    let cursor = 0
    
    // 为每个分片创建一个下载任务
    const promises = downloadedBytes.map((v, index) => {
      // 计算当前分片的起始位置
      cursor = chunkSize * index
      
      // 发起分片下载请求
      return fetch(url, {
        method: 'GET',
        credentials: 'omit',
        headers: {
          // 设置Range头指定下载范围
          Range: `bytes=${cursor}-${Math.min(cursor + chunkSize - 1, contentLength)}`,
        },
      }).then(response => {
        // 获取响应体的ReadableStream
        const reader = response?.body?.getReader();

        // 创建新的ReadableStream来处理下载数据
        return new ReadableStream({
          async start(controller) {
            // 持续读取数据
            while (true) {
              // 读取数据块
              const { done, value } = await reader?.read() || {}
              
              // 如果读取完成，关闭流
              if (done) {
                controller.close()
                break
              }
              
              // 更新当前分片的已下载字节数
              downloadedBytes[index] += value?.byteLength || 0
              
              // 计算整体下载进度
              const progress = (downloadedBytes.reduce((acc, v) => acc + v, 0) / contentLength) * 100;
              
              // 调用进度回调函数
              options?.onProcess?.(progress, contentLength)
              
              // 将数据块加入流中
              controller.enqueue(value);
            }
          }
        });
      })
      // 将ReadableStream转换为Response对象
      .then(stream => new Response(stream))
      // 将Response转换为ArrayBuffer
      .then(response => response.arrayBuffer())
    })
    // ... 后续代码处理合并分片数据
    return Promise.all(promises)
      .then(chunks => {
        const zipName = getZipFileNameFromUrl(url, 'file.zip')
        const mergedArrayBuffer = mergeArrayBuffers(chunks)
        return new File([mergedArrayBuffer], zipName, { type: 'application/octet-stream' })
      })
  } else {
    throw 'Partial download failed'
  }
}
```
- 设置分片数量，默认是4
- 发送一个range请求只请求0-0字节额，来获取文件总共的大小，fetch，从响应头content-range当中解析文件总共大小的
- 计算每一个分片的大小，为每一个分片创建一个fetch请求，使用range头部指定下载的范围，使用readablestream处理下载流，实时更新下载的进度，为每一个分片数据转换成为ArrayBuffer
- 等待所有的分片下载完成proiseall使用mergeArrayBuffers函数合并所有的分片的数据，将合并后的数据创建为file对象返回，
## 跨域
credentials: 'omit' 是 Fetch API 中的一个配置选项，用于控制是否在请求中发送凭据（如 cookies、HTTP 认证等）。具体来说：

credentials: 'omit'：表示在请求中不发送任何凭据。即不会包含 cookies、HTTP 认证信息等。

credentials: 'include'：表示在请求中总是发送凭据，即使请求是跨域的。

credentials: 'same-origin'：表示仅在请求与目标 URL 同源时发送凭据。

在你的代码中，credentials: 'omit' 的作用是确保在发送请求时不会携带任何用户凭据，这通常用于以下场景：

跨域请求：当请求的目标 URL 与当前页面的域名不同时，避免发送不必要的 cookies 或其他敏感信息。
安全性：在某些情况下，你可能不希望将用户的身份验证信息（如 cookies）发送到第三方服务器。
## ReadableStream

使用 ReadableStream 在处理大文件或流式数据时有以下几个好处：

1. 内存效率
   
传统方式：如果使用传统的 fetch 和 response.arrayBuffer()，整个文件会被一次性加载到内存中。对于大文件，这可能导致内存占用过高，甚至引发内存不足的问题。

ReadableStream：ReadableStream 允许你以流式的方式处理数据，数据可以分块读取和处理，从而显著减少内存占用。
3. 实时处理
传统方式：必须等待整个文件下载完成后才能开始处理。
ReadableStream：可以在数据到达时立即处理，实现实时处理。例如，在下载过程中可以实时更新进度条或进行其他操作。
4. 更好的用户体验
传统方式：用户需要等待整个文件下载完成后才能看到结果。
ReadableStream：可以在数据到达时立即显示部分结果，提升用户体验。例如，视频或音频文件可以在下载过程中开始播放。
5. 支持中断和恢复
传统方式：如果下载中断，可能需要重新下载整个文件。
ReadableStream：可以更容易地实现断点续传，只需从上次中断的地方继续下载。
6. 灵活性
传统方式：处理方式较为固定，难以适应复杂的需求。
ReadableStream：提供了更高的灵活性，可以根据需要自定义数据处理逻辑。


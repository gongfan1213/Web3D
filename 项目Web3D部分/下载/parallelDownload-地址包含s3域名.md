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

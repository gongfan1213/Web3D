# fetchZipToSTLFile 函数详细解析

这个函数主要用于下载 ZIP 文件并解压出其中的 STL 文件，支持两种下载方式：并行分片下载和普通下载。让我们逐部分分析：

## 1. 函数参数说明

```typescript
export const fetchZipToSTLFile = async ({
  url,                    // ZIP文件的下载地址
  onProcess,             // 下载进度回调函数
  onDownloadFinish,      // 下载完成回调函数
  abortController        // 用于取消下载的控制器
})
```

## 2. ZIP 处理函数

```typescript
const processZip = async (blob: Blob) => {
  // 根据URL生成ZIP文件名
  const zipName = getZipFileNameFromUrl(url, 'file.zip')
  
  // 创建ZIP文件对象
  const zipFile = new File([blob], zipName, {
    type: 'application/octet-stream'
  });
  
  // 通知下载完成
  onDownloadFinish?.(zipFile)
  
  // 动态导入jszip库并解压文件
  const JSZip = await import('jszip')
  const zip = new JSZip.default();
  
  // 解压并处理STL文件
  return zip.loadAsync(blob)
    .then(zip => {
      // 筛选出所有STL文件
      const stlFiles = Object.keys(zip.files).filter(v => /\.stl$/i.test(v))
      // 将每个STL文件转换为File对象
      return Promise.all(stlFiles.map(v => {
        return zip.files[v].async('arraybuffer')
          .then(arrayBuffer => {
            return new File([arrayBuffer], v, {
              type: 'application/octet-stream'
            })
          })
      }))
    })
}
```

## 3. 下载逻辑

### 3.1 并行分片下载（S3）
```typescript
if (typeof url === 'string' && /\.s3\./.test(url)) {
  // 对S3链接使用并行下载
  parallelDownload(url, { onProcess })
    .then(zipFile => processZip(zipFile))
    .then(resolve)
    .catch(err => {
      reject('Not support parallel download')
    })
}
```

### 3.2 普通下载（非S3）
```typescript
fetch(url, { signal: abortController.signal })
  .then(response => {
    if (!response.ok) throw new Error('Get file failed');
    return response;
  })
  .then(response => {
    // 获取文件大小
    const contentLength = response.headers.get('content-length');
    let downloadedBytes = 0;

    // 使用流式读取
    const reader = response?.body?.getReader();
    
    // 创建可读流
    return new ReadableStream({
      async start(controller) {
        while (true) {
          const { done, value } = await reader?.read() || {}
          if (done) {
            controller.close()
            break
          }
          // 更新下载进度
          downloadedBytes += value?.byteLength || 0
          const progress = (downloadedBytes / Number(contentLength)) * 100;
          onProcess?.(progress, Number(contentLength))
          controller.enqueue(value);
        }
      }
    });
  })
```

## 4. 下载流程

1. **判断下载方式**：
   - 如果 URL 包含 `.s3.`，使用并行分片下载
   - 否则使用普通的 fetch 下载

2. **并行下载流程**：
   - 使用 `parallelDownload` 函数分片下载
   - 下载完成后调用 `processZip` 处理

3. **普通下载流程**：
   - 使用 fetch API 下载
   - 通过 ReadableStream 处理下载进度
   - 支持通过 abortController 取消下载

4. **ZIP 处理流程**：
   - 下载完成后解压 ZIP 文件
   - 提取所有 STL 文件
   - 将 STL 文件转换为 File 对象

## 5. 特点

1. **流式处理**：
   - 使用 ReadableStream 处理大文件
   - 实时反馈下载进度

2. **并行下载**：
   - 对 S3 链接支持并行分片下载
   - 提高大文件下载速度

3. **错误处理**：
   - 完整的错误处理链
   - 支持下载失败后降级

4. **可中断**：
   - 支持通过 AbortController 取消下载
   - 避免资源浪费

5. **进度回调**：
   - 提供下载进度回调
   - 支持 UI 进度展示

这个函数设计得比较完善，既考虑了性能（并行下载），又考虑了用户体验（进度展示）和错误处理，是一个很好的大文件下载和处理示例。





```js
/**
 * 下载zip，解压返回压缩文件内STL文件
 * @param url              上传地址
 * @param onProcess        获取下载进度回调
 * @param abortController  请求中断控制器
 * @param onDownloadFinish 下载完成回调
 * @returns {File}
 */
export const fetchZipToSTLFile = async ({
  url,
  onProcess,
  onDownloadFinish,
  abortController
}: {
  url: string, // 文件下载地址
  onProcess?: (v: number, total: number) => void, // 下载进度回调函数
  onDownloadFinish?: (file: File) => void, // 下载完成回调函数
  abortController: AbortController // 用于中断请求的控制器
}): Promise<File[] | void> => {
  // 处理下载的ZIP文件，解压并返回其中的STL文件
  const processZip = async (blob: Blob) => {
    // 从URL中获取ZIP文件名，如果无法获取则使用默认文件名
    const zipName = getZipFileNameFromUrl(url, 'file.zip')
    // 将Blob转换为File对象
    const zipFile = new File([blob], zipName, {
      type: 'application/octet-stream'
    });
    // 调用下载完成回调函数
    onDownloadFinish?.(zipFile)
    // 动态导入JSZip库
    const JSZip = await import('jszip')
    const zip = new JSZip.default();

    // 使用 loadAsync 方法加载 ZIP 数据
    return zip.loadAsync(blob)
      .then(zip => {
        // 过滤出ZIP文件中所有以.stl结尾的文件
        const stlFiles = Object.keys(zip.files).filter(v => /\.stl$/i.test(v))
        // 寻找并解压 STL 文件
        return Promise.all(stlFiles.map(v => {
          // 将每个STL文件解压为ArrayBuffer
          return zip.files[v].async('arraybuffer')
            .then(arrayBuffer => {
              // 将ArrayBuffer转换为File对象
              return new File([arrayBuffer], v, {
                type: 'application/octet-stream'
              })
            })
        }))
      })
  }

  // 返回一个Promise，处理下载逻辑
  return new Promise<File[]>((resolve, reject) => {
    // 判断URL是否支持分片下载（仅支持包含.s3.的域名）
    if (typeof url === 'string' && /\.s3\./.test(url)) {
      // 使用parallelDownload进行分片下载
      parallelDownload(url, { onProcess })
        .then(zipFile => processZip(zipFile)) // 下载完成后处理ZIP文件
        .then(resolve) // 解析Promise
        .catch(err => {
          reject('Not support parallel download') // 如果分片下载失败，返回错误
        })
    } else {
      // 如果不支持分片下载，直接返回错误
      reject('Not support parallel download')
    }
  }).catch(() => fetch(url, { signal: abortController.signal }) // 如果分片下载失败，尝试普通下载
    .then(response => {
      // 检查响应是否成功
      if (!response.ok) {
        throw new Error('Get file failed');
      }
      return response;
    })
    .then(response => {
      // 获取文件总大小
      const contentLength = response.headers.get('content-length');
      let downloadedBytes = 0;

      // 获取响应体的ReadableStream
      const reader = response?.body?.getReader();

      // 创建一个新的ReadableStream来处理下载数据
      return new ReadableStream({
        async start(controller) {
          while (true) {
            // 读取数据块
            const { done, value } = await reader?.read() || {}
            if (done) {
              // 如果读取完成，关闭流
              controller.close()
              break
            }
            // 更新已下载字节数
            downloadedBytes += value?.byteLength || 0
            // 计算下载进度并调用回调函数
            const progress = (downloadedBytes / Number(contentLength)) * 100;
            onProcess?.(progress, Number(contentLength))
            // 将数据块加入流中
            controller.enqueue(value);
          }
        }
      });
    })
    .then(stream => new Response(stream).blob()) // 将ReadableStream转换为Blob
    .then(processZip) // 处理ZIP文件
    .catch(error => {
      // 捕获并记录错误
      ConsoleUtil.error('error:', error);
    })
  )
}
```
优先尝试使用 parallelDownload 进行分片下载，适用于大文件。

普通下载：如果分片下载失败，则回退到普通下载方式。

进度监控：通过 onProcess 回调函数实时更新下载进度。

中断支持：使用 AbortController 支持中断下载。

ZIP 解压：使用 JSZip 库解压 ZIP 文件，并提取其中的 STL 文件。


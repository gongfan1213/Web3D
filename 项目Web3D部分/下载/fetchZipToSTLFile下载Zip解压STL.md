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


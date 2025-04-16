# unzipMake.worker.ts
`<mcfile name="unzipMake.worker.ts" path="d:\code\shixicode\src\templates\UploadMake\unzipMake.worker.ts"></mcfile>` 是一个 Web Worker 文件，主要用于在后台线程中处理 ZIP 文件的解压和内容提取。以下是该文件的主要功能和工作流程：

1. **主要功能**：
   - 解压 ZIP 文件
   - 处理 ZIP 文件中的配置文件
   - 处理 ZIP 文件中的文本文件
   - 处理 ZIP 文件中的缩略图
   - 将处理结果返回给主线程

2. **核心模块**：
   - `JSZip`: 用于解压 ZIP 文件
   - `handleConfig`, `handleTxt`, `handleThumbnail`: 自定义的处理函数

3. **工作流程**：
   - 接收主线程传递的消息，获取需要解压的文件
   - 使用 JSZip 加载并解压文件
   - 并行处理配置文件、文本文件和缩略图
   - 将处理结果通过 `postMessage` 返回给主线程

4. **关键代码分析**：
```typescript:d:\code\shixicode\src\templates\UploadMake\unzipMake.worker.ts
onmessage = (e) => {
  const file = e.data.file;
  const zip = new JSZip.default();

  zip.loadAsync(file.originFile)
    .then((zipContent) => Promise.all([
      handleConfig(zipContent, file),
      handleTxt(zipContent, file),
      handleThumbnail(zipContent, file)
    ]))
    .then(([configData, txtData, thumbnailData]) => {
      postMessage({
        message: 'success',
        file,
        configData,
        txtData,
        thumbnailData
      })
    })
    .catch((error) => {
      postMessage({
        message: 'error',
        file,
      })
    });
}
```

5. **特点**：
   - 使用 Web Worker 实现后台处理，避免阻塞主线程
   - 使用 Promise.all 实现并行处理
   - 提供错误处理机制
   - 通过 postMessage 与主线程通信

这个文件主要用于处理包含 3D 打印相关文件的 ZIP 包，提取其中的配置、文本和缩略图信息，并将处理结果返回给主线程。这种设计可以提高应用的响应性能，特别是在处理大文件时。


# unzip.worker.ts

`<mcfile name="unzip.worker.ts" path="d:\code\shixicode\src\templates\Upload\unzip.worker.ts"></mcfile>` 是一个 Web Worker 文件，主要用于在后台线程中处理 ZIP 文件的解压和内容提取。以下是该文件的详细功能讲解：

---

### **主要功能**
1. **解压 ZIP 文件**：
   - 使用 `JSZip` 库加载并解压用户上传的 ZIP 文件。
   - 支持异步操作，避免阻塞主线程。

2. **提取和处理文件内容**：
   - 从 ZIP 文件中提取配置文件、文本文件和缩略图。
   - 使用自定义的 `handleConfig`、`handleTxt` 和 `handleThumbnail` 函数处理提取的内容。

3. **返回处理结果**：
   - 将处理后的数据通过 `postMessage` 返回给主线程。
   - 如果解压或处理失败，返回错误信息。

---

### **核心代码解析**
```typescript:d:\code\shixicode\src\templates\Upload\unzip.worker.ts
onmessage = (e) => {
  const file = e.data.file;
  const zip = new JSZip.default();
  
  zip.loadAsync(file.originFile)
    .then((zipContent) => Promise.all([
      handleConfig(zipContent, file),
      handleTxt(zipContent, file),
      handleThumbnail(zipContent, file)
    ]))
    .then(([configData, txtData, thumbnailData]) => {
      postMessage({
        message: 'success',
        file,
        configData,
        txtData,
        thumbnailData
      })
    })
    .catch((error) => {
      // 解压缩失败
      postMessage({
        message: 'error',
        file,
      })
    });
}
```

1. **接收消息**：
   - 通过 `onmessage` 监听主线程发送的消息，获取需要解压的文件。

2. **加载 ZIP 文件**：
   - 使用 `JSZip.loadAsync` 异步加载 ZIP 文件。

3. **并行处理文件**：
   - 使用 `Promise.all` 并行处理配置文件、文本文件和缩略图。
   - 每个处理函数（`handleConfig`、`handleTxt`、`handleThumbnail`）负责提取和解析特定类型的内容。

4. **返回结果**：
   - 如果处理成功，通过 `postMessage` 返回包含配置、文本和缩略图数据的对象。
   - 如果失败，返回错误信息。

---

### **特点**
1. **异步处理**：
   - 使用 `Promise` 和 `async/await` 实现异步操作，避免阻塞主线程。

2. **模块化设计**：
   - 将不同文件类型的处理逻辑分离到独立的函数中，便于维护和扩展。

3. **错误处理**：
   - 捕获解压和处理过程中的错误，并返回明确的错误信息。

4. **与主线程通信**：
   - 通过 `postMessage` 将处理结果返回给主线程，实现线程间通信。

---

### **适用场景**
- 处理包含 3D 打印相关文件的 ZIP 包（如 `.3mf` 文件）。
- 提取配置文件（如 `Slic3r_PE.config`）和打印参数。
- 提取缩略图用于预览。

---

### **总结**
`unzip.worker.ts` 是一个高效的后台解压工具，专门用于处理 ZIP 文件中的特定内容。通过 Web Worker 实现后台操作，确保主线程的流畅性，同时通过模块化设计提高了代码的可维护性和扩展性。


# screen.worker.ts

`<mcfile name="screenshot.worker.ts" path="d:\code\shixicode\src\templates\Upload\screenshot.worker.ts"></mcfile>` 是一个 Web Worker 文件，主要用于在后台线程中加载 3D 模型并生成截图。以下是该文件的详细功能讲解：

---

### **主要功能**
1. **加载 3D 模型**：
   - 支持加载 `.stl` 和 `.obj` 格式的 3D 模型文件。
   - 使用 Babylon.js 的 `SceneLoader.ImportMesh` 方法加载模型。

2. **模型处理**：
   - 调整模型的中心点，使其居中显示。
   - 设置模型的材质和颜色，确保渲染效果一致。

3. **场景设置**：
   - 创建默认相机，并调整相机的视角和限制。
   - 添加光照，确保模型在场景中可见。

4. **渲染截图**：
   - 在离屏 canvas 上渲染模型。
   - 通过 `setTimeout` 等待渲染完成，然后返回结果。

5. **与主线程通信**：
   - 使用 `postMessage` 将截图结果或错误信息返回给主线程。

---

### **核心代码解析**
```typescript:d:\code\shixicode\src\templates\Upload\screenshot.worker.ts
onmessage = (e) => {
  const file = e.data.file;
  const canvas = e.data.canvas;
  const CLEAR_COLOR = Babylon.Color4.FromHexString('#44474A')

  async function load(url, scene, canvas, options, onFinish) {
    await new Promise<void>(resolve => {
      Babylon.SceneLoader.ImportMesh(
        '',
        '',
        url,
        scene,
        (meshes) => {
          meshes.forEach(mesh => {
            const boundingInfo = mesh.getBoundingInfo()
            // 修改模型的中心点
            mesh.setPivotPoint(
              boundingInfo.boundingBox.maximumWorld
                .add(boundingInfo.boundingBox.minimumWorld)
                .divide(new Babylon.Vector3(2, 2, 2)),
            )
          })
          resolve()
        },
        (e) => { ConsoleUtil.log('progress', e.total, e.loaded) },
        (err) => {
          ConsoleUtil.error(err)
          postMessage({ message: 'error', file })
        },
      )
    })

    if (!scene.activeCamera) {
      scene.createDefaultCamera(true, true, true)
    }

    // 控制相机
    const camera = scene.activeCamera
    camera.upperBetaLimit = null
    camera.lowerBetaLimit = null
    camera.beta = Math.PI / 4

    // 添加光照
    const light = new Babylon.HemisphericLight('hl', Babylon.Vector3.Zero(), scene)
    light.groundColor = Babylon.Color3.Black()
    light.intensity = 1

    // 设置材质
    const meshes = scene.meshes
    scene.clearColor = options?.clearColor ?? CLEAR_COLOR
    meshes.forEach((mesh) => {
      if (mesh.material === null) {
        const standardMaterial = new Babylon.StandardMaterial('default', scene)
        mesh.material = standardMaterial
      }
      const material = mesh.material
      material.backFaceCulling = false
      material.twoSidedLighting = true
      material.diffuseColor = Babylon.Color3.FromHexString('#C8BCB7')
      material.specularColor = Babylon.Color3.White()
      material.useObjectSpaceNormalMap = true
      material.specularPower = 0.5
    })

    onFinish(scene, scene.activeCamera || null)
  }

  if (file.preview) {
    postMessage(file.preview)
  } else {
    const engine = new Babylon.Engine(canvas, true)
    const scene = new Babylon.Scene(engine)
    load(
      file.originFile,
      scene,
      canvas,
      {
        angle: Math.PI / 1.2 + Math.PI,
        fileExtension: file.name.endsWith('.stl') ? '.stl' : '.obj'
      },
      (scene, camera) => {
        engine.runRenderLoop(() => {
          scene.render()
        })
        setTimeout(() => {
          postMessage({ message: 'success', file })
        }, 1000)
      },
    )
  }
}
```

---

### **工作流程**
1. **接收消息**：
   - 通过 `onmessage` 监听主线程发送的消息，获取需要处理的文件和 canvas。

2. **加载模型**：
   - 使用 `Babylon.SceneLoader.ImportMesh` 加载模型文件。
   - 调整模型的中心点，使其居中显示。

3. **设置场景**：
   - 创建默认相机，并调整相机的视角和限制。
   - 添加光照，确保模型在场景中可见。
   - 设置模型的材质和颜色。

4. **渲染截图**：
   - 在离屏 canvas 上渲染模型。
   - 通过 `setTimeout` 等待渲染完成，然后返回结果。

5. **返回结果**：
   - 如果处理成功，通过 `postMessage` 返回截图结果。
   - 如果失败，返回错误信息。

---

### **特点**
1. **后台处理**：
   - 使用 Web Worker 在后台线程中处理模型加载和渲染，避免阻塞主线程。

2. **模块化设计**：
   - 将模型加载、场景设置和渲染逻辑分离到独立的函数中，便于维护和扩展。

3. **错误处理**：
   - 捕获加载和渲染过程中的错误，并返回明确的错误信息。

4. **与主线程通信**：
   - 通过 `postMessage` 将处理结果返回给主线程，实现线程间通信。

---

### **适用场景**
- 生成 3D 模型的预览截图。
- 在后台处理复杂的 3D 渲染任务，确保主线程的流畅性。

---

### **总结**
`screenshot.worker.ts` 是一个高效的后台渲染工具，专门用于加载 3D 模型并生成截图。通过 Web Worker 实现后台操作，确保主线程的流畅性，同时通过模块化设计提高了代码的可维护性和扩展性。



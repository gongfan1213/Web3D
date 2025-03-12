# ModelPreview 组件的 load 函数详细解析

这个 load 函数是 ModelPreview 组件的核心加载逻辑，用于加载和渲染 3D 模型。让我们分步骤详细分析：

## 1. 函数参数说明

```typescript
export async function load(
  url: string | ArrayBufferView | File[],    // 模型文件的URL或数据
  scene: Scene,                              // Babylon场景对象
  canvas: HTMLCanvasElement,                 // 画布元素
  options: LoadOptions | null,               // 加载选项
  onFinish: (scene: Scene, camera?: ArcRotateCamera | null) => void,  // 加载完成回调
  loadFn?: (scene: Scene, options: LoadOptions | null, camera?: ArcRotateCamera) => void,  // 自定义加载函数
  onLoadProgress?: (progress: number, total: number) => void,  // 加载进度回调
)
```

## 2. 文件加载处理

### 2.1 S3文件处理
```typescript
if (typeof url === 'string' && /(\.s3\.)|(\.cloudfront\.)/.test(url)) {
  // 对S3或CloudFront的URL使用并行下载
  const stlFile = await parallelDownload(url, { 
    splitNumber: 4, 
    onProcess: onLoadProgress 
  })
  if (stlFile) {
    urlOrBuffer = [stlFile]
  }
}
```

### 2.2 多文件加载
```typescript
if (Array.isArray(urlOrBuffer)) {
  await Promise.all(urlOrBuffer.map((file) => {
    return new Promise<void>(resolve => {
      SceneLoader.ImportMesh(
        '',           // meshName：空字符串表示加载所有网格
        '',           // 文件路径前缀
        file,         // 文件对象
        scene,        // 场景对象
        (meshes => {  // 加载成功回调
          // 处理模型命名
          const name = file.name.replace(/^.+_/, '').replace(
            options?.fileExtension ? options.fileExtension : '.stl', 
            ''
          )
          // 设置每个网格的属性
          meshes.forEach(mesh => {
            mesh.id = name
            mesh.name = name
            // 计算并设置模型中心点
            const boundingInfo = mesh.getBoundingInfo()
            mesh.setPivotPoint(
              boundingInfo.boundingBox.maximumWorld
                .add(boundingInfo.boundingBox.minimumWorld)
                .divide(new Vector3(2, 2, 2))
            )
          })
          resolve()
        })
      )
    })
  }))
}
```

## 3. 场景设置

### 3.1 相机设置
```typescript
if (!scene.activeCamera) {
  scene.createDefaultCamera(true, true, true)
}

const camera = scene.activeCamera as ArcRotateCamera
// 设置相机参数
camera.upperBetaLimit = null    // 取消上下视角限制
camera.lowerBetaLimit = null
camera.beta = Math.PI / 4       // 设置初始视角
// 设置缩放限制
options?.upperRadiusLimit !== undefined &&
  (camera.upperRadiusLimit = options?.upperRadiusLimit)
options?.lowerRadiusLimit !== undefined &&
  (camera.lowerRadiusLimit = options?.lowerRadiusLimit)
```

### 3.2 灯光设置
```typescript
const light = new HemisphericLight('hl', new Vector3(0, 0, -1), scene)
light.groundColor = Color3.Black()
light.intensity = 1
```

## 4. 模型处理

```typescript
meshes.forEach((mesh) => {
  // 设置模型旋转
  if (typeof options?.angle !== 'undefined') {
    mesh.rotate(Axis.X, options.angle, Space.LOCAL)
  } else {
    mesh.rotate(Axis.X, Math.PI / 2, Space.LOCAL)
  }

  // 设置材质
  if (!loadFn) {
    if (mesh.material === null) {
      const standardMaterial = new StandardMaterial('default', scene)
      mesh.material = standardMaterial
    }
    const material = mesh.material as StandardMaterial
    // 设置双面渲染
    material.backFaceCulling = false
    material.twoSidedLighting = true

    // 设置材质属性
    material.diffuseColor = Color3.FromHexString('#C8BCB7')  // 漫反射颜色
    material.specularColor = Color3.White()                   // 高光颜色
    material.useObjectSpaceNormalMap = true                  // 使用对象空间法线贴图
    material.specularPower = 0.5                            // 高光强度
  }
})
```

## 5. 渲染设置

```typescript
// 注册渲染前回调，用于更新灯光方向
scene.registerBeforeRender(() => {
  const camera = scene.activeCamera as ArcRotateCamera
  light.direction.copyFrom(camera.rotation)
})
```

## 主要功能特点

1. **文件加载支持**：
   - 支持 S3/CloudFront 文件并行下载
   - 支持多文件批量加载
   - 支持 STL 等多种格式

2. **模型处理**：
   - 自动计算和设置模型中心点
   - 支持自定义模型旋转角度
   - 统一的命名规则处理

3. **场景优化**：
   - 自适应相机设置
   - 灯光跟随相机旋转
   - 双面渲染支持

4. **可定制性**：
   - 支持自定义加载函数
   - 可配置的材质属性
   - 灵活的加载选项

5. **进度反馈**：
   - 支持加载进度回调
   - 完成回调处理

这个 load 函数设计得比较完善，既考虑了性能（并行下载、批量处理），又提供了良好的可定制性和用户体验。

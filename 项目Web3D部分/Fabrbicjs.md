# Fabric.js API 在项目中的具体实现分析

通过查看项目代码，该项目主要在以下几个方面使用了 Fabric.js API：

## 1. 画布初始化和管理

<mcfile name="App.tsx" path="d:\code\shixicode\src\templates\2dEditor\App.tsx"></mcfile>

```typescript
// 初始化画布和编辑器
Promise.all([import('./core'), import('./utils/event/notifier'), import('fabric')]).then(
  ([coreModule, notifierModule, fabricModule]) => {
    const Editor = coreModule.default;
    setFabric(fabricModule.fabric);
    const canvas = new fabricModule.fabric.Canvas(canvasRef.current, {
      preserveObjectStacking: true,
      controlsAboveOverlay: true,
      selection: true,
    });
    setCanvas(canvas);
    // ...
  }
);
```

## 2. 图层管理和操作

<mcfile name="Layer/index.tsx" path="d:\code\shixicode\src\templates\2dEditor\components\Layer\index.tsx"></mcfile>

```typescript
// 处理图层顺序
const handleZIndex = (allObjects: fabric.Object[]) => {
    const allObjectLength = allObjects.length;
    allObjects.forEach((item: any, index: number) => {
      const zIndex = allObjectLength - index;
      item.set({
        [CustomKey.ZIndex]: zIndex,
      });
      item.sendToBack();
    });
    canvasEditor?.workspaceSendToBack();
    canvasEditor?.canvas.renderAll();
};
```

## 3. 渐变效果实现

<mcfile name="MainUiTopToolImage.tsx" path="d:\code\shixicode\src\templates\2dEditor\components\MainUi\MainUiTopTool\MainUiTopToolImage.tsx"></mcfile>

```typescript
const gradientColor2FabricGradient = (cssGradient: string) => {
    const angle = Number(cssGradient.substring(cssGradient.indexOf('(') + 1, cssGradient.indexOf('deg')));
    // 创建渐变对象
    return new fabric.Gradient({
      type: 'linear',
      coords: {
        x1,
        y1,
        x2,
        y2,
      },
      colorStops: [
        { offset: 0, color: color1 },
        { offset: 1, color: color2 },
      ],
    });
};
```

## 4. 控件插件系统

<mcfile name="ControlPlugin/index.ts" path="d:\code\shixicode\src\templates\2dEditor\core\plugin\ControlPlugin\index.ts"></mcfile>

```typescript
class ControlPlugin {
  public canvas: fabric.Canvas;
  public editor: IEditor;
  static pluginName = 'ControlPlugin';

  constructor(canvas: fabric.Canvas, editor: IEditor) {
    this.canvas = canvas;
    this.editor = editor;
    // 设置控件精度
    fabric.Object.NUM_FRACTION_DIGITS = 8;
  }
}
```

## 5. 工作区管理

<mcfile name="WorkspacePlugin.ts" path="d:\code\shixicode\src\templates\2dEditor\core\plugin\WorkspacePlugin.ts"></mcfile>

```typescript
class WorkspacePlugin {
  public canvas: fabric.Canvas;
  public editor: IEditor;
  
  // 缩放到指定点
  zoomToPoint(point: fabric.Point, value: number) {
    this.canvas.zoomToPoint(point, value);
  }
  
  // 设置视口变换
  setViewportTransform(vpt: number[]) {
    this.canvas.setViewportTransform(vpt);
  }
}
```

## 6. 素材管理

<mcfile name="MaterialPlugin.ts" path="d:\code\shixicode\src\templates\2dEditor\core\plugin\MaterialPlugin.ts"></mcfile>

```typescript
class MaterialPlugin {
  public canvas: fabric.Canvas;
  public editor: IEditor;
  
  constructor(canvas: fabric.Canvas, editor: IEditor) {
    this.canvas = canvas;
    this.editor = editor;
  }
}
```

## 主要功能实现：

1. **基础编辑功能**
   - 对象的选择、移动、缩放、旋转
   - 图层的添加、删除、排序
   - 组合与解组操作

2. **高级渲染效果**
   - 渐变填充
   - 图层混合模式
   - 滤镜效果

3. **工作区管理**
   - 画布缩放
   - 视口变换
   - 网格对齐

4. **事件处理**
   - 鼠标事件
   - 键盘事件
   - 选择事件

5. **插件系统**
   - 控件插件
   - 素材插件
   - 工作区插件

6. **性能优化**
   - 对象缓存
   - 渲染优化
   - 事件节流

这个项目通过对 Fabric.js API 的封装和扩展，实现了一个功能完整的在线编辑器，支持复杂的图形编辑和图层管理功能。

---
layout: post
title: Metal - Draw a View’s Contents（2）
date: 2019-08-06
Author: Xiaoye
tags: [Metal]
excerpt_separator: <!--more-->
toc: true
---

上一篇对于 Metal、GPU、渲染管线整个有个大概的了解后，这一篇针对具体的 [Metal Sample Code - Using Metal to Draw a View’s Contents](<https://developer.apple.com/documentation/metal/basic_tasks_and_concepts/using_metal_to_draw_a_view_s_contents>) 加深对 Metal 具体使用上的理解

<!--more-->

先上一个 [Demo](<https://github.com/Xiaoye220/Demos/tree/master/MetalDemo>)

接下来按照 Sample Code 的顺序讲讲其中自己对于其中代码的理解

#### 1.创建 MTKView

`MTKView` 可以使用 `Metal` 去处理很多细节，以下就是一个 `MTKView` 创建的过程，当然只实现了简易的功能

1. Metal 使用 GPU 能力需要依赖 device，一个 `MTLDevice` 对象代表一个可以执行指令的 GPU，MTLDevice 的创建很昂贵、耗时。使用 `MTLCreateSystemDefaultDevice()`  获取系统默认设备对象，可以根据其是否为 nil 判断当前设备是否支持 Metal。在模拟器上，需要 iOS 13.0 以上，真机需要 iOS 8.0 以上才支持 Metal
2. 使用一个颜色去清除 view 的 contents，个人感觉应该就是类似 PS 里面油漆桶的功能
3. 设置使 view 只有在调用 setNeedsDisplay() 才会进行绘制，避免一些动画
4. 声明一个 _renderer 对象实现了代理 `MTKViewDelegate`，用于控制 view 什么时候进行绘制。在需要绘制 `MTKView` 时会调用代理方法 `func draw(in view: MTKView)`，我们需要在其中编写绘制的实现
5. 该代理方法 `func mtkView(_ view: MTKView, drawableSizeWillChange size: CGSize)` 在 view 的 size 改变时调用

```swift
var _view: MTKView!
var _renderer: Renderer!

func initMTKView() {
    _view = MTKView(frame: self.view.frame)
    self.view.addSubview(_view)

    // (1)
    _view.device = MTLCreateSystemDefaultDevice()

    // (2)
    _view.clearColor = MTLClearColorMake(0.0, 0.5, 1.0, 1.0)

    // (3)
    _view.enableSetNeedsDisplay = true

    // (4)
    _renderer = DVCRenderer(with: _view)

    if _renderer == nil {
        print("Renderer initialization failed")
        return
    }

    // (5)
    _renderer.mtkView(_view, drawableSizeWillChange: _view.drawableSize)

    _view.delegate = _renderer
}
```



#### 2. Renderer 实现

在本例中 `Renderer` 即 `MTKViewDelegate` ，提供 draw 方法去绘制 view 的 contents

要看懂这个代码，需要先理解几个概念，这也是我自己个人的理解，如果觉得有问题可以提

* `textures`：也叫 `render targets`，表示内存中包含图像数据，如 color、depth、stencil 信息，可以被 GPU 访问的很多数据块。GPU 也会保存图像处理结果至 textures 中，在显示 contents 时会从获取其中一个 texture 进行渲染
* `commands`：表示 GPU 进行图像处理的命令，我的理解就是比如一组图像数据，GPU 通过这些命令处理后输出新的一组图像数据
* `render pass`：是 `textures` + `commands` 

这里我一个具象的比喻就是

* `textures` 是**画布**，提供作画的场所
* `commands` 是**画笔**，要画什么颜色，什么图案需要画笔来实现
* `render pass` 就是一整个**绘画过程** 



那么下面是对代码的解析

1. 为了创建 `render pass`，需要 `MTLRenderPassDescriptor`，包含了 textures，以及如何处理 textures（比如自定义的一些 commands，本例中是没有的）
2. 处理所有的 `commands` ，将  `MTLRenderPassDescriptor` 编码出所有的 GPU 能处理的 commands 后放入缓冲区
3. 立刻调用 `endEncoding()`，因为本例中没有额外的 commands 去处理 texture，仅仅将 texture 清空
4. 在 Metal 中通过一个 `drawable` 对象来管理 textures，通过 drawable 对象的 present 方法来显示 textures。这里获取 MTKView 自带的 drawable 对象
5. 可以理解为添加一条 command（当所有 commands 处理图像完毕后，显示内容）
6. 当前 command 缓冲区 commands 添加完毕，提交处理

```swift
class Renderer: NSObject, MTKViewDelegate {
    
    var mtkView: MTKView
    var device: MTLDevice
    var commandQueue: MTLCommandQueue
    
    init(with mtkView: MTKView) {
        self.mtkView = mtkView
        self.device = mtkView.device!
        self.commandQueue = self.device.makeCommandQueue()!
    }
    
    
    /// 当 mtkView 的 contents 的 size 改变时会调用该方法
    /// 使用自动布局时，屏幕旋转也会调用
    func mtkView(_ view: MTKView, drawableSizeWillChange size: CGSize) {
        
    }
    
    /// 当需要更新 mtkView 的 contents 是调用该方法
    func draw(in view: MTKView) {

        // (1)
        guard let renderPassDescriptor: MTLRenderPassDescriptor = view.currentRenderPassDescriptor else { return }
        
        // (2)
        let commandBuffer: MTLCommandBuffer = commandQueue.makeCommandBuffer()!
        let commandEncoder: MTLRenderCommandEncoder = commandBuffer.makeRenderCommandEncoder(descriptor: renderPassDescriptor)!
		// (3)
        commandEncoder.endEncoding()
        
        // (4)
        let drawable: MTLDrawable = view.currentDrawable!
        
        // (5)
        commandBuffer.present(drawable)
        // (6)
        commandBuffer.commit()
    }
    
}
```


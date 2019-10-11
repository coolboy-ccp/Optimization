#  离屏渲染
## What
* GPU, 在屏幕缓冲区外，另外开辟一个帧缓冲区。
* CPU, 如果我们重写了drawRect方法，并且使用任何Core Graphics的技术进行了绘制操作，就涉及到了CPU渲染。整个渲染过程由CPU在App内同步地完成，渲染得到的bitmap(位图)最后再交由GPU用于显示。
## 性能消耗
1. 新缓冲区的创建
2. 上下文切换: 
![context](/context.png)
先是从当前屏幕（On-Screen）切换到离屏（Off-Screen），等到离屏渲染结束以后，将离屏缓冲区的渲染结果显示到屏幕上有需要将上下文环境从离屏切换到当前屏幕。
## Why
有些效果(mask, border, corner, shadow) 被认为不能直接呈现于屏幕，而需要在别的地方做额外的处理预合成。图层属性的混合体没有预合成之前不能直接在屏幕中绘制，所以就需要屏幕外渲染。屏幕外渲染并不意味着软件绘制，但是它意味着图层必须在被显示之前在一个屏幕外上下文中被渲染（不论CPU还是GPU）。
## Reason
* Any layer with a mask (layer.mask)
* Any layer with layer.masksToBounds/view.clipToBounds being true
* Any layer with layer.allowsGroupOpacity set to YES and layer.opacity is less than 1.0
* Any layer with a drop shadow (layer.shadow*)
* Any layer with layer.[shouldRasterize](/shouldRasterize.md)] being true
* Any layer with layer.cornerRadius, layer.edgeAntialiasingMask, layer.allowsEdgeAntialiasing(边缘抗锯齿)
iOS9.0后, 设置png图片的圆角不会触发离屏渲染
* 文本
* drawRect:
## Solution
### cornerRadius
1. 图片
   * 使用CoreGraphics预先设置图片圆角, 可以在子线程进行
   * UI直接给圆角图
2. 其他
   自定义视图，使用CAShapeLayer和UIBezierPath绘制自定义视图
### shadow
使用shadowPath代替shadowOffset等属性, shadowPath可通过贝塞尔曲线绘制


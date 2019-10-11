#  光栅化
## 效果
当shouldRasterize设成true时，开启shouldRasterize后,CALayer会被光栅化为bitmap,layer的阴影等效果也会被缓存到bitmap中，等下次使用时不会再重新去渲染了。实现圆角本身就是在做颜色混合（blending），如果每次页面出来时都blending，消耗太大，这时shouldRasterize = yes，下次就只是简单的从渲染引擎的cache里读取那张bitmap，节约系统资源。

## 注意点
* 如果我们更新已光栅化的layer,会造成大量的offscreen渲染。因此CALayer的光栅化选项的开启与否需要我们仔细衡量使用场景。只能用在图像内容不变的前提下的。
   * 用于避免静态内容的复杂特效的重绘，如UIBlureEffect
   * 用于避免多个view嵌套的复杂view的重绘
* 不要过度使用,系统限制了缓存的大小为2.5X Screen Size.如果过度使用,超出缓存之后,同样会造成大量的offscreen渲染。
* 被光栅化的图片如果超过100ms没有被使用,则会被移除。因此我们应该只对连续不断使用的图片进行缓存。对于不常使用的图片缓存是没有意义,且耗费资源的。
* 设置适当的rasterizationScale，否则在retina的设备上这些视图会成锯齿状




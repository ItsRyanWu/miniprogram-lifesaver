# 小程序（微信）优雅踩坑指南


### 1. 程序化的动态阻止事件冒泡与默认行为

在 DOM 中是一个非常基础且刚需的操作，但在小程序中却是非常隐晦。实际上，你无法用运行在 App Service 层的 JS 代码去动态判断并阻止事件的冒泡与默认行为，你仅可以使用运行在 Webview 层的 WXS （原始且被精简的类 ES5 语言）代码来[实现][1]。


### 2. _Android_ `<video/>` 视频画面宽高比例失调，画面被横向拉伸

`<video/>` 在声明 `Object-fit: cover | contain` 后将开启视频画面尺寸自适应功能，该功能是基于 video 文件的宽高二维尺寸数据进行计算并适配的。

需要注意的是每一个视频文件的尺寸数据其实都分两种，而决定这两种尺寸产生的对应元素也各有一个。第一个叫做显示纵横比 (display aspect ratio) 即该视频文件在最终输出成图像传达到你的眼中时的那个正确的纵横比例，正确意味着此时的画面纵横比例就是完全等于视频创作者创造该视频时设计的视觉比例。生活中常见的影像文件显示比例有 4:3, 16:9 等。 第二个叫做储存纵横比 (storage aspect ratio) 即当一个有着固定显示纵横比的视频被投放在不同单个像素纵横比的显示器上时，它所占的所有物理像素数量在 X 轴与 Y 轴两个维度的比例。因为现代显示器的单个像素纵横比总是 1:1 （即正方形），所以我们很容易忽视这个储存纵横比。显示纵横比影响着视频文件的显示尺寸 (display size)，储存纵横比影响着视频文件的储存尺寸 (storage size)。

在小程序的开发工具模拟器和 iOS 真机中，客户端采用的是视频文件的显示尺寸。而在 Android 真机中，客户端采用的却是视频文件的储存尺寸。所以当一个视频在 Premiere Pro 等视频编辑器中最终被渲染成文件时，如果视频创作者采用的渲染设置中的像素纵横比 (Pixel Aspect Ratio) 被设置为 “NTSC”, “PAL” 等不为 1:1 的选项时，视频输出画面在 Android 上就会被横向拉伸（画面被压扁），此时画面不协调的 BUG 就会产生。

所以该 BUG 的最终解决方案就是在视频编辑器中的渲染设置里将基础设置中的像素纵横比改为 1:1 (正方形像素)。


### 3.  _iOS_ `<video/>` 等原生组件同层渲染失败条件与监听处理

众所周知小程序实际上是一种 hybrid app，无法避免的涉及到 native 组件与 web 组件在同一视口中的叠加渲染。理论上 native 组件在垂直于屏幕的图层优先级上肯定是要高于 web 组件，因为 webview 本身就处于 native 应用提供的容器里。有关小程序可以实现在 hybrid 应用中实现不同视图层组件的同层渲染的技术细节介绍很少，这篇 [来自官方社区的文章][2] 也仅能解答部分疑惑。总结于这篇文章中的有价值的内容主要是：

1. 尽量避免在支持同层渲染的原生组件上用 CSS 属性做影响文档结构的变化。
	2. `bind:rendererror` 事件是唯一可以用来监听同层渲染失败的事件，你可以用这个事件来做亡羊补牢式的体验优化，即错误已经发生，如何最大程度挽回已经丢失的用户体验？从我的个人经验来说可以做两点：1. 向后台发送错误统计信息；2. 用且仅能用 `wx.showModal` 方法触发原生 modal 组件向用户解释目前到底发生了什么并且提供按钮引导用户 `relaunch` 到当前页面来试探此次同层渲染错误是否为偶然发生（实际上的确有偶然发生的情况）。

	还有一点不成系统的个人经验是就减少原生组件在所在自定义组件的嵌套层级，或者干脆避免在自定义组件中使用原生组件而是直接在页面组件中直接使用，不然这会使同层渲染变得非常难以捉摸，即使是同一平台的不同机型，同一机型的不同元素尺寸… 都会成为导致同层渲染失败的因素。而发生了这种情况你也可以试试替换或去除一些对排版影响比较重的 CSS 属性（如将 `left` / `top` 替换为 `transform` 来达到相同的视觉效果）来努力争取一下好的结果。


### 4. CSS 变量 `env(safe-area-inset-bottom)` 的非常规表现

在 Android 平台中，该变量并非不支持，而是为 0。这意味着你无法使用 `@supports (padding-bottom: env(safe-area-inset-bottom))` 这样的 CSS 特性检测功能以及 `env(safe-area-inset-bottom, 20px)` 这样的CSS 变量 fallback 功能。

为什么需要这些功能？有时你需要在 `env(safe-area-inset-bottom)` 值的基础上进行修改从而得出另一个 CSS 变量，比如 `env(safe-area-inset-bottom) - 10px` 。可是在 Android 平台中该变量将无法避免的成为最终表现为 `-10px` 的变量，这将对 UI 的适配造成负面的影响。CSS `max()` 或者 `min()` 仅支持 iOS 平台的兼容局限性也使得它在这个场景中无法发挥作用。针对这个问题，我的解决方案是用 `min-height` CSS 属性来防止以上例子中的 `-10px` 对垂直方向上的布局造成负面影响，但应用的场景非常有限。


### 5. _iOS_ CSS `transition-delay` 偶现闪烁

可以使用 CSS animation 创建空白的 keyframes 填充在真正要表现的动画之前来绕过该问题。

但该解决方案的缺点是他无法从视觉上真实的表现出 easing-function 的结果，因为空白关键帧也属于动画的一部分，但这部分人眼不可见的动画却占用了 easing 效果的时间轴，使得真正要表现的动画却不是从 easing-function 的起始点按照曲线输出效果的，而是从中途开始。所以这就需要另外对 easing-function 进行调整，尽量弥补这一遗憾。


### 6. 声明 `virtualHost: true` 时自定义组件实例不具备被选择以及被观测的特性

在 Component 的 Options 字段中声明 `virtualHost: true` 是为了使该组件更加轻量化，从而免去外部组件容器的无法直接作用于该组件内部根节点的麻烦。

但官方文档中并没有注明的是，如果你使用了 `virtualHost` ，同时也就意味着你的组件将无法在被 `this.selectComponent` 获取到实例对象，也无法在 Devtool 的 Inspector 中被观测到组件内部的数据情况。


### 7. 声明 `styleIsolation: shared` 时 `externalClasses` 属性无效

小程序对于组件样式的管理策略原本是一个不与外界产生任何作用的孤岛，但在自定义组件的 JSON 配置中声明 `styleIsolation: shared` 或者 `styleIsolation: apply-shared` 会对该组件以及外界的 CSS 作用域产生不同的影响。而仅当你使用 `shared` 去开启一个自定义组件最大程度的样式作用域时 `externalClasses` 所起到的自定义外部 class 名的作用是无效的。


### 8. `<image/>` 开启 `widthFix` 或 `heightFix` 模式时，尺寸样式自动计算的响应速度差强人意

可以看得出来 `widthFix` 与 `heightFix` 是小程序团队从提升开发体验角度设计的 API，而且这种特性相较于 `object-fit: contain` 式的特性的确具备专属于它的优势与应用场景，简单来说应用这个功能之后你不用担心图片在保持完整显示长边的情况下留有过多的空隙，这使得布局可以在保持灵活响应式的同时又可以做到恰如其分。

但现实是小程序这个特殊的 API 其实就是在当图片在渲染完成后，根据其中一个已经被设置为固定维度的尺寸数据以及 metadata 中的原始比例数据来自动计算出另一个维度的数据并将该数据设置到 style 属性中。这整个过程从开始到应用结束是有肉眼可见的延迟的，并且在糟糕的场景中该延迟会导致文档进行布局重排，页面内被影响到的元素会出现位置闪烁，而图片组件本身会出现尺寸从被拉伸到比例正常的尺寸闪烁。解决方案是：1. 仍然使用 `aspectFit` 作为实现图片尺寸自适应的方法，只不过开发者需要预先根据图片的原始比例给图片的容器赋予合理的比例以使其能够更好地利用展示面积。


### 9. _iOS_ `font-family` 覆盖字符范围不全导致中英文混合排版时字重异常

在小程序中为内容段落实现自定义英文字体样式时，不仅要在该段落的 `font-family` CSS 属性中声明你想应用的英文字体包的名称，还需要在其后继续增加类似于 `-apple-system ` 之类的系统字体，来使得没有匹配到该英文字体包的中文字符可以 fallback 到后面你所指定的中文字体包。否则，在 iOS 设备上会出现应用该英文字体的段落中的中文字重过重且无法用 `font-weight` 控制的 BUG。


### 10. `<input/>` 的 `focus` 属性设置为 `true` 在页面初始挂载时有几率失效

小程序官方文档废弃了 `<input/>` 组件的 `auto-focus` 属性转而建议通过将 `focus` 属性固定设置为 `true` 的方式来实现文本框自动聚焦，但该 API 在页面初始挂载后有一定几率起不到自动聚焦的作用，也就是在页面渲染完成后键盘没有自动弹出。

建议在页面的 `show` 生命周期钩子函数中设置一个不小于 500ms 延迟的定时器来使 `focus` 在页面渲染完成的一段时间后被设置为 `true` 来解决这个不稳定 BUG。


### 11. 使用自定义组件 `model:` 指令进行双向数据绑定时如值为非原始类型则会产生无限循环导致内存溢出

小程序组件的双向数据绑定不支持非原始值类型的数据，主要原因是小程序组件内部逻辑不支持对引用数据类型的深度对比来终止数据流的循环所以会导致无限循环内存溢出。如想要实现引用数据类型的双向数据绑定可以采用最基本的父组件向下传参 + 子组件向上发布事件的方式。


### 12. _Mac_ 自定义 Tabbar 在切换页面时消失

macOS 下的小程序自定义 Tabbar 在切换页面时并不会为下一个页面挂载渲染实例，导致仅有首个 Tabbar 页面会有 Tabbar 而其他页面却没有。[官方已确认此 BUG][3]


### 13. 部分微信封装的组件对于 Flexbox 弹性尺寸的支持不友好

`<picker-view/>`, `<scroll-view/>` 等微信封装的组件应该避免使用 CSS Flexbox 弹性的控制其尺寸。否则在某些边界情况下，小程序在渲染此类组件时可能会出现该类型组件尺寸不符合预期、布局错乱的情况。但此时又难以用 Web 的知识与经验去解释以及 debug。


### 14. _iOS_ 同一页面内 `<video/>` 组件数量上限

在 iOS 中若同一页面内有超过 3 个 `<video/>` 则存在其中个别 `<video/>` 无法加载播放的 BUG。规避方法为将 `<video/>` 数量限制在 3 个以内。


### 15. _iOS_ `<video/>` 事件触发不对等

在 iOS 中当 `<video/>` 开启 `autoplay` 和 `loop` 两个选项时会只触发一次 `play` 事件和无数次 `ended` 事件。


### 16. `<picker-view/>` 事件触发不对等

在 `<picker-view/>` 中单击当前选中项会触发 `pickStart` 事件但紧接着并不会触发 `pickEnd` 事件。此 BUG 产生的原因或许是 `pickStart` 事件由 `touchStart` 一个交互事件即可触发而 `pickEnd` 则必须在当前选中项发生变动后触发。规避方法为在 `<picker-view/>` 上绑定 `tap` 事件来有条件的模拟 `pickEnd` 事件。


### 17. 自定义组件内 `<text/>` 的 slot 无法为其内部组件提供视图更新能力

在自定义组件内若将 `<slot/>` 放置于 `<text/>` 节点内则无法使 slot 内部的组件在 runtime 进行基于状态数据变化的视图更新。




---- 

Todo:

1. 根级滚动容器与 `<scroll-view/>` 的优劣对比
2. _iOS_ `<scroll-view/>` 特定情况下卡死
3. `<scroll-view/>` 自动滚动功能特定情况下偶现失效
4. 使用 custom tabbar 时的性能差异、created 预先触发逻辑、维度数据获取
5. 使用纯 CSS 逻辑来计算与给页面制造 custom tabbar 的预留空间以及相比之下使用 `getboundingClientRect` API 的劣势
6. `<input/>` 激活时滚动试图 placeholder 偏移
7. _iOS_ `<input/>` 激活时特殊情况下内部区域可滚动
8. `wx.onKeyboardHeightChange` 在键盘收起时不触发
9. _Android_ 数字键盘触发唤起系统原生键盘导致键盘重叠
10. `this.animate` 单独动画某些属性无效（如 `background-color`）
11. _Android_ 三星 Samsung 机型 `windowHeight` 异常
12. _iOS_ Flexbox 部分表现反常
13. Video 组件自动横屏导致在安卓上该页面横屏
14. iOS 14 Video 组件可被 overflow 滚动
15. CSS pointer-events: none 作用于安卓端的原生组件无效
16. _Android_ 当前小程序在从另一个小程序跳转回来后 autoplay 的 Video 停止播放
17. `<Swiper/>` 设置 `display-multiple-items` 值后若 `current` 值使其超出滚动范围则 `<SwiperItem/>` 会全部消失
18. _iOS_ Data 字段中未初始化设置视图需要的变量可能会导致该组件视图渲染失败进而导致在其之后的节点渲染全部失败，页面白屏。



[1]:	https://developers.weixin.qq.com/miniprogram/dev/framework/view/interactive-animation.html#%E5%AE%9E%E7%8E%B0%E6%96%B9%E6%A1%88
[2]:	https://developers.weixin.qq.com/community/develop/article/doc/000c4e433707c072c1793e56f5c813
[3]:    https://developers.weixin.qq.com/community/develop/doc/0002008ca3039864d68b9425c56400
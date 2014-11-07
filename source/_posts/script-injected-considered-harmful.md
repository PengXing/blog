title: 脚本注入的异步script可能有害
date: 2014-11-07
tags: 
 - frontend
 - javascript
 - performance
comments: true
categories:
 - frontend
 - performance
---

From：https://www.igvita.com/2014/05/20/script-injected-async-scripts-considered-harmful/?utm_source=feedburner&utm_campaign=Feed%3A+igvita+%28igvita.com%29&utm_content=feed

同步脚本会强制浏览器阻塞DOM树的构建、加载script和执行，当然，这是众所周知的，这也是为什么一直宣传使用异步脚本，下面是一个例子

```html
<!-- BAD: blocking external script -->
<script src="//somehost.com/awesome-widget.js"></script>

<!-- GOOD: remote script is loaded asynchronously -->
<script>
    var script = document.createElement('script');
    script.src = "//somehost.com/awesome-widget.js";
    document.getElementsByTagName('head')[0].appendChild(script);
</script>
```

有什么区别呢？在这个BAD的例子中，我们阻塞了DOM树的构建，等待加载完script并且执行完成之后，才会继续处理剩下的文档。在第二个例子中，通过执行JavaScript插入script标签来加载script。不同点就是：脚本注入不会阻塞网络。

## 脚本注入，不一定快

内联脚本挺好的，但是有一个很重要的性能上的缺陷（也是经常被忽视的）：内联脚本被阻塞执行知道CSSOM建立完成。浏览器要加载的脚本会做什么，所以在CSS加载完成、处理完成、CSSOM建立完成并且可以被操作之前，浏览器会一直block JS的加载，因为JS可以操作CSSOM。一图顶千言，可以看下面的例子

![CSS阻塞脚本加载](https://1-ps.googleusercontent.com/s/www.igvita.com/posts/14/xasync-injected.png.pagespeed.ic.Nz8cY9Q-Ow.png)

这个[例子](http://jsbin.com/qefefiyi/9/quiet)在页面顶部加载CSS，在页面底部加载JavaScript，换句话说，这个例子符合所有的性能规范，除了脚本不能在CSSOM构建号之前执行，这导致延迟处理HTML，和JavaScript请求的发送。结果就是脚本在页面初始化完毕被执行，大概在3.5s时刻左右。

现在，让我们对比一下上面所说的BAD的例子，使用[两个阻塞Script标签](http://jsbin.com/qefefiyi/8/quiet)：

![阻塞Script标签](https://1-ps.googleusercontent.com/s/www.igvita.com/posts/14/xasync-blocking.png.pagespeed.ic.Qf2_CnkjCV.png)

发生什么了？两个Script文件都被提前加载，并且在页面初始化后被执行，大概在2.7s时刻左右，请注意页面上的时间，脚本还是在CSS ready之后才被执行的，但是因为脚本是提前获取的，在CSSOM构建完成之后立即执行了JavaScript，节省了大概1s的时间。难道我们之前都做错了吗？

在我们回答这个问题之前，先看一下另外一个例子，这次用[`async`属性](http://jsbin.com/qefefiyi/7/quiet)：

```html
<script src="http://udacity-crp.herokuapp.com/time.js?rtt=1&a" async></script>
<script src="http://udacity-crp.herokuapp.com/time.js?rtt=1&b" async></script>
```

![async属性](https://1-ps.googleusercontent.com/s/www.igvita.com/posts/14/xasync-async.png.pagespeed.ic.-aH-Uzp-RY.png)

`async`属性告诉浏览器不要阻止DOM树的构建，脚本加载完成就会执行，大概在1.6s的时刻

下面是一个总结

* 脚本注入：`脚本执行 ~3.7s     onload ~3.7s`
* 阻塞脚本：`脚本执行 ~2.7s     onload ~2.7s`
* async属性：`脚本执行 ~1.7s    onload ~2.7s`

所以，为什么我们提倡这种用法这么久了呢？

1. `async`属性有兼容性问题，IE8/9 Android 2.2/2.3，这些旧的浏览器就会忽略这个属性，并且会将这些script当成阻塞脚本看待，这是一个很大的倒退，但是这也引入了下一个观点
2. 所有的现代浏览器都有一个`预加载扫描器`，甚至IE8/9 Android 2.2/2.3，当浏览器处理文档被阻塞的时候，它唯一的责任就是查找那些可以尽快加载的资源

脚本注入方式在任何方便都比不过`<script async>`，它存在的唯一理由是因为`<script async>`不能完全兼容，并且，预加载扫描器不会重新扫描。但是那个时代已经过去了，我们需要更新我们的观念，用`async`属性代替脚本注入，换句话来说，脚本注入的异步脚本被认为是有害的。

注意`预加载扫描器`只能预加载那些具有`src/href`属性的script和link标签，预加载扫描器不能执行内联JavaScript块，这就意味着任何脚本注入的静态文件都不会被雨加载，因此最好的加载方式是：
```html
<!-- BAD: the pre async / pre preload scanner era -->
<script>
    var script = document.createElement('script');
    script.src = "//somehost.com/awesome-widget.js";
    document.getElementsByTagName('head')[0].appendChild(script);
</script>

<!-- GOOD: simpler, faster, and better all around -->
<script src="//somehost.com/awesome-widget.js" async></script>
```

1. `<script src="...">`

缺点：
 * 阻塞DOM树的构建
 * 执行被CSSOM阻塞
优点：
 * 可以被预加载器发现
 * 顺序执行

当执行顺序有影响的时候使用，把这些脚本放在页面最末尾

2. `<script async src="...">`

缺点：
 * 不是顺序执行
优点：
 * 不会阻塞DOM树的构建
 * 不会被CSSOM阻塞执行
 * 可被预加载器发现

可以在页面的任何地方书写，对于无顺序执行的脚本最为理想

## 那`defer`属性呢 

`defer`在`async`属性以前就存在，理论上，它是保证脚本不会阻塞parser，并且会在`DOMContentLoaded`事件之前执行，并且按照他们的插入顺序，不幸运的是实际上这个执行顺序的实现有bug，但是，`defer`还是一个有用的字段，在那些不支持`async`属性的浏览器中，`defer`也能达到同样的效果，我们可以结合`defer`和`async`属性来使用

```html
<!-- Modern browsers will use 'async', older browsers will use 'defer' -->
<script src="//somehost.com/awesome-widget.js" async defer></script>
```

## 但是，等一下。。。
澄清一下，并不是所有的内联JavaScript都不应该写，有时候它还是非常有用的。下面是一些小应该要记住的：

1. **async属性不保证执行顺序**。脚本在被加载完成之后立即执行，和他们的顺序和在文档中的位置没有关系，如果对你的项目来说有影响，你能推迟他们的执行，或者让你的代码变成和顺序无关。可以研读一下这篇文档：[串行执行异步函数](http://stackoverflow.com/questions/6963779/whats-the-name-of-googla-analytics-async-design-pattern-and-where-is-it-used)
2. **第一点中讲的那种解决放哪尕诺要求初始化一些变量，这意味着我们需要内联脚本，所以，我们又回来了吗？**。没有，如果你把你的内联代码放在CSS声明前面，这些内联代码就会立即执行。
3. **我们应该把所有的JavaScript放在CSS前面吗？**。不要，你要让你的`<head>`标签尽可能 ，让浏览器尽快处理CSS和页面内容。

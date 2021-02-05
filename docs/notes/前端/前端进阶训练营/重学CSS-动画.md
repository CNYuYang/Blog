# 动画

CSS 当中控制表现的无非就是三类：

1. 控制元素位置和尺寸的信息
2. 控制位置和最后实际看到的渲染信息
3. 交互与动画的信息

接下来就和大家一起来学习另外的一些 CSS 的属性。

## Animation

首先介绍一下 CSS 里面最直接的一些特性，先来讲讲 `Animation`。`Animation` 它包含着两个部分：


**使用 `@keyframes` 去定义动画的关键帧**

```css
@keyframes myKeyFrame {
  from { background: red; }
  to { background: yellow; }
}
```

> 首先我们可以使用 `keyframes` 这个 `@rule` 来定义一个关键帧。然后使用 `from` 和 `to`，它们里面定义的都是 `CSS declaration`（CSS 属性和值得声明）

使用 `animation` 属性去使用关键帧的部分

```css
div {
  animation: myKeyFrame 5s infinite;
}
```

> 在具体的 CSS 规则里面我们可以设置 animation 这个属性。Animation 这个属性它又分为几个部分。这里我们给 `div` 标签定义了一个 `animation` 关键帧，也就是我们上面预定义的 `myKeyFrames`。然后我们给它定义了 `5` 秒，并且让它是一个 `infinite` 的循环模式，`infinte` 就是说这个动画是无限循环的。

好我们看一下实际的例子是怎么样的：

```html
<style>
  @keyframes myKeyFrame {
    from {
      background: red;
    }
    to {
      background: yellow;
    }
  }
  div {
    animation: myKeyFrame 5s infinite;
  }
</style>

<div style="height: 100px; width: 100px"></div>
```

这里的代码与我们刚刚讲到的是一样的，`style` 标签里面含有我们设定的 `myKeyFrame`。然后给 `div` 标签加入了 `animation` 属性并且绑定了我们预设置的 `myKeyFrame` 关键帧。运行的效果如下：

![](./images/09-01.gif)

> 如果我们打开 “控制台” 我们会发现这个元素的 `background` 属性是没有在更变的，但是如果我们去取这个元素的 `computed` 属性的话，就会发现是一直在不断变化的。这个就是 CSS animation 的基本用法。

## Animation 属性

接下来我们来详细看一下 `Animation` 有哪些属性，它的属性有 6 个组成的部分：

- *animation-name* —— 时间曲线
- *animation-duration* —— 动画的时长
- *animation-timing-function* —— 动画的时间曲线
- *animation-delay* ——动画开始前的延迟
- *animation-iteration-count* —— 动画的播放次数
- *animation-direction* —— 动画的方向

## Keyframes 定义

```css
@keyframes myKeyframe {
  0% { top: 0; transition: top ease }
  50% {top: 30px; transition: top ease-in }
  75% { top 10px; transition: top ease-out  }
  100% { top: 0; transition: top linear }
}
```

Keyframes 的定义是可以使用`% (百分比)`也可以使用 `from to`。而 `from` 大致相当于 `0%` ，`to` 大致相当于 `100%`。

每一个关键帧里面我们都是可以定义很多的属性。有一个常见的技巧就是在这个里面去定义 `transition` 而不是使用 animation 的 timing-function，来让这个值发生改变。

这样的话，每个关键帧之间的 timing-function 都可以用不一样的。不像 animation 中，一旦指了一个值，那么它的整个 timing-function 就确定了，也就没有办法分段去指定了。

### Transition 使用

Transition 的使用与 animation 差不多，它也的属性一共有 4 条：

- *transition-property* —— 要变换的属性
- *transition-duration* —— 变换的时长
- *transition-timing-function* —— 时间曲线
- *transition-delay* —— 延迟

## Cubic-bezier 三次贝塞尔曲线

`Timing-function` 它其实来自于一个三次贝塞尔曲线，我们所有的 `timing-function` 都跟三次贝塞尔曲线有相关。

我们可以通过 [cubic-bezier.com](https://cubic-bezier.com/#.17,.67,.83,.67) 的网站开始了解 `cubic bezier`。


---
title: CSS容易忽略的一些知识点
date: 2019-08-10 17:27:40
tags: 
  - CSS
categories:
  - CSS
---
## <code>inline</code>元素、<code>inline-block</code>元素、<code>block</code>元素的区别

1. <code>inline</code>元素根据宽度横向排列，<code>block</code>元素默认独占一行；
2. <code>inline</code>元素无法设置上下外边距(<code>margin</code>)、<code>width</code>、<code>height</code>，<code>block</code>元素可以设置四个方向的外边距和元素的<code>width</code>、<code>height</code>；
3. <code>inline-block</code>元素则融合了<code>inline</code>元素和<code>block</code>元素，可以像<code>inline</code>元素横向排列以及像<code>block</code>元素设置四个方向的外边距以及<code>width</code>、<code>height</code>；

效果图如下：

![](1.jpeg)

代码如下：

```
<div>
    <div style="display: block;width: 100px;height: 100px;background-color: red;margin: 10px;padding: 10px;border: 1px solid grey;">block</div>
    <div style="display: inline-block;width: 100px;height: 100px;background-color: blue;margin: 10px;padding: 10px;border: 1px solid grey;">inline-block</div>
    <div style="display: inline;width: 100px;height: 100px;background-color: yellow;margin: 10px;padding: 10px;border: 1px solid grey;">inline</div>
</div>
```

我们给<code>inline</code>元素设置四个方向外边距，只有左右的外边距才显示出了效果。

## <code>flex-grow</code>容易忽略的坑

我们先看看<code>flex-grow</code>的定义：

<code>flex-grow</code>属性定义项目的放大比例，默认为0，即如果存在剩余空间，也不放大。

如果所有项目的<code>flex-grow</code>属性都为1，则它们将等分剩余空间（如果有的话）。如果一个项目的<code>flex-grow</code>属性为2，其他项目都为1，则前者占据的剩余空间将比其他项多一倍。

根据定义我们可以得知，当在<code>flex</code>容器内给容器内的项目设置不同的<code>flex-grow</code>，可以根据比例设置项目的空间；

先看第一个例子：

<code>flex</code>容器宽度为<code>780px</code>，容器内有三个项目，第一个项目固定宽度<code>480px</code>，剩下的两个项目根据比例分配。

效果图如下：

![](2.jpeg)

代码如下：

```
<div style="display: flex;color: grey;width: 780px;">
    <div style="height:100px;flex-basis: 480px;background-color: red;">width480px</div>
    <div style="height:100px;flex-grow: 2;background-color: blue;">flex-grow: 2</div>
    <div style="height:100px;flex-grow: 1;background-color: yellow;">flex-grow: 1</div>
</div>
```

从效果图得知，除了固定宽度的项目，另外两个项目的宽度并未按照我们所想的那样分配。

重新去看定义，分配“_剩余空间_”似乎并不是我们理解的那样，具体这个“_剩余空间_”是如何计算我并未具体去研究，在这里先说说如何解决这个问题。

1. 当剩余两个项目内不存在内容的时候，分配的空间就是正确的；
   效果图如下：
   ![](3.jpeg)
   代码如下：

```
<div style="display: flex;color: grey;width: 780px;">
    <div style="height:100px;flex-basis: 480px;background-color: red;">width480px</div>
    <div style="height:100px;flex-grow: 2;background-color: blue;"></div>
    <div style="height:100px;flex-grow: 1;background-color: yellow;"></div>
</div>
```

当我们把除了固定宽度外的项目的内容去掉，分配的空间比例就是正确的，但是这种解决方法肯定不是我们想要的。

2. 给剩余两个项目设置<code>flex-basic: 0;</code>

效果图如下：
![](4.jpeg)
代码如下：

```
<div style="display: flex;color: grey;width: 780px;">
    <div style="height:100px;flex-basis: 480px;background-color: red;">width480px</div>
    <div style="height:100px;flex-grow: 2;background-color: blue;flex-basis: 0;">flex-grow: 2</div>
    <div style="height:100px;flex-grow: 1;background-color: yellow;flex-basis: 0;">flex-grow: 1</div>
</div>
```

在这里当我们给除了固定宽度外的项目加上<code>flex-basic: 0</code>后，分配的空间就是我们所期望的那样了。

如果我们需要使用<code>flex-grow</code>来分配<code>flex</code>容器内的项目，一定要注意设置<code>flex-basic</code>。因为这里的“_剩余空间_”和<code>flex-basic</code>相关。

下面我们看看<code>flex-basic</code>的定义：

<code>flex-basic</code>属性定义了在分配多余空间之前，项目占据的主轴空间（main size）。浏览器根据这个属性，计算主轴是否有多余空间。它的默认值为auto，即项目的本来大小。

当我们一个<code>flex</code>容器内的项目同时存在<code>flex-basic</code>和<code>flex-grow</code>，这个项目的宽度为<code>flex-basic</code>的值加上<code>flex-grow</code>所分配到的空间。

还是以上面的代码举例，假如我们给两个项目的<code>flex-basic</code>设置值为<code>30px</code>和<code>60px</code>

则剩余两个容器的宽度分别为：

<code>width1 = ((780 - 480 - 30 - 60) * 2 / 3) + 30 = 170</code>

<code>width2 = ((780 - 480 - 30 - 60) * 1 / 3) + 60 = 130</code>

效果图如下：

![](5.jpeg)
代码如下：

```
<div style="display: flex;color: grey;width: 780px;">
    <div style="height:100px;flex-basis: 480px;background-color: red;">width480px</div>
    <div style="height:100px;flex-grow: 2;background-color: blue;flex-basis: 30px;">flex-grow: 2</div>
    <div style="height:100px;flex-grow: 1;background-color: yellow;flex-basis: 60px;">flex-grow: 1</div>
</div>
```

## 多个<code>class</code>样式的时候如何取值？

当某个<code>div</code>元素上<code>class</code>的值为<code>a b c</code>的时候，最后的样式是如何计算的？

```
<div class="a b c"></div>
```

这个问题我们讨论的前提是同样的样式属性；

1. 不考虑权重的情况下，<code>a b c</code>最终决定元素的样式为加载的顺序，哪个<code>class</code>最后加载则显示为哪个<code>class</code>的效果，和书写顺序无关。

* 当<code>a b c</code>都在同一个文件的情况下，哪个定义在最后，则以最后的为准；

```
.c {
    background-color: green;
}

.b {
    background-color: yellow;
}

.a {
    background-color: red;
}
```

* 当<code>a b c</code>在不同的文件的情况下，哪个文件最后加载，则以最后的为准；

```
    <link rel="stylesheet" href="cssC.css">
    <link rel="stylesheet" href="cssB.css">
    <link rel="stylesheet" href="cssA.css">
```

2. 如果某个<code>class</code>中设置了<code>!important</code>，则直接以<code>!important</code>的为准。

3. 其他时候则按照选择器的权重来计算样式。

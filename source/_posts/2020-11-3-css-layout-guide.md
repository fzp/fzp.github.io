---
title: CSS 布局漫游
date: 2020-11-03
tags:
  - CSS layout
---

本文是对CSS多种布局的一个总结。

<!--more-->

## 点: 盒子模型、postion、float

最早的网页布局方式有两个要点：

  1. 盒子模型
  2. HTML普通流

每个html元素都是一个盒子，可以通过设置它的Margin, border, padding, content等改变盒子的各种间距。
HTMl普通流是默认放置盒子的方式。盒子分为行内元素和块状元素。多个行内元素会放置在同一行，块状元素单独占据一行。

为了让网页的布局不局限于HTML的普通流，于是有了posistion和float两个属性。

position属性用于设置一个根节点。让元素能够根据设置的偏移量进行偏移。

float属性用于让盒子向左向右水平移动，直到它的外边缘碰到包含框或另一个浮动框的边框为止。

更具体的细节就不多说了。都是很原始的方法。

实际上，这种方法用于布局其实并不好用。一些比如垂直居中的布局也很不好实现。

这种传统布局相当于让你把盒子一个个放到合适的位置。特别是现在注重响应式编程的现在，这个布局方式不能够适应各种不同尺寸的屏幕。

## 线：flex布局

flex布局相当于把一组盒子看作是一维的数组放到一个容器（另外的html元素）中，横向或者竖向排列开来。由于盒子能够在容器中进行缩放，所以称作flex布局。

了解flex布局最好的方式是[玩游戏](https://flexboxfroggy.com/)。

![flex_layout.png](flex_layout.png)

flex容器（flex container）中有若干个项目（flex items）. 项目按照主轴（main axis）依次排列。交叉轴（cross ）与主轴垂直，用于调整项目在该方向的位置。

通过调整flex容器和项目的属性，我们可以调整项目在容器中的布局。简单地讲，容器属性定义了方向，换行和对齐方式，项目属性主要定义了缩放。

### 容器属性

* flex-direction： 主轴方向
* flex-wrap：一条主轴排不下是否换行
* flex-flow： \<flex-direction\> \|\| \<flex-wrap\>
* justify-content： 主轴对其方式（左对齐，右对齐，居中，两端对齐， 每个项目两侧的间隔相等。）
* align-items： 交叉轴对其方式
* align-content： 多轴线的对齐方式

### 项目属性

* order： 定义项目顺序，数值小的项目在前面 
* flex-grow： 定义放大比例，在主轴上有剩余空间的时候使用。
* flex-shrink： 定义缩小比例。
* flex-basis： 定义如何计算剩余空间。
* flex：none \| [ \<'flex-grow'\> \<'flex-shrink'\>? \|\| \<'flex-basis'\> ]
* align-self： 允许单个项目在交叉轴上有不同的对其方式。覆盖align-items

## 面：grid布局

grid布局则是一种二维布局。这种布局方式的表现形式是跟我们要设计的主体（网页）是一致的，都是一个平面。因此，这就省略了前面布局方式跟设计主体不一致带来的额外代码。

同样，了解grid布局的最好方式还是[玩游戏](https://codepip.com/games/grid-garden/).

下面是来自游戏的一个例子。 `#garden`的属性创建了一个5*5的格子图， `#water`属性表明了要占据的格子。从第2条垂线到第6条垂线，从第一条横线到第6条横线围成的空间。

``` CSS
#garden {
  display: grid;
  grid-template-columns: 20% 20% 20% 20% 20%;
  grid-template-rows: 20% 20% 20% 20% 20%;
}

#water {
grid-column: 2/6;
grid-row:1/6;
}
```

![grid_layout.png](grid_layout.png)

主要的思想就是这么简单，其他的属性，比如格子的缩放和内容的对齐，可以参考官方文档或者其他blog。

经典布局方式如何用grid布局或者flex布局实现可以参考链接1.

## 参考链接

1. https://www.ruanyifeng.com/blog/2020/08/five-css-layouts-in-one-line.html
2. http://www.ruanyifeng.com/blog/2019/03/grid-layout-tutorial.html
3. https://www.ruanyifeng.com/blog/2015/07/flex-grammar.html
4. https://www.cnblogs.com/coffeedeveloper/p/3145790.html

[1. 简介](#简介)

[2. @import的缺点](#@import的缺点)

[3. 模块化的核心@use](#模块化的核心@use)

[3.1 `@use`的基本用法](#`@use`的基本用法)

[3.2 `@use`与`@import`的区别](#`@use`与`@import`的区别)

[3.3 配置样式库的基础变量](#配置样式库的基础变量)

[4. 给库开发者使用的利器@forward](#给库开发者使用的利器@forward)

[4.1 `@forward`的基本用法](#`@forward`的基本用法)

[4.2 用`show`/`hide`控制成员是否可见](#用`show`/`hide`控制成员是否可见)

[4.3 给不同的子模块添加前缀](#给不同的子模块添加前缀)

[5. sass内置模块](#sass内置模块)

[6. 兼容性](#兼容性)

[7. 迁移指南](#迁移指南)

[8. 参考资料](#参考资料)

# 简介

2019年十月一号，sass团队推出了sass的模块化机制，通过新关键词`@use`、`@forward`，变量、mixin和函数从此拥有了命名空间。并且，sass对已有的内置函数进行了归类和整理，分类到了各个内置模块下。

引入模块化机制，让sass向更成熟的阶段迈进了很大的一步。目前sass的各个实现中，仅Dart Sass 1.23.0完全支持这些新特性。sass团队宣称两到三年后才会完全废弃`@import`等旧语法，现在处于新旧语法共存的过渡时期。
 
 ![compatibility](https://feedback.xingshulinimg.com/1574650108097)

sass的模块化机制显然是一个major的提升，那么，如此大费工夫，它能够解决什么痛点呢？让我们先来看看之前的`@import`所带来的问题吧！

# @import的缺点

`@import`主要有以下5个缺点：

1. 无法知道变量、mixin、函数具体是在哪里定义的。比如说，a.scss文件中定义了变量$height，b.scss文件中引入了a，c.scss文件又引入了b，那么在c文件中，$height是可用的，但无法确定其来源。

2. 嵌套import会导致重复的css代码，还可能引发奇怪的副作用。设想这样一个场景，一个页面中动态引入了一个组件，页面本身需要加载page.css，组件的样式由component.css决定，而这两个样式表的源scss文件中都用到了common.scss，那么在动态引入组件的时候，common.css中的样式就会被重复加载，可能对原有的样式造成覆盖。

3. 因为没有命名空间，css中的选择器又天然是全局的，为了避免撞名，不敢使用简写的`class name`，因此起名总是非常冗长。

4. 没有私有函数的概念。库作者无法确保他们的私有工具函数不会被使用者直接获取，直接使用私有函数可能导致混淆和向后兼容的问题。

5. `@extend`规则可能会影响到样式中的一切选择器，而不是仅仅是作者所希望的那些。（对`@extend`不熟悉的童鞋可以看看这篇文章[the-benefits-of-inheritance-via-extend-in-sass](https://www.sitepoint.com/the-benefits-of-inheritance-via-extend-in-sass/)来具体了解下其使用场景和优缺点。）这一点会在下一节详细论述。

天天写`@import`，没想到它存在的问题还真不少。那么，sass又推出了什么样的功能，能够替代`@import`呢？下面，我们来详细解剖一下sass的模块化机制。首先，就从`@import`的替代者———— `@use`说起。

# `@use`, 模块系统的核心

## `@use`的基本用法

`@use`将取代`@import`，使css，变量，mixin, 函数都可以在不同的样式表中复用。一个样式表文件就是一个模块，其命名空间会基于文件名自动生成，也可以进行自定义命名。变量、mixin和函数会默认在该命名空间下使用。见下例：

```scss
@use "bootstrap" as b;

.element {
  background-color: b.$body-bg;
  @include b.float-left;
}
```
如果使用了`as *`，那么这一模块就处于全局命名空间，可以直接使用其中的变量和mixin。*但注意，如果多个模块暴露出的变量命名重复，并且都使用了`as *`，那么sass会报错[Error: This mixin is available from multiple global modules:]。*

```scss
@use "bootstrap" as *;

.element {
  @include float-left;
}
```
## `@use`与`@import`的区别

`@use`和`@import`的区别在于：

1. 不管使用了多少次样式表，@use都只会引入和执行一次。
2. 与全局使用相反，@use是有命名空间的，而且只在当前样式表中生效。
3. 以`—`或者`_`开头的命名空间被认为是私有的，不会被引入其他样式表。

```scss
/* buttons.scss */
$_height: 20px;

/* tryuse.scss */
@use 'buttons';

.my-buttoon {
  background-color: buttons.$color;
  height: buttons.$_height;
}
```
![私有命名空间](https://qa-feedback.xingshulinimg.com/1572518233040)

4. 如果一个样式表A包括@extend，那么该扩展的作用域仅包括它本身和被它引入的上游模块中的样式规则，而不是引入了样式表A的下游模块。如下例：

```scss
// _upstream.scss
.a {@extend .b};
.b {c: d};

// downstream.scss
@use "upstream";
.b {x: y};

// css结果
.b, .a {
  c: d;
}

.b {
  x: y;
}

// 如果采用@import，则
.b, .a {
  c: d;
}

.b, .a {
  x: y;
}
```
这是很容易理解的，`@import`约等于直接把上游模块的代码粘贴过来，那么在下游文件中对.b的任何css都会被应用于.a选择器中。而`@use`是模块化的，下游文件中的修改不会影响到上游文件。

注意，**具有命名空间的成员只包括变量、mixin和函数。被`@extend`扩展的选择器是不具有命名空间的，占位符选择器也没有命名空间**。使用了`@use`之后，占位符选择器的表现与普通选择器一样，下游模块的代码都不会污染使用了`@extend`的选择器。

由于`@extend`仍然不具备命名空间，所有上游模块的代码均可以影响到全局，而且很难分清到底哪个扩展样式是来自于哪个样式文件。因此，笔者认为仍需要谨慎使用`@extend`，可以更多地利用css层叠继承样式来完成`@extend`的功能。

## 配置样式库的基础变量

在`@import`的时代，很多样式库都需要用户配置全局变量来覆盖用`!default`定义的默认变量。在改为使用`@use`之后，变量不再是全局的了，因此配置变量可以使用更有针对性的`with`语句。

```scss
// bootstrap.scss
$paragraph-margin-bottom: 1rem !default;

p {
  margin-top: 0;
  margin-bottom: $paragraph-margin-bottom;
}

// my-style.scss
@use 'bootstrap' with ($paragraph-margin-bottom: 1.5rem)
```

这样，在my-style.scss文件中，被引用的bootstrap.scss文件里$paragraph-margin-bottom的值就被设成了1.5rem。`with`语句只允许设置被引入模块中已经被定义的默认变量（即使用了`!default`的变量），这样还可以保护用户免于输入错误。

*但要注意的是，一个模块只能被配置一次，那就是在第一次引入的时候。* sass中的引入顺序很重要，最好在入口文件中按顺序引入需要的模块，这样前一个被引入的模块会在下面的模块被引入之前编译好配置项。

# 给库开发者使用的利器@forward

## `@forward`的基本用法

`@forward`语句可以引入另一个模块的所有变量、mixins和函数，将它们直接作为当前模块的API暴露出去，而不会真的在当前模块增加代码。这样，库作者可以更好地在不同源文件之间拆分代码。不同于`@use`，`@forward`不会给变量添加命名空间。

```css
/* bootstrap.scss */
@forward "functions";
@forward "variables";
@forward "mixins";
```

注意，此时生成的bootstrap.css文件中，是不包含"functions"、"variables"、"mixins"代码的，也不能直接在bootstrap.scss文件中使用这些模块。而是需要在另一个文件中`@import`或者`@use`bootstrap模块，再去使用这些方法。**bootstrap.scss文件类似于一个传输中转站，把上下游的成员变量无缝连接起来。**

*注意，直接写在上游模块的样式仍然会被`@forward`进来。见下例：*

```css
/* upstream.scss */
...
footer {
  height: pow(2, 3) * 1px;
  font-weight: map.get($font-weights, "medium");
}

/* downstream.scss */
@forward "upstream.scss";

/* 生成的downstream.css */
footer {
  height: 8px;
  font-weight: 500;
}
```

## 用`show`/`hide`控制成员是否可见

```css
@forward "functions" show color-yiq;
@forward "functions" hide assert-ascending;
```

通过控制`show`和`hide`，可以决定模块中的哪些成员对引入后的模板可见。对隐藏的变量，在下游文件中不可以使用，相当于模块私有成员。

## 给不同的子模块添加前缀

大多数情况下，一个样式库会存在一个入口文件index.scss，然后在index.scss中引入其他的子文件。这种结构类似于`多合一`模块。那么，如果要在某一文件中`@forward`多个子模块，就可以使用as子句，来使子模块下的成员自动带上前缀以区分。

```css
/* material/_index.scss */
@forward "theme" as theme-*;
@forward "func" as func-*;

/* downstream.scss */

@use 'material' as *;

p {
  color: $theme-white;
}
/* 或者，命名空间从父到子 */

@use 'material';

p {
  height: material.func-pow(4, 3) * 1px;
  color: material.$theme-white;
}

/* 也可以在多合一模块中为theme相关的变量重新定义值 */
@use "material" with ($theme-primary: blue);

/* 这等价于： */
@use "material/theme" with ($primary: blue);
```

# sass内置模块

sass引入模块化后，还有一个很大的改变，就是把原来的内置函数归类到了内置模块中。这些内置模块包括：
`sass:math, sass:color, sass:string, sass:list, sass:map, sass:selector, and sass:meta`。使用时需要先`@use`引用内置模块，才能使用模块中的方法。

```scss
@use "sass:color";

.button {
  $primary-color: #6b717f;
  color: $primary-color;
  border: 1px solid color.scale($primary-color, $lightness: 20%);
}
```
代码会被编译为：

```css
.button {
  color: #6b717f;
  border: 1px solid #878d9a;
}
```
这些模块被引入的时候需要加上命名空间，这样可以大大避免sass内置函数与用户自定义函数及css原生方法的命名冲突。此后，sass新增函数也会更加方便。

某些函数在新的内置模块中的命名不同于原先作为全局函数的命名。首先，已经拥有了手动命名空间的内置函数已经被移除了前缀。比如`map-get()、change-color()等`，现在重命名为`map.get()，color.change()`。此外，一些容易被混淆的函数名称也被重新命名了，比如`unitless()`现在改为了`math.is-unitless()`，`comparable()`改为了`math.compatible()`。

color相关的函数，如`lighten()`, `darken()`, `saturate()`, `desaturate()`, `opacify()`, `fade-in()`, `transparentize()`, 和`fade-out()`的行为都非常反直觉，它们对属性的增减是按照静态比例的。因此，这些方法都从新的内置color模块中移除了，取而代之的则是`color.adjust()`函数和`color.scale()`函数。

以下是`color.adjust()`和`color.scale()`的函数签名：

> color.adjust($color,
>  $red: null, $green: null, $blue: null,
>  $hue: null, $saturation: null, $lightness: null,
>  $alpha: null)

>  color.scale($color,
>  $red: null, $green: null, $blue: null,
>  $hue: null, $saturation: null, $lightness: null,
>  $alpha: null)

*二者的区别在于`color.adjust()`是以固定值来改变颜色的属性，而`color.scale()`是动态缩放的。*对`color.adjust()`，$red, $green, 和$blue需要在-255和255之间取值，$hue的值要么以`deg`为单位，要么无单位，$saturation和$lightness需要在-100%和100%之间取值，$alpha需要在-1和1之间取值。对`color.scale()`，取值均需在-100%到100%之间。

下面是一个使用`color.adjust()`的例子：

```css
@use "sass:color";

$c: rgba(144, 233, 12);

p:first-child {
  background: color.adjust($c, $green: 150);
  /* 生成的颜色为rgb(144, 255, 12)，注意生成的rgb中各项取值在0-255之间*/
}

p:nth-child(2) {
  background: $c;
}
```

内置模块`meta`中有一个新增的内置mixin，是`meta.load-css($url，$with:())`。 该mixin可以把$url中css样式全部包含进来。*注意，$url中的函数，变量和mixin在`meta.load-css()后`的scss中并不可用*。 `@meta.load-css`类似于`@use`，可以取代嵌套的`@import`，但只会返回编译后的css代码，还可以在代码中动态使用。

```css
/* index.scss */
@use 'sass:meta';

@function geturl($theme) {
  @return $theme + '.scss';
}

@each $theme in ('dark', 'light') {
  [data-theme='#{$theme}'] {
    @include meta.load-css(geturl($theme));
  }
}

/* _dark.scss */
$base-color:#000 !default;

p {
  background: $base-color;
  color: #fff;
}
```
`meta.load-css()`的第二个参数可以接受一系列配置项，如：

```scss
// 在加载之前配置'theme/dark'文件中的$base-color变量
@include meta.load-css(
  'theme/dark', 
  $with: ('base-color': rebeccapurple)
);
```
**注意，`load-css`中的url不是真的网络地址，而是一个scss文件的相对路径或者绝对路径，就像传给`@use`的参数一样**。

# 兼容性

sass生态系统不会在一夜之间就迁移到`@use`，所以在当下它会和`@import`共存一段时间。它们会以这样的方式兼容：

- 如果一个文件中包含`@import`，它本身又被别的文件以`@use`的方式引用，那么这个文件的全局命名空间下的所有内容构成一个独立的模块。这个模块的成员使用该文件的命名空间，被正常引用。

- 如果一个文件包含`@use`，它本身又被别的文件以`@import`的方式引用，那么所有暴露在该文件公共API上的东西都被会添加到样式表的全局空间中。*注意，该文件私有API并不会自动暴露出来，即以`@use`被引入的上游文件中的API。*这样，一个库可以给导出的东西特定命名，即使对于`@import`而不是`@use`这个库的用户，特定命名也生效。

为了使库作者能在拥有独立命名空间的基础上，保持他们现存的`@import`向的API，新增了对只可`import`文件的支持。如果一个文件`file`被命名为`file.import.sass`，那么它只可被`@import`使用，并且只需写`@import "file"`。

sass团队将会让`@use`和`@import`在未来一段很长的时间内共存，来帮助整个生态流畅地完成迁移。当然，为了简洁、性能和css兼容性，团队最终的目标是完全抛弃掉`@import`。时间线具体如下：

- 在`Dart Sass`和`LibSass`都支持模块系统的一年后，或`Dart Sass`支持模块系统的两年后（以较早者为准，最晚于2021年10月1日），将降级废弃(*deprecate*)`@import`以及全局的核心函数，这些核心函数将只能通过sass内置的模块来调用。

- 该弃用生效一年后（最晚于2022年10月1日），我们将完全放弃对@import和大多数全局函数的支持。到时会发布一个大版本的更新。

简而言之，对`@import`的兼容还会支持至少两年，实际上可能接近三年。

# 迁移指南

随着新模块系统的发布，sass自动迁移工具也发布了出来。它可以帮助你轻松地将大多数样式表迁移到最新版本。安装后，只需在应用内执行如下命令即可

```shell
sass-migrator module --migrate-deps <path/to/style.scss>
```

`--migrate-deps`标志是指需不需要一并迁移那些被引用进来的上游模块。迁移器将自动选取通过`webpack`的`node_modules`语法导入的文件，但也可以使用`--load-paths`标志传递显式的加载路径。

如果想知道将要进行的更改而不实际进行更改，可以同时传递`--dry-run`标志和`--verbose`标志，以使迁移器仅打印出将进行的更改，而不会将其保存到磁盘。

如果想要对一个sass仓库进行迁移，可以执行：

```sh
sass-migrator module --migrate-deps --forward=all <path/to/index.scss>
```

`--forward`标志告诉迁移器添加`@forward`规则，以便用户仍然可以通过`@use`库名来加载库中所有的mixins，变量和函数。如果给库添加了手动命名空间（为了避免名称冲突），则可以通过传递`--remove-prefix`标志删除它。 甚至可以通过传递`--forward=prefixed`来仅仅`@forward`具有前缀的模块成员。

sass并不是孤立的。在实际项目中，往往经由`sass-loader`转换，与其他`loader`一起配合`webpack`打包。接下来，就让我们看看如何在项目中进行sass的迁移。

sass有三种实现：Dart Sass，libsass，和目前已不再支持的Ruby Sass。

- Dart Sass，用Dart语言写的sass实现，于2016年11月1日发布alpha版本。版本1.23.0之后完全支持模块化机制。
- libsass，用c/c++实现的sass，使用非常广泛。`node-sass`是绑定了`libsass`的nodejs库，可以极快的将.scss 文件编译为.css文件。
- Ruby Sass，是最初的Sass实现，但是2019年3月26日被停止了，以后也不会再支持，使用者需要迁移到别的实现上。

从实现上看，显然Dart Sass一骑绝尘，率先支持了模块化机制。相信它将会凭借优异的性能和快速的开发速度，成为sass用户的新宠。

要在webpack中使用Dart Sass也很简单，只需为sass-loader使用恰当的实现即可。根据[sass-loader的文档](https://webpack.js.org/loaders/sass-loader/#root)，只需要在项目中同时安装`sass-loader`和`sass`(即Dart Sass实现，没错，人家的命名就是这么霸气，就叫sass！)即可。

```js
// package.json
{
  "devDependencies": {
    "sass-loader": "^7.2.0",
    "sass": "^1.22.10"
  }
}
```

如果同时安装了`node-sass`和`sass`，那么`sass-loader`会默认使用`node-sass`，可以通过`implementation`选项来改变。

```js
{
  test: /\.s[ac]ss$/i,
  use: [
    'style-loader',
    'css-loader',
    {
      loader: 'sass-loader',
      options: {
        // 优先选用`dart-sass`
        implementation: require('sass'),
      },
    },
  ]
}
```

这样，就可以成功地在项目中使用sass模块语法了。笔者在一个`vue`项目中亲测是可以的，所用到的`webpack`版本是4.41.2，`sass-lodaer`版本8.0.0，`sass`版本为1.23.7。如果在升级sass后遇到`this.getResolve is not a function`的报错，可以考虑升级`webpack`版本试试。

# 参考资料

1. [sass官网](https://sass-lang.com/)
2. [sass官宣: the-module-system-is-launched](http://sass.logdown.com/posts/7858341-the-module-system-is-launched)
3. [introducing-sass-modules](https://css-tricks.com/introducing-sass-modules/)
4. [the-benefits-of-inheritance-via-extend-in-sass](https://www.sitepoint.com/the-benefits-of-inheritance-via-extend-in-sass/)
5. [sass占位符文档](https://sass-lang.com/documentation/style-rules/placeholder-selectors)
6. [为什么要避免使用extend](https://www.sitepoint.com/avoid-sass-extend/)
7. [sass-loader官方文档](https://webpack.js.org/loaders/sass-loader/#root)

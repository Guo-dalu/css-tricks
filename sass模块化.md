https://css-tricks.com/introducing-sass-modules/
http://sass.logdown.com/posts/7858341-the-module-system-is-launched

 and today we're excited to announce that it's available in Dart Sass 1.23.0。sass 1.23.0 compiled with dart2js 2.5.1

@import/@extend 的缺点
1. 无法知道变量、mixin、函数具体是在哪里定义的，因为a.scss文件中的东西在import了a文件的任何文件中都可用。

2. 多次import会导致重复的css代码，还可能引发奇怪的副作用。因为每次stylesheet被import进来，都会从0开始重新加载。

3. 不敢使用简写的`class name`，因为不知道样式表会不会撞名，因此起名总是非常冗长。

4. 库作者无法确保他们的私有工具函数不会被使用者直接获取，直接使用私有函数可能导致混淆和向后兼容的问题。

5. @extend 规则可能会影响到样式中的一切选择器，而不是仅仅是作者所希望的那些。关于extend的使用场景、优点缺点，具体可以看看这篇文章[the-benefits-of-inheritance-via-extend-in-sass](https://www.sitepoint.com/the-benefits-of-inheritance-via-extend-in-sass/)。滥用`@extend`会造成css更加复杂、体积更大，多个语义上不相关的选择器都继承了同一基类。逐一检查每个选择器的css并不现实。只有当所要继承的基类与父类直接相关时，才需要用到@extend。剩下的应当交给css本身的层叠继承规则。

模块系统完全向后兼容。现有功能并没有移除或者退化。模块系统可与@import共存，以使旧代码可以轻松地逐步迁移。sass最终会完全摆脱@import，但那将是很久之后，会留出迁移代码的时间。

# `@use`, 模块系统的核心

@use将取代@import，使css，变量，mixin, 函数都可以在不同的样式表中复用。一个样式表文件就是一个模块，具有命名空间。变量、mixin和函数会默认在该命名空间下使用。见下例：

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

`@use`和`@import`的区别在于：

1. 不管使用了多少次样式表，@use都只会引入和执行一次。
2. 与全局使用相反，@use是有命名空间的，而且只在当前样式表中生效。
3. 以—或者_开头的命名空间被认为是私有的，不会被引入其他样式表。

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

4. 如果一个样式表A包括@extend，那么该扩展起作用的范围只包括它本身和被它引入的上游模块中的样式规则，而不是引入了样式表A的下游模块。如下例：

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

另外注意，占位符选择器没有命名空间。但使用了
`@use`之后，占位符选择器的表现与普通选择器一样，下游模块的代码都不会污染使用了`@extend`的选择器。[sass占位符文档](https://sass-lang.com/documentation/style-rules/placeholder-selectors)。简单来说，占位符就是一种以%开头的特殊选择符，可以被扩展，但本身并不会存在于输出后的css文件中。

关于@extend和@mixin的优劣讨论可以参见[这篇文章: 为什么要避免使用extend](https://www.sitepoint.com/avoid-sass-extend/)，可以说，在sass模块化出来之前，广泛使用`@extend`都不被认为是一个好主意。而现在，相对于变量、`@mixin`和`@function`，`@extend`仍然不具备命名空间，所有上游模块的代码均可以影响到全局，而且很难分清到底哪个扩展样式是来自于哪个样式文件。因此，笔者认为仍需要谨慎使用`@extend`，可以更多地利用css层叠继承样式来完成`@extend`的功能。

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

这样，在my-style.scss文件中，被引用的bootstrap.scss文件里$paragraph-margin-bottom的值就被设成了1.5rem。`with`语句只允许设置被引入模块中已经被定义的默认变量（即使用了`!default`的变量），因此，可以保护用户免于输入错误。

# `@forward`，给库开发者

`@forward`语句可以引入另一个模块的所有变量、mixins，和函数，直接作为当前模块的一部分API暴露出去，而不会真正在当前模块增加代码。这样，库作者可以更好地在不同源文件之间拆分代码。不同于`@use`，`@forward`不会给变量加任何命名空间。

```scss
// bootstrap.scss
@forward "functions";
@forward "variables";
@forward "mixins";
```

注意，此时生成的bootstrap.css文件中，是不包含"functions"、"variables"、"mixins"代码的，也不能直接在bootstrap.scss文件中使用这些模块。而是需要在另一个文件中`@import`或者`@use`bootstrap模块，再去使用这些方法。bootstrap.scss文件类似于一个传输中转站，把上下游的成员变量无缝连接起来。

*注意，直接写在上游模块的样式仍然会被`@forward`进来。见下例*

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

## 控制命名是否可见

```css
@forward "functions" show color-yiq;
@forward "functions" hide assert-ascending;
```

通过控制`show`和`hide`，可以决定模块中的哪些成员对引入后的模板可见。对隐藏的变量，在下游文件中不可以使用，相对于模块私有成员。

## 额外前缀

如果在多合一模块中`@forward`子模块，则可能需要向该子模块手动添加命名空间。可以使用as子句来添加前缀：

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
## 内置模块

新的模块系统也加入了内置模块来维护已经存在的sass方法。这些内置模块包括：
`sass:math, sass:color, sass:string, sass:list, sass:map, sass:selector, and sass:meta`

```scss
@use "sass:color";

.button {
  $primary-color: #6b717f;
  color: $primary-color;
  border: 1px solid color.scale($primary-color, $lightness: 20%);
}
```

```css
.button {
  color: #6b717f;
  border: 1px solid #878d9a;
}
```
因为这些模块被引入的时候需要加上命名空间，因此会大大避免sass函数与css方法的命名冲突。此后，sass新增函数也会更加方便。

### 重命名及移除部分函数

某些函数在内置模块中的命名和全局函数中的命名可能不同。已经拥有了手动命名空间的内置函数已经被移除了前缀。比如`map-get()、change-color()等`，现在重命名为`map.get()，color.change()`。同时，一些容易被混淆的函数名称也被重新命名了，比如`unitless()`现在改为了`math.is-unitless()`，`comparable()`改为了`math.compatible()`。

color相关的函数，如`lighten()`, `darken()`, `saturate()`, `desaturate()`, `opacify()`, `fade-in()`, `transparentize()`, 和`fade-out()`的行为都非常反直觉，它们对属性的增减是按照静态比例的。因此，这些方法都从新的内置color模块中移除了，取而代之的则是`color.adjust()`函数和`color.scale()`函数。

> color.adjust($color,
>  $red: null, $green: null, $blue: null,
>  $hue: null, $saturation: null, $lightness: null,
>  $alpha: null)

>  color.scale($color,
>  $red: null, $green: null, $blue: null,
>  $hue: null, $saturation: null, $lightness: null,
>  $alpha: null)

*二者的区别在于`color.adjust()`是以固定值来改变颜色的属性而`color.scale()`是动态缩放的。*对`color.adjust()`，$red, $green, 和$blue需要在-255和255之间取值，$hue的值要么以`deg`为单位，要么无单位，$saturation和$lightness需要在-100%和100%之间取值，$alpha需要在-1和1之间取值。对`color.scale()`，取值均需在-100%到100%之间。

下面是一个使用color.adjust的例子

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

### `meta.load-css()`

新的模块系统带有一个新的内置mixin，称作`meta.load-css($url，$ with:())`。 该mixin使用给定的$url动态加载模块并包含其CSS（css中的函数，变量和mixin不可用）。 它可以取代嵌套的`@import`，还可以用于动态导入模块。

## `@import`兼容性

sass生态系统不会在一夜之间就迁移到`@use`，所以在当下它会和`@import`共存一段时间。它们会以这样的方式兼容：

- 如果一个文件中包含`@import`，它本身又被别的文件以`@use`的方式引用，那么这个文件的全局命名空间下的所有内容构成一个独立的模块。这个模块的成员使用该文件的命名空间，被正常引用。

- 如果一个文件包含`@use`，它本身又被别的文件以`@import`的方式引用，那么所有暴露在该文件公共API上的东西都被会添加到样式表的全局空间中。*注意，该文件私有API并不会自动暴露出来，即以`@use`被引入的上游文件中的API。*这样，一个库可以给导出的东西特定命名，即使对于`@import`而不是`@use`这个库的用户，特定命名也生效。

为了使库作者能在拥有独立命名空间的基础上，保持他们现存的`@import`向的API，新增了对只可`import`文件的支持。如果一个文件`file`被命名为`file.import.sass`，那么它只可被`@import`使用，并且只需写`@import "file"`。

## 自动迁移

随着新模块系统的发布，sass自动迁移工具也发布了出来。它可以帮助你轻松地将大多数样式表迁移到最新版本。安装后，只需在应用内执行如下命令即可

```shell
sass-migrator module --migrate-deps <path/to/style.scss>
```

`--migrate-deps`标志是指需不需要一并迁移那些被引用进来的上游模块。迁移器将自动选取通过Webpack的`node_modules`语法导入的文件，但也可以使用`--load-paths`标志传递显式的加载路径。

如果想知道将要进行的更改而不实际进行更改，可以同时传递`--dry-run`标志和`--verbose`标志，以使迁移器仅打印出将进行的更改，且不会将其保存到磁盘。

如果想要对一个sass仓库进行迁移，可以执行：

```sh
sass-migrator module --migrate-deps --forward=all <path/to/index.scss>
```
`--forward`标志告诉迁移器添加`@forward`规则，以便用户仍然可以通过单个@use加载库定义的所有mixins，变量和函数。如果为了避免名称冲突，给库添加了手动命名空间，则可以通过传递`--remove-prefix`标志删除它。 甚至可以通过传递`--forward=prefixed`来选择仅`@forward`具有前缀的模块成员。


## 未来计划

sass团队将会让`@use`和`@import`在未来一段很长的时间内共存，来帮助整个生态流畅地完成迁移。当然，为了简洁、性能和css兼容性，团队最终的目标是完全抛弃掉`@import`。时间线是这么安排的：

- 在`Dart Sass`和`LibSass`都支持模块系统的一年后，或`Dart Sass`支持模块系统的两年后（以较早者为准，最迟于2021年10月1日），我们将降级废弃(*deprecate*)@import以及全局的核心函数，这些核心函数将只能通过sass内置的模块来调用。

- 该弃用生效一年后（最晚于2022年10月1日），我们将完全放弃对@import和大多数全局函数的支持。到时会发布一个大版本的更新。

简而言之，对`@import`的兼容还会支持至少两年，实际上可能接近三年。
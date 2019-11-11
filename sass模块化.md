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

注意，此时生成的bootstrap.css文件中，是不包含"functions"、"variables"、"mixins"代码的，也不能直接在bootstrap.scss文件中使用这些模块。而是需要在另一个文件中`@import`或者`@use`bootstrap模块，再去使用这些方法。也就是说，bootstrap.scss文件相当于一个传输中转站，把上下游的代码无缝连接起来，但本身并不承载代码。

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

/* downstream.scss */

@use 'material' as *;

p {
  height: b-pow(4, 3) * 1px;
  color: $b-white;
}
/* 或者，命名空间从父到子 */

@use 'tryuse';

p {
  height: tryuse.b-pow(4, 3) * 1px;
  color: tryuse.$b-white;
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

### 函数重命名

某些函数在内置模块中的命名和全局函数中的命名可能不同。已经拥有了手动命名空间的内置函数已经被移除了前缀。比如`map-get()、change-color()等`，现在重命名为`map.get()，color.change()`。同时，一些容易被混淆的函数名称也被重新命名了，比如`unitless()`现在改为了`math.is-unitless()`，`comparable()`改为了`math.compatible()`。

缺点：整体引入，没有摇树优化。
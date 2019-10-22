https://css-tricks.com/introducing-sass-modules/
http://sass.logdown.com/posts/7858341-the-module-system-is-launched

@import 的缺点
1. 无法知道变量、mixin、函数具体是在哪里定义的，因为a.scss文件中的东西在import了a文件的任何文件中都可用。

2. 多次import会导致重复的css代码，还可能引发奇怪的副作用。因为
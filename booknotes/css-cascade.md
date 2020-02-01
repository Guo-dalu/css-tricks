[css-cascade](https://wattenberger.com/blog/css-cascade)

# four level/type of the importance

首先看类型，以下4种类型，重要性依次降低。

1. transition (active)
2. !important
3. animation (active)
4. normal

# origin

类型相同，看这个规则是在哪里规定的。

1. website
2. user
3. browser 

> 注意，`!important`的优先级在这里是反过来的，即如果浏览器默认设置了`!important`规则，那么它将战胜网站设置的`!important`规则。

# specificity

分为4个等级，优先级逐渐降低：

1. inline
2. id
3. class | attribute | pseudo-class，比如`[checked]`/`[href="https://lala.com"]`/`:hover`
4. type | pseudo-element，比如`::before`/`::selection`

*注意，这里优先级是可以叠加的，类似一个id选择器等于100，一个tag选择器等于1那样的加法*

# position 

写在后面的css规则优先级更高。
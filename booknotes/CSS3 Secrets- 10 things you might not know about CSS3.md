# Bouncing transitions

cubic-bezier(0, 1, 1, 5)

比最终效果超出一点，营造出一种回弹的效果

# Flexible ellipses

给border-radius设置成百分比, 注意高的基准是元素高度，宽的基准是元素宽度

# Multiple outlines

设置多个box-shadow，注意间隔插入与背景色相同的box-shadow。
```css
div {
  box-shadow: 0 0 0 5px rebeccapurple, 0 0 0 15px aquamarine, 0 0 0 22px gold;
  background: aquamarine;
}
```

# make pointer events pass through

`pointer-events`，其实是一个不在css规范中的svg属性，可以取auto, none, inherit。默认为auto。
`pointer-events: none`会禁止鼠标在元素上的行为，元素不会相应鼠标动作，这样，位于其底层的元素就会接收到鼠标事件，就如同上面覆盖的那一层元素不存在一样。可用于取消触发Hover效果等。

![下拉框示例](https://qa-feedback.xingshulinimg.com/1579580022096)
如上图，右侧的图片按钮被设置了`pointer-events: none`之后，就会有“点击穿透”的效果，点击时就可以直接触发input事件了。

如果不支持该属性，可以这么去hack：

```js
function noPointerEvents(element) {
  $(element).bind('click mouseover', (event) => {
    this.style.display = 'none'
    const { x, y } = event
    const under = document.elementFromPoint(x, y)
    this.style.display = ''
    event.stopPropogation()
    event.preventDefault()
    $(under).trigger(event.type)
  })
}
```

# adjusting tab size

`tab-size: <Number>`

# styling based on sibling count

`:only-child`选择器， 等价于 `:first-child:last-child`。

可通过同时让元素满足first-of-type和nth-last-of-type，来确认兄弟元素的个数，进行选择。以下代码将兄弟元素大于等于2个的所有p元素设置成了红字。

```css
p:first-of-type:nth-last-of-type(2),
p:first-of-type:nth-last-of-type(2) ~ p {
  color: red;
}
```
这种方式可应用于多个图片排列的场景。

# custom checkboxes & radio buttons

隐藏原始的input，用`input + label ::before`来写一个自己的checkbox/radio。注意加伪类`:focus/:checked`。

# more cursors for better UX

cursor: none | no-drop | not-allowed | zoom-in | zoom-out | ...

# background patterns with pure css

```css
div {
  background: linear-gradient(0, black 20%, gray 80%);
  background: linear-gradient(0, black 50%, gray 50%); /* 一半黑一半灰色, 截然不同的两半 */
}
```

![菱形格子](https://qa-feedback.xingshulinimg.com/1579597021714)

# background positioning tricks

![](https://qa-feedback.xingshulinimg.com/1579597398618)

通过设置`background-origin: content-box`也可以起到类似的效果(原始值为padding-box)。
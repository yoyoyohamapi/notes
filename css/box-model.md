# Box Model（盒模型）

CSS Box Model 是使用 CSS 布局的理论前提。Web 页面中的每个元素都是一个矩形，如果我们要在页面中放置（布局）这些矩形，那么单纯的考虑矩形的长宽是不够的，我们还要考虑：

- 各个矩形之间的间距，即外边距：Margin
- 矩形边框的粗细：Border Width
- 矩形中内容与边框的距离，即内边距：Padding

因此，在布局时，将元素单纯地抽象为矩形不够，它更适合被抽象为一个盒子：

![](https://i0.wp.com/css-tricks.com/wp-content/csstricks-uploads/thebox.png?resize=570%2C248)

那么，盒子的宽高就是：

|---------|--------|
|Width|width + padding-left + padding-right + border-left + border-right|
|Height|height + padding-top + padding-bottom + border-top + border-bottom|

## 个场景下，盒子的默认宽度

### Block Level 的相对位置盒子宽度

如果盒子的位置是 `static` 或者 `relative` 的，那么：

- 如果没有声明 `width`，那么盒子宽度将充满父元素，同时 padding 和 border 也会向内填充
- 如果声明了 `width: 100%;`，那么盒子的 padidng 和 border 就会向外填充，内容溢出父元素

![](https://i0.wp.com/css-tricks.com/wp-content/csstricks-uploads/weird-1.png?resize=570%2C360)

这也意味着，盒子的默认宽度不是 `100%`，

## 参考资料https://detail.tmall.com/item.htm?id=564889583433&spm=a1z0k.6846577.0.0.52682a069ZtdzD&_u=t2dmg8j26111&skuId=4254181944493

- [The CSS Box Model](https://css-tricks.com/the-css-box-model/)

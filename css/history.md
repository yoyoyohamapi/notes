# CSS 历史

1989 年，万维网（World Wide Web）诞生时，我们拥有了 HTML，它够描述一个网站的内容，但网站作者无法像使用 Word 那样，自由地对网站内容进行布局。

## RRP: Rob Raisch's stylesheet Proposal

第一个为 HTML 声明样式的提议是 Robert Raish 的 [RRP(Rob Raisch's stylesheet
proposal)](http://1997.webhistory.org/www.lists/www-talk.1993q2/0445.html):

```
@BODY fo(fa=he,si=18)
```

这段代码设置了：

- 字体（fo）为 `helvetica`
- 字号（si）为 18

> 这种样式声明并不是很直接，它被设计的短小紧凑的原因的还是首先于当时的网络条件——没有 Gzip，连接速度也慢的可怕。

这个协议也无法对字号这样的样式声明其单位，因为该协议任何数值相关的样式都是由上下文动态决定的，这也是因为当时 RRP 除了支持图形化界面的浏览器之外，还要支持命令行的浏览器（如 [Lynx](<https://www.wikiwand.com/en/Lynx_(web_browser)>)）。

但是 Mosaic 浏览器的作者 Marc Andreessen 并没有在 Mosaic 中实现 RRP 协议，而是使用了 HTML tag 来定义样式，如 `<FONT>` 等。

## PWP: Pei-Yuan Wei's stylesheet Proposal

Mosaic 虽然是最早流行开来的浏览器，但并不是第一个图形化界面的浏览器，Pei-Yuan Wei 在短短四天之内写出的 [ViolaWWW](https://www.wikiwand.com/en/ViolaWWW) 要比 Mosasic 还早。

Pei-Yuan Wei 在这个浏览器中创造了和如今 CSS 类似的，支持嵌套结构的 Stylesheet Language：

```
(BODY fontSize=normal
      BGColor=white
      FGColor=black
  (H1   fontSize=largest
        BGColor=red
        FGColor=white)
)
```

PWP 使用了缩进而不是选择器（selectors）来描述层次，当今流行的 Sass、Stylus 支持的缩进表示，其灵感可能也是来自与此：

```scss
@mixin button-base() @include typography(button) @include ripple-surface
  @include ripple-radius-bounded display: inline-flex position: relative height:
  $button-height border: none vertical-align: middle &: hover cursor: pointer &:
  disabled color: $mdc-button-disabled-ink-color cursor: default pointer-events:
  none;
```

PWP 也描述了如何在 HTML 文档中引入外部的样式表：

```html
<link rel="STYLE" href="URL_to_a_stylesheet" />
```

特性优异的 ViolaWWW 却不支持 Windows，导致它在浏览器历史上也只是昙花一现。

## Web 之前的 Stylesheet

在 Internet 之前，就已经有了描述一篇文档其样式的需求了。1987 年，美国国防部整研究是否能让 SGML（HTML 的前身）更加易于存储和传输，成立了研究小组 CALS（Continuous Acquisition and Life-cycle Support）。CALS 创建了一门名叫 [FOSI(Formatting Output
Specification Instance)](http://xml.coverpages.org/fosiagnw.html) 的语言来描述 SGML 文档的样式：

```html
<outspec>
  <docdesc>
    <charlist>
      <font size="12pt" bckcol="white" fontcol="black">
    </charlist>
  </docdesc>
  <e-i-c gi="h1"><font size="24pt" bckcol="red", fontcol="white"></e-i-c>
  <e-i-c gi="h2"><font size="20pt" bckcol="red", fgcol="white"></e-i-c>
  <e-i-c gi="a"><font fgcol="red"></e-i-c>
  <e-i-c gi="cmd kbd screen listing example"><font style="monoser"></e-i-c>
</outspec>
```

> e-i-s 即表示 element-in-context

FOSI 为人称道的是引入了 `em` 单位，这是我们现在 CSS 中常用的尺寸单位。

## 图灵完备的 Stylesheet

DSSSL 是一门图灵完备的，用来定义文档样式的函数式编程语言：

```
(element H1
  (make paragraph
    font-size: 14pt
    font-weight: 'bold))
```

由于它是一门编程语言，因此你当然可以在其中函数：

```
(define (create-heading heading-font-size)
  (make paragraph
    font-size: heading-font-size
    font-weight: 'bold))

(element h1 (create-heading 24pt))
(element h2 (create-heading 18pt))
```

但是和任何函数式编程语言面临的问题一样，括号太多了，对于新手开发者门槛也过高了。

## 为什么样式是自顶向下定义的

在早期的网络环境下，我们希望在文档被完整加载之前，页面就应该被渲染出来，如果使用了 DSSSL 这样的语言来定义样式，那么加载文档的同时，就需要实时的去更新文档，同理，这也是为什么 CSS 标准一直没有采纳万众期待的 Parent Selector，因为子元素加载后，我们又需要修改父元素的样式。

针对此，1995 年，Bert Bos 提出了一个新的样式定义语言：

```
*LI.prebreak: 0.5
*LI.postbreak: 0.5
*OL.LI.label: 1
*OL*OL.LI.label: A
```

该语言使用了 `.` 声明直接的孩子节点，使用 `*` 声明了祖先。

这门语言甚至支持控制元素的行为：

```
*A.anchor: !HREF
```

上面这段代码就可以控制链接的跳转地址为其 `HREF` 属性，别惊讶于此，因为在 JavaScript 时代之前，我们是没有办法通过脚本实现这样的行为的。

1994 年 C.M. Sperberg-McQueen 也提出了一个函数式的语言提议来定义类似的行为：

```
(style a
  (block #f)     ; format as inline phrase
  (color blue)   ; in blue if you’ve got it
  (click (follow (attval 'href)))  ; and on click, follow url
```

## PSL96

1996 年的 PSL（Presentation Specification Language）已经和 CSS 非常相似了：

```
H1 {
  fontSize: 20;
}
```

同 DSSSL 一样，成也灵活性，败也灵活性，不同浏览器可能会有非常大的实现差异。另外，它们也都只活跃在学术界，而在商业浏览器领域，从未掀起过水花。

```
LI {
  if (ChildNum(Self) == round(NumChildren(Parent) / 2 + 1)) then
    VertPos: Top = Parent.Top;
    HorizPos: Left = LeftSib.Left + Self.Width;
  else
    VertPos: Top = LeftSib.Actual Bottom;
    HorizPos: Left = LeftSib.Left;
  endif
}
```

## CSS 前身 - CHSS

CSS 的直接前身是 Håkon W Lie 于 1994 年提出的 CHSS（Cascading HTML Style Sheets）：

```
h1.font.size = 24pt 100%
h2.font.size = 20pt 40%
```

规则里面最引人瞩目的是百分数，它描述了当前的样式表的样式贡献占比（ownership），例如，如果之前的一个样式表定义了：

```
h2.font.size = 30pt 60%
```

那么最终的 `h2` 的样式就是 `30*0.6 + 20*0.4 = 26pt`。

CHSS 的贡献也在于引入了 Cascading 的概念，允许一个网站加载并且融合多个 Stylesheet。

## 脱颖而出的为什么是 CSS？

经过对提议的简化，以及与 Bert Bos 合作之后，Håkon Lie 于 1996 年发布了第一版 CSS 规范。

如果一门技术足够强大，但是只能被专家掌握，那么它也很难流行开来，CSS 最终能够脱颖而出，也是胜在了简单，解析简单，写起来容易，读起来也不费力气。另外，有别于其他样式定义语言，CSS 的 `cascading` 特性能够融合（通过继承或者重写）不同样式诉求。

CSS 不少规则的引入也是偶然，例如上下文选择器（contextual selectors）：

```css
body ol li {
}
```

支持这个特性只是因为 Netscape 浏览器已经有了一个能够删除超链接图片边框的方法，CSS 要想实现浏览器这个功能就需要上下文选择器：

```css
a {
  img {
    border: none;
  }
}
```

这个特性让 CSS 的实现推迟了非常之久，因为当时绝大多数浏览器解析 HTML 后，并没有输出一个 HTML tag stack，导致要实现上下文选择器，就需要重新设计 parser。

各种挑战让 CSS 知道 1997 年仍不可用，知道 2000 年 5 月份也没被任何浏览器完整实现，直到规范发布的 15 年后（2009 年），浏览器对 CSS 的支持才达到标准。

## 鹿死谁手

IE 3 以支持 CSS 作为卖点退出，迫于竞争压力，Netscape 4 也决定支持这门语言，但是它是将 CSS 能力嵌入到了 JavaScript 中，而不是再多支持一门语言：

```js
tags.H1.color = "blue";
tags.p.fontSize = "14pt";
with (tags.H3) {
  color = "green";
}

classes.punk.all.color = "#00FF00";
ids.z098y.letterSpacing = "0.3em";
```

因此，你可以使用函数来定义动态样式：

```js
evaluate_style() {
  if (color == "red"){
    fontStyle = "italic";
  } else {
    fontWeight = "bold";
  }
}

tag.UL.apply = evaluate_style();
```

> 这种 CSS In JS 的理念，在如今的 React 社区中也颇为流行。

当时，JavaScript 还是个新生儿，而 CSS 在社区已经声名鹊起了，Netscape 的做法看起来就有些本末倒置了，因此，当它向标准委员会提交 JSSS 时，委员会对此 “充耳不闻”，三年之火，Netscape 也放弃了对 JSSS 的支持。

2000 年发布的 IE 5.5 最终带来了对 CSS1 的完整支持，但各个浏览器对 CSS 的支持在往后十年仍然 bug 不断。

## 参考资料

- [The Languages Which Almost Became CSS](https://eager.io/blog/the-languages-which-almost-were-css/)
- [A Look Back at the History of CSS](https://css-tricks.com/look-back-history-css/)
- [A brief history of CSS until 2016](https://www.w3.org/Style/CSS20/history.html)

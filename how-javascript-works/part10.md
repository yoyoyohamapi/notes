# a追踪 DOM 变化

## 使用 MutationObserver 

```js
const mutationObserver = new MutationObserver(mutations => {
  mutations.forEach(mutation => {
    console.log(mutation)
  })
})

mutationObserver.observe(document.documentElement, {
  attributes: true,
  characterData: true,
  childList: true,
  subtree: true,
  attributeOldValue: true,
  characterDataOldValue: true
})

document.querySelector('#sample-div').removeAttribute('class')
```

## 使用 CSS Animation

以追踪 DOM 插入为例：

```html
<div id=”container-element”></div>
```

CSS：

```css
@keyframes nodeInserted { 
 from { opacity: 0.99; }
 to { opacity: 1; } 
}

#container-element * {
 animation-duration: 0.001s;
 animation-name: nodeInserted;
}
```

JS：

```js
const insertionListener = function(event) {
  // Making sure that this is the animation we want.
  if (event.animationName === "nodeInserted") {
    console.log("Node has been inserted: " + event.target);
  }
}

document.addEventListener(“animationstart”, insertionListener, false); // standard + firefox
document.addEventListener(“MSAnimationStart”, insertionListener, false); // IE
document.addEventListener(“webkitAnimationStart”, insertionListener, false); // Chrome + Safari
```


# vite-plugin-shadow-dom-css

Vite Shadow DOM CSS 插件。

- 能够将 CSS 插入到 Shadow DOM 中
- 开发环境支持热更新

## 为什么要使用此插件

当你试图使用 Vite 作为构建工具，来编写和 Web Components 相关的组件的时候，你会发现样式永远无法生效： 

```js
import myStyle from './my-style.css';
export class MyElement extends HTMLElement {
  constructor() {
    this.attachShadow({ mode: 'open' });
    myStyle(this.shadowRoot).mount();
    this.shadowRoot.innerHTML = `
      <h1>Hello world</h1>
    `;
  }
}
customElements.define('my-element', MyElement);
```

```html
<html>
  <head>
    <style>
      /*...*/
    </style>
  </head>
  <body>
    <my-element>
      #shadow-root
      	<h1>Hello world</h1>
    </my-element>
  </body>
</html>
```

在开发模式中，我们会在 `<head>` 中得到由 Vite 生成的 `<style>`；在生产模式中，我们将得到 .css 文件，这些 .css 文件需要有我们额外通过 `<link>` 标签输出。这两种模式都不能让 CSS 工作在 Shadow DOM 中，因为 Shadow DOM 是一个隔离的样式环境。

### 使用 CSS in JS

要解决这样的问题，你需要使用 `CSS in JS` 的方案，它对 Shadow DOM 非常友好。

```js
import myStyle from './my-style.css?inline';
export class MyElement extends HTMLElement {
  constructor() {
    this.attachShadow({ mode: 'open' });
    myStyle(this.shadowRoot).mount();
    this.shadowRoot.innerHTML = `
      <h1>Hello world</h1>
      <style>${myStyle}</style>
    `;
  }
}
customElements.define('my-element', MyElement);
```

```html
<my-element>
  #shadow-root
    <h1>Hello world</h1>
    <style>
      /*...*/
    </style>
</my-element>
```

### 问题

#### 依赖失去控制

并不是所有的外部组件都使用了 CSS in JS，这些组件会导致我们需要花费大量的时间处理样式丢失问题：

```js
import '@ui/my-button';
```

```js
// @ui/my-buttom
import './style.css';
// ...
```

#### 无法热更新

当使用 `inline` CSS 后，Vite 的热更新与自动刷新功能都无法工作。

## 使用

```js
// vite.config.js
import { defineConfig } from 'vite'
import shadowDomCss from 'vite-plugin-shadow-dom-css'

export default defineConfig({
  plugins: [shadowDomCss()]
})
```

```js
// src/main.js
import myStyle from 'virtual:style-provider!./my-style'; // 删除了 .css 后缀名，避免 Vite 内部插件二次处理
export class MyElement extends HTMLElement {
  constructor() {
    this.attachShadow({ mode: 'open' });
    this.shadowRoot.innerHTML = '<h1>Hello world</h1>';
    myStyle(this.shadowRoot).mount();
  }
}
customElements.define('my-element', MyElement);
```

### 注意事项

由于 Vite 的限制，不得不使用如下方式：

* CSS 路径之前需要增加 `virtual:style-provider!` 前缀
* CSS 路径不能包含 `.css` 字符，一旦这样会被 Vite 内置 CSS 后处理插件进行加工

### 通配符

对于 [vite-plugin-style-import](https://github.com/anncwb/vite-plugin-style-import) 这样的插件，为我们提供了按需引入组件 CSS 的方式，我们可以结合 vite-plugin-shadow-dom-css 来管理样式

```js
// src/main.js
import app from './app';
import allStyleProvider from 'virtual:style-provider?query!./**';

const allStyleProvider = myStyle(document.head);
allStyleProvider.mount();
```

```js
// src/app.js
import 'virtual:style-provider!./button/button-style';
import 'virtual:style-provider!./dialog/dialog-style'; 
```

## API


```js
import myStyle from 'virtual:style-provider!./my-style';

/* ... */
const styleProvider = myStyle(shadowRoot);
// 挂载样式
styleProvider.mount();
// 卸载样式
styleProvider.unmount();
```


## 为什么这么设计这个插件

虽然 Web Components 充满了希望，但如今要在工程中使用它将会面临非非常多的问题，而样式的工程化首当其冲。Vite 与其他流行构建工具并没有很好的解决这样的问题，而开发社区中只有零星的文章讨论到它，所以这个插件是一次解决问题的尝试。

这个插件实现过程中遇到诸多困难，这也许是 Vite 的插件设计导致的：

* Vite 和 Webpack 不同，它没有 Loader 这样的概念，这意味着插件之间几乎无法组合使用
* Vite 的内部插件会自动处理 `.css` 文件，所以 vite-plugin-shadow-dom-css 必须通过奇怪的虚拟路径才能绕开它，否则经过 Vite 的内部插件加工后的 CSS 文件将不可用

## 限制

* 不支持 CSS 格式之外的文件
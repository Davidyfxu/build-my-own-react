# React 源码

**来源**：[https://pomb.us/build-your-own-react/](https://pomb.us/build-your-own-react/)

## Review

JSX: 像babel一样，把代码转换成`createElement`的方法形式，传递给tag、props和children元素。（children在此情况下是一个string，但通常是包含多元素的array）

```jsx
const element = <h1 title="foo">Hello</h1>
const element = {
  type: "h1",
  props: {
    title: "foo",
    children: "Hello",
  },
};
```

ReacDOM.render: React改变DOM节点的地方。

为了避免疑惑，用element代表react elements (**React.createElement**创建)，用node代表DOM elements (**document.createElement(element.type)**创建)。用node方法和element方法可以互相转换，但node方法更类似于原生前端语法。ReactDOM库内置的方法把JSX转成了原生。

## The `createElement` Function

手写createElement函数，着重处理children中的元素是string还是object这两种情况，如果是string还需要转换成object。

```jsx
function createElement(type, props, ...children) {
  return {
    type,
    props: {
      ...props,
      children: children.map((child) =>
        typeof child === "object" ? child : createTextElement(child)
      ),
    },
  };
}
function createTextElement(text) {
  return { type: "TEXT_ELEMENT", props: { nodeValue: text, children: [] } };
}
```

我们手造一个Didact替代React

```jsx
const Didact = { createElement };
const element = Didact.createElement(
  "div",
  { id: "foo" },
  Didact.createElement("a", null, "bar"),
  Didact.createElement("b")
);
```

转换成jsx来描述，就是`<div id='foo'><a>bar</a><b /></div>`类型。（**需要完成下面一个step才能正常渲染页面**）

## The `render` Function

手写render函数，把render加入Didact声明中。初期只考虑create，不考虑delete和update。在render阶段，我们用一个递归来处理children。

注意：

1. 对于文本类型的dom，需要和createElement一样做三元处理。
2. 分配属性给node节点 `dom[name] = element.props[name]`。

```jsx
function render(element, container) {
  const dom =
    element.type === "TEXT_ELEMENT"
      ? document.createTextNode("")
      : document.createElement(element.type);
  // 加入props
  const isProperty = (key) => key !== "children";
  Object.keys(element.props)
    .filter(isProperty)
    .forEach((name) => {
      dom[name] = element.props[name];
    });
  // 递归
  element.props.children.forEach((child) => render(child, dom));
  container.appendChild(dom);
}
```

**到此阶段，可以正常渲染页面了。**

## Concurrent Mode



## Fibers



## Render and Commit Phases



## Reconciliation



## Function Components



## Hooks

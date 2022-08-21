# React 源码

**来源**：[https://pomb.us/build-your-own-react/](https://pomb.us/build-your-own-react/)

**优秀译文**：
https://www.sytone.me/build-your-own-react#fd94056e7c204c98a675504357f44cfe

https://juejin.cn/post/6884968140892176397

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

**并发模式**：上一节中，我们采用递归的方式来渲染，但万一由于递归太深中途渲染中断，则浏览器会被迫终止，因此我们需要划分小单元。假如某个单元需要被停止，我们就应该让浏览器停止这个小单元。

**requestIdleCallback**：写一个loop作为延时器，当主线程空闲的时候浏览器调用callback（React不再使用此方法，而是使用scheduler调度，但原理一致）。requestIdleCallback需要一个deadline参数，用来得知浏览器再次控制的时间还有多久。**浏览器将在主线程空闲时进行回调，而不是指定回调何时运行。**

**performUnitOfWork**：实现工作，并返回下一个工作单元。**要开始循环检查之前，我们需要设置第一个工作单元。**

```jsx
let nextUnitOfWork = null;
function workLoop(deadline) {
  let shouldYield = false;
  while (nextUnitOfWork && !shouldYield) {
    nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
    shouldYield = deadline.timeRemaining() < 1;
  }
  requestIdleCallback(workLoop);
}
requestIdleCallback(workLoop);
function performUnitOfWork(nextUnitOfWork) {
  // TODO
}
```

## Fibers

**Fiber树**：工作单元的数据结构。

**原因**：这块之前我始终没有理解，为什么有了element还要为每个element分配一个Fiber，而每个Fiber才会最终成为一个工作单元。element是一个对象的结构，仅仅记录了jsx的基本信息，而Fiber是一个双向链表+多叉树的结构，便于双向迭代/递归。设计这个数据结构的目标之一是：使查找下一个工作单元变得更加容易。这就是为什么**每一个 Fiber 都会链接到其第一个子节点，下一个兄弟姐妹节点和其父节点。**

![img](https://cdn.nlark.com/yuque/0/2022/png/26911683/1660711952926-fac8a007-5ce6-4fbe-a0b4-58c7b8e9310a.png)

在`render`阶段，假设我们形成了element tree，如下。

```jsx
Didact.render(
  <div>
    <h1>
      <p />
      <a />
    </h1>
    <h2 />
  </div>,
  container
)
```

在`Didact.render`中，将创建Root Fiber并把它设为`nextUnitOfWork`，然后剩下的工作便是`performUnitOfWork`函数。

对于每个Fiber：

1. 把element加入DOM。
2. 为每个element子节点创建Fiber。
3. 挑选下一个合适的工作单元。

对于上图，root先寻找child，接着div->h1->p->a->h1->h2->div->root，工作结束

开始改造render，形成createDom的方法，保留原本dom的生成方法。

## Render and Commit Phases

**分阶段的原因**：每次我们处理一个element然后加一个新节点到DOM中，但浏览器可能在我们结束渲染整棵树前终止，这就有可能导致不完整UI，给用户带来歧义。因此，我们需要删去部分可变DOM。

为了实现这个优化，我们需要追踪Fiber树的根节点，用`wipRoot`来描述。一旦完成了整棵树render，我们才会commit整棵树到DOM。

`commitRoot`: 递归添加节点到dom。(中序遍历)

```jsx
function commitWork(Fiber) {
  if (Fiber) {
    const domParent = Fiber.parent.dom;
    domParent.appendChild(Fiber.dom);
    commitWork(Fiber.child);
    commitWork(Fiber.sibling);
  }
}

function commitRoot() {
  // add nodes to dom
  commitWork(wipRoot.child);
  wipRoot = null;
}
```

## Reconciliation

用协调器来做更新或删除节点的操作。为了update或delete，我们需要将最新的render的Fiber tree和committed的Fiber tree做diff。

因此，我们需要在commit结束后，保留上次committed的Fiber tree的指针，同时加一个`alternate`属性给每一个Fiber，这个属性连接着旧的Fiber（committed Fiber），方便之后做diff。

提取`performUnitOfWork`函数到`reconcileChildren`函数，用于协调新老fibers的diff。

`reconcileChildren`函数：同时迭代老Fiber的Children和新调度的elements，忽略两者的数据结构，`element`是将渲染到DOM的内容，`oldFiber`是上次渲染到DOM的内容。

`React Diff`原则：

1. 老Fiber和新element有同样的type -> 保持原DOM，更新props
   基于老Fiber的DOM和新element的props创建新Fiber，同时加一个新属性`effectTag`（设为UPDATE）到Fiber之后commit阶段使用。
2. 有不同的type且element为全新的 -> 新建DOM
   `effectTag` 设置为PLACEMENT。
3. 有不同的type且有老Fiber -> 删除DOM
   把effectTag加入老Fiber设为DELETION，需要加一个deletions的array来记录需要删除的Fiber，当Commit所有改变到DOM时，我们从array中提取Fiber做删除。

> 注意：React也使用keys促进协调器的Diff加速。

对`commitWork`函数做effectTag识别处理，对不同effectTag分别做增删改操作。

## Function Components

函数组件不同点：

1. 函数组件的Fiber没有DOM
2. children来自于函数，而不是props

在`performUnitOfWork`中加一个判断函数组件的函数，用`updateFunctionComponent`函数来做额外处理。

**Commit时需要寻找DOM node**：因为函数组件Fiber没有DOM，所以需要做额外处理。

1. 修改寻找 DOM 父节点的逻辑：顺着 Fiber Tree 向上找直到找到有 DOM node 的 Fiber。
1. 删除节点的时候，我们需要向下找到带DOM节点的child将其删除。

`/** @jsx Didact.createElement */`作用：当babel转义jsx时，将使用我们定义的功能。



## Hooks

`Didact.useState` 用数组代替链表

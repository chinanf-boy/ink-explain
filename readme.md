# ink

「 对CLI做出-React。 使用组件构建和测试您的CLI输出. 」

[![explain](http://llever.com/explain.svg)](https://github.com/chinanf-boy/Source-Explain)
    
Explanation

> "version": "0.4.1"

[github source](https://github.com/vadimdemedes/ink)

~~[english](./README.en.md)~~

---

### 本次例子

-[官方](https://github.com/vadimdemedes/ink#usage)

<details>

```jsx
const {h, render, Component, Text} = require('ink');

class Counter extends Component {
	constructor() {
		super();

		this.state = {
			i: 0
		};
	}

	render() {
		return h('div', {}, [
			h('div', {}, []),
			h('div', {}, [
				h(Text, {blue: true}, '~/Projects/ink ')
			]),
			h('div', {}, [
				h(Text, {red: true}, 'λ '),
				h(Text, {green: true}, 'node '),
				h(Text, {}, 'media/example')
			]),
			h(Text, {green: true}, `${this.state.i} tests passed`)
		]);
	}

	componentDidMount() {
		console.log()
		this.timer = setInterval(() => {
			if (this.state.i === 50) {
				process.exit(0); // eslint-disable-line unicorn/no-process-exit
			}

			this.setState({
				i: this.state.i + 1
			});
		}, 100);
	}

	componentWillUnmount() {
		clearInterval(this.timer);
	}
}

render(h(Counter));

```

<p align="center">
  <img src="media/demo.svg" width="600">
</p>

</details>


---

本目录

---

可以看到例子运行, `render(h(Counter));` 首先触发 `h`

---

## 0. h

`ink/lib/h.js`

> 将-`Component`/`"div/span/br"`-变成-`Vnode`

<details>

``` js
'use strict';

const flatten = require('lodash.flattendeep');
const VNode = require('./vnode');

// class 其实也是 函数的一种
module.exports = (component, props, ...children) => {
	if (typeof component !== 'function' && typeof component !== 'string') {
		throw new TypeError(`Expected component to be a function, but received ${typeof component}. You may have forgotten to export a component.`);
	}

	props = props || {};

	const readyChildren = [];

	// 孩子 
	if (children.length > 0) {
		props.children = children;
	}

	// 对那些 h(Component) / 其他类型 孩子改造
	flatten(props.children).forEach(child => {
		if (typeof child === 'number') {
			// 数字 孩子
			child = String(child);
		}

		if (typeof child === 'boolean' || child === null) {
			// 异类孩子
			child = '';
		}

		if (typeof child === 'string') {
			// 字符串孩子
			if (typeof readyChildren[readyChildren.length - 1] === 'string') {
				// 最后孩子是否同样是 字符串
				// 合并
				readyChildren[readyChildren.length - 1] += child;
			} else {
				readyChildren.push(child);
			}
		} else {
			// 剩下的就是 h(Component) -> new VNode 孩子
			readyChildren.push(child);
		}
		// 不管怎么样都放进 readyChildren
	});

	props.children = readyChildren;
// 把一系列的 选项打入 props 直接交给 VNode
	return new VNode(component, props);
};

```

- VNode

> 1. 有两种组件 div/span/br string 类型 | 通过 extend Component 类型

> 2. VNode类可以说扩展-Component类, 也就是只是其中一个 VNode.component = component 属性

``` js
const getComponent = name => {
	switch (name) {
		case 'div': return Div; // 相当于 孩子 + 换行
		case 'span': return Span; // 相当于 孩子
		case 'br': return Br; // // 相当于 换行
		default: return null;
	}
};

class VNode {
	constructor(component, props = {}) {
		
		const ref = props.ref;
		delete props.ref;

		// 组件类
		this.component = typeof component === 'string' ? getComponent(component) : component;
		// 分析选项
		this._props = transformProps(props);
		this._children = [];
		// 指定的组件
		this.ref = ref;
		this.instance = null;
	}

	// 定义 - 属性规则
	get props() {
		return this._props;
	}

	set props(nextProps) {
		this._props = transformProps(nextProps);
		return this._props;
	}

	get children() {
		return this._children;
	}

	set children(nextChildren) {
		this._children = flatten(arrify(nextChildren));
		return this._children;
	}

	// 在一定时候, 需要将  Component 类 实例化
	createInstance(props) {
		// 只会将 继承 Component 的类实例化
		if (isClassComponent(this.component)) {
			this.instance = new this.component(props, {});
		}
	}
}

if (process.env.NODE_ENV !== 'production') {
	// 确定一个 不可变类型
	VNode.prototype.$$typeof = Symbol.for('react.element');
}

module.exports = VNode;

```

---

通过 `h` 和 `VNode` 的 洗刷

例子🌰中的 `Counter` -> `VNode`

要⚠️注意的是 `Coumter.render` 里面的 渲染, 并没有触发。

VNode(Counter) 只是一个很原型的 虚拟节点「VNode」, 没有孩子

👇下一步, 我们把 统一好的`VNode` -> [`render(VNode)`](#1-render)

> 但是在 `render` 中, 主要说得是, 接管终端 输出/ 输入

</details>


---

## 1. render

`ink/index.js`

代码 30-119

> 接管终端 输出/ 输入, 和重覆盖

<details>

``` js
exports.render = (tree, options) => {
    // tree == h(Counter)
	if (options && typeof options.write === 'function') {
		options = {
			stdout: options
		};
	}

	const {stdin, stdout} = Object.assign({
		stdin: process.stdin, // cli-输入
		stdout: process.stdout // cli-输出
	}, options);

	const log = logUpdate.create(stdout); // log-update 是 对 通过覆盖终端中的前一个输出进行记录。

	const context = {};
	let isUnmounted = false;
	let currentTree;

    readline.emitKeypressEvents(stdin);
    // 相应于接收到的输入触发 'keypress' 事件。

	if (stdin.isTTY) {
        stdin.setRawMode(true);
//把 tty.ReadStream 配置成原始模式。
// 在原始模式中，输入按字符逐个生效，但不包括修饰符。
	}

	const update = () => {
		const nextTree = build(tree, currentTree, onUpdate, context); // 确定✅-下一个树
		log(renderToString(nextTree)); // 覆盖

		currentTree = nextTree;
	}; // 更新终端视图

	const onUpdate = () => { // 给予 diff 更新函数
		if (isUnmounted) {
			return;
		}

		update();
	};

	update(); // 先运行一边

	const onKeyPress = (ch, key) => {
		if (key.name === 'escape' || (key.ctrl && key.name === 'c')) {
			exit();// 退出
		}
	}; // 终端输入键-监控-触发事件

	if (stdin.isTTY) {
		stdin.on('keypress', onKeyPress); // 监控输入键
		stdout.on('resize', update); // 监控 终端重载
	}

	const consoleMethods = ['dir', 'log', 'info', 'warn', 'error'];

	consoleMethods.forEach(method => {
		const originalFn = console[method];

		console[method] = (...args) => {
			log.clear();
			log.done();
			originalFn.apply(console, args);
			update();
		};

		console[method].restore = () => {
			console[method] = originalFn;
		};
	}); // 控制 console.** 函数 对终端视图的输出

	const exit = () => {
		if (isUnmounted) { // 已经拆了
			return;
		}

		if (stdin.isTTY) {
			stdin.setRawMode(false); // 默认模式
			stdin.removeListener('keypress', onKeyPress); // 移除
			stdin.pause(); // 输入暂停
			stdout.removeListener('resize', update); // 移除
		}

		isUnmounted = true; // 拆了
        build(null, currentTree, onUpdate, context);
        // 终端视图-归零
		log.done(); // 最重要的是这个, 放开终端输出

		consoleMethods.forEach(method => console[method].restore()); // 把原来的 换回去
	};

	return exit;
};

```

- [readline.emitKeypressEvents](http://nodejs.cn/api/readline.html#readline_readline_emitkeypressevents_stream_interface)

> 相应于接收到的输入触发 'keypress' 事件。重置所有事件, 然后自定义事件

- [stdin.setRawMode(true)](http://nodejs.cn/api/tty.html#tty_readstream_setrawmode_mode)

> 配置成原始模式。在原始模式中，输入按字符逐个生效，但不包括修饰符。

- [require('log-update')](https://github.com/sindresorhus/log-update)

> 说白了, 所有的类React, 组件定义等等, 最后都输出到 `log-update` 这个控制终端输出的库中, 通过覆盖终端中的前一个输出进行记录。用于渲染进度条，动画等! `log-update by sindresorhus`

- [build](#2-build)

> 用来建立新的-视图树`Tree`, 也就`❤️最重要❤️`的

- [renderToString](#4-rendertostring)

> 把 视图树`VNode 类` 变成 `String`， 给 `log-update` 使用

> `log(renderToString(nextTree));`

</details>

---

## 2. build

`ink/lib/index.js`

> 用来建立新的-视图树`Tree`

代码 19-23

``` js
const noop = () => {};

const build = (nextTree, prevTree, onUpdate = noop, context = {}) => {
    // 初始化 prevTree undefaulted
	return diff(prevTree, nextTree, onUpdate, context);
};
```

那么到了这里-`diff`-「有关 比较新旧树 触发生命周期钩子, 改变-虚拟树」

> 但在这例子中, 仅仅是自己与自己的状态比较, [`有关setState 的触发机制`](#5-setstate)

---

---

## 3. diff

`ink/lib/diff.js`

我们前面已经说明了, 在如何接管-终端输出和输入, 已达到覆盖的动画等效果

> 对比-视图树, 返回新或者没有变的树🌲

<details>

>⚠️ onUpdate 也跟进来了, 这是有关[setState 触发重覆盖的问题](#5-setstate)

``` js
const diff = (prevNode, nextNode, onUpdate, context) => {
// 初始化时, prevNode 未声明
	if (typeof nextNode === 'number') { // 如果是 数字
		if (prevNode instanceof VNode) {
			unmount(prevNode);
		}

		return String(nextNode); // 转 字符串
	}

	if (!nextNode || typeof nextNode === 'boolean') { 
        // 如果是 正负
		if (prevNode instanceof VNode) {
			unmount(prevNode);
		}

		return null; // 返回 🈳️
	}

	if (typeof nextNode === 'string') { // 如果 字符串
		if (prevNode instanceof VNode) {
			unmount(prevNode);
		}

		return nextNode; // 返回 下个🌲
	}

    let isPrev = true; // 如果true , 不需要更新 
    // 其实改为 isPrev 可能好理解点, just do it

	if (!(prevNode instanceof VNode)) { // 如果上一个不是VNode
		mount(nextNode, context, onUpdate);
		isPrev = false; // 如果false, 更新-新的
	}

	if (isPrev && prevNode.component !== nextNode.component) { // 直接对比 Component 类
        unmount(prevNode); 
        // 移除旧, 触发 componentWillUnmount
        mount(nextNode, context, onUpdate); 
        // 加载新的 注意⚠️ onUpdate 是覆盖-终端输出的关键
        // 触发 componentWillMount
		isPrev = false;
    } 
    
// 如果到这里 isPrev 还是 == true, 说明自己和自己,判断还没有结束
//⏰ 下面这部分, 就开始细分更新, 和 剩下组件事件钩子的运行

    // 对比 Component props
    const shouldUpdate = isPrev && shouldComponentUpdate(prevNode, getProps(nextNode), getNextState(prevNode));

// 为什么是 prevNode, 因为还是 isPrev , 这里就自己和自己比较的开始了

    // 旧树的孩子
	const prevChildren = isPrev ? [].slice.call(prevNode.children) : [];

	if (isPrev && !isEqualShallow(getProps(prevNode), getProps(nextNode))) {
		componentWillReceiveProps(prevNode, getProps(nextNode));
	}

	if (shouldUpdate) { // 应该重新渲染
		rerender(prevNode, context); // ???
	}

    // 下一个树孩子
	const nextChildren = isPrev ? prevNode.children : nextNode.children;

	const length = Math.max(prevChildren.length, nextChildren.length);
	const reconciledChildren = [];

	for (let index = 0; index < length; index++) { 
        // 递归把 孩子 整回来
        const childNode = diff(prevChildren[index], nextChildren[index], onUpdate, context);
        
		reconciledChildren.push(childNode);
	}

	if (isPrev) {
        // 旧树
		prevNode.children = reconciledChildren;

		if (shouldUpdate) {
            // 触发更新-旧树
			componentDidUpdate(prevNode);
		}
	} else {
        // 新树
        nextNode.children = reconciledChildren;
         // 触发更新
		componentDidMount(nextNode);
    }
    // 反正 componentDidMount 是一定要触发的

	return isPrev ? prevNode : nextNode;
};
```

#### 3.1 `mount`

> 1. 将 vnode.component 实例化 vnode.instance

> 2. 触发 载入前钩子-componentWillMount

> 3. _render()触发->生成孩子虚拟树

``` js
const mount = (vnode, context, onUpdate) => {
	const props = getProps(·);
	checkPropTypes(vnode.component, props);

	if (isClassComponent(vnode.component)) {
// 1. 继承Component的类
		vnode.createInstance(props);
		vnode.instance._onUpdate = onUpdate;

		// context 是 一个全局上下文
		vnode.instance.context = Object.assign(context, vnode.instance.getChildContext());
		// 2.
		vnode.instance.componentWillMount();
		// 3.
		vnode.children = vnode.instance._render();
	} else {
// 关于 孩子 换行之类
// div/span/br
		vnode.children = vnode.component(props, context);
	}
};
```

#### 3.2 `unmount`

> 1. 触发 卸载前-componentWillUnmount

> 2. 清空 实例

> 3. 全部孩子去掉

> 4. 去掉指定组件

``` js
const unmount = vnode => {
	if (isClassComponent(vnode.component)) {
		componentWillUnmount(vnode);
		vnode.instance = null;
	}

	vnode.children.forEach(childVNode => {
		diff(childVNode, null);
	});

// 去掉全局变量
	if (isClassComponent(vnode.component) && vnode.ref) {
		vnode.ref(null);
	}
};
```

---

🧠 在`diff函数`我们需要什么? `return isPrev ? prevNode : nextNode;`

那么 我们需要一个改造好的 虚拟树。

这么一堆的判断, 生命周期钩子触发, Instance化, 但我们返回的也就是-VNode类, 

只是 `VNode.instance` 不再是 `null`, 

`VNode.children` 也 `_render()触发` 回来 变成 `VNode`树


</details>


---

## 4. renderToString

> 把 `Vnode树` 变成 `String`

<details>

`/ink/lib/render-to-string.js`

``` js
'use strict';

const StringComponent = require('./string-component');

const renderToString = vnode => {
	if (!vnode) {
// 1.
		return '';
	}

	if (typeof vnode === 'string') {
// 2.
		return vnode; // children 返回
	}

	if (Array.isArray(vnode)) { // 数组VNode, h(Component, props, children)
	// children 都是数组, 所有通过这样 join('') 组合
		return vnode
			.map(renderToString)
			.join('');
	}

	if (vnode.instance instanceof StringComponent) {

// 真正停止的递归终止条件, 
// 1. ''
// 2. String
// 3. 是 继承 StringComponent Text

// 也就是 Text, 作者定义的字符串组件
// 可以看到本项目例子中 Text, 

		// 我们上面 3.1 mount 就说了 vnode.children 是 vnode.instance._render() 赋予的
		// 4.1 StringComponent children 如何是字符串
		const children = renderToString(vnode.children);
		// 4.2  染色字符串
		return vnode.instance.renderString(children); 
	}

	// 递归-孩子
	return renderToString(vnode.children);
};

module.exports = renderToString;

```


#### 4.1 `StringComponent children 如何是字符串`

`ink/lib/string-component.js`

``` js
class StringComponent extends Component {
	render() {
		return this.props.children;
	}
}
```

> 可以看到[本项目例子中 Text](#本次例子)

> h(Text, {blue: true}, '~/Projects/ink ') =>

> '~/Projects/ink ' == props.children =>

> vnode.instance._render() 触发 StringComponent.render() =>

> vnode.children = this.props.children == '~/Projects/ink '

---

#### 4.2 `染色字符串`

`ink/lib/components/text.js`

代码 22-36

``` js
// 没错, 为什么可以设置颜色, 就是 chalk 输出颜色库
// h(Text, {blue: true}, '~/Projects/ink ')

class Text extends StringComponent {
	renderString(children) {
		Object.keys(this.props).forEach(method => {
			if (this.props[method]) {
				if (methods.includes(method)) {
					children = chalk[method].apply(chalk, arrify(this.props[method]))(children);
				} else if (typeof chalk[method] === 'function') {
					children = chalk[method](children);
				}
			}
		});

		return children; // 染好的字符串
	}
}
```

</details>


---

## 5. setState

> Component 组件中改变 状态的函数, 却也是触发, 重覆盖的函数

怎么做到, 点击⬇️

<details>

让我们回到 Component 类的定义

`ink/lib/component.js`

代码 15-27

``` js
	setState(nextState, callback) {
		if (typeof nextState === 'function') {
			nextState = nextState(this.state, this.props);
		}

		this._pendingState = Object.assign({}, this._pendingState || this.state, nextState);

		if (typeof callback === 'function') {
			this._stateUpdateCallbacks.push(callback);
		}

		this._enqueueUpdate(); // 《=== 触发
	}
```

代码 70-72

``` js
	_enqueueUpdate() {
		enqueueUpdate(this._onUpdate); 
		// 把 组件的 _onUpdate 放进去

//🤔 _onUpdate 怎么来的, 其实本质就是
// `ink/index.js` 61-67

	// const onUpdate = () => {
	// 	if (isUnmounted) {
	// 		return;
	// 	}

	// 	update();
	// };

// 在做比较-diff-函数时就一直在变量中流传
// diff(prevTree, nextTree, onUpdate, context);
	}

```

代码 3

``` js
const {enqueueUpdate} = require('./render-queue');

```

`ink/lib/render-queue.js`

``` js
'use strict';

const options = require('./options');

const queue = [];

const rerender = () => {
	while (queue.length > 0) {
		const callback = queue.pop();
		callback();
	}
};

exports.rerender = rerender;

exports.enqueueUpdate = callback => {
	queue.push(callback);
	options.deferRendering(rerender); 
	// <=== 函数在下一次事件轮询调用
	// 也就是 onUpdate 的调用
};

```

`ink/lib/options.js`

``` js
'use strict';

module.exports = {
	deferRendering: process.nextTick //<===
};

```

- [`process.nextTick`](http://nodejs.cn/api/process.html#process_process_nexttick_callback_args)

> 一旦当前事件轮询队列的任务全部完成，在next tick队列中的所有callbacks会被依次调用。

</details>
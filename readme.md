# ink

「 对CLI做出-React。 使用组件构建和测试您的CLI输出. 」

[![explain](http://llever.com/explain.svg)](https://github.com/chinanf-boy/Source-Explain)
    
Explanation

> "version": "0.4.1"

[github source](https://github.com/vadimdemedes/ink)

~~[english](./README.en.md)~~

---

本次例子-[官方](https://github.com/vadimdemedes/ink#usage)

<details>

```jsx
const {h, render, Component, Text} = require('ink');

// h -> 用来替换 JSX
class Counter extends Component {
	constructor() {
		super();

		this.state = {
			i: 0
		};
	}

	render() {
		return (
			<Text green>
				{this.state.i} tests passed
			</Text>
		);
	}

	componentDidMount() {
		this.timer = setInterval(() => {
			this.setState({
				i: this.state.i + 1
			});
		}, 100);
	}

	componentWillUnmount() {
		clearInterval(this.timer);
	}
}

// <Counter /> -> h(Counter)
render(<Counter/>);
```

<p align="center">
  <img src="media/demo.svg" width="600">
</p>

</details>


---

本目录

---

可以看到例子运行, `render(<Counter/>);` 首先触发

## 1. render

`ink/index.js`

代码 30-119

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

> 用来建立新的-视图树`Tree`

- [renderToString](#4-rendertostring)

> 把 视图树`Tree` 变成 `String`， 给 `log-update` 使用

> `log(renderToString(nextTree));`

</details>

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

---

## 3. diff

`ink/lib/diff.js`

我们前面已经说明了, 在如何接管-终端输出和输入, 已达到覆盖的动画等效果

> 对比-视图树, 返回新或者没有变的树🌲

<details>

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
		unmount(prevNode); // 移除旧
		mount(nextNode, context, onUpdate); // 加载新的
		isPrev = false;
    } 
    
// 如果到这里 isPrev 还是 == true, 说明判断还没有结束
//⏰ 下面这部分, 就开始细分, 更新, 和 组件事件钩子的运行

    // 对比 Component props
    const shouldUpdate = isPrev && shouldComponentUpdate(prevNode, getProps(nextNode), getNextState(prevNode));

    // 为什么是 prevNode???

    // 旧树的孩子
	const prevChildren = isPrev ? [].slice.call(prevNode.children) : [];

	if (isPrev && !isEqualShallow(getProps(prevNode), getProps(nextNode))) {
		componentWillReceiveProps(prevNode, getProps(nextNode));
	}

	if (shouldUpdate) { // 应该重新渲染
		rerender(prevNode, context);
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
            // 应该更新
			componentDidUpdate(prevNode);
		}
	} else {
        // 新树
		nextNode.children = reconciledChildren;
		componentDidMount(nextNode);
	}

	return isPrev ? prevNode : nextNode;
};
```


</details>


## 4. renderToString

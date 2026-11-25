---
title: 前端开发知识库：Vue 2 / Vue 3 / React 三框架入门与实践
topic: 教学知识库
sub_topic: 前端开发（Vue 2 / Vue 3 / React）
doc_type: kb-doc
level: 入门
chapters: 12
chars: 134989
build_date: '2026-11-25'
---

# 前端开发知识库：Vue 2 / Vue 3 / React 三框架入门与实践

从 HTML/CSS/JS/TS 前置知识讲到工程化，再分别深入 Vue 2、Vue 3、React 的核心 API 与原理，覆盖响应式、状态管理、性能优化与测试部署，最后做三框架对比与迁移、双版本项目实战。

> 适合人群：零基础或刚入门的开发者、需要带新人的工程师；每章独立成节，可按需跳读，也适合整篇灌入 RAG 系统做检索。

## 目录

1. 学习地图与前置知识
2. 工程化与工具链
3. Vue 2 核心
4. Vue 2 响应式原理与生态
5. Vue 3 核心
6. Vue 3 响应式原理与状态管理
7. React 基础
8. React Hooks
9. React 进阶与状态管理
10. 三框架对比与迁移
11. 性能优化与调试
12. 测试、CI 与项目实战

## 1. 学习地图与前置知识

### 1.1 前端全景：六层结构决定排错方向

前端（Front-end，运行在浏览器中、直接面向用户的那部分程序）常被误解成「把设计稿翻译成页面」。真实链路是从网络响应、文档解析、样式计算、脚本执行一直到像素绘制，Vue 2、Vue 3、React 只是其中「框架层」的一个可选件。把链路拆成六层，看到报错先判断它属于哪一层，比直接搜索错误信息快得多。

| 层次 | 职责 | 代表技术 | 故障时的典型表现 |
| --- | --- | --- | --- |
| 运行环境层 | 解析文档、执行脚本、绘制像素 | 浏览器内核、DOM/BOM | 白屏、滚动卡顿、跨浏览器不一致 |
| 结构层 | 描述内容语义与层级 | HTML | 表单无法键盘操作、屏幕阅读器读不出内容 |
| 表现层 | 布局、配色、响应式 | CSS | 元素重叠、横向滚动条、移动端错位 |
| 行为层 | 交互逻辑、请求数据、改状态 | JavaScript、TypeScript | 点击无反应、`xxx is not a function` |
| 框架层 | 组件化、声明式渲染、状态管理 | Vue 2 / Vue 3 / React | 数据变了视图不变、组件重复渲染 |
| 工程层 | 依赖管理、构建、检查、部署 | npm、Vite、Webpack | 本地正常线上白屏、产物体积异常 |

层与层之间是依赖关系而不是并列关系：结构层没有语义，可访问性无从谈起；表现层不懂盒模型，布局技巧就只能靠背属性；行为层不理解异步，数据请求就会时序错乱。三套框架的差别集中在「怎么描述 UI、怎么追踪数据变化」，对前四层的要求完全一致，这也是前端岗位面试里框架题之外必有 JavaScript 与 CSS 题的原因。

同一个故障在不同层有不同的排查入口，认清这一点能省掉大量猜测。页面错位时，先在开发者工具里看盒模型示意与计算后的样式，而不是去翻组件代码；数据没渲染时，先确认接口返回与状态值，再怀疑框架的响应式；本地正常线上白屏时，先看构建产物与资源路径，而不是重装依赖。把「现象 → 层」的映射记住，等于拿到一张排错地图。

可以用四条标准自检地基：能否不查资料写出带校验的表单页；能否说出同时含 `setTimeout` 与 `Promise` 的代码的输出顺序；能否只用 Flex 或 Grid 做一个自适应卡片列表；遇到「样式不生效」能否在五分钟内定位到是优先级、层叠顺序，还是选择器压根没匹配。四条都做到，框架层的学习就只剩语法与心智模型。

顺序上，HTML 与 CSS 可以并行推进，JavaScript 的语言核心必须打牢再进框架，TypeScript 放到能写组件之后再补，收益最高——此时类型报错能立刻对应到具体的视图行为。每个阶段都应该有可交付的产物：结构阶段是一个能被键盘完整操作的表单页，表现阶段是一个在 320px 到 1440px 宽度下都不崩的列表页，行为阶段是一个带加载态与错误态的数据列表，框架阶段才是把它们组件化。没有产物的学习很容易停在「看懂了但写不出」。

框架也不是必需品。页面只有静态展示、交互仅是锚点跳转时，原生 HTML 加少量脚本更快、更小、更容易被搜索引擎收录；只有当同一份状态需要在多处视图间同步、或者组件数量多到手工更新 DOM 会漏改时，框架的收益才盖过它的学习与构建成本。

### 1.2 HTML 语义化：结构本身就是信息

语义化（Semantic HTML，用含义与内容匹配的标签描述结构，而不是一律用 `div` 承载）解决的是「机器和辅助技术能否读懂这份文档」。它的读者有三类：搜索引擎的抓取程序、屏幕阅读器（Screen Reader，把页面内容朗读给视障用户的软件），以及三个月后接手这份代码的人。

| 场景 | 随手写法 | 语义写法 | 换来的能力 |
| --- | --- | --- | --- |
| 顶部导航 | `<div class="nav">` | `<header>` 加 `<nav>` | 辅助技术可直接跳到导航区 |
| 正文主体 | `<div class="content">` | `<main>` | 一个页面只允许一个，可快速定位 |
| 独立文章 | `<div class="post">` | `<article>` | 可被聚合器识别为独立内容单元 |
| 点击提交 | `<div onclick>` | `<button type="button">` | 可聚焦、回车与空格触发、保留 disabled |
| 跳转链接 | `<div onclick="location.href">` | `<a href>` | 支持新标签打开、可被爬虫跟踪 |

判定 `article` 与 `section` 有一条可操作的标准：把这段内容单独拎出来发布到别处（RSS、搜索结果、聚合页），语义仍然成立就用 `article`；它必须依附于上下文才能理解，只是文档中的一个主题分组，就用 `section`。两者的共同要求是必须有标题，没有标题的 `section` 等于一个名字更好听的 `div`。

几条能被工具验证的硬规则：表单控件必须有可关联的标签；标题按 `h1` 到 `h6` 逐级使用，不要为了字号跳级；图片写 `alt`，纯装饰图写 `alt=""` 让辅助技术忽略；可点击处用 `button`，跳转用 `a href`，`div` 加 `onclick` 键盘 Tab 到不了、回车也不触发。

```html
<!-- 标签、必填、错误提示都能被辅助技术感知 -->
<form novalidate>
  <label for="email">邮箱</label>
  <input id="email" type="email" required aria-describedby="email-error" />
  <p id="email-error" role="alert">邮箱格式不正确</p>
  <button type="submit">订阅</button>
</form>
```

验证语义是否到位，有三个不装插件的动作：禁用 CSS 后刷新，若页面变成一堆分不清主次的文本，说明结构只靠样式撑着；只用 Tab 键走完全部交互，走不到或没有焦点样式的地方就是缺陷；打开浏览器的无障碍（Accessibility）面板或跑一次 Lighthouse，看有无「缺少标签」「对比度不足」这类告警。

### 1.3 CSS 盒模型、Flex、Grid 与选择器优先级

盒模型（Box Model，元素占据空间时由内到外的四层结构：内容区 content、内边距 padding、边框 border、外边距 margin）是 CSS 布局的第一性原理，两种计算方式决定了「设置的宽度到底包不包括内边距与边框」：

| 声明 | `box-sizing: content-box`（默认） | `box-sizing: border-box` |
| --- | --- | --- |
| `width: 100px; padding: 10px; border: 2px` | 内容区 100px，占位 124px | 内容区 76px，占位 100px |
| 实际影响 | 加内边距就撑破父容器 | 宽度即最终占位，排版可预测 |

```css
/* 项目样式的第一段：统一盒模型，写宽度时不必心算 padding 与 border */
*, *::before, *::after { box-sizing: border-box; }
```

另一个高频错位来源是外边距折叠（Margin Collapse，相邻块级元素的上下外边距合并为一个并取较大值），表现为「分别设了 20px，看起来只有 20px」。父元素加 `display: flow-root` 形成块级格式化上下文（BFC，Block Formatting Context，一块独立计算内部布局的区域）即可隔断折叠。

Flex（Flexible Box，一维弹性布局，把子元素沿一条主轴排列并按剩余空间伸缩）解决「一行内怎么分布、怎么对齐」：主轴对齐用 `justify-content`，交叉轴对齐用 `align-items`，间距用 `gap`。注意 `flex: 1` 等价于 `flex: 1 1 0%`，而 `flex: auto` 是 `1 1 auto`，后者的基准宽度取内容宽度，所以内容长度不一时两列宽度会对不齐。

Grid（Grid Layout，二维网格布局，同时定义行与列）解决页面骨架与卡片墙，一行声明就能得到自适应列数：

```css
/* 每列最小 240px，能塞几列塞几列，剩余空间平均分配 */
.cards { display: grid; grid-template-columns: repeat(auto-fill, minmax(240px, 1fr)); gap: 16px; }
```

`auto-fill` 保留空列，`auto-fit` 折叠空列：容器很宽而卡片很少时，前者卡片仍保持 240px，后者被拉伸填满整行。选哪个取决于卡片该不该占满。

两者的分工可以一句话概括：只在一条方向上分配空间用 Flex，需要同时约束行与列、或者希望子元素自动对齐成栅格用 Grid。实战里常混用——页面骨架用 Grid 划分区域，某个区域内部的导航条用 Flex 排列。

选择器优先级（Specificity，多条规则命中同一元素时决定哪条生效的权重）用三元组 `(ID 数, 类/属性/伪类数, 元素/伪元素数)` 表示，从左到右逐位比较且不进位，类再多也压不过一个 ID；行内 `style` 高于任意选择器，`!important` 再高一层。

```css
/* 三条规则同时命中同一个链接，权重最高者生效 */
.nav a        { color: #666; }  /* (0,1,1) */
.nav li .link { color: #06c; }  /* (0,2,1)：多一个类，胜出 */
nav ul li a   { color: #f60; }  /* (0,0,4)：元素再多也压不过类 */
```

定位「样式不生效」的标准动作是打开开发者工具的 Styles 面板：被划掉的声明就是被更高优先级覆盖了，鼠标悬停能看到最终生效的规则来自哪个文件哪一行；若都没有被划掉，说明选择器没匹配上元素，去 Elements 面板核对 `class` 拼写与嵌套层级。

优先级只是层叠（Cascade，浏览器决定同一元素最终样式的仲裁流程）的一环，完整顺序是：先看来源与重要性（带 `!important` 的用户样式、作者样式、浏览器默认样式），再看优先级，最后看出现顺序，后出现的胜出。因此同一份样式在不同引入顺序下表现可能不同，构建产物的样式文件顺序漂移也会造成「线上样式和本地不一样」。继承（Inheritance，子元素自动获得父元素上可继承的属性值）是另一条独立的规则，`color`、`font-size`、`line-height` 会继承，`margin`、`border`、`display` 不会，误以为宽度也会继承是最常见的理解偏差。

### 1.4 JavaScript 作用域、闭包、原型链与 this

作用域（Scope，一个变量能被访问到的代码范围）分全局、函数、块级三种。`var` 只有函数作用域并存在变量提升（Hoisting，声明被提到作用域顶部、赋值留在原地），`let` 与 `const` 有块级作用域和暂时性死区（Temporal Dead Zone，声明语句之前访问会直接抛 `ReferenceError`）。

```js
// var：循环里只有一份 i，三个回调读到的都是循环结束后的值
for (var i = 0; i < 3; i++) setTimeout(() => console.log('var', i)); // 3 3 3
// let：每次迭代创建新的绑定，回调各自持有当次的值
for (let i = 0; i < 3; i++) setTimeout(() => console.log('let', i)); // 0 1 2
```

闭包（Closure，函数与它定义时所处词法环境的组合，使函数在环境退出后仍能访问其中的变量）是最常用的抽象手段，私有状态、函数工厂、防抖节流都靠它：

```js
function createCounter() {
  let count = 0;                          // 被闭包持有，外部无法直接改写
  return { inc: () => ++count, value: () => count };
}
const c = createCounter();
c.inc(); c.inc();
console.log(c.value());                   // 2
```

闭包不会自动释放它引用的变量。大数组、DOM 节点或定时器留在闭包里，外部又长期持有这个函数，内存就降不下来，排查办法是在开发者工具的 Memory 面板做两次快照对比，看保留树（Retainers）里是谁在持有这个函数。

原型链（Prototype Chain，每个对象通过内部 `[[Prototype]]` 指向另一个对象，属性查找沿这条链向上直到 `null` 的机制）解释了「属性从哪来」：

```js
function Person(name) { this.name = name; }
Person.prototype.say = function () { return `I am ${this.name}`; };
const p = new Person('Ada');
p.say();                                        // "I am Ada"，say 在原型上
Object.getPrototypeOf(p) === Person.prototype;  // true
p.hasOwnProperty('say');                        // false，是继承来的
```

`class` 与 `extends` 只是这套机制的语法糖。工程上要避免在构造函数里逐个实例定义方法（每个实例一份，无法复用），也要避免把整个 `prototype` 直接赋成对象字面量，那会丢掉 `constructor` 指向。给内置对象挂方法（例如往 `Array.prototype` 上加 `sum`）看起来方便，但会污染所有数组、与将来的标准方法冲突，也会让 `for...in` 遍历出意外结果，属于明确的反模式。

`this`（函数被调用时确定的执行上下文引用，指向谁由调用方式决定，而不是由定义位置决定）是排查成本最高的一类问题：

| 绑定方式 | 调用写法 | `this` 指向 |
| --- | --- | --- |
| 默认绑定 | `fn()` | 严格模式为 `undefined`，非严格为 `window` |
| 隐式绑定 | `obj.fn()` | `obj` |
| 显式绑定 | `fn.call(obj)` / `apply` / `bind` | 传入的那个对象 |
| new 绑定 | `new Fn()` | 新建的实例 |
| 箭头函数 | `() => {}` | 定义时外层作用域的 `this`，`call` 也改不了 |

```js
const timer = {
  count: 0,
  start() {
    // 错误：回调中的 this 是 window 或 undefined，count 被写到了全局
    setTimeout(function () { this.count++; }, 100);
    // 正确：箭头函数继承 start 的 this
    setTimeout(() => { this.count++; }, 100);
  },
};
```

`this` 出问题的表现很集中：报错说某个方法不存在，或者状态改到了看不见的地方。排查只要两步——先看调用点，`obj.fn()` 与 `const f = obj.fn; f()` 的结果完全不同；再看函数是不是回调，回调被框架或定时器调用时，隐式绑定已经丢失。框架里大量回调场景（事件处理、请求回调、数组方法）都会重置 `this`，所以现代代码更倾向用箭头函数而不是 `bind`。

### 1.5 事件循环与异步：代码到底按什么顺序执行

事件循环（Event Loop，JavaScript 单线程执行完当前任务后，不断从任务队列里取出回调执行的调度机制）是理解所有「顺序问题」的总钥匙。单线程是为了避免多个线程同时修改同一份 DOM，耗时操作交给浏览器或 Node 的其他线程，完成后把回调排进队列。

队列分两级：宏任务（Macrotask）含整段脚本、`setTimeout`、I/O 回调、用户交互回调；微任务（Microtask）含 `Promise.then/catch/finally`、`queueMicrotask`、`MutationObserver`。规则只有一条：每执行完一个宏任务，就把微任务队列清空，然后才可能渲染页面、再取下一个宏任务。

```js
console.log('1');
setTimeout(() => console.log('2'), 0);
Promise.resolve().then(() => console.log('3'));
queueMicrotask(() => console.log('4'));
console.log('5');
// 输出 1 5 3 4 2：1、5 是当前宏任务的同步代码，3、4 是微任务，2 属于下一个宏任务
```

Promise（Promise，表示异步操作最终完成或失败的对象，状态从 pending 单向转为 fulfilled 或 rejected 且不可逆）把回调嵌套改写成链式调用；`async/await` 是它的语法糖，函数在 `await` 处暂停、恢复后继续执行剩余代码。`try/catch` 只能捕获被 `await` 的那个拒绝，忘了 `await` 就变成未处理的 Promise 拒绝。`catch` 写在链尾能兜住前面所有环节，但会让「哪一步失败」的信息变模糊，排错时更愿意逐段加日志或在中间插入一次 `catch` 后重新抛出。

```js
// 串行：总耗时约为两个请求之和，后者依赖前者结果时必须这样写
const user = await fetchUser(id);
const orders = await fetchOrders(user.id);
// 并行：互不依赖就并发发出，总耗时约为最慢的那一个
const [me, cfg] = await Promise.all([fetchUser(id), fetchConfig()]);
```

真实排错按现象对号入座：在 `forEach` 里写 `await`，回调返回的 Promise 被丢弃，循环不等请求完成就往下走，改用 `for...of` 或 `Promise.all(list.map(fn))`；`setTimeout(fn, 0)` 实际延迟几十毫秒，因为定时器只保证「不早于」，主线程忙时只能排队；控制台出现未处理的 Promise 拒绝却定位不到来源，给 `window.addEventListener('unhandledrejection')` 与 `'error'` 加统一上报才能拿到真实堆栈。

微任务还有一个容易忽视的副作用：它在同一个宏任务内被反复追加时，队列永远清不空，页面就永远等不到渲染机会。用 `Promise` 递归模拟轮询很容易写出这种死循环式的「假死」，界面卡住但 CPU 不满。需要按帧推进的任务应该用 `requestAnimationFrame`，需要让出主线程时用 `setTimeout` 分段，而不是在微任务里递归。

### 1.6 TypeScript 入门：把类型错误拦在编译期

TypeScript（微软推出的 JavaScript 超集，在编译期检查类型，再输出可在浏览器与 Node 运行的普通 JavaScript）的价值不在「写起来更严谨」这种感受，而在三点可验证的收益：重命名字段时编译器会把所有漏改处列出来；编辑器能补全对象属性与函数签名；接口定义本身就是团队共享的数据契约。Vue 3 与 React 的官方脚手架如今默认就是 TypeScript 模板。

入门只需掌握六件事：基础类型注解、对象接口、联合与字面量类型、可选与只读修饰、泛型（Generics，把类型当作参数传入，让函数在多种类型下复用且保留类型信息）、`unknown` 与类型收窄。

```ts
interface User {
  id: number;
  name: string;
  email?: string;                    // 可选属性
  readonly role: 'admin' | 'member'; // 只读 + 字面量联合，写错值直接报错
}

function first<T>(list: T[]): T | undefined { return list[0]; }

function greet(input: unknown): string {
  if (typeof input === 'string') return `hello ${input}`; // 收窄后才能用字符串方法
  return 'hello stranger';
}

const admin: User = { id: 1, name: 'Ada', role: 'admin' };
const picked = first([admin]);        // 类型是 User | undefined，不能直接取属性
const name = picked?.name ?? '匿名';  // 可选链加空值兜底，结果收窄为 string
// admin.role = 'member';             // 编译错误：readonly
```

`interface` 适合描述对象契约且支持声明合并，`type` 能表达联合、交叉等更复杂的组合；团队常见约定是「对象结构用 `interface`，其余用 `type`」。

`tsconfig.json` 里几个字段直接决定检查强度与产物形态：

| 字段 | 作用 | 建议值 |
| --- | --- | --- |
| `target` | 输出代码的语法版本 | `ES2020` |
| `module` / `moduleResolution` | 模块格式与解析策略 | `ESNext` / `Bundler` |
| `strict` | 一次打开全部严格检查，含 `strictNullChecks` | 新项目必开 `true` |
| `jsx` | JSX 的编译方式 | React 用 `react-jsx`，Vue 用 `preserve` |
| `noUncheckedIndexedAccess` | 索引访问返回类型自动带 `undefined` | `true`，提前发现越界 |
| `skipLibCheck` | 跳过第三方声明文件检查以加快速度 | `true` |

把类型检查接进流水线只需一条命令：`tsc --noEmit` 只做检查不产出文件，适合放在提交钩子或 CI 里，真正的打包交给构建工具。第三方库没有声明时先装 `@types/包名`，仍没有就写 `declare module '包名'` 兜底，而不是把整个文件降级成 `any`。

类型系统也有边界，不必追求把每个值都写成精确类型。处理外部输入（接口响应、`localStorage`、URL 参数）时，用 `unknown` 接住再做校验，比假设一个接口类型更安全，因为编译期无法证明服务端真的返回了那个结构；只有在边界处校验一次，内部才可以放心用具体类型。反过来，为了一个只用一次的临时对象写三层泛型，收益为负——类型是用来降低维护成本的，不是用来展示技巧的。

### 1.7 常见坑

1. 用 `div` 加 `onclick` 冒充按钮：鼠标能点，键盘 Tab 走不到、回车不触发，还丢掉了 `disabled` 与表单语义。凡可点击处优先 `button`，跳转优先 `a href`。
2. 忘记统一 `box-sizing`：给元素加内边距后宽度溢出，移动端出现横向滚动条。项目入口第一段 CSS 就写 `*, *::before, *::after { box-sizing: border-box; }`。
3. 把对象方法当回调直接传：`setTimeout(obj.method, 100)` 会丢掉 `obj`，`this` 变成 `undefined`。用箭头函数包一层或 `bind(obj)`。
4. 在 `forEach` 里 `await`：回调返回的 Promise 被丢弃，循环不等待，后续逻辑读到空数据。要么 `for...of` 串行，要么 `Promise.all(list.map(...))` 并发。

## 2. 工程化与工具链

### 2.1 工程化解决的是四类具体麻烦

一个页面加一个 `<script src="app.js">`，双击 index.html 就能跑。文件超过五十个、参与的人超过三个，这套做法立刻失效，暴露出的都是同一类麻烦：

依赖：用了 lodash，怎么记录版本、怎么让同事装到同一份代码、半年后换台机器还能装回来。模块：三千行塞一个文件改不动，拆开又要按顺序引脚本，顺序错了就报 undefined。语言：TypeScript、Sass、JSX 浏览器都不认，必须先编译。交付：本地正常、线上白屏，问题一定出在产物或路径上。

工程化（Build Engineering，用工具链把源码稳定地转换成可部署产物的过程）就是给这四件事建立可重复的流程。它的验收标准很朴素：换一个人、换一台机器、换一个时间，得到同一份产物。

「本地能跑」从来不是交付标准。工程化真正买到的是一份契约：任何人执行同样两条命令得到同样的站点，出错时能定位到哪一环偏了，而不是靠某个人记得那台机器的特殊配置。

脚手架（Scaffold，用一条命令生成项目骨架的工具）把这些配置提前写好。Vue 官方是 `create-vue`，React 侧 CRA（Create React App）已停止维护，主流做法是 Vite 模板。要不要上工程化，判断标准是「代码要不要给别人看、要不要上线、要不要多人改」，三者占一条就需要；一个下午写完就丢的验证页面直接引 CDN 更快。

这四类麻烦的解法在工具上各自独立，却必须在同一个项目里配成一套约定。缺哪一环，它都会在交付那天以白屏或装不上的形式还回来。

### 2.2 包管理器与锁文件

包管理器（Package Manager，负责解析、下载、记录、还原第三方依赖的工具）主流有三款：npm、yarn、pnpm。命令几乎可以互换，差别在安装策略与磁盘占用。

版本号先要读懂。`^1.4.2` 允许升到 1.x 内最新（不含 2.0.0），`~1.4.2` 只允许 1.4.x，`1.4.2` 是精确版本。`^` 让团队自动吃到补丁，也让「昨天还好好的」成为可能，真正兜住这件事的是锁文件。

锁文件（Lockfile，记录每个依赖的精确版本、下载地址与完整依赖树的文件）：npm 用 `package-lock.json`，yarn 用 `yarn.lock`，pnpm 用 `pnpm-lock.yaml`。它必须提交到 Git，冲突时不要手改，删掉重装生成即可。CI 里不要用 `npm install`，用 `npm ci`（只按锁文件安装、先清空 node_modules、不改锁文件）：锁文件与 package.json 不一致时直接失败，而不是悄悄改掉。

依赖要写在正确的字段里，写错会让库的使用者装不上东西：

| 字段 | 含义 | 典型内容 |
| --- | --- | --- |
| `dependencies` | 运行时真正 import 的包 | vue、axios |
| `devDependencies` | 只在构建与测试时用 | vite、eslint、vitest |
| `peerDependencies` | 声明由宿主项目提供、自己不打包 | 组件库声明 vue |
| `optionalDependencies` | 装不上也能继续 | fsevents |

| 维度 | npm | yarn | pnpm |
| --- | --- | --- | --- |
| node_modules 布局 | 扁平提升 | 扁平提升 | 符号链接 + 内容寻址存储 |
| 幽灵依赖 | 普遍存在 | 普遍存在 | 基本杜绝 |
| 磁盘占用（多项目） | 每项目一份 | 每项目一份 | 全局一份，硬链接复用 |
| CI 安装命令 | `npm ci` | `yarn install --frozen-lockfile` | `pnpm install --frozen-lockfile` |
| monorepo 支持 | workspaces | workspaces | workspaces，过滤能力强 |
| 常见问题 | 锁文件频繁冲突 | v1 与 berry 差异大 | 部分工具对软链不友好 |

幽灵依赖（Phantom Dependency，代码里 import 了没写进 package.json 的包，只因为它被别人的依赖顺带装到了顶层）是扁平布局的副作用。本地能跑、单独部署就报模块找不到，多半是它。pnpm 的严格结构会直接把这个隐患暴露在开发期。

多个包共用一个仓库时用工作区（Workspace，在一个仓库里管理多个包、共享依赖与工具链的机制）：根目录 package.json 声明包路径，子包之间用 `workspace:*` 互相引用，`pnpm -r build` 按拓扑顺序批量构建。单包项目不必上，两个以上包、且改动经常同时发生时才划算。

升级要有纪律。小版本可以随手升，主版本单独开分支，先跑通构建与测试再合；把 `pnpm outdated` 放进每月例行事项，比攒一年再一次性升级省事得多。

```ini
# .npmrc —— 镜像与安装行为的项目级约定，随仓库提交
registry=https://registry.npmmirror.com/   # 内网或国内加速源，避免 npmjs 超时
strict-peer-dependencies=true               # peer 版本冲突直接失败，不让问题带进产物
```

```bash
# 安装并记录到 dependencies（运行时依赖）
pnpm add vue
# 装到 devDependencies（仅构建期需要，如 vite、eslint）
pnpm add -D vite
# 校验锁文件与 package.json 是否一致，CI 前置检查
pnpm install --frozen-lockfile
```

遇到 `ERESOLVE unable to resolve dependency tree`，不要去加 `--legacy-peer-deps` 掩盖。先跑 `npm ls react` 看清楚谁装了几份、谁要求了什么范围，再升级冲突的那一方，最后删掉 node_modules 与锁文件重装一次确认。安装慢的原因通常在网络与源，而不是机器性能，先把 registry 配好再排查其他。

### 2.3 模块化：ESM 与 CJS

模块化（Module System，把代码拆成显式导入导出的文件单元的机制）现在有两套标准并存。ESM（ECMAScript Modules，语言内置的 `import`/`export`，静态可分析）是新代码的默认选择；CJS（CommonJS，Node.js 早期方案，用 `require` 同步加载、`module.exports` 导出）仍大量存在于老库和配置文件里。

| 维度 | ESM | CJS |
| --- | --- | --- |
| 语法 | `import` / `export` | `require` / `module.exports` |
| 解析时机 | 静态，编译期确定依赖 | 运行时同步求值 |
| Tree-shaking | 支持，未用导出可被删除 | 基本不支持 |
| 顶层 await | 支持 | 不支持 |
| 浏览器 | `<script type="module">` 原生可用 | 需打包器转换 |
| 文件扩展名 | `.mjs`，或 `type: module` 下的 `.js` | `.cjs`，或默认 `.js` |

package.json 里的 `"type": "module"` 决定 `.js` 被当作哪种模块，这是 `require is not defined` 报错的头号原因。发布库时用条件导出（Conditional Exports，按 `import`/`require` 分别指向不同产物）同时兼容两种消费者：

```json
{
  "name": "my-utils",
  "type": "module",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.mjs",
      "require": "./dist/index.cjs"
    }
  }
}
```

两种写法的对照（JavaScript）：ESM 写 `import { sum } from './math.js'`，导出 `export function sum(a, b) { return a + b }`；CJS 写 `const { sum } = require('./math')`，导出 `module.exports = { sum }`。混用是单向的：ESM 可以 import CJS（把 `module.exports` 视为默认导出），CJS 里 require ESM 在旧版 Node 会抛 `ERR_REQUIRE_ESM`。同时打进两份产物还可能出现双包危险（Dual Package Hazard，ESM 与 CJS 各持一份实例，导致单例与计数器失效），解法是让其中一份只做包装、状态只存在一处。

tree-shaking（摇树，把没有被引用的导出从产物里删掉的优化）能生效的前提是模块无副作用。库作者要在 package.json 里写 `"sideEffects": false`，或者列出真有副作用的文件（如 `["*.css"]`）；漏写这一项，打包器为求安全会整块保留，import 一个函数却带进几十 KB 的组件代码。

循环引用在 CJS 里表现为拿到半成品 exports，读到 undefined；ESM 的活绑定（Live Binding，导入的是变量的引用而非拷贝）能缓解，但正确做法仍是拆公共模块。另外，动态 `import()` 返回 Promise，构建器会据此切出独立 chunk，这是路由懒加载的实现基础：

```ts
// 只有访问到该路由时才下载对应代码块，首屏不必背全部页面的体积
const routes = [{ path: '/report', component: () => import('./views/Report.vue') }]
```

浏览器原生 ESM 要求服务器给出正确 MIME（`.js` 必须是 `text/javascript`），用 `file://` 直接打开会因跨域被拦，所以开发期必须起 dev server。Vite 另提供 `import.meta.glob`，用于一次性注册某个目录下的全部模块（路由表、图标集、Markdown 文件），省掉手写十几行 import。

### 2.4 Vite 与 Webpack：定位差异

Webpack（模块打包器，从入口递归分析依赖，把 JS、CSS、图片都打成 bundle 再交给开发服务器）自 2014 年起是事实标准，loader 与 plugin 生态极全。Vite（开发期用浏览器原生 ESM 按需编译、生产期交给 Rollup 打包的构建工具）2020 年出现，解决的是「项目一大，冷启动和热更新就慢」。

差的根源是启动方式：Webpack 开发时先把整个依赖图打包完，才响应第一次请求，耗时随代码量线性增长；Vite 只起一个 HTTP 服务，浏览器请求哪个模块就即时编译哪个，第三方依赖用 esbuild 预构建成少量 ESM 文件，冷启动通常在几百毫秒级。

热更新（HMR，Hot Module Replacement，改文件后只替换变化的模块并保留页面状态）的机制也不同。Webpack 依赖 HMR runtime 沿依赖图往上找 accept 边界，找不到就整页刷新；Vite 基于 ESM 的 import 关系精确失效单个模块。Vue、React 的官方插件会给组件自动加自接受边界，所以改模板时表单里已填的内容不会丢——这是判断 HMR 是否真的生效的最快办法。

Vite 有两个开发期细节值得先知道。依赖预构建的结果缓存在 `node_modules/.vite`，改过 lockfile 或升级依赖后若控制台报 `504 (Outdated Optimize Dep)`，删掉这个目录重启即可。本地需要 HTTPS（对接要求安全上下文的接口、调试手机）时，配 `server.https` 的证书，或用 `vite --host` 起局域网地址让手机直接访问。

loader 与 plugin 的分工值得记住：loader 把非 JS 文件翻译成模块，plugin 在构建生命周期钩子上做整体处理（压缩、注入变量、拆包）。Vite 用一套插件 API 统一了这两件事，多数 Rollup 插件可以直接用。

生产构建仍然要打包。浏览器能直接跑 ESM，但一个中型项目拆出几百个模块就是几百次请求，HTTP/2 也扛不住这种瀑布；打包、压缩、加哈希才是线上形态。Vite 默认目标为现代浏览器，要兼容旧环境需装 `@vitejs/plugin-legacy`，代价是多出一套体积可观的兼容包。

拆包（Code Splitting，把产物切成多个文件按需加载）是两套工具共有的能力，配法不同：Webpack 用 `optimization.splitChunks` 定义缓存组，Vite 用 `build.rollupOptions.output.manualChunks` 指定哪些依赖合并进同一个 chunk。默认策略通常够用——业务代码随路由懒加载切分，第三方依赖单独成块，这样改一行业务代码不会让整个 vendor 的哈希失效。

同一件事在两套工具里的配置对照：

| 需求 | Vite | Webpack 5 |
| --- | --- | --- |
| 别名 `@` | `resolve.alias` | `resolve.alias` |
| 开发代理 | `server.proxy` | `devServer.proxy` |
| 环境变量注入 | 自动读 `.env`，用 `import.meta.env` | `dotenv` + `DefinePlugin` |
| 产物目录 | `build.outDir` | `output.path` |
| 文件名哈希 | 默认带哈希 | `[name].[contenthash:8].js` |
| CSS 处理 | 内置，直接 `import './a.css'` | 需 `css-loader` + `style-loader` |

```ts
// vite.config.ts（Vite 5+）
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import { fileURLToPath, URL } from 'node:url'

export default defineConfig({
  plugins: [vue()],
  // 别名要和 tsconfig.json 的 paths 一起改，否则编辑器红波浪线、构建却能过
  resolve: { alias: { '@': fileURLToPath(new URL('./src', import.meta.url)) } },
  server: {
    port: 5173,
    // 开发期把 /api 转发到后端，绕开同源限制，业务代码里仍写相对路径
    proxy: { '/api': { target: 'http://127.0.0.1:8080', changeOrigin: true } },
  },
  build: { outDir: 'dist', sourcemap: false, chunkSizeWarningLimit: 800 },
})
```

选型不复杂：新项目用 Vite；需要 Module Federation 微前端、复杂 loader 链，或必须兼容 IE11（要 Babel 加持的 Webpack 4/5），继续用 Webpack。已有项目能跑就别为了「快」重写配置，收益通常抵不过迁移风险。

### 2.5 目录结构与路径别名

目录组织有两种思路：按类型分（`components/`、`views/`、`utils/`）和按业务域分（`features/order/`）。中小项目按类型足够；业务域清晰、多人并行开发时按域分，能减少跨目录的改动冲突。

```text
src/
├── main.ts            # 入口：创建应用、注册路由与全局插件
├── App.vue
├── router/index.ts    # 路由表
├── stores/            # Pinia / Redux 状态
├── api/               # 请求封装，统一加拦截器与错误处理
├── components/        # 跨页面复用组件
├── views/             # 路由级页面
├── composables/       # 组合式函数（Vue）或自定义 Hook（React）
└── assets/            # 会被构建处理的静态资源
```

`public/` 与 `src/assets/` 的区别容易踩：`public/` 下的文件原样拷贝到产物根目录，不参与哈希，引用必须写绝对路径 `/logo.png`；`assets/` 下的文件参与构建，小图会被内联成 base64，引用走 import 或相对路径。

命名的统一比目录形状更重要。组件文件用大驼峰并与组件名一致（`TodoList.vue` 里就是 `TodoList`），组合式函数与自定义 Hook 用 `use` 前缀，类型集中放在 `types.ts`。目录层级控制在三层以内，超过三层往往说明模块边界没划清。也尽量别让 `index.ts` 背十几个导出，它在搜索里没有辨识度。

### 2.6 ESLint 与 Prettier：分工而不是叠加

ESLint（静态检查器，找出可能出错的写法，如未定义变量、错误的 hook 依赖，部分问题可自动修复）管正确性；Prettier（格式化器，只按规则重排版，不做对错判断）管排版。两者都管格式必然互相打架：ESLint 要求单引号，Prettier 改成双引号，保存一次改两遍。

正确分工是 Prettier 独占格式，ESLint 只留逻辑规则，并用 `eslint-config-prettier` 关掉所有格式类规则。ESLint 9 起使用扁平配置：

```js
// eslint.config.js —— 扁平配置，数组顺序即应用顺序
import js from '@eslint/js'
import vue from 'eslint-plugin-vue'
import prettier from 'eslint-config-prettier' // 必须放最后，覆盖掉冲突的格式规则

export default [
  js.configs.recommended,
  ...vue.configs['flat/recommended'],
  { rules: { 'no-console': ['warn', { allow: ['error', 'warn'] }] } },
  prettier,
]
```

配套的 npm scripts 与提交前钩子，顺序是先格式化再 lint，因为格式化会移动行号，反过来会让错误位置对不上：

```json
{
  "scripts": {
    "lint": "eslint . --max-warnings=0",
    "format": "prettier --write .",
    "check": "prettier --check . && eslint ."
  },
  "lint-staged": { "*.{js,ts,vue,tsx}": ["prettier --write", "eslint --fix"] }
}
```

`--max-warnings=0` 很关键：默认警告不导致失败，团队过两周就会对满屏黄色视而不见。落地顺序建议三步：先只开 Prettier + `lint-staged`（一次性格式化提交单独一个 commit，别混进业务改动），再打开推荐规则集修掉存量，最后才按需追加严格规则。提交信息建议用约定式提交（Conventional Commits，以 `feat:`、`fix:` 前缀标明改动类型），变更日志与版本号能据此自动生成，`git log` 也变成可读的发布记录。

### 2.7 环境变量与多环境

Vite 的规则只有一条：只有 `VITE_` 开头的变量会注入客户端代码，通过 `import.meta.env` 读取；内置的还有 `MODE`、`BASE_URL`、`DEV`、`PROD`。同名变量按 `.env` → `.env.local` → `.env.[mode]` → `.env.[mode].local` 的顺序覆盖，后者优先。`.local` 结尾的文件进 `.gitignore`，用来放个人调试配置。

| 命令 | mode | 实际生效的文件 |
| --- | --- | --- |
| `vite` | development | `.env`、`.env.local`、`.env.development` |
| `vite build` | production | `.env`、`.env.local`、`.env.production` |
| `vite build --mode staging` | staging | `.env`、`.env.local`、`.env.staging` |

```bash
# .env.development —— 开发环境走本地代理
VITE_API_BASE=/api
VITE_APP_TITLE=待办清单
```

```ts
// src/api/request.ts
const base = import.meta.env.VITE_API_BASE ?? '/api' // 变量缺失时兜底，不把 undefined 拼进 URL

export async function getTodos() {
  const res = await fetch(`${base}/todos`)
  if (!res.ok) throw new Error(`HTTP ${res.status}`) // fetch 对 4xx/5xx 不抛错，必须自己判断
  return res.json()
}
```

注意这些值是构建期替换，不是运行时读取。`if (import.meta.env.PROD)` 在产物里早已被裁掉一个分支，同一份 dist 无法靠改环境变量切换到测试环境，要换就重新构建。CI 里也不要提交带真实地址的 `.env.production`，把变量放在流水线的环境变量配置里，由构建命令读入，权限与轮换都归平台管。

生产环境不要放任何密钥。前端环境变量会被替换成字符串字面量写进产物，等于公开发布，`VITE_API_TOKEN` 这类命名只会制造虚假的安全感，真实的密钥操作应该放在服务端网关或后端代持。Webpack 那一侧等价的做法是 `dotenv` 读文件加 `new webpack.DefinePlugin({ 'process.env.APP_ENV': JSON.stringify(...) })`。

另一种常见误用是把环境差异写进代码分支，比如按域名判断「这是测试环境就走测试接口」。几周之后就没人说得清线上到底走了哪条路。差异只应存在于 `.env` 文件与构建命令里，代码对所有环境保持同一份逻辑。

### 2.8 构建产物与部署

`vite build` 产出 `dist/index.html` 与 `dist/assets/` 下带内容哈希的文件。哈希的作用是缓存分级：文件名随内容变，assets 可以放心设一年缓存；只有 index.html 保持不缓存，用户刷新就能拿到新版本引用的新文件名。这个组合让发版不需要清 CDN。

部署到子路径（如 `/todo/`）时必须设 `base: '/todo/'`，否则 index.html 里引用的还是 `/assets/xxx.js`，线上直接白屏。单页应用使用 history 路由时，服务端还要把未知路径回退到 index.html，否则刷新 `/todo/42` 得到 404：

```nginx
server {
  listen 80;
  root /usr/share/nginx/html;

  location /assets/ {
    expires 1y;
    add_header Cache-Control "public, immutable";   # 文件名含哈希，可长缓存
  }

  location / {
    try_files $uri $uri/ /index.html;               # history 路由回退，刷新不 404
  }
}
```

部署形态常见三种：静态托管（Vercel、Netlify、对象存储加 CDN，最省事，改 `base` 与回退规则即可）、Nginx 容器（配置如上，适合内网与多环境）、Node 服务端渲染（要跑一个常驻进程，缓存策略完全不同）。

CI 里构建与安装必须锁死：

```bash
# .github/workflows/deploy.yml 中的关键片段（Shell）
pnpm install --frozen-lockfile   # 锁文件被改过会直接失败，而不是装出另一棵树
pnpm build                       # 产物输出到 dist/
```

本地验证产物用 `pnpm preview` 起一个静态服务器，它的行为比直接双击 index.html 更接近线上，能提前暴露路径与回退问题。

发布前看构建日志里每块的体积，超过 500 kB 的块用路由级动态 `import()` 拆出去；需要定位依赖占比就接 `rollup-plugin-visualizer`，它会把每个包在产物里占多大画成一张图，先砍最大的那块。

sourcemap 的取舍也要定下来。生产环境建议 `sourcemap: 'hidden'`：产物里不写 `//# sourceMappingURL`，把生成的 .map 文件单独上传给错误监控平台，线上访客拿不到源码，你又能还原压缩后的堆栈。直接开 `true` 等于公开源码，关掉 `false` 则线上报错只剩一行无意义的列号。

### 2.9 一次典型排错：白屏与依赖装不上

工程问题按固定顺序排查，比凭感觉点来点去快得多。以「刚拉下代码、装完依赖、`pnpm dev` 起来后页面白屏」为例：

1. 只读控制台的第一条错误。后面的报错多半是连锁反应，从最后一条读起会被带偏。
2. 区分是开发期问题还是产物问题：`pnpm dev` 正常、`pnpm build && pnpm preview` 异常，方向就在构建配置、`base` 或环境变量替换。
3. 看网络面板的 MIME：模块请求 404 多为路径别名或文件名大小写；`.js` 请求返回 `text/html`，说明请求打到了回退规则上，静态资源路径已经错了。
4. 清缓存复测：删 `node_modules/.vite`，浏览器硬刷新，排除缓存造成的假象。

| 现象 | 常见原因 | 处理动作 |
| --- | --- | --- |
| 启动即报 `Cannot find module` | 依赖没装或写错字段 | 重装并检查 dependencies |
| 白屏，assets 全部 404 | `base` 与部署路径不符 | 设 `base: '/子路径/'` 后重新构建 |
| 刷新子路由 404 | 服务端缺少回退规则 | 配 `try_files ... /index.html` |
| 变量拿到 undefined | 名字没加 `VITE_` 前缀 | 改名并重启 dev server |

### 2.10 常见坑

- **CI 用 `npm install` 而不是 `npm ci`**：锁文件被悄悄改写，装出的依赖树和本地不一致，本地永远复现不了线上问题。同理，别用 `--legacy-peer-deps` 掩盖 peer 冲突，那只是把错误留给下一台机器。
- **本地正常、线上白屏**：先开 DevTools 的 Network，看 assets 是否 404。子路径部署忘了配 `base`、history 路由缺少 `try_files` 回退，占这类问题的大半；Windows 不区分大小写、Linux 区分，`import Header from './header.vue'` 也会本地能跑、线上 404。
- **`require is not defined` 或 `Cannot use import statement outside a module`**：package.json 的 `type` 与文件扩展名不匹配，把文件改成 `.cjs`/`.mjs` 或显式声明模块类型。
- **密码写进 `.env` 并提交**：立刻轮换密钥，补 `.gitignore`，再用 `git log -p -- .env` 确认历史里是否已泄露。

## 3. Vue 2 核心

Vue 2 是渐进式 JavaScript 框架（Progressive Framework，指既能用一个 script 标签增强现有页面、也能配合构建工具写完整单页应用的框架）。它的核心思路是「声明式渲染 + 响应式数据」：开发者只描述数据长什么样、界面应该长什么样，框架负责在数据变化时把 DOM 更新到正确状态。Vue 2 的组件默认用选项式 API 书写，模板语法接近原生 HTML，所以从传统模板渲染或 jQuery 迁移过来的团队上手成本最低。一个 Vue 2 组件的完整知识面包括五块：模板怎么写、选项怎么分、生命周期怎么走、组件之间怎么通信、表单怎么双向绑定。

### 3.1 模板语法与指令

模板（Template，写在 HTML 里的声明式视图描述）会被 Vue 2 编译成渲染函数（Render Function），渲染函数产出虚拟 DOM（Virtual DOM，用普通 JS 对象描述真实 DOM 结构的树），最后由框架把差异打到真实 DOM 上。模板里只能写三类内容：插值、指令、表达式。

插值 `{{ }}` 只能输出文本，不能用在 HTML 属性上。属性绑定用 v-bind（简写 `:`），事件绑定用 v-on（简写 `@`）。指令（Directive，以 `v-` 开头、在模板中声明式操作 DOM 或数据的特殊属性）是模板表达力的来源，常见的有 v-text、v-html、v-bind、v-on、v-if、v-show、v-for、v-model、v-once。其中 v-html 直接插入 HTML 字符串，只对可信内容使用，否则存在跨站脚本注入风险；v-once 让节点只在首次渲染时求值，适合页脚、静态说明这类永不变化的内容。事件修饰符（Modifier，用点号接在指令后、修饰其行为的后缀）里最常用的五个是 `.stop` 阻止冒泡、`.prevent` 阻止默认行为、`.self` 只在该元素自身触发时响应、`.once` 只响应一次、`.native` 监听组件根元素的原生事件（Vue 2 专有，Vue 3 已移除）。

模板表达式有硬边界：`{{ }}` 和指令的值里只能写单个 JavaScript 表达式，不能写 if、for、var 这类语句，也不能直接访问全局对象（模板作用域被限制在组件实例上，要访问 window 上的东西得先挂到 data 或 computed）。运算超过两三个运算符的表达式就该挪到 computed 或 methods 里，模板只负责展示。

class 与 style 绑定支持对象语法和数组语法，且与静态 class 合并而不是覆盖：`:class="{ active: isActive, 'text-danger': hasError }"` 按布尔值决定是否加上某个类名，`:class="[baseClass, isActive ? 'active' : '']"` 按条件拼接，`:style="{ fontSize: size + 'px' }"` 直接写内联样式对象。这套写法比在 JS 里拼字符串更不容易漏空格、也更容易被静态检查发现。

条件渲染有两种写法，取舍取决于「切换频率」而不是「代码长短」：

| 维度 | v-if | v-show |
| --- | --- | --- |
| 条件为假时的 DOM | 节点不存在，组件被销毁 | 节点存在，仅 `display: none` |
| 切换开销 | 高，涉及创建销毁子组件与事件 | 低，只改一个样式属性 |
| 初始渲染开销 | 低，条件为假时不渲染 | 高，无论如何先渲染再隐藏 |
| 适用场景 | 条件很少变化、需要懒加载或重置状态 | 显隐频繁、需要保留内部状态 |
| 能否用在 template 上 | 可以，不产生额外节点 | 不可以，必须有真实元素 |

列表渲染有两个硬规则。第一，v-for 必须配 `:key`，key 要唯一且稳定。第二，不要与 v-if 写在同一个元素上：Vue 2 中 v-for 优先级高于 v-if，会先把整个列表循环一遍再逐个判断，既浪费性能，又让 v-if 访问不到 v-for 的临时变量。正确做法是先用计算属性过滤，再渲染过滤后的结果。

```html
<template>
  <div class="todo">
    <!-- visibleTodos 是已过滤好的计算属性，渲染阶段不再做判断 -->
    <ul v-if="visibleTodos.length">
      <li v-for="item in visibleTodos" :key="item.id">
        <span :class="{ done: item.done }">{{ item.title }}</span>
        <!-- .stop 阻止冒泡，.prevent 阻止默认行为，可链式叠加 -->
        <button @click.stop.prevent="remove(item.id)">删除</button>
      </li>
    </ul>
    <p v-else>暂无待办</p>
    <!-- 动态参数：属性名与事件名由变量决定，2.6+ 支持 -->
    <a :[linkKey]="url" @[eventName]="onJump">跳转</a>
  </div>
</template>
```

新手最常见的三个模板错误是：把 `{{ }}` 写进 HTML 属性里（属性必须用 v-bind 绑定，插值在属性上不生效）、在 v-bind 的值里漏掉引号导致变量名被当成字符串、以及多层嵌套的 v-for 只给外层写了 key 却漏掉内层。排查方法是打开浏览器控制台的 Vue Devtools，看组件树里渲染出的节点数量与预期是否一致。

### 3.2 选项式 API：data、computed、watch、methods

选项式 API（Options API，把组件逻辑按 data、methods、computed、watch 等固定选项分类组织的写法）用「数据放在哪一类」来切分代码。四个核心选项的职责是：data 声明响应式数据源，methods 放由交互触发的动作，computed 放从别处派生出来的值，watch 放数据变化后要执行的副作用。

data 必须是函数，不能是对象。组件是可复用的，若写成 `data: { count: 0 }`，同一组件的所有实例会共享同一个对象，改一个实例的 count 全都跟着变；`data() { return { count: 0 } }` 每次创建实例都返回新对象，才是正确写法。根实例可以写对象，组件不行。

计算属性（Computed Property，由其他响应式数据派生、结果会被缓存的值）与普通方法的区别在缓存：只要依赖没变，多次访问 computed 都直接返回上次结果；methods 只要在模板里被调用，每次重渲染都重新执行。因此凡是在模板里展示的派生值都该用 computed。computed 默认只有 getter，需要被赋值时写成 `{ get, set }` 对象形式，这样 v-model 也能绑到它上面。注意 computed 里不能有异步操作和副作用，它是「算出一个值」，不是「做一件事」。

侦听器（Watcher，监听某个数据变化并执行副作用的选项）适合处理「数据变了要做事」：发请求、写 localStorage、联动非响应式对象。它支持 `immediate`（首次绑定立即执行一次）和 `deep`（深度监听对象内部变化），也支持用字符串路径 `'form.address.city'` 精确监听嵌套字段。要注意 deep 监听时新旧值指向同一个对象引用，直接比较 `newVal === oldVal` 永远相等，需要自己记录快照。

| 需求 | 选择 | 原因 |
| --- | --- | --- |
| 模板中展示派生值 | computed | 有缓存，依赖不变不重算 |
| 派生值还要被 v-model 写回 | computed 的 get/set | 读与写收敛到一处 |
| 数据变化后做副作用（请求、存储） | watch | computed 里不应有副作用 |
| 用户交互触发的动作 | methods | 由事件驱动，不需要缓存 |
| 初始化后立刻执行一次 | watch + `immediate: true` | 比在 created 里手写调用更不易漏 |

多字段联动只用一个 watcher，不要一个字段一个：比如「省份变了重新查城市、城市变了重新查门店」，正确做法是同时监听这两个字段的同一个 handler，参数里识别变化来源，否则一次联动会发出两次重复请求。另外 watch 的 handler 里做异步请求时，要防止旧请求覆盖新结果：给每次请求打一个自增序号，回调里比对当前序号，不一致就丢弃。

```js
export default {
  data() {
    // 组件必须用函数：每次返回新对象，实例之间互不干扰
    return { keyword: '', todos: [], loading: false };
  },
  computed: {
    // 依赖 keyword 与 todos，两者都不变时直接复用缓存
    visibleTodos() {
      const k = this.keyword.trim().toLowerCase();
      return k ? this.todos.filter(t => t.title.toLowerCase().includes(k)) : this.todos;
    },
    keywordModel: {
      get() { return this.keyword; },
      set(v) { this.keyword = v; }   // 供 v-model 写回
    }
  },
  watch: {
    keyword: {
      immediate: true,               // 首次绑定立即执行一次
      handler(val) { this.loading = true; this.fetchTodos(val); }
    },
    'form.address.city'(val) {       // 路径写法监听深层字段
      this.cityChanged = true;
    }
  },
  methods: {
    async fetchTodos(kw) { /* 请求细节省略 */ },
    remove(id) { this.todos = this.todos.filter(t => t.id !== id); }
  }
};
```

写这四个选项时有一个共同禁忌：不要在 methods、computed、watch 的 handler 里使用箭头函数。Vue 2 靠 `this` 把选项绑定到组件实例上，箭头函数不绑定自己的 `this`，会捕获外层的 `this`（通常是 undefined 或 window），结果就是 `this.todos` 报 undefined。同理，`setTimeout(() => {}, 100)` 里可以用箭头函数（因为它要的正是外层 `this`），而选项本身必须用普通函数。

排查「computed 明明依赖变了却不更新」的顺序是：先确认被依赖的数据本身是响应式的（对象新增的键、按索引赋值都需要 `$set`，这种赋值在 Vue Devtools 里看起来值变了，但没有触发依赖通知），再确认 computed 里没有对依赖做提前返回的短路（比如依赖在 if 分支里才被读，条件不变就永远不收集该依赖），最后检查是否在 computed 里直接改了别的数据导致循环。

### 3.3 生命周期钩子

生命周期钩子（Lifecycle Hook，组件在创建、挂载、更新、销毁各阶段自动调用的函数）决定「代码该写在哪一步」。八个钩子的时机与典型用途如下：

| 钩子 | 触发时机 | 典型用途 |
| --- | --- | --- |
| beforeCreate | 实例初始化完成，数据观测尚未开始 | 极少使用，插件注入场景 |
| created | 数据、方法、计算属性就绪，尚未挂载 | 发初始请求、初始化非 DOM 状态 |
| beforeMount | 模板编译完成，尚未插入页面 | 极少使用 |
| mounted | 已挂载，`$el` 可用 | 操作 DOM、初始化图表等第三方库 |
| beforeUpdate | 数据已变，DOM 尚未重渲染 | 更新前读取旧 DOM 状态 |
| updated | DOM 已重渲染完成 | 依赖新 DOM 的测量 |
| beforeDestroy | 销毁前，实例与数据仍可用 | 清定时器、解绑事件、取消请求 |
| destroyed | 实例已销毁 | 日志与资源回收的收尾 |

created 与 mounted 的分工是最常被问的问题：两者都能发接口请求，created 里发能更早拿到数据、少一次无谓的渲染；但只有在 mounted 之后才能拿到真实 DOM、初始化依赖尺寸的第三方库（图表、地图、富文本编辑器）。如果组件需要在服务端渲染环境里运行，created 里不能访问 window、document，这类代码必须挪到 mounted。

父子组件的挂载顺序是固定的：父 beforeCreate → 父 created → 父 beforeMount → 子 beforeCreate → 子 created → 子 beforeMount → 子 mounted → 父 mounted。也就是说子组件的 mounted 先于父组件的 mounted 执行，父组件在 mounted 里一定能拿到已经挂载好的子组件。销毁顺序相反：父 beforeDestroy → 子 beforeDestroy → 子 destroyed → 父 destroyed。这个顺序解释了一个常见现象：父组件在 mounted 里调用子组件的 `$refs` 方法一定安全，在 created 里调用则一定拿到 undefined。

`$nextTick` 是配套的必需工具。Vue 2 的 DOM 更新是异步批量的：改完数据立刻读 DOM 拿到的还是旧值，框架会把这些更新排进微任务队列，等本轮同步代码跑完再统一打补丁。`this.$nextTick(cb)` 或 `await this.$nextTick()` 的回调在 DOM 更新之后执行，是「改数据 → 立刻操作新 DOM」的唯一正确时机。比如新增一条待办后要让列表滚到底部，就必须写在 nextTick 里，否则拿到的还是加进去之前的高度。

```js
export default {
  data() { return { list: [], timer: null, req: null }; },
  mounted() {
    this.timer = setInterval(this.poll, 5000);            // 定时器必须在销毁前清掉
    this.$nextTick(() => { this.$refs.list.scrollTop = 0; }); // 此时 DOM 已更新
  },
  beforeDestroy() {
    clearInterval(this.timer);
    this.timer = null;
    if (this.req && this.req.cancel) this.req.cancel();    // 取消未完成请求，避免回调操作已销毁实例
    window.removeEventListener('resize', this.onResize);   // 全局监听同理
  }
};
```

生命周期还有两个与报错相关的钩子：errorCaptured 捕获后代组件抛出的错误，返回值设为 false 可以阻止错误继续向上冒泡；根实例上配置 `Vue.config.errorHandler = (err, vm, info) => {}` 可以把所有组件错误统一上报到监控平台，info 里会带上出错阶段（如 render、created hook）。生产环境里把这两处接起来，比让用户看到白屏再靠复现去猜要高效得多。

### 3.4 组件化：props、$emit、插槽、动态组件

组件（Component，把模板、逻辑、样式封装成可复用单元的最小粒度）是 Vue 应用的基本单位。父子通信有三条主干：父传子用 props，子传父用自定义事件 `$emit`，内容分发用插槽。

props 遵循单向数据流：父组件的数据变化会流向子组件，反过来不行。子组件直接改 prop 会收到警告，而且父组件下次重渲染时这次修改会被覆盖，形成难以复现的 bug。需要在子组件内做本地编辑时，用 data 拷贝初始值或用 computed 派生，改动后通过事件把结果抛回父组件。props 可以声明类型、是否必填、默认值，类型不匹配只警告、不阻断运行；对象类型的默认值必须写成工厂函数，否则所有实例共享同一个默认对象。

| 通信方案 | 方向 | 适用场景 | 关键约束 |
| --- | --- | --- | --- |
| props + `$emit` | 父到子、子到父 | 明确的父子关系 | prop 只读，事件名用 kebab-case |
| `.sync` | 父子双向 | 单个字段回写 | 事件名必须是 `update:字段名` |
| v-model | 父子双向 | 表单类控件 | Vue 2 中每个组件只能有一个 |
| 具名/作用域插槽 | 父决定子内部渲染 | 布局与列表内容定制 | 数据由子组件向上抛出 |
| `$attrs` / `$listeners` | 跨层级透传 | 包装第三方组件 | 2.4+，配合 `v-bind="$attrs"` |
| provide / inject | 祖先到后代 | 主题、国际化、全局配置 | 默认非响应式 |
| 事件总线 | 任意组件（含兄弟） | 少量跨组件事件 | 必须手动 `$off`，否则泄漏 |

事件总线（Event Bus，用一个空的 Vue 实例充当全局事件中转站）在中小项目里解决兄弟组件通信最省事：在任意组件里 `$on` 订阅、`$emit` 触发。它的代价是订阅关系不受组件树约束，一旦忘记在 beforeDestroy 里 `$off`，销毁的组件仍会被回调，既报错又泄漏，所以事件名要足够具体、在全局唯一，并且订阅与解绑必须成对出现。

```js
// bus.js：一个只做事件中转的空实例
import Vue from 'vue';
export default new Vue();

// 订阅方：组件内 this.$bus.$on / beforeDestroy 里 $off
beforeDestroy() { this.$bus.$off('todo:refresh', this.load); }
```

父组件要直接调用子组件的方法时，用 ref 与 `$refs`：在子组件标签上写 `ref="list"`，之后 `this.$refs.list.reload()` 即可。这是一种绕过单向数据流的应急手段，只适合命令式操作（打开弹窗、重新加载、聚焦输入框），业务数据仍应走 props 与事件；过度使用会让组件耦合变紧、也没法在测试里单独渲染。给组件写 name 还有另一个用途：递归渲染自身，树形菜单组件在自己的模板里再引用自己，没有 name 就无法递归。

插槽（Slot，父组件向子组件指定位置插入内容的机制）解决「结构复用但内容不同」的问题。默认插槽是 `<slot></slot>`，具名插槽是 `<slot name="head">`，作用域插槽则把子组件内部的数据通过 `:row="row"` 抛给父组件，由父组件决定怎么渲染这一行。Vue 2.6 起统一用 `v-slot:name="slotProps"`，等价的旧写法是 `slot-scope`。作用域插槽是封装表格、列表、树这类组件时唯一能做到「逻辑复用、样式自定义」的手段，值得单独练熟。

```html
<!-- 子组件 DataTable.vue：只负责循环，单元格长什么样交给父组件 -->
<template>
  <table>
    <thead><slot name="head" :count="rows.length"></slot></thead>
    <tbody>
      <tr v-for="row in rows" :key="row.id">
        <slot name="row" :row="row">{{ row.name }}</slot>  <!-- 父组件不传时用默认内容 -->
      </tr>
    </tbody>
  </table>
</template>
```

```html
<!-- 父组件使用作用域插槽，解构出子组件抛上来的数据 -->
<data-table :rows="users">
  <template v-slot:head="{ count }">共 {{ count }} 人</template>
  <template v-slot:row="{ row }">
    <td>{{ row.name }}</td>
    <td>{{ row.age }}</td>
  </template>
</data-table>
```

动态组件用 `<component :is="comp">` 按变量切换渲染哪个组件。默认每次切换都会销毁旧组件、重建新组件，组件内部的表单内容、滚动位置全部丢失；套一层 keep-alive 可以缓存组件实例，`include`/`exclude` 按组件 name 限定缓存范围，`max` 限制缓存个数。被缓存的组件会多出 activated 与 deactivated 两个钩子，需要「每次进入都刷新数据」的逻辑就写在 activated 里。注意 keep-alive 只对直接子组件生效，包裹多层 div 会失效。

```html
<keep-alive :include="['TodoList', 'TodoStats']" :max="5">
  <component :is="currentTab" @change="onTabChange" />
</keep-alive>
```

### 3.5 表单与自定义 v-model

v-model 只是语法糖：在 input、textarea 上等价于 `:value` 加 `@input`；在 checkbox、radio 上是 `:checked` 加 `@change`；在 select 上是 `:value` 加 `@change`。所以「双向绑定」的底层仍是单向数据流加事件回写，理解这一点才能排查「输入框打不出字」「绑定的值不更新」这类问题。多选框绑定数组时，v-model 指向的数组会自动增删选中值；`<select multiple>` 同理，绑定的也必须是数组而不是字符串。

表单修饰符有三个：`.lazy` 把同步时机从 input 事件改成 change 事件（失焦或回车才更新），`.number` 用 parseFloat 转换、转换失败时保留原字符串，`.trim` 去掉首尾空格。Vue 2 不会把自定义修饰符自动传给子组件，自定义组件里要用修饰符得自己透传，实际项目里通常改为用 props 传配置。

自定义 v-model 的约定只有两条：组件声明 `value` 这个 prop，并在需要更新时 `$emit('input', 新值)`。满足这两条，父组件就能直接用 v-model 绑定。一个组件在 Vue 2 里只能有一个 v-model，因为默认 prop 名固定是 value、事件名固定是 input；需要多个双向绑定时用 `.sync`，每个字段一个 `update:字段名` 事件。若控件的天然属性不是 value（例如用自己的 checked 语义），可以用 model 选项改掉默认名。

```html
<!-- 子组件 MyInput.vue：接受 value，通过 input 事件回写 -->
<template>
  <div class="field">
    <input :value="value" @input="$emit('input', $event.target.value)" />
    <button @click="$emit('input', '')">清空</button>
  </div>
</template>
<script>
export default {
  name: 'MyInput',
  props: { value: { type: String, default: '' } }
};
</script>
```

```js
// 父组件模板里：<my-input v-model="form.name" /> 等价于下面两行
// <my-input :value="form.name" @input="form.name = $event" />

// 多个双向绑定用 .sync：<my-panel :title.sync="form.title" />
// 子组件里对应 this.$emit('update:title', newTitle)

// 改默认 prop 与事件名：父组件写 <my-check v-model="agree">
export default {
  model: { prop: 'checked', event: 'change' },
  props: { checked: { type: Boolean, default: false } }
};
```

### 3.6 常见坑

1. data 写成对象。组件里写 `data: { count: 0 }` 会让同一组件的所有实例共享同一份数据，改一个全变，必须写成 `data() { return { count: 0 } }`。
2. 直接给对象加键或按索引赋值。`this.obj.newKey = 1`、`this.arr[2] = 'x'` 不会触发视图更新，要用 `this.$set(this.obj, 'newKey', 1)` 和 `this.$set(this.arr, 2, 'x')`，删除用 `this.$delete`，数组也可用 splice。排查方法：数据看着变了但页面没动，先在控制台打印再确认赋值方式。
3. 用数组下标当 key。列表插入、删除、排序时下标会整体偏移，导致 DOM 复用错位，典型表现是输入框里的内容串到了下一行；key 必须取业务 id，没有 id 时要在数据进入列表前生成稳定 id。
4. 忘记清理副作用。定时器、window 事件、事件总线订阅、未完成的请求都要在 beforeDestroy 中清理，否则组件销毁后回调仍会执行，控制台报「Cannot read property of null」，同时造成内存泄漏。

## 4. Vue 2 响应式原理与生态

### 4.1 响应式解决的是什么问题

在 Vue 2 里执行 `this.count++`，页面上所有用到 count 的地方都会自动变。这件事由响应式系统（Reactivity System，数据被修改后自动找出所有依赖它的位置并重新执行它们的机制）完成，它由三个角色协作：

- Observer（观察者）：组件初始化时递归遍历 data 里的每个属性，用 `Object.defineProperty` 把普通属性改写成带 getter/setter 的属性。
- Dep（Dependency，依赖收集器）：每个属性配一个，内部是一个订阅者数组，负责「记住谁用过我」和「值变了通知谁」。
- Watcher（订阅者）：渲染函数、computed、watch 都被包装成 Watcher。它求值一次，在求值过程中把自己登记到读到的每个属性的 Dep 里。

一次完整的链路是：组件渲染 → 读到 `this.msg` → 触发 msg 的 getter → `Dep.target` 指向当前渲染 Watcher → 该 Watcher 被收进 msg 的 Dep → 用户写 `this.msg = 'x'` → 触发 setter → `dep.notify()` → 各 Watcher 进入异步调度队列 → 队列 flush 时统一 patch 真实 DOM。

之所以要这么绕，是因为浏览器没有「属性变了通知我」的原生能力。没有响应式框架时，开发者只能自己 `document.querySelector('#count').textContent = count`，状态与视图的对应关系散落在几十处，漏改一处就是 bug。响应式系统把「状态到视图」的同步变成框架职责，代价是框架必须知道每个状态被谁读过，这就是依赖收集存在的理由。

这套机制的学名叫 MVVM（Model-View-ViewModel，用中间层把数据模型与视图连接起来的模式），Vue 实例就是那个 ViewModel。要注意响应式只负责「让 Watcher 重跑」，真正把新旧虚拟节点差异写到浏览器上的是虚拟 DOM 与 patch，两件事必须分开看：响应式决定「什么时候更新」，虚拟 DOM 决定「更新哪一块」。反过来也一样，只懂其中一半，遇到「数据变了但 DOM 没变」和「DOM 重绘了但值不对」就会归错因。

必须记住一个前提：Vue 2 是在初始化时「一次性递归转换」整个 data 对象的，而不是访问某个属性时才惰性处理。这条设计选择是它三个先天限制的总根源。

### 4.2 用 Object.defineProperty 手写最小实现

原语只有一句话：在 getter 里收集依赖，在 setter 里派发更新。下面这段代码可直接在浏览器控制台运行。

```javascript
// mini-reactivity.js：Vue 2 响应式核心的最小骨架
function defineReactive(obj, key, val) {
  const dep = new Dep();                 // 一个属性一个收集器
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get() {
      if (Dep.target) dep.depend();      // 谁在读我，就把谁记下来
      return val;
    },
    set(newVal) {
      if (newVal === val) return;        // 值没变就不通知，省掉一次无谓渲染
      val = newVal;
      dep.notify();                      // 值变了，通知全部订阅者
    }
  });
}

let uid = 0;
class Dep {
  constructor() { this.id = uid++; this.subs = []; }
  depend() {
    const t = Dep.target;
    if (t && !this.subs.includes(t)) this.subs.push(t);  // 去重，一次渲染只记一次
  }
  notify() { this.subs.slice().forEach(w => w.update()); }  // 拷贝后遍历，防回调里改数组
}
Dep.target = null;

class Watcher {
  constructor(getter, cb) {
    this.getter = getter;
    this.cb = cb;
    this.value = this.get();             // 首次求值，顺便完成依赖收集
  }
  get() {
    Dep.target = this;                   // 声明「现在求值的是我」
    const v = this.getter();             // 求值期间读到谁，就依赖谁
    Dep.target = null;                   // 必须还原，否则依赖挂到错误的 Watcher 上
    return v;
  }
  update() { this.value = this.get(); if (this.cb) this.cb(this.value); }
}
```

上面为了好读把 `Dep.target` 写成了单个变量，真实实现用的是 `targetStack` 栈：computed 的 getter 里可能再读另一个 computed，嵌套求值会把当前订阅者覆盖掉，只有入栈出栈才能正确还原。复刻实现时若发现「computed 依赖 computed 时更新错乱」，八成就是漏了这个栈。

把递归入口补上就完整了：Observer 负责把一个对象（含嵌套子对象）整体转成响应式，并在对象上挂一个不可枚举的 `__ob__` 属性，用来反查「这个对象被观察过没有」以及「它的收集器在哪」。

```javascript
class Observer {
  constructor(value) {
    this.value = value;
    this.dep = new Dep();                // 对象/数组级别的 dep，Vue.set 靠它通知
    Object.defineProperty(value, '__ob__', { value: this, enumerable: false });
    if (Array.isArray(value)) { value.__proto__ = arrayMethods; this.observeArray(value); }
    else this.walk(value);               // 深层递归就发生在这里，数据越深启动越慢
  }
  walk(obj) { Object.keys(obj).forEach(k => defineReactive(obj, k, obj[k])); }
  observeArray(arr) { arr.forEach(i => observe(i)); }
}
```

真实源码里 `defineReactive` 的 getter 还会调用子对象收集器的 `childOb.dep.depend()`，这正是 `this.$set(this.form, 'a', 1)` 能生效的关键：set 时给新键补一次 `defineReactive`，再借用 `__ob__.dep` 通知依赖过父对象的渲染 Watcher。手写版本常漏掉这一步，于是 set 之后视图仍然不动。

Vue 2 用 defineProperty 而不是 Proxy 是兼容性约束下的选择，不是设计偏好：Vue 2 要覆盖到 IE9，而 Proxy 是 ES6 引入的语言级拦截，无法用 polyfill 补齐，defineProperty 则是 ES5 就有的 API。代价就是前面说的三件事天然做不到——新增键、删除键、数组下标；Vue 3 换成 Proxy 一次性解决了它们，也是这次重写的最大动机。

数组要单独处理：`Object.defineProperty` 拦不住 `arr[0] = x` 和 `arr.length = 0`，Vue 2 的解法是把数组原型上的 7 个变更方法（push、pop、shift、unshift、splice、sort、reverse）包一层，先调原生逻辑再手动通知。

```javascript
// 数组方法拦截：不改原生 Array.prototype，只继承一个新对象
const arrayMethods = Object.create(Array.prototype);
['push', 'pop', 'shift', 'unshift', 'splice', 'sort', 'reverse'].forEach(method => {
  const original = Array.prototype[method];
  Object.defineProperty(arrayMethods, method, {
    value: function mutator(...args) {
      const result = original.apply(this, args);
      const ob = this.__ob__;            // 挂载在数组上的 Observer 实例
      if (ob) ob.dep.notify();           // setter 拦不到，只能在这里补一刀
      return result;
    },
    enumerable: false, writable: true, configurable: true
  });
});
```

### 4.3 三大缺陷：现象、根因、解法

| 缺陷 | 现象 | 根因 | 正确做法 |
| --- | --- | --- | --- |
| 新增属性 | `this.obj.a = 1` 后视图不变 | 新键没有 getter/setter，不在监听范围 | `this.$set(this.obj, 'a', 1)` 或整体替换 `this.obj = {...this.obj, a: 1}` |
| 删除属性 | `delete this.obj.a` 后视图不变 | delete 不触发 setter | `this.$delete(this.obj, 'a')` |
| 数组下标与长度 | `arr[0] = 1`、`arr.length = 0` 无效 | 元素访问不走 setter，长度赋值无钩子 | `arr.splice(0, 1, 1)`、`arr.splice(0)` |

`Vue.set(target, key, value)` 的内部就是给新键补一次 `defineReactive`，然后再调 `target.__ob__.dep.notify()`。所以它必须拿到响应式对象本身：对冻结对象调用只会得到一条 readonly 警告且不生效，对没被 observe 的普通对象调用则退化成普通赋值、不产生响应式。对数组用 `$set(arr, 3, 'x')` 是可行的，内部会转成 `arr.splice(3, 1, 'x')`，因此与数组拦截方法共用同一条通知路径。

除了这三条，还有一个容易被忽略的代价：因为初始化要递归遍历整个 data，嵌套越深、字段越多，启动就越慢；而且被冻结的对象、Map、Set 这类原生集合完全不在监听范围内，Vue 2 没有对 Map/Set 的响应式支持，`state.map.set(k, v)` 永远不会触发更新，只能用普通对象加 `$set` 兜住。实践中的三条对策是：用 `Object.freeze()` 标记纯展示的大常量（冻结对象会被 observe 跳过，省下一整轮递归）；把不用响应的数据挪出 data，挂到 `this.$options` 或外部模块；大列表分页加载，而不是一次塞进 data 让初始化卡住两三秒。

一个高频真实场景：表格行内编辑。行对象是 `row`，输入框绑 `row.status`；如果为了存临时错误信息写成 `this.rows[index].errors = [...]`，页面不会有任何反应，因为 errors 是一个新键。排查时不要先怀疑 v-model，打开 Vue Devtools 看该字段有没有出现在响应式树里，一眼就能定性。

### 4.4 异步更新队列、三类 Watcher 与 nextTick

Vue 2 的 DOM 更新是异步批量的。同一个事件循环里连续改 10 次数据，Watcher 靠 id 去重（`has[id]` 判断），最终只 patch 一次。队列在 `nextTick` 里 flush，`nextTick` 的降级顺序是 `Promise.then`（微任务）→ `MutationObserver` → `setImmediate` → `setTimeout`。

Watcher 在 Vue 2 里有三种，理解它们的差别就理解了「computed 为什么有缓存」：

| 类型 | 创建者 | 特点 |
| --- | --- | --- |
| 渲染 Watcher | 组件挂载时 | 每个组件一个，求值即执行 `updateComponent`，走 patch |
| computed Watcher | `computed` 定义 | 惰性求值，靠 dirty 标记判断缓存是否失效 |
| 用户 Watcher | `watch` / `this.$watch` | 支持 deep、immediate，回调拿到 newVal 与 oldVal |

computed 的缓存就是一个布尔量：依赖变化时只把 `dirty` 置 true 而不立刻计算，下次读取时发现 dirty 才重新求值。所以 `computed` 里放耗时计算是安全的，而 `methods` 每次重渲染都会重跑一遍。`deep: true` 则会为被监听对象递归创建依赖，在大对象上是明确的性能负担，能监听具体路径就不要开 deep。`this.$watch` 返回一个 unwatch 函数，在非组件场景（比如监听路由或 store）要手动调用它，否则闭包一直活着。

```javascript
// 数据改完立刻读 DOM 拿到的一定是旧值
export default {
  data() { return { list: ['a'] } },
  methods: {
    async add() {
      this.list.push('b');                 // 数据已变，渲染任务已入队
      console.log(this.$el.textContent);   // 'a'：此刻页面还没变
      await this.$nextTick();              // 等队列 flush
      console.log(this.$el.textContent);   // 'ab'：这才是新内容
    }
  }
}
```

依赖这个机制的第三方库（图表、富文本编辑器、地图 SDK）必须在 `$nextTick` 或 `updated` 之后初始化，否则容器尺寸和 DOM 节点都还不对。另一个常被忽略的边界是：`$nextTick` 只保证「本轮渲染队列已 flush」，它不保证图片加载完成、也不保证子组件的异步请求已返回。涉及子组件内部 DOM 的操作，要在子组件 `mounted` 里 emit，或者写 `this.$refs.child.$nextTick(...)`，否则拿到的仍是子组件的初始状态。

### 4.5 vue-router：模式、守卫、传参

vue-router（Vue 官方路由库，把 URL 映射到组件并在切换时管理浏览器历史）在 Vue 2 用 3.x 版本。模式的选择直接决定部署方式：

| 模式 | URL 形态 | 原理 | 部署要求 |
| --- | --- | --- | --- |
| hash | `/#/user/1` | 监听 hashchange | 纯静态托管即可 |
| history | `/user/1` | pushState + popstate | 服务端必须把所有路径回落到 index.html |
| abstract | 无 URL | 内存栈 | SSR、单元测试、非浏览器环境 |

history 模式最常见的线上事故是刷新 404：Nginx 要加 `try_files $uri $uri/ /index.html;`，否则用户直接访问或刷新 `/user/1` 时服务端找不到文件。

```javascript
// router/index.js：路由定义与守卫配置
const router = new VueRouter({
  mode: 'history',
  base: process.env.BASE_URL,
  scrollBehavior: () => ({ x: 0, y: 0 }),          // 切换路由回到顶部
  routes: [
    { path: '/user/:id', name: 'user', props: true, // 把 params 解耦成 props，组件更好测
      meta: { requiresAuth: true },
      component: () => import(/* webpackChunkName: "user" */ '@/views/User.vue'),
      beforeEnter: (to, from, next) => next() },    // 路由独享守卫
    { path: '*', component: () => import('@/views/404.vue') }  // 通配必须放最后
  ]
});

router.beforeEach((to, from, next) => {
  if (to.meta.requiresAuth && !store.state.user) next({ name: 'login', query: { redirect: to.fullPath } });
  else next();                                      // 任何分支都必须调用 next，否则导航永久挂起
});
```

完整执行顺序：失活组件 `beforeRouteLeave` → 全局 `beforeEach` → 复用组件 `beforeRouteUpdate` → 路由独享 `beforeEnter` → 异步组件解析 → 激活组件 `beforeRouteEnter` → 全局 `beforeResolve` → 导航确认 → 全局 `afterEach` → DOM 更新 → `beforeRouteEnter` 的 next 回调。`beforeRouteEnter` 是唯一拿不到 `this` 的守卫，只能用 `next(vm => {...})` 访问实例。

传参有两条路，选错就是刷新丢数据的经典 bug：`params` 走路径段（`/user/1`），必须先在 path 里声明 `:id`，否则按 name 跳转时刷新即丢失；`query` 走查询串（`/search?q=vue`），刷新安全但会弄脏 URL。更隐蔽的坑是组件复用：`/user/1` 跳到 `/user/2` 是同一个组件，`created`、`mounted` 不会再执行，必须用 `beforeRouteUpdate` 或 `watch: { '$route'(to) {...} }` 重新拉数据。

权限类项目还会用 `router.addRoutes()`（3.x）动态挂载路由：它必须在登录成功后、通配 404 之前插入，接在 404 后面会导致新加的路由永远匹配不到。另外 3.1 之后 `push` 返回 Promise，跳到当前所在路由会 reject 出 `NavigationDuplicated`，未 catch 时控制台出现未处理的 promise 拒绝，需要在调用处补 `.catch(() => {})` 或者显式处理该分支。

还有两块几乎必然遇到：一是嵌套路由，父级模板里放一层 `<router-view>` 承接子路由，子路由的 path 不要带前导斜杠（写 `path: 'profile'` 而不是 `'/profile'`），否则会被当作绝对路径挂到根上，导致父级布局整个消失；二是用 `<keep-alive>` 缓存列表页后，`mounted` 只在首次进入时执行，刷新数据要改到 `activated` 钩子里，否则用户从详情页返回时看到的还是旧列表。

### 4.6 Vuex：单向数据流与模块化

Vuex（Vue 官方集中式状态管理库，用一份全局 store 承载跨组件共享状态）在 Vue 2 配 3.x 版本。它的约束不是风格问题：state 只能由 mutation 同步修改，异步逻辑写在 action，getter 负责派生数据。这样任何一次状态变化都能被 devtools 记录并回溯，出问题时可以「重放」到崩溃前一步。

```javascript
// store/index.js：Vue 2 + Vuex 3 的完整结构
import Vue from 'vue';
import Vuex from 'vuex';
Vue.use(Vuex);

export default new Vuex.Store({
  strict: process.env.NODE_ENV !== 'production',  // 严格模式会深比较快照，生产环境关掉
  state: { user: null, cart: [] },
  getters: {
    cartCount: state => state.cart.length,        // 类似 computed，结果有缓存
    total: (state) => state.cart.reduce((s, i) => s + i.price * i.qty, 0)
  },
  mutations: {
    SET_USER(state, user) { state.user = user; }, // 唯一允许改 state 的地方，必须同步
    ADD_CART(state, item) { state.cart.push(item); }
  },
  actions: {
    async fetchUser({ commit }, id) {             // 异步、可组合、返回 Promise
      const res = await fetch(`/api/users/${id}`);
      commit('SET_USER', await res.json());
    }
  },
  modules: {
    order: {
      namespaced: true,                           // 调用必须带前缀：dispatch('order/add')
      state: () => ({ list: [] }),                // 模块 state 写成函数，便于复用与测试
      mutations: { PUSH(state, o) { state.list.push(o); } },
      actions: { add({ commit }, o) { commit('PUSH', o); } },
      getters: { count: state => state.list.length }
    }
  }
});
```

引入之前先判断是否真的需要全局状态，选错会让简单页面背上不必要的调试成本：

| 通信场景 | 推荐方案 | 注意点 |
| --- | --- | --- |
| 父子之间 | props / $emit | 无全局副作用，边界最清晰 |
| 跨层级少量 | provide / inject | 注入对象本身是响应式的才会更新 |
| 全局共享且要调试 | Vuex | 有 devtools 时间旅行，代价是模板代码多 |
| 一次性事件广播 | 事件总线 `$bus` | 必须在 `beforeDestroy` 里 `$off`，否则重复监听 |

组件里用 `mapState`、`mapGetters`、`mapMutations`、`mapActions` 减少样板代码。表单双向绑定最容易踩线：直接 `v-model` 绑 state 会在严格模式下抛出 `do not mutate vuex store state outside mutation handlers`，正确写法是用带 get/set 的 computed 做中间人。

```javascript
computed: {
  userName: {
    get() { return this.$store.state.user.name; },           // 读：直接取
    set(v) { this.$store.commit('SET_USER', { ...this.$store.state.user, name: v }); }
  }
}
```

落地到工程上要注意两点：模块分散到 `store/modules/order.js` 等文件后，`namespaced` 漏写会让 `dispatch('order/add')` 静默失败；持久化不要手写在组件里，用 `store.subscribe((mutation, state) => localStorage.setItem('cart', JSON.stringify(state.cart)))` 统一做，且只存必要字段，整份 state 序列化会把 token 也写进 localStorage。

### 4.7 mixins：合并规则与替代方案

mixin（混入，把一组选项对象合并进组件，用于复用逻辑）在 Vue 2 是官方复用手段，但它的合并规则是隐式的，出问题极难排查。

| 选项 | 合并规则 |
| --- | --- |
| data | 递归合并，组件与后注册的 mixin 覆盖先注册的 |
| 生命周期钩子 | 全部保留，mixin 的钩子先于组件的执行 |
| methods / computed / watch | 同键时组件覆盖 mixin |
| props / components / directives | 同 data，按合并策略覆盖 |

两个坑：多个 mixin 定义同名方法时静默覆盖，谁生效取决于注册顺序；mixin 里的属性会凭空出现在 `this` 上，一年后没人知道 `this.canEdit` 从哪来、由谁赋值。所以命名要加业务前缀（如 `perm_canEdit`），并且优先用 renderless 组件（只提供逻辑、通过作用域插槽把状态交给使用方渲染的组件）或纯工具函数替代。一个典型的翻车过程：三个列表页共用 `listMixin`，里面封装了分页、筛选和请求。某天其中一个页面要改分页步长，开发者直接在 mixin 里改了默认值，另外两个页面同时在线上变了行为，而评审时 diff 只显示一行改动，影响面完全看不出来。mixin 的危险不在写法，而在「改一处、动几处」这件事在代码里没有痕迹。Vue 3 的 Composition API 正是针对这个「来源不可见」缺陷给出的替代方案。

### 4.8 过滤器：能用与不能用的地方

过滤器（Filter，在模板中把值格式化输出的函数）用管道符 `|` 串联，只能出现在插值 `{{ }}` 和 `v-bind` 表达式里，不能用于 `v-model`、`v-on`、`v-for` 的数据源。

```javascript
// 全局注册 + 局部注册，都必须是纯函数
Vue.filter('currency', (v, symbol = '¥') =>
  `${symbol}${Number(v).toFixed(2).replace(/\B(?=(\d{3})+(?!\d))/g, ',')}`);

export default {
  filters: {
    dateTime(v) {
      if (!v) return '-';                       // 空值兜底，否则 Invalid Date 直接上屏
      const d = new Date(v), p = n => String(n).padStart(2, '0');
      return `${d.getFullYear()}-${p(d.getMonth() + 1)}-${p(d.getDate())}`;
    }
  }
};
```

```html
<!-- 管道可串联，过滤器参数从第二个形参开始 -->
<span>{{ amount | currency('$') }}</span>
<span>{{ createdAt | dateTime }}</span>
```

报错 `Failed to resolve filter: xxx` 只有两个原因：注册名与使用名大小写不一致，或者局部过滤器写在子组件里却被父组件模板使用（模板作用域按组件隔离）。过滤器在 Vue 3 已被移除，迁移时改用 methods 或 computed，所以存量项目不要再往里加业务逻辑。

一个实用判断标准：过滤器只适合「纯展示的格式化」，不适合承担数据转换。把 199 显示成「3分19秒」、把状态码转成中文文案，都是典型用法；但在过滤器里发请求、改 store、塞业务规则，就会得到一段只存在于模板里、脱离组件无法复用也无法单测的逻辑。

### 4.9 排查手册：五个高频现象的定位顺序

响应式问题有固定套路：先在 Vue Devtools 里确认 state 是否真的变了，再看组件有没有重渲染，最后才怀疑 DOM 层。

| 现象 | 先查什么 | 常见根因 |
| --- | --- | --- |
| 数据改了视图不动 | devtools 里字段值变了吗 | 新增属性、数组下标赋值、`Object.freeze` 对象 |
| 视图变了数据没变 | `v-model` 绑的是不是 computed | computed 只写了 get 没写 set |
| 刷新页面 404 | 服务端对未知路径的响应 | history 模式缺 fallback 配置 |
| 路由参数刷新后丢失 | path 里有没有声明 `:id` | 用了 name + params 但未声明参数 |
| 控制台报 mutation 违规 | 组件里有没有直接赋值 state | `v-model` 直接绑 store 的 state |

三步定位法的顺序不可颠倒：先确认响应式（数据是否被 observe），再确认渲染（`updated` 是否触发），最后确认路由与服务端。绝大多数「视图不更新」都不是框架 bug，而是改了一个没有被 defineProperty 包裹的属性。时间紧时最快的验证手段是在控制台执行 `vm.$forceUpdate()`：强制刷新后视图正常，说明问题一定出在依赖收集侧，与渲染逻辑无关。

### 4.10 常见坑

- 以为改完数据 DOM 立刻变：同步读到的永远是旧 DOM，任何依赖新 DOM 的操作（图表初始化、取元素尺寸、聚焦）都要放进 `$nextTick`。
- 用下标或 length 改数组：`arr[i] = x` 与 `arr.length = 0` 完全无响应，统一走 `splice` 或整体替换。
- 直接给 state 赋值：`this.$store.state.user.name = 'x'` 在开发环境报错、生产环境静默生效（严格模式被关掉），状态流转从此无法追踪。
- 路由守卫漏写 `next()`：某个 if 分支没调 next，点击后页面毫无反应且控制台不报错，只能逐行看守卫内的分支。
- 把大对象塞进 data 且不需要响应：初始化递归遍历会让首屏变慢，用 `Object.freeze()` 或挪出 data，只读的配置表、省市区字典都属于这一类。

## 5. Vue 3 核心

### 5.1 Composition API 要解决的问题

Vue 3 最大的变化不是运行速度，而是组织代码的方式。Vue 2 的选项式 API（Options API，按 data、computed、methods 等固定选项分区书写组件逻辑的写法）有一个隐含前提：一个组件只负责一件小事。一旦组件长到三五百行，同时处理「列表加载 + 搜索筛选 + 弹窗编辑 + 权限判断」，这个前提就不成立了。

具体会暴露出三个问题。

第一，逻辑被选项切碎。一个「搜索」功能，状态写在 data，派生值写在 computed，处理函数写在 methods，初始化写在 mounted，清理写在 beforeDestroy，五处分散，改一次需求要在文件里上下翻五遍。

第二，复用只能靠 mixin。mixin 的 data 与方法会直接合并进组件，命名冲突时后引入的静默覆盖先引入的；组件里出现一个 `loading`，看不出它来自哪个 mixin；mixin 之间还能互相依赖，形成看不见的耦合。

第三，this 是动态上下文。类型系统无法从 `this` 推导成员，编辑器补全和重构都靠猜；函数脱离 `this` 就不能独立复用，未被使用的选项也很难被 tree-shaking（摇树优化，打包时删除从未执行的代码）剔除。

Composition API（组合式 API，用普通函数把同一个功能的响应式状态、派生值与副作用写在一起的写法）正是针对这三点：按功能而不是按选项组织代码，用函数而不是 mixin 复用，不依赖 this。

| 维度 | 选项式 API | 组合式 API |
| --- | --- | --- |
| 代码组织 | 按 data / computed / methods 分区 | 按功能聚合成一个函数 |
| 逻辑复用 | mixin、作用域插槽 | 自定义组合函数（composable） |
| 命名冲突 | mixin 静默覆盖 | 函数内部变量，天然隔离 |
| 类型推导 | this 动态，推导弱 | 普通变量，推导完整 |
| 打包 | 选项难以剔除 | 未调用的函数可被 tree-shaking |
| 上手成本 | 结构固定，新手友好 | 需先理解响应式 API |

两者不是替代关系：Vue 3 仍完整支持选项式写法，同一个项目甚至同一个组件里可以混用。组合式 API 的价值在复杂组件与逻辑复用，简单展示型组件用选项式反而更短。

还有一个隐性差别：选项式 API 的复用单位是「一个被合并进来的对象」，组合式 API 的复用单位是「一个普通函数」。普通函数能传参、能返回值、能在单元测试里脱离组件直接调用，也能嵌套进另一个函数，这让同一份逻辑既能给组件用，也能给测试或另一个组合函数用。

### 5.2 setup 与 script setup

setup 是组合式 API 的唯一入口，在组件实例创建时、beforeCreate 之前执行。此刻 props 已解析完成，但 this 还不可用，这是新手最常撞的一堵墙。

它的签名是 `setup(props, context)`。props 是响应式对象，直接解构会丢失响应性；context 包含 attrs、slots、emit、expose。返回值有两种含义：返回对象时，对象上的属性暴露给模板；返回函数时，该函数被当作组件的渲染函数。

```vue
<script>
import { ref, onMounted } from 'vue'

export default {
  props: { keyword: { type: String, default: '' } },
  setup(props, { emit }) {
    const list = ref([])
    const loading = ref(false)

    async function load() {
      loading.value = true                       // JS 中必须通过 .value 读写
      const res = await fetch(`/api/items?q=${props.keyword}`)
      list.value = await res.json()
      loading.value = false
    }

    onMounted(load)                              // 必须在 setup 同步执行期间注册
    return { list, loading, load }               // 返回对象 = 暴露给模板
  }
}
</script>
```

`<script setup>` 是编译期语法糖：顶层声明的变量、函数、import 进来的组件自动暴露给模板，不用写 return。它编译后等价于 setup 函数体，运行时开销更低，模板里的类型推导也最完整。

```vue
<script setup>
import { ref, computed, onMounted } from 'vue'
import ItemCard from './ItemCard.vue'           // 无需注册，模板中直接用

const props = defineProps({ keyword: { type: String, default: '' } })
const emit = defineEmits(['select'])             // 声明事件，供校验与类型推导

const list = ref([])
const loading = ref(false)
const count = computed(() => list.value.length)

async function load() {                              // 只需普通函数，无需 this
  loading.value = true
  const res = await fetch(`/api/items?q=${props.keyword}`)
  list.value = await res.json()
  loading.value = false
}
onMounted(load)
</script>

<template>
  <ItemCard v-for="it in list" :key="it.id" :item="it" @click="emit('select', it)" />
  <p>共 {{ count }} 条</p>
</template>
```

四个只在 `<script setup>` 中可用的编译器宏（无需 import）：defineProps 声明 props、defineEmits 声明事件、defineExpose 显式暴露给父组件 ref 的成员、defineOptions（3.3+）声明 name 与 inheritAttrs 等组件选项。这些宏只能在 `<script setup>` 顶层直接调用，写进 if 分支或函数体里不会按预期工作，也不能用 import 引入。

`<script setup>` 中的组件默认是封闭的：父组件通过模板 ref 拿到的实例只能访问 defineExpose 暴露的内容。这比 Vue 2 的「实例上什么都能取」更可控，也避免了内部实现被外部依赖。

```vue
<!-- 子组件：只对外暴露两个方法，其余内部状态不可见 -->
<script setup>
function focus() { /* ... */ }
function reset() { /* ... */ }
defineExpose({ focus, reset })
</script>
```

```vue
<!-- 父组件：模板 ref 拿到的是被裁剪过的实例 -->
<script setup>
const panel = ref(null)
onMounted(() => panel.value?.focus())      // 只能调用 focus / reset
</script>

<template>
  <UserPanel ref="panel" />
</template>
```

`<script setup>` 允许与普通 `<script>` 同时存在于一个 .vue 文件中，后者用于声明 name、inheritAttrs 等组件选项，或放置只执行一次的模块级代码。需要注意的是 props 解构：`const { keyword } = defineProps(...)` 在 Vue 3.5 之前会让 keyword 退化成初始快照、丢失响应性，3.5 起才支持响应式 props 解构；跨版本兼容的写法是保留 props 对象，或写成 `const keyword = computed(() => props.keyword)`。

### 5.3 ref、reactive、toRefs、shallowRef

Vue 3 提供两条创建响应式数据的路，选错的直接后果是「数据变了、页面不动」，且不报任何错。

ref（引用包装，把任意值包进一个带 .value 的对象，靠读写 .value 触发追踪）适合基本类型，也能包对象。在 JS 里必须写 `.value`，在模板里自动解包。

reactive（深层响应式代理，用 Proxy 把对象整体变成可追踪对象）只接受对象、数组、Map、Set，返回 Proxy，属性直接访问。解构会丢失响应性，因为解构拿到的是求值后的普通值，与源对象断开。

toRefs 把 reactive 对象的每个属性转成指向原属性的 ref，专门用于安全解构。shallowRef 与 shallowReactive 只追踪浅层：shallowRef 只有整体替换 `.value` 才触发，适合存大数组、图表实例、第三方类实例，避免深层代理的开销与副作用。

| API | 接受类型 | 访问方式 | 解构后 | 模板解包 | 典型场景 |
| --- | --- | --- | --- | --- | --- |
| ref | 任意 | `.value` | 仍是 ref | 自动 | 基本类型、需要整体替换的单值 |
| reactive | 对象 / 集合 | 直接访问 | 丢失响应性 | 不适用 | 表单对象、成组状态 |
| shallowRef | 任意 | `.value` | 仍是 ref | 自动 | 大列表、ECharts 实例 |
| toRefs(对象) | 对象 | `.value` | 仍是 ref | 自动 | 组合函数返回值 |

```javascript
import { reactive, toRefs } from 'vue'

const state = reactive({ page: 1, size: 20 })

// 错误：解构出的是普通数字，改 page 不会影响 state，页面也不更新
// const { page, size } = state

// 正确：toRefs 让每个属性都变成指向原对象的 ref
const { page, size } = toRefs(state)
page.value++                                    // 等价于 state.page++
```

两条容易忽略的规则：给 ref 赋一个对象时，内部会用 reactive 处理它，所以 `ref({ a: 1 }).value.a` 是响应式的，而 shallowRef 不会；reactive 对象的顶层属性如果是 ref 会自动解包，但数组元素与 Map、Set 里的 ref 不会解包，取出来还得写 `.value`。

选择可以简化为三条：单个基本类型、需要整体替换的值一律用 ref；一组互相关联的字段用 reactive 包成对象；从组合函数往外返回时用 ref 或 toRefs 的产物，不要直接返回 reactive 对象——接收方一旦解构就会静默失去响应性。第三方实例、只读配置、纯展示用的大数组交给 shallowRef 或 markRaw，能省掉一层深层代理。toRef 是 toRefs 的单属性版本，用来从对象里安全取出某个字段，属性当前不存在时它也能生成一个可写 ref。

### 5.4 computed 与 watch/watchEffect

computed（计算属性，由其他响应式数据派生出新值并缓存结果）是惰性求值的：依赖不变时反复读取只返回缓存，依赖一变就标脏、下次读取时重算。不要在里面写副作用，比如发请求或改其他状态。

```javascript
import { ref, computed } from 'vue'

const price = ref(100)
const count = ref(2)
const total = computed(() => price.value * count.value)   // 依赖不变则不重算

const first = ref('Li'), last = ref('Lei')
const fullName = computed({                               // 可写计算属性
  get: () => `${first.value} ${last.value}`,
  set: (v) => { [first.value, last.value] = v.split(' ') }
})
```

watch（侦听器，显式指定侦听源，值变化时执行回调）能拿到新值与旧值，默认惰性，可配置 immediate、deep、flush、once。watchEffect（副作用侦听，立即执行一次并自动收集回调里读到的所有响应式依赖）写法更短，但没有显式依赖列表，也拿不到旧值。

| 特性 | computed | watch | watchEffect |
| --- | --- | --- | --- |
| 首次执行 | 否，首次读取时才求值 | 否，immediate 可改 | 是，立即执行一次 |
| 结果缓存 | 是 | 不适用 | 不适用 |
| 依赖来源 | 自动收集 | 显式声明 | 自动收集 |
| 拿到旧值 | 无 | 有 | 无 |
| 返回值 | 派生值 | 停止函数 | 停止函数 |
| 典型用途 | 派生展示数据 | 联动请求、监听路由参数 | 依赖多且只需「变了就做」 |

```javascript
import { ref, watch, watchEffect } from 'vue'

const keyword = ref('')
const page = ref(1)

const stop = watch(
  [keyword, page],                                  // 多源侦听，回调参数是数组
  ([kw, p], [oldKw, oldP], onCleanup) => {
    if (!kw) return
    const timer = setTimeout(() => fetchList(kw, p), 300)   // 简易防抖
    onCleanup(() => clearTimeout(timer))            // 下次触发前清理上一次的副作用
  },
  { immediate: true, flush: 'post' }                // flush: pre 默认 / post / sync
)

// 自动收集依赖：keyword 变了就改标题
watchEffect(() => { document.title = `${keyword.value} - 列表` })

stop()                                              // 不再需要时手动停；卸载时自动停
```

三个实用细节。侦听 reactive 对象的某个属性必须写成 getter：`watch(() => state.page, cb)`；直接写 `watch(state.page, cb)` 传入的是求值后的数字，侦听静默失效。侦听 reactive 对象本身时默认是深层侦听；侦听 ref 包的对象则要显式 `{ deep: true }`。`flush: 'post'` 让回调在 DOM 更新之后执行，适合读取元素尺寸。

computed 只适合纯计算，不要在里面发请求：它的求值时机由读取者决定，一次渲染里可能读取多次，副作用会被重复触发。与输入框联动搜索这类场景要额外处理竞态——旧请求可能比新请求回得更晚，把界面覆盖成过期数据：

```javascript
let controller
watch(keyword, async (kw) => {
  controller?.abort()                                  // 取消上一次未完成的请求
  controller = new AbortController()
  const res = await fetch(`/api/items?q=${kw}`, { signal: controller.signal })
  list.value = await res.json()
})
```

`watch` 还支持 `{ once: true }`（3.4+），回调执行一次后自动停止，适合「拿到首屏数据后初始化一次」的场景；组件卸载时框架会自动停止该组件注册的所有侦听，手动调用返回的 stop 函数只在需要提前停止时才有必要。

### 5.5 用组合函数封装与复用逻辑

组合函数（Composable，以 use 开头、内部调用 Vue 响应式 API 并返回一组响应式状态的普通函数）是组合式 API 落地的复用载体。它的结构很固定：内部创建状态、内部注册生命周期、最后返回需要暴露的部分。

```javascript
// composables/useList.js
import { ref, onMounted, onBeforeUnmount } from 'vue'

export function useList(fetcher, { immediate = true } = {}) {
  const list = ref([])
  const loading = ref(false)
  const error = ref(null)

  async function reload() {
    loading.value = true
    error.value = null
    try {
      list.value = await fetcher()
    } catch (e) {
      error.value = e                     // 错误留在函数内部，调用方只读状态
    } finally {
      loading.value = false
    }
  }

  if (immediate) onMounted(reload)       // 依赖组件上下文的注册留在函数内部
  onBeforeUnmount(() => { /* 取消请求、清定时器 */ })

  return { list, loading, error, reload } // 返回 ref，调用方解构也安全
}
```

```javascript
// 组件中使用：一行接入，其余状态与流程都藏在函数里
const { list, loading, error, reload } = useList(
  () => fetch('/api/items').then(r => r.json())
)
```

几条约定值得遵守：函数名与文件名都以 use 开头，一眼能认出是组合函数；返回值用对象而不是数组，调用方按需解构、顺序无关；内部创建的定时器、事件监听、请求必须配对清理；不要在失去组件上下文的普通回调里调用 onMounted，钩子会无处注册，需要脱离组件作用域时改用 effectScope。

与 mixin 的本质差异是：组合函数不共享隐式命名空间，两个 useList 重命名后可以并存；依赖关系写在参数和返回值上，是显式的；不会因为合并顺序不同产生不同结果。需要留意的代价是，组合函数在每个调用处都真实执行一遍、状态互不共享，跨组件共享状态应该把状态提到模块顶层或交给 Pinia。

### 5.6 生命周期 API

组合式生命周期以 onXxx 函数形式提供，且必须在 setup 同步执行阶段调用：放到 await 之后或异步回调里不会报错，但钩子不执行，因为当前实例上下文已经失效。

| 选项式钩子 | 组合式 API | 触发时机 |
| --- | --- | --- |
| beforeCreate / created | setup 函数体 | 实例创建、props 解析之后 |
| beforeMount | onBeforeMount | 首次挂载前，模板已编译、DOM 未插入 |
| mounted | onMounted | DOM 已插入文档 |
| beforeUpdate | onBeforeUpdate | 数据变化、DOM 更新之前 |
| updated | onUpdated | DOM 更新之后 |
| beforeDestroy | onBeforeUnmount | 卸载前，适合清定时器、解绑事件 |
| destroyed | onUnmounted | 卸载之后 |
| activated / deactivated | onActivated / onDeactivated | 被 keep-alive 激活或缓存 |
| errorCaptured | onErrorCaptured | 捕获后代组件抛出的错误 |

onMounted 可以在同一组件内注册多次，按注册顺序执行；把注册与清理写在相邻位置是最不容易漏的写法。

```javascript
import { onMounted, onBeforeUnmount, ref } from 'vue'

const el = ref(null)
function onResize() { /* 重新计算布局 */ }

onMounted(() => {
  window.addEventListener('resize', onResize)
  console.log(el.value.offsetWidth)      // 只有 mounted 之后才拿得到真实 DOM 尺寸
})
onBeforeUnmount(() => window.removeEventListener('resize', onResize))
```

需要针对 props 变化做副作用时，用 `watch(() => props.x, cb)`，不要挂在 onUpdated 上：后者只要组件因任何原因重渲染就会执行，条件判断会越写越复杂。

钩子的执行顺序也值得记牢：挂载阶段父组件的 onBeforeMount 先于子组件的 onMounted，卸载阶段子组件的 onBeforeUnmount 先于父组件的 onUnmounted。onMounted 回调里不能假设所有子组件都已挂载完成，异步组件或 Suspense 下的子组件尤其如此，依赖子组件内部状态的初始化应放到 nextTick 之后或改用事件通知。

两个只在特定场景出现的钩子：onServerPrefetch 用于服务端渲染时在服务端等待异步数据，纯客户端项目用不到；onRenderTracked 与 onRenderTriggered 只在开发构建中生效，能打印出「哪个依赖被收集、是哪次写入触发了重渲染」，是排查多余渲染最直接的工具。

### 5.7 模板与组件写法的变化

第一，支持多根节点（Fragment）。Vue 2 要求模板有唯一根元素，为了满足它多包一层 div，经常破坏 flex 或 grid 布局；Vue 3 允许写多个并列根节点。代价是多根组件不会自动继承父组件传入的非 prop 属性（class、style、原生事件），必须用 `v-bind="$attrs"` 显式指定落到哪个节点上。单根组件默认仍自动继承，可用 `inheritAttrs: false` 关闭。

```vue
<!-- 多根组件：手动决定父组件传下来的 class、事件落在哪个节点 -->
<script setup>
defineOptions({ inheritAttrs: false })     // 关闭自动继承
</script>

<template>
  <header v-bind="$attrs">标题</header>     <!-- attrs 只落到 header -->
  <main>正文</main>
</template>
```

第二，v-model 机制重写。Vue 2 在组件上固定用 value 属性配 input 事件；Vue 3 改为 modelValue 属性配 update:modelValue 事件，并支持参数，一个组件可以挂多个。

```vue
<!-- 子组件 Child.vue -->
<script setup>
const props = defineProps({
  modelValue: String,        // 对应父组件的 v-model
  title: String              // 对应父组件的 v-model:title
})
const emit = defineEmits(['update:modelValue', 'update:title'])
</script>

<template>
  <input :value="modelValue" @input="emit('update:modelValue', $event.target.value)" />
  <input :value="title" @input="emit('update:title', $event.target.value)" />
</template>
```

```vue
<!-- 父组件：同时绑定两个 v-model -->
<Child v-model="text" v-model:title="head" />
```

`.sync` 修饰符已被 `v-model:xxx` 取代。自定义修饰符通过 `v-model.capitalize` 传入，子组件用 `modelModifiers` 这个 prop 读取。表单项少、逻辑简单时，Vue 3.4+ 的 defineModel 宏能省掉 props 与 emit 的样板代码。

```vue
<!-- 父组件 -->
<Child v-model.capitalize="text" />

<!-- 子组件：读取修饰符并决定是否转换大小写 -->
<script setup>
const props = defineProps({
  modelValue: String,
  modelModifiers: { default: () => ({}) }   // 修饰符统一收在这里
})
const emit = defineEmits(['update:modelValue'])
function onInput(e) {
  let v = e.target.value
  if (props.modelModifiers.capitalize) v = v.charAt(0).toUpperCase() + v.slice(1)
  emit('update:modelValue', v)
}
</script>
```

第三，其他高频差异要记住：过滤器 filter 移除，改用 computed 或方法；$listeners 合并进 $attrs；$children 移除，需要访问子组件就显式用 ref；v-if 与 v-for 同时用在同一元素上时优先级反转（Vue 3 中 v-if 优先），此时 v-if 拿不到 v-for 的临时变量，正确做法是用 computed 先过滤出列表再渲染。

### 5.8 Teleport、Suspense、Fragment

Teleport（传送门，把一段子节点渲染到 DOM 树中另一个指定位置的组件）解决「弹窗被父级 overflow: hidden、transform 或 z-index 层级困住」的经典问题。属性 to 接受 CSS 选择器或真实元素，disabled 为 true 时退回原位渲染。

```vue
<!-- Modal.vue：弹窗实际渲染到 body 下，脱离父级层叠上下文 -->
<script setup>
defineProps({ open: Boolean })
defineEmits(['close'])
</script>

<template>
  <Teleport to="body">
    <div v-if="open" class="mask" @click.self="$emit('close')">
      <div class="dialog"><slot /></div>
    </div>
  </Teleport>
</template>
```

Teleport 只改变渲染位置，不改变组件层级：provide / inject 照常生效，组件树中的父子关系不变；而 DOM 事件冒泡按真实 DOM 结构走。多个 Teleport 指向同一目标时按挂载顺序追加，卸载时各自移除自己插入的节点。`to` 的目标必须在挂载时已存在于文档中，目标元素由异步渲染产生时会报 Target not found，此时可先用 `disabled` 渲染到原位，等目标就绪再切换。

Suspense（悬念组件，等待子树中的异步依赖就绪后再渲染，就绪前显示 fallback 内容）主要配合顶层 await 的 `<script setup>` 或 defineAsyncComponent 使用。

```vue
<template>
  <Suspense>
    <template #default><UserPanel /></template>
    <template #fallback><p>加载中…</p></template>
  </Suspense>
</template>
```

```javascript
// UserPanel.vue：顶层 await 让组件变成异步组件
const data = await fetch('/api/user').then(r => r.json())
```

Suspense 在 Vue 3 中仍标记为实验性，API 可能在次要版本调整，且它只处理「等待」不处理「失败」。生产项目更稳的做法是把 loading 与 error 状态写进组件自身，或用 defineAsyncComponent 的 loadingComponent、errorComponent、delay、timeout 四个选项控制异步组件的加载与超时。

使用时还有两个约束：default 插槽里只能有单个根节点，多根会直接警告且 fallback 行为异常；Suspense 只等待它内部第一层异步依赖，嵌套的异步组件要各自处理，否则会出现「外层已就绪、内层还在转圈」的空窗。

Fragment（片段，Vue 3 允许组件有多个根节点，编译后对应一个不可见的片段容器）不是需要手写的标签，而是多根节点的运行时表示。它带来两个副作用：多根组件的属性透传需要显式指定；对多根组件取 $el 拿不到单一根 DOM 节点。需要操作 DOM 时应给具体元素加 ref，在 v-for 中收集元素引用则要用函数式 ref 把元素推进数组。

### 5.9 常见坑

- 解构 reactive 丢响应性。`const { page } = reactive({ page: 1 })` 之后 page 只是一个快照数字，改它页面不动、也不报错。用 toRefs 或 toRef 解构，或者始终保留对象引用。
- 在异步回调里注册生命周期。setup 中 await 之后调用 onMounted 不报错但不会执行。把注册动作放在第一个 await 之前，或抽成独立同步函数调用。
- 忘记写 .value。ref 在 JS 里漏写 .value 不报错，只会得到 ref 对象本身，于是 `if (loading)` 永远为真、`loading = false` 直接把变量覆盖成布尔值。模板里不需要 .value，恰恰加剧了肌肉记忆的混乱。
- 用 watch 侦听 reactive 属性却没写成 getter。`watch(state.page, cb)` 传入的是数字，侦听静默失效且无警告，必须写 `watch(() => state.page, cb)`。

## 6. Vue 3 响应式原理与状态管理

### 6.1 Proxy 为什么能替代 Object.defineProperty

Vue 2 用 `Object.defineProperty` 把每个属性改写成 getter/setter，Vue 3 换成了 Proxy（代理，在目标对象外面建一层拦截层，读属性、写属性、删除属性、`in` 判断都会先进入拦截器）。核心原语没变，仍然是「读时收集依赖、写时通知更新」，变的是拦截的粒度和时机。

```javascript
// mini-reactivity.js：用 Proxy 重建响应式骨架
const targetMap = new WeakMap();      // target -> Map<key, Set<effect>>
let activeEffect = null;
const effectStack = [];               // 嵌套 effect 时保存外层 effect

function reactive(target) {
  return new Proxy(target, {
    get(obj, key, receiver) {
      track(obj, key);                                   // 读时收集依赖
      const res = Reflect.get(obj, key, receiver);
      // 惰性代理：只有真正被访问到的嵌套对象才包装，初始化不再递归
      return typeof res === 'object' && res !== null ? reactive(res) : res;
    },
    set(obj, key, value, receiver) {
      const old = obj[key];
      const ok = Reflect.set(obj, key, value, receiver);
      if (!Object.is(old, value)) trigger(obj, key);     // 值真的变了才通知
      return ok;
    },
    deleteProperty(obj, key) {
      const had = key in obj;
      const ok = Reflect.deleteProperty(obj, key);
      if (had) trigger(obj, key);                        // 删除也能触发，这是关键改进
      return ok;
    }
  });
}
```

用 Proxy 改写还有一个不易察觉的好处：`get` 里可以按 key 返回不同结果，比如对数组做特殊处理、对 `__v_isReactive` 这类内部标记直接短路，而不必像 Vue 2 那样去改原型。代价是 Proxy 无法 polyfill，且拦截器必须老老实实返回布尔值或者用 `Reflect` 转发，少写一个 return 就会在 `'use strict'` 下静默失败。

两者的差异直接决定了写法上的差别：

| 能力 | Object.defineProperty（Vue 2） | Proxy（Vue 3） |
| --- | --- | --- |
| 新增属性 | 拦截不到，必须 `Vue.set` | 天然拦截，直接赋值 |
| 删除属性 | 拦截不到，必须 `Vue.delete` | `deleteProperty` 拦截 |
| 数组下标赋值 `arr[0]=x` | 拦截不到，靠改写 7 个数组方法补丁 | 天然拦截 |
| Map / Set | 不支持 | 支持 |
| 初始化成本 | 递归遍历全部属性，数据越大越慢 | 惰性，访问到才代理 |
| 兼容性 | IE9+ | 不支持 IE |

一句话总结：Proxy 让「新增/删除属性不响应」这类经典坑从语言层面消失，代价是放弃低版本浏览器。惰性代理的收益在大列表上最明显：Vue 2 初始化一张一万行的表格，要把十万个属性逐个改写成 getter/setter，首屏会明显卡顿；Vue 3 只代理表格这个外层对象，行数据要等你渲染到那一屏时才被包装。另外 Proxy 只能代理对象，数字、字符串、布尔这类基本类型无法代理，这正是 `ref` 存在的根本原因。

### 6.2 依赖收集与触发：targetMap 的三层结构

响应式系统要回答两个问题：某个属性被哪些 effect（副作用函数，读取响应式数据并在数据变化时重新执行的函数）用过；某个 effect 用过哪些属性。第二个问题是为了清理：`flag ? a : b` 这种分支切换后，effect 不该再订阅已经不再读取的 `a`。

```javascript
function track(target, key) {
  if (!activeEffect) return;                 // 不在任何 effect 内读取，无需收集
  let depsMap = targetMap.get(target);
  if (!depsMap) targetMap.set(target, (depsMap = new Map()));
  let dep = depsMap.get(key);
  if (!dep) depsMap.set(key, (dep = new Set()));
  dep.add(activeEffect);
  activeEffect.deps.push(dep);               // 反向记录，供 cleanup 使用
}

function trigger(target, key) {
  const depsMap = targetMap.get(target);
  const dep = depsMap && depsMap.get(key);
  if (!dep) return;
  new Set(dep).forEach(e => (e.scheduler ? e.scheduler() : e.run()));  // 拷贝后遍历
}

function effect(fn, options = {}) {
  const e = function () {
    if (!e.active) return;                   // 已被 stop 的 effect 不再执行
    cleanup(e);                              // 每次重跑前清空旧依赖
    activeEffect = e;
    effectStack.push(e);
    try { return fn(); } finally {
      effectStack.pop();
      activeEffect = effectStack[effectStack.length - 1] || null;  // 恢复外层 effect
    }
  };
  e.deps = []; e.active = true; e.scheduler = options.scheduler;
  e.run = e; e();                            // 首次执行，顺便完成依赖收集
  return e;
}

function cleanup(e) {
  e.deps.forEach(dep => dep.delete(e));      // 断开不再使用的 key
  e.deps.length = 0;
}
```

内存里的结构是这样三层：

```
targetMap: WeakMap
  └─ 响应式对象 → Map
       └─ 属性名 → Set<effect>   // 用 Set 保证同一属性只记一次
```

外层用 WeakMap 是为了不阻止垃圾回收：原始对象被销毁时，这条依赖记录跟着消失。dep 用 Set 而不是数组也有实际原因：同一个 effect 在一次求值里可能多次读到同一个属性（模板里 `{{ a }}` 写两次就是两次），数组会重复登记，卸载时容易漏删；Set 的 `add`、`delete` 都幂等，清理逻辑才写得对。`effectStack` 解决的是嵌套问题：effect A 内部执行 effect B 时，`activeEffect` 必须指向 B，B 跑完要还原成 A，否则 B 期间读到的属性会错误地记到 A 头上。

Proxy 还能拦截 `in` 判断和 `for...in` 遍历（对应 `has` 与 `ownKeys` 两个陷阱），所以 `Object.keys(state)` 这种写法在新增属性后也能重新执行，Vue 2 里同样写法要靠 `Vue.set` 收尾。反过来要清楚：`track` 只在 effect 执行期间生效，在普通函数或事件回调里读响应式数据不会留下依赖记录，这是排查「改了数据却没触发更新」的第一个检查点。

组件卸载时，Vue 调用该渲染 effect 的 `stop()`，把这个 effect 从所有 dep 集合里删掉，避免已销毁组件的渲染函数继续被触发。默认不传 `onStop` 就没法回收，这也是手动写 `watchEffect` 时要在 `onUnmounted` 里停掉的原因。需要成批管理副作用时用 `effectScope()`：

```javascript
import { effectScope, effect } from 'vue';

const scope = effectScope();
scope.run(() => {          // 作用域内创建的 effect / computed 全部归属这个 scope
  effect(() => console.log(a.value));
  effect(() => console.log(b.value));
});
scope.stop();              // 一次性停掉作用域内所有副作用，不用逐个记录
```

Pinia 的每个 store 内部就用 `effectScope` 管理自己的 computed 和 watch，store 被销毁时能整体回收，不会留下悬空订阅。

### 6.3 调度器：改三次为什么只渲染一次

同步执行 effect 会带来一个严重问题：`count++; count++; count++` 如果每次赋值都立刻重新渲染，就等于渲染三次。Vue 3 给渲染 effect 配了 scheduler（调度器，决定 effect 什么时候执行、以什么顺序执行的函数），把更新推进微任务队列。

```javascript
const queue = new Set();        // Set 天然去重：同一组件的多次修改只入队一次
let isFlushing = false;
const resolvedPromise = Promise.resolve();

function queueJob(job) {
  queue.add(job);
  if (!isFlushing) {
    isFlushing = true;
    resolvedPromise.then(flushJobs);   // 当前同步代码全部跑完后再统一更新
  }
}

function flushJobs() {
  try {
    // 按组件 uid 排序：父组件先于子组件更新，保证父传子的 props 在同一轮里是对的
    [...queue].sort((a, b) => a.uid - b.uid).forEach(job => job());
  } finally {
    queue.clear();
    isFlushing = false;
  }
}
```

队列里同一个 job 只会执行一次，但如果某个 job 执行时又触发了新 job，新 job 会追加到当前队列末尾并在同一轮 flush 里继续处理，而不是等到下一个微任务；正是因为「同一轮内可能反复追加」，Vue 才需要那个递归上限来兜底。

三个结论可以直接指导写代码：第一，同步代码里连续改同一个状态，只会触发一次组件更新；第二，更新发生在微任务（microtask，当前宏任务结束后、下一个宏任务开始前执行的队列），所以 `nextTick` 之后能拿到最新 DOM，而 `setTimeout(fn, 0)` 也能拿到但时机更晚；第三，如果更新里又改了状态，会在 `flushJobs` 的下一轮继续排队，不能无限排队，Vue 会用一个递归计数上限直接抛错。

`nextTick` 本身并不神秘，它就是包了一层 Promise：

```javascript
// 简化实现：正在 flush 时返回新的微任务，避免更新中读不到最新结果
export function nextTick(fn) {
  const p = isFlushing ? Promise.resolve() : resolvedPromise;
  return fn ? p.then(fn) : p;
}
```

`watch(source, cb, { flush: 'post' })` 里的回调也走同一条队列：排在组件更新之后执行，因此在回调里能读到已经渲染完成的 DOM；默认的 `flush: 'pre'` 排在更新之前，此时 DOM 还是旧的；`flush: 'sync'` 完全退化成同步执行，只在需要精确测量中间态时使用，滥用会让一次循环里触发几十次回调。

顺便说清 `watch` 与 `watchEffect` 的差异：`watchEffect` 靠「先跑一遍函数」来收集依赖，函数里因为条件不成立而没被执行到的分支不会被订阅；`watch` 显式声明依赖，即便本轮没读到某个值也能追踪变化，排查「为什么没触发」时更容易定位。

### 6.4 reactive 与 ref 的取舍

`ref` 的实现思路是把任意值包成一个对象 `{ value }`，再用 getter/setter 拦截 `value`，从而绕开「Proxy 不能代理基本类型」的限制：

```javascript
function ref(raw) {
  const r = {
    get value() { track(r, 'value'); return raw; },
    set value(v) { if (!Object.is(raw, v)) { raw = v; trigger(r, 'value'); } }
  };
  return r;
}
```

| 使用场景 | 推荐 | 理由 |
| --- | --- | --- |
| 数字、字符串、布尔 | `ref` | Proxy 代理不了基本类型 |
| 多层嵌套对象 | `reactive` | 不用到处写 `.value` |
| 需要整体替换的对象 | `ref` 或 `shallowRef` | `reactive` 直接赋新对象会断开代理 |
| 大数组、图表实例、地图实例 | `shallowRef` | 深层代理开销大，且常被第三方实例内部方法改坏 |
| 解构后仍要响应 | `toRefs` / `storeToRefs` | 解构只是取值快照 |

四个高频嵌套陷阱必须记住：

```javascript
import { reactive, toRefs, shallowRef } from 'vue';

const state = reactive({ count: 0 });
const { count } = state;        // 陷阱一：解构出来的是数字快照，改 count 不触发更新
const refs = toRefs(state);     // 正确做法：得到保持连接的 ref，模板里写 refs.count.value

state = { count: 1 };           // 陷阱二：这只是给变量重新赋值，新对象没有代理，页面不动
Object.assign(state, { count: 1 });   // 正确做法：改原对象本身

const store = reactive({ list: [ref(1)] });
console.log(store.list[0]);     // 陷阱三：数组/Map 里的 ref 不会自动解包，打印出 RefImpl 对象

const big = shallowRef({ tree: hugeData });   // 陷阱四：只有 .value 这一层响应
big.value = { tree: newData };                 // 替换整个值有效
big.value.tree.name = 'x';                     // 深层修改无效，需要 triggerRef(big)
```

另一个常见取舍是 `reactive([])` 和 `ref([])`：两者都能让数组响应，但 `reactive([])` 不能整体替换，`list = newList` 之后就断了；`ref([])` 需要写 `.value`，却可以直接换掉整个数组。列表类状态建议统一用 `ref`，因为分页、筛选、下拉刷新之后整体替换数组是最常见的操作。

模板里 `ref` 会自动解包（`{{ msg }}` 而不是 `{{ msg.value }}`），但脚本里除了顶层变量之外一律要写 `.value`。这条规则还有一处容易误判的反向情况：把一个 `ref` 放进 `reactive` 对象后，通过属性访问会拿到解包后的值，`state.foo` 不是 ref 而是值本身，此时对它赋值 `state.foo = 1` 会经由 reactive 的 setter 写回原 ref，看起来「不用 .value」，但如果把 `state.foo` 单独解构出来，又变回普通值。所以最稳的写法是：团队内约定所有状态都用 `ref`，只在明确需要「对象属性整体代理」时才用 `reactive`，减少两套心智模型混用。`toRaw(obj)` 可以拿到原始对象，修改原始对象不会触发更新，它适合传给不需要响应的第三方库。

### 6.5 编译优化：静态提升、patchFlag 与 Block Tree

Vue 3 的运行时之所以更快，一半功劳在编译器。编译器把模板编译成渲染函数时，会提前标出哪些部分是静态的、哪些是动态的，运行时只处理动态部分。

```html
<template>
  <div class="box">
    <h2>这段文字永远不变</h2>
    <p :id="id">{{ msg }}</p>
  </div>
</template>
```

编译结果（简化为调用形式）：

```javascript
const _hoisted_1 = { class: "box" };       // 静态属性提升到模块作用域，只创建一次
const _hoisted_2 = _createElementVNode("h2", null, "这段文字永远不变");  // 静态节点只创建一次

function render(_ctx) {
  return _createElementBlock("div", _hoisted_1, [
    _hoisted_2,                             // 直接引用常量，完全跳过 diff
    _createElementVNode("p", { id: _ctx.id }, _toDisplayString(_ctx.msg), 9 /* TEXT, PROPS */)
  ]);
}
```

`9` 就是 patchFlag（补丁标记，编译期写在虚拟节点上的数字，告诉运行时这个节点哪部分是动态的）。

| 值 | 常量名 | 含义 |
| --- | --- | --- |
| 1 | TEXT | 只有文本动态，比较时跳过属性 |
| 2 | CLASS | class 动态 |
| 8 | PROPS | 属性动态，需要完整比对 props |
| 16 | FULL_PROPS | 存在动态 key，无法静态判断 |
| 32 | NEED_HYDRATION | SSR 水合节点 |
| 512 | NEED_PATCH | 带 ref 或指令，需要额外 patch |

Block Tree（块树，把动态子节点收集成扁平数组的机制）更进一步：`openBlock()` 在渲染函数里开启一个块，所有带 patchFlag 的动态节点被收进当前块的 `dynamicChildren` 数组。父组件更新时不需要逐层遍历整棵虚拟 DOM 树，只遍历这个扁平数组。

```javascript
// 编译产物背后的收集逻辑（简化）
let currentBlock = null;
const blockStack = [];

function openBlock() { blockStack.push((currentBlock = [])); }
function closeBlock() { blockStack.pop(); currentBlock = blockStack[blockStack.length - 1] || null; }

function createElementBlock(tag, props, children, patchFlag) {
  const vnode = createVNode(tag, props, children, patchFlag);
  if (patchFlag > 0 && currentBlock) currentBlock.push(vnode);  // 只收动态节点
  vnode.dynamicChildren = currentBlock;
  return vnode;
}
```

代价是 `v-if`、`v-for` 这类结构会打断收集：模板里的一个 `v-for` 列表本身可能整体增删，编译器让它自己成为一个新的 Block 边界，这样外层块里只保留这个列表 vnode，列表内部再各自收集，动态节点一个都不会漏。除了这些，编译器还会处理 `v-once`（节点只在首次渲染时创建，之后永久跳过更新）和 `v-memo`（依赖数组未变化就复用整棵子树），它们适合渲染成本高但很少变化的长列表行。反过来说，如果模板里动态绑定铺满，patchFlag 只能退化成 `FULL_PROPS`，逐项比对 props，编译优化的收益会大幅缩水，此时瓶颈一般在业务计算而不在 diff。

判断标准很简单：想让 diff 变快，就让模板里动态绑定的数量尽量少，动态节点越集中，扁平数组越短。这些优化只对模板语法生效：用 `h()` 手写的渲染函数没有编译期信息，patchFlag 只能是 0，diff 退回全量比较，所以性能敏感的组件不要为了「灵活」而绕过模板。

### 6.6 diff 与最长递增子序列

同层子节点比较时，先用四个指针 `oldStart`、`oldEnd`、`newStart`、`newEnd` 做双端对比：新头与旧头相等就一起后移，新尾与旧尾相等就一起前移，新尾与旧头相等说明节点被移到了前面，新头与旧尾相等说明节点被移到了后面，四种情况都不成立才进入中间乱序段。乱序段用 key 建一张「key → 旧下标」的映射表，逐个判断新节点是新建、复用还是移动。

决定移动方案时用到最长递增子序列（Longest Increasing Subsequence，在一个序列中找长度最长且值严格递增的子序列）：把每个可复用节点在旧列表中的位置写成数组，求出的子序列就是「相对顺序本来就没变」的节点，让它们原地不动，其余节点移动，移动次数最少。

```javascript
// 求「不需要移动」的下标集合，复杂度 O(n log n)
function getSequence(arr) {
  const p = arr.slice();          // p[i] 记录 i 的前驱下标，供回溯
  const result = [0];             // result 存当前递增子序列的下标
  for (let i = 1; i < arr.length; i++) {
    const cur = arr[i];
    if (cur === 0) continue;      // 0 表示新节点，没有旧位置
    const last = result[result.length - 1];
    if (arr[last] < cur) { p[i] = last; result.push(i); continue; }  // 接在队尾
    let start = 0, end = result.length - 1;                          // 二分替换
    while (start < end) {
      const mid = (start + end) >> 1;
      if (arr[result[mid]] < cur) start = mid + 1; else end = mid;
    }
    if (cur < arr[result[start]]) {
      if (start > 0) p[i] = result[start - 1];
      result[start] = i;
    }
  }
  let len = result.length, idx = result[len - 1];
  while (len-- > 0) { result[len] = idx; idx = p[idx]; }   // 回溯出完整下标
  return result;
}
```

Vue 2 的双端 diff 没有这一步：它靠「把节点挪到旧指针之前、再推进指针」逐个处理，列表整体反转时要移动 n 次。Vue 3 先求出不动点集合，再一次性按新顺序 `insertBefore`，移动次数降到理论最小值。

朴素做法是为每个位置求「以它结尾的最长长度」，即动态规划 O(n²)：1000 个节点要做约 100 万次比较，4000 个节点就不可接受了。上面的贪心 + 二分解法把复杂度压到 O(n log n)，并且用 `p` 数组记录前驱，最后回溯出完整下标，而不是只知道长度。

例如把列表从 `[a, b, c, d]` 改成 `[a, b, d, c]`，只移动一个节点就够，而不是重建整段。这条逻辑依赖 key：如果 key 用数组下标，新旧节点会被判定成同一批，最长递增子序列的结果看似正确，实际复用了错误的组件实例和输入框状态。列表必须用稳定唯一的业务 id 作 key。

### 6.7 Pinia：Vue 3 的状态管理

Pinia（Vue 官方推荐的状态管理库，把共享状态定义成可组合的 store，天然支持 TypeScript 与 SSR）取代了 Vuex。一个 store 就是一个被响应式包裹的单例，跨组件 import 即可共享。

```javascript
// stores/cart.js —— setup 风格：像写组件一样写 store
import { defineStore } from 'pinia';
import { computed, ref } from 'vue';

export const useCartStore = defineStore('cart', () => {
  const items = ref([]);                 // state：ref 或 reactive
  const count = computed(() => items.value.reduce((s, i) => s + i.qty, 0));  // getter
  function add(product) {                // action：同步异步都直接写这里
    const hit = items.value.find(i => i.id === product.id);
    hit ? hit.qty++ : items.value.push({ ...product, qty: 1 });
  }
  async function checkout(api) {
    await api.submit(items.value);
    items.value = [];                    // 不需要 mutation，直接改
  }
  return { items, count, add, checkout };
});
```

```javascript
// 组件内使用
import { storeToRefs } from 'pinia';
import { useCartStore } from '@/stores/cart';

const cart = useCartStore();
const { count, items } = storeToRefs(cart);   // 解构 state/getter 必须包一层
const { add } = cart;                         // action 直接解构不会丢 this
add({ id: 1, name: '键盘', price: 199 });
```

同一套 API 还有 options 风格，写法和 Vuex 更接近，适合把老项目平滑迁过来：

```javascript
// stores/user.js —— options 风格
import { defineStore } from 'pinia';

export const useUserStore = defineStore('user', {
  state: () => ({ name: '', token: '' }),        // 必须是函数，否则多次实例化会共享同一份对象
  getters: { isLogin: (state) => !!state.token },
  actions: {
    async login(api, form) {
      this.token = (await api.login(form)).token;  // action 里 this 指向 store
    }
  }
});
```

两种风格可以在不同 store 里混用，同一个项目内最好统一，否则新人读代码要在两套写法之间来回切换。持久化也不必写在业务里，用插件统一接管：

```javascript
// main.js：给所有 store 加上 localStorage 持久化
pinia.use(({ store }) => {
  const key = `pinia:${store.$id}`;
  const saved = localStorage.getItem(key);
  if (saved) store.$patch(JSON.parse(saved));
  store.$subscribe((_mutation, state) => {
    localStorage.setItem(key, JSON.stringify(state));   // 每次变更后落盘
  });
});
```

与 Vuex 的差异集中在「少了一层仪式」：

| 维度 | Vuex 4 | Pinia |
| --- | --- | --- |
| store 定义 | 一个 options 对象 + modules 嵌套 | 每个 store 一个 `defineStore` |
| 修改 state | 必须 `commit` 一个 mutation | action 内直接赋值 |
| 异步 | action 里 commit mutation | action 直接改 |
| TypeScript | 需手写类型、泛型较绕 | 自动推断，无需额外声明 |
| 命名空间 | `namespaced: true` 手工开启 | 天然隔离，默认扁平结构 |
| 开发体验 | 模板冗长、devtools 按 mutation 追踪 | 直接追踪 action，支持热更新 |
| 体积 | 较大 | 约 1.5 KB |

还有两个容易踩的边界：setup 风格的 store 没有内置 `$reset`，要重置状态得自己写一个 `reset()` 函数，或者改用 options 风格；两个 store 互相引用时，不要在模块顶层直接调用对方的 `useXxxStore()`，那会形成循环依赖并在启动时报错，正确做法是在 action 函数内部按需调用。

设计上有三点值得注意：store 用 `reactive()` 包装后存进单例表，所以任何组件拿到的都是同一份；服务端渲染时必须在每个请求里调用 `createPinia()`，不能复用全局实例，否则不同用户的会话状态会互相污染；持久化通过 `pinia.use()` 插件统一拦截，不要在业务代码里手动读写 localStorage。

### 6.8 常见坑速查

状态管理的选型顺序可以这样判断：只在一个组件内用，就 `ref`；父子传两层，用 props/emit；跨页面共享且要持久化的，才引入 Pinia。过早把局部状态塞进全局 store，会让后面每次改动都要跨文件追踪数据来源。

调试验证时还有一招很有效：在浏览器 devtools 的组件面板里直接修改状态，观察哪一层组件重新渲染。如果数据变了、视图没动，问题在响应式收集（解构、shallowRef、toRaw）；如果整块组件都在重渲染，问题在组件拆分和依赖范围。

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| 列表删中间一项后，后面输入框内容错位 | key 用了 index | 换成稳定唯一 id |
| 解构 store 后页面不更新 | 解构得到的是快照 | 用 `storeToRefs` 或 `toRefs` |
| 重新赋值 reactive 对象后失效 | 变量指向了新普通对象 | `Object.assign` 改原对象，或改用 `ref` |
| 直接改第三方图表实例的配置无效 | 被 `shallowRef` 包住，深层没被代理 | 替换 `.value` 或调用 `triggerRef` |
| 控制台报 Maximum recursive updates exceeded | 在 effect / watch 里改了自身依赖的数据 | 拆出独立状态或用 `nextTick` 跳出本轮 |
| 模板里 ref 忘记写 `.value` | 仅在模板顶层自动解包 | 脚本内一律显式写 `.value` |

## 7. React 基础

### 7.1 组件是一个返回 UI 描述的函数

React（React，Meta 维护的用于构建用户界面的 JavaScript 库）并不直接操作 DOM。它要求你把界面写成「在给定数据下应该长什么样」的函数，再由 React 把这份描述同步到浏览器。这份描述叫元素（Element，一个只记录类型和属性的普通 JavaScript 对象），它不含真实 DOM 节点。

```jsx
// 函数组件：输入 props，输出一段 UI 描述
function Welcome(props) {
  return <h1>你好，{props.name}</h1>;
}

// 等价的底层写法，看清 JSX 编译后到底是什么
const same = React.createElement('h1', null, '你好，', props.name);
```

JSX 只是 `React.createElement` 的语法糖。浏览器不认识尖括号，所以必须经过编译（Vite 项目默认由 @vitejs/plugin-react 用 Babel/SWC 完成）。记住这一点，「为什么这里要花括号」「为什么不能写 if 语句」这类问题都可以自己推导出来。

一个 React 18 应用的最小启动代码：

```jsx
// main.jsx
import { createRoot } from 'react-dom/client';
import App from './App';

const root = createRoot(document.getElementById('root')); // 先拿根容器
root.render(<App />);                                     // 再渲染整棵树
```

React 18 之前是 `ReactDOM.render(<App />, container)`。18 起必须先 createRoot 再 render，只有走新 API 才能启用并发特性与完整的自动批处理。

这种风格叫声明式（Declarative，你只描述目标状态，由框架算出最小的变更步骤）。与之相对的是命令式（Imperative，你亲自查找节点并逐步修改）。同样是把计数显示到页面：

```js
// 命令式：每一步都要自己算，状态散落在 DOM 里，改需求就要重写流程
const el = document.querySelector('#count');
el.textContent = String(n);

// 声明式：描述「n 是几就该显示几」，n 变化后由 React 重新执行并比对差异
return <span>{n}</span>;
```

声明式带来的直接收益是状态与视图一一对应：界面对不上，一定是某个 state 不对，而不是「某次 DOM 操作漏了」。代价是所有渲染逻辑必须是纯的——同一个输入必须得到同一份输出，不能在渲染过程中改 state、发请求、写全局变量。这条约束在整个 React 体系里反复出现，也是后面排查「渲染两次」「死循环」类问题的判据。

### 7.2 JSX 的七条硬规则

JSX 不是模板语言，是表达式的方言，编译器对写法有硬性约束。

| 规则 | 错误写法 | 正确写法 |
| --- | --- | --- |
| 必须有唯一根节点 | `return <h1/><p/>` | `return <><h1/><p/></>` |
| 标签必须闭合 | `<img src={x}>` | `<img src={x} />` |
| 属性用小驼峰 | `class="a"` / `for="b"` | `className="a"` / `htmlFor="b"` |
| 只能插入表达式 | `{if (ok) {}}` | `{ok ? <A/> : null}` |
| 属性值二选一 | `className="{'a'}"` | `className={'a'}` 或 `className="a"` |
| 注释放在花括号里 | `<!-- 注释 -->` | `{/* 注释 */}` |
| style 传对象 | `style="color:red"` | `style={{ color: 'red' }}` |

```jsx
function Profile({ user, tags }) {
  const cls = user.vip ? 'card card--vip' : 'card'; // 类名用表达式拼
  return (
    <>
      {/* Fragment 当根节点，不会产生多余的 DOM 层级 */}
      <div className={cls}>
        <img src={user.avatar} alt={user.name} />
        <h2>{user.name}</h2>
        <p style={{ marginTop: 8, color: '#666' }}>{/* 属性名同样小驼峰 */}
          {tags.length > 0 ? tags.join('、') : '暂无标签'}
        </p>
      </div>
      <label htmlFor="bio">简介</label>
      <input id="bio" defaultValue={user.bio} disabled />
    </>
  );
}
```

花括号内只接受表达式（有返回值的东西）：`{1 + 1}`、`{cond ? a : b}`、`{list.map(fn)}` 都合法；`{for (...)}`、`{const x = 1}` 不合法。需要写语句时，把逻辑提到 return 之前。布尔属性直接写属性名即可，`disabled` 等价于 `disabled={true}`。注意 `value` 与 `defaultValue` 的区别：前者把输入框交给 React 控制，后者只是 HTML 初始值。

条件分支复杂时，不要把它们塞进 JSX。JSX 里嵌套三层三元表达式，几乎没人能一次读懂，正确做法是在 return 之前用 if 算出要渲染的节点：

```jsx
function Panel({ status, data }) {
  if (status === 'loading') return <Spinner />;   // 提前返回，JSX 保持扁平
  if (status === 'error') return <ErrorTip />;
  if (!data) return null;                          // 返回 null 表示什么都不渲染
  return <Chart data={data} />;
}
```

`null` 与空 Fragment `<> </>` 的区别是：`null` 不产生任何节点，也不占位；Fragment 只是省略包裹元素，本身不生成 DOM。列表项里返回 `null` 会保留位置语义，而返回 Fragment 会多一层合并逻辑。

### 7.3 函数组件与 props

组件名首字母必须大写。小写开头会被当成原生标签名，React 直接把 `<card />` 渲染成未知 HTML 元素而不报错，这类问题最难查。

props（属性，父组件传给子组件的只读数据）是单向流动的：父传子靠 props，子通知父靠回调函数。

```jsx
// 解构 + 默认值，是当下最简洁的声明方式
function Card({ title, footer = null, children }) {
  return (
    <section className="card">
      <header>{title}</header>
      <div className="card__body">{children}</div>
      {footer}
    </section>
  );
}

// 标签之间的内容自动收集进 children
<Card title="订单详情" footer={<button>关闭</button>}>
  <p>共 3 件商品</p>
</Card>
```

三条必须遵守的约定：

1. props 只读。子组件里写 `props.title = 'x'` 不会生效，严格模式下会直接报错。需要改变就调用父组件传下来的回调，让父组件去改自己的 state。
2. 不要整体展开未知 props 到 DOM。`<div {...props} />` 会把 `onClick`、`data-*` 之外的自定义属性也塞进 DOM，控制台出现 unknown prop 警告。要透传就显式挑字段。
3. 组件复用靠把可变部分提成 props。同一个组件里写死「按钮文案」「接口地址」，就无法复用；把它们参数化，组件才有价值。

`children` 是特殊 props，类型可以是字符串、元素、数组甚至函数（渲染属性模式）。它让组件具备「外壳固定、内容自定义」的能力，是布局类组件的主要实现手段。

还有一个容易忽略的事实：函数组件的函数体会在**每次渲染时重新执行**。父子组件之间，父组件重渲染默认会连带子组件重渲染；子组件自身 state 变化也会重渲染。于是组件体内任何一句代码都会被反复执行，包括 `const config = {...}` 这种对象字面量——它每次都是新引用，这也解释了为什么把对象当 prop 传下去容易让子组件无谓重渲染。相对的结论是：组件体内只做「由 props 和 state 算出视图」的计算，发请求、订阅事件、改 DOM 这类副作用都交给专门的钩子处理。

### 7.4 state 与批处理：一次渲染一个快照

state（组件状态，组件内部会随交互变化、变化后触发重新渲染的数据）是组件自己的记忆。函数组件用 `useState` 声明：

```jsx
const [count, setCount] = useState(0); // 返回当前值与修改函数
```

关键认知是：**每一次渲染都是一张快照**。渲染函数执行期间读到的 `count` 是一个常量，本次渲染内不会变。所以下面这段代码的行为和直觉不同：

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  function wrong() {
    setCount(count + 1);
    setCount(count + 1);   // 两次都基于同一快照的 count，结果只加 1
    console.log(count);    // 打印旧值：setState 不会改写当前这次渲染的变量
  }

  function right() {
    setCount(c => c + 1);  // 函数式更新，参数是最新待提交的值
    setCount(c => c + 1);  // 两次叠加，结果加 2
  }

  return <button onClick={right}>{count}</button>;
}
```

批处理（Batching，把同一次事件中产生的多次状态更新合并成一次渲染）是 React 的性能基础。React 18 起，事件回调、Promise、setTimeout 里的更新都会自动批处理；React 17 及以前只有 React 事件处理器内部批处理，`setTimeout` 中两次 set 会渲染两次。所以「在定时器里连续 set 会闪两下」这类现象，先确认 React 版本。

更新对象与数组时必须换新引用，因为 React 用 `Object.is` 比较前后值，引用没变就认为没变化并跳过渲染：

```jsx
// 错误：原地改，引用不变，界面不更新
user.name = '李四';
setUser(user);

// 正确：创建新对象
setUser({ ...user, name: '李四' });

// 数组的增、删、改，都产出新数组
setList([...list, item]);                                        // 增
setList(list.filter(x => x.id !== id));                          // 删
setList(list.map(x => x.id === id ? { ...x, done: true } : x));  // 改某一项
```

状态组织有一条经验：互不相关的字段拆成多个 `useState`，会一起变化的字段合成一个对象。这样更新时只需复制一小块，不会因为一个字段变化就让整个大对象换引用。

`useState` 的初始值是每次渲染都会求值的表达式，如果它很贵（读 localStorage、解析 JSON、遍历数组），要改成惰性初始化，传一个函数进去，React 只在首次渲染时调用一次：

```jsx
const [list, setList] = useState(() => JSON.parse(localStorage.getItem('list')) || []);
```

另外，可推导的数据不要放进 state。`total = list.reduce(...)`、`filtered = list.filter(...)`、`isVip = user.level > 3` 这类值都能由现有的 state 当场算出，多存一份就意味着多一个需要同步的地方，一旦漏同步就会出现「列表删了但总数没变」的 bug。真正该进 state 的只有两类：用户输入的值，以及需要跨越多次渲染保留的值。

### 7.5 事件系统与受控表单

React 用合成事件（SyntheticEvent，对各浏览器原生事件做统一封装、提供一致属性的对象）处理交互，它抹平了浏览器差异，也让事件可以跨平台复用。

事件写法的四个要点：

- 事件名用小驼峰：`onClick`、`onChange`、`onKeyDown`。
- 传函数引用，不传字符串：`onClick={handleClick}`，而不是 `onClick="handleClick()"`。
- 阻止默认行为用 `e.preventDefault()`。`return false` 在 React 里无效，这是从原生 DOM 思维迁移过来最常见的错。
- 需要传参时包一层箭头函数：`onClick={() => del(item.id)}`。写成 `onClick={del(item.id)}` 会在渲染时立刻执行，导致无限循环。

从 React 17 起，事件不再统一委托到 `document`，而是委托到挂载的根容器。这让多个 React 版本共存、微前端嵌套场景下的行为更可预期，也意味着在根容器之外手动派发的事件不会再被捕获。

受控组件（Controlled Component，值由 React state 驱动、用户输入必须经过 onChange 回写 state 的表单元素）是 React 表单的默认做法：

```jsx
function LoginForm({ onSubmit }) {
  const [form, setForm] = useState({ user: '', pwd: '' });

  function handleChange(e) {
    const { name, value } = e.target;           // 用 name 复用同一个处理器
    setForm(prev => ({ ...prev, [name]: value }));
  }

  function handleSubmit(e) {
    e.preventDefault();                         // 不阻止就会整页刷新，state 全丢
    onSubmit(form);
  }

  return (
    <form onSubmit={handleSubmit}>
      <input name="user" value={form.user} onChange={handleChange} />
      <input name="pwd" type="password" value={form.pwd} onChange={handleChange} />
      <button type="submit">登录</button>
    </form>
  );
}
```

受控组件必须同时给 `value` 和 `onChange`。只给 `value` 会让输入框变成只读，控制台给出警告；这是排查「输入框打不进字」的第一检查点。

| 维度 | 受控组件 | 非受控组件 |
| --- | --- | --- |
| 数据存放 | React state | DOM 节点自身 |
| 取值方式 | 直接读 state | ref.current.value |
| 初始值 | `value` + `onChange` | `defaultValue` / `defaultChecked` |
| 实时校验、联动 | 天然支持 | 需手动监听 |
| 适用场景 | 绝大多数业务表单 | 文件上传、超简单表单、非 React 库集成 |

受控组件在字段很多的大表单上有明显开销：每敲一个字符都会触发一次重渲染，如果父组件里还带着长列表，输入就会卡。可行的缓解手段是把表单区块独立成子组件，让 state 只影响这一小块；或者对长列表使用虚拟滚动。另一条底线是不要在 `onChange` 里做请求，输入框每变一次就发一次搜索请求，既打爆接口也打乱结果顺序，正确做法是加防抖（Debounce，等用户停止输入一段时间后才真正触发）。

合成事件对象在早期版本里会被回收复用，事件回调中如果异步访问 `e.target`，需要先 `e.persist()`，或者干脆在回调开头把需要的值取出来存到局部变量：

```jsx
function handleChange(e) {
  const value = e.target.value;   // 先取值，后续异步逻辑只依赖这个字符串
  setTimeout(() => search(value), 300);
}
```

`e.target` 与 `e.currentTarget` 也常被混淆：前者是实际触发事件的元素（可能是子节点），后者是绑定了处理器的那个元素。做事件委托时取错一个，拿到的就是子元素的属性，表现为「点按钮却读到了按钮里 span 的 textContent」。

### 7.6 条件渲染与列表渲染

条件渲染有三种写法，按可读性选择：

```jsx
{isLoading && <Spinner />}              // 短路：左边为真才渲染
{count > 0 && <Badge n={count} />}      // 用比较运算，保证左边是布尔值
{isVip ? <VipPanel /> : <NormalPanel />} // 二选一
{list.length === 0 ? <Empty /> : <List data={list} />}
```

第一个写法有著名陷阱：如果左边是数字 `0`，`0 && <X/>` 的结果是 `0`，React 会把 0 渲染到页面上，界面上莫名多出一个「0」。所以用 `&&` 时保证左边是布尔值，比如 `count > 0`，或写成 `Boolean(count) && ...`。

列表渲染用 `map` 返回元素数组，每个元素必须带 key（React 用来在两次渲染之间识别同一列表项、从而复用其 DOM 与组件状态的标识）。key 只需在兄弟节点之间唯一，不必全局唯一，也不作为 props 传给子组件。

```jsx
// 反例：用下标当 key。删除第 1 项后，原第 2 项的下标变成 0，
// React 认为是「同一个节点内容变了」，于是复用了旧节点及其内部状态
{todos.map((t, i) => <Row key={i} data={t} />)}

// 正例：用业务主键
{todos.map(t => <Row key={t.id} data={t} />)}
```

用下标作 key 的后果很具体：列表项里有输入框时，删除中间一项会导致下方输入框的内容错位；列表项有展开/选中状态时会串位；带动画的列表会闪。下标作 key 只在「列表静态、不排序、不增删、项内无状态」时可以接受。

key 还有第二个用途：主动改变 key 就能强制 React 卸载并重新挂载该组件，用它重置内部状态非常有效。比如切换用户时希望详情面板的表单清空，只要把用户 id 接到 key 上（`<Profile key={user.id} user={user} />`），换人时组件整棵重建，内部 state 与输入框内容一并归零，比在组件里逐个字段手动清空可靠得多。

另一个常见缺失是 `map` 里忘了 `return`：

```jsx
{list.map(t => { <Row data={t} /> })}         // 错误：箭头函数体用了大括号却没 return
{list.map(t => <Row data={t} />)}             // 正确：简写返回
{list.map(t => { return <Row key={t.id} data={t} />; })} // 正确：显式 return
```

undefined 被 `map` 出来后，React 会静默跳过，界面一片空白但不报错，排查时先看渲染函数返回值。

列表与筛选条件组合时，把链式处理提到 return 之前，写成一个有名字的变量，既避免 JSX 里塞一长串方法调用，也方便在它前面加一行 `console.log` 定位问题：

```jsx
function UserList({ users, keyword, onlyActive }) {
  const shown = users
    .filter(u => (onlyActive ? u.active : true))
    .filter(u => u.name.includes(keyword));
  if (!shown.length) return <p>没有匹配的用户</p>;
  return <ul>{shown.map(u => <li key={u.id}>{u.name}</li>)}</ul>;
}
```

注意不要为了「优化」在渲染过程中修改源数组（`users.sort()` 会原地排序并改变 props），需要排序就写 `[...users].sort(...)`。

### 7.7 样式方案怎么选

| 方案 | 写法 | 隔离性 | 动态样式 | 主要代价 |
| --- | --- | --- | --- | --- |
| 全局 CSS | `className="card"` | 靠 BEM 约定 | 弱 | 命名冲突、死代码难清 |
| CSS Module | `className={s.card}` | 编译期加哈希，强 | 中 | 需要构建支持，跨文件复用类名麻烦 |
| CSS-in-JS | `styled.div\`\`` | 组件级，强 | 强 | 运行时开销、包体积 |
| 原子化 CSS | `className="flex gap-2"` | 天然无冲突 | 强 | 类名长、学习成本 |
| 内联 style | `style={{...}}` | 天然隔离 | 强 | 不支持伪类、媒体查询、动画 |

工程上最常见的组合是 CSS Module 或原子化 CSS 做基础样式，内联 style 只用于运行时才知道的数值（位置、宽度百分比）。

```jsx
// 条件类名：用数组过滤，比嵌套三元表达式可读得多
const cls = ['btn', size && `btn--${size}`, disabled && 'is-disabled']
  .filter(Boolean)
  .join(' ');

<div className={cls} style={{ width: `${pct}%` }} />
```

注意 `className` 只接受字符串，不能传数组或对象（Vue 允许数组语法）。从 Vue 转过来时写 `className={[a, b]}`，React 会把它转成 `"a,b"` 这样的字符串，样式静默失效。

内联 style 有两个细节值得记住：数值型的属性 React 会自动补 `px`（`{ width: 100 }` 等于 `width: 100px`），但 `lineHeight`、`opacity`、`zIndex`、`flex` 这类无单位属性例外；自定义 CSS 变量不能直接写 `--main-color` 这种键，要写成 `style={{ '--main-color': '#333' }}`，键名必须加引号。此外，把「哪个元素高亮」「弹窗开着还是关着」这类状态存进 style 对象再往下传，等于把状态藏在了样式里，调试时很难跟踪，这类开关应该单独作为布尔 state 或 props 存在。

### 7.8 组件拆分的三个信号

拆分不是按行数切的，出现下面任一信号就该拆：

1. **重复**：同样的结构在第二处出现，抽成组件。
2. **状态归属不清**：多个兄弟组件需要同一份数据，用状态提升（Lifting State Up，把数据放到最近的共同父组件，由父组件持有并向下传值和回调）。
3. **渲染范围过大**：一处小改动导致整块区域重渲染，拆细后才好做局部优化。

```jsx
// 容器组件：只管数据与行为
function TodoPage() {
  const [list, setList] = useState([]);
  const remove = id => setList(l => l.filter(x => x.id !== id));
  return <TodoList list={list} onRemove={remove} />;
}

// 展示组件：只负责渲染，入参确定则输出确定，便于单测与复用
function TodoList({ list, onRemove }) {
  if (!list.length) return <p>暂无待办</p>;
  return <ul>{list.map(t => <TodoItem key={t.id} item={t} onRemove={onRemove} />)}</ul>;
}

function TodoItem({ item, onRemove }) {
  return (
    <li>
      {item.text}
      <button onClick={() => onRemove(item.id)}>删除</button>
    </li>
  );
}
```

对应的目录组织，按功能而非按文件类型分组，改一个需求只在一个目录里动：

```text
src/
  components/     # 跨页面复用的通用组件（Button、Modal）
  features/
    todo/         # 一个业务域的组件、hooks、请求、样式放在一起
      TodoPage.jsx
      TodoList.jsx
      useTodos.js
  hooks/          # 通用自定义 Hook
  layouts/        # 页面骨架
  pages/          # 路由级页面，只做组装
```

拆分也要防止过度。把每个 `<div>` 都做成组件，会出现两个新问题：props 钻透（Prop Drilling，数据从高层一路透传到深层，中间层只是转发）和状态放错层（把本可以局部持有的 state 提到顶层，导致一处输入整页重渲染）。判断标准很实用：一个组件如果只有一处使用、没有内部状态、props 不超过两个，就留在原地；只有当它开始被复用、或开始变得难以单独阅读时，才值得提取。中间层只是转发 props 超过两三层，就该考虑用组合（把节点作为 children 传下去）或上下文来打破链条。

### 7.9 常见坑速查

| 现象 | 根因 | 处理 |
| --- | --- | --- |
| 界面不更新，state 明明改了 | 原地修改对象/数组，引用没变 | 用展开、filter、map 产出新引用 |
| 输入框打不进字 | 只给了 `value` 没给 `onChange` | 补 onChange，或改用 `defaultValue` |
| 页面上莫名出现一个 0 | `0 && <X/>` 返回了 0 | 左边改成比较表达式或布尔值 |
| 删除列表项后内容错位 | 用下标当 key | 换成业务唯一 id |
| 点一次按钮执行了多次 | `onClick={fn(id)}` 渲染时即调用 | 包成箭头函数 `() => fn(id)` |
| 表单提交后整页刷新 | 忘了 `e.preventDefault()` | submit 处理器首行阻止默认行为 |
| `<card />` 什么都没渲染 | 组件名小写，被当原生标签 | 首字母大写 |
| setTimeout 里连续 set 渲染两次 | React 17 只批处理事件内更新 | 升级 18，或手动 `unstable_batchedUpdates` |

四条最该记住的规则：props 只读，改数据靠回调；state 用新引用替换旧引用；key 用稳定业务 id，不用下标；`children` 与 props 一样，是组件复用的入口而不是特例。

## 8. React Hooks

### 8.1 Hooks 是什么，为什么需要它

Hooks（钩子，让函数组件在不写 class 的前提下使用状态与生命周期能力的函数集合）是 React 16.8 引入的 API。在它出现之前，只有 class 组件能持有 state，于是有三类固定痛点反复出现：

- 逻辑复用只能靠高阶组件（Higher-Order Component，接收组件返回新组件的函数）与 render props，层层包裹后组件树变成嵌套地狱，某个字段到底从哪一层传进来极难追；
- 一个功能常被拆到 componentDidMount、componentDidUpdate、componentWillUnmount 三个钩子里，不相关的代码挤在同一个钩子，相关的代码隔了几十行；
- this 绑定的丢失、绑定顺序、生命周期调用的微妙差异，都和业务逻辑无关，却天天制造 bug。

Hooks 把「状态」和「副作用」变成可调用的函数：同一功能的声明、订阅、清理写在一起，组件逻辑还能抽成自定义 Hook 复用。这也是函数组件 + Hooks 成为 React 主流写法、类组件逐步退居兼容角色的原因。

| Hook | 作用 | 返回值 | 典型场景 |
| --- | --- | --- | --- |
| useState | 声明一个参与渲染的状态 | `[值, 设置函数]` | 计数器、表单输入、开关 |
| useEffect | 渲染提交后执行副作用，可返回清理函数 | 无 | 请求数据、订阅事件、定时器 |
| useMemo | 缓存计算结果 | 缓存的值 | 上万条数据的过滤排序 |
| useCallback | 缓存函数引用 | 缓存的函数 | 传给 memo 子组件的回调 |
| useRef | 跨渲染保存可变对象或 DOM 引用 | `{ current }` | 定时器 id、输入框聚焦 |
| useContext | 读取上层 Context 的值 | Context 当前值 | 主题、登录用户、购物车 |
| useReducer | 用 reducer 管理复杂状态 | `[state, dispatch]` | 多字段关联的表单、状态机 |

使用 Hooks 有两条硬规则（Rules of Hooks，React 官方规定的调用约束）：

1. 只在函数组件或自定义 Hook 的顶层调用，不能放进 if、for、while、try，也不能放在回调里；
2. 只在 React 函数中调用，普通工具函数里不能调用。

原因不是风格洁癖。React 判断「这是第几个 Hook」完全依赖调用顺序，内部用链表按顺序存放每个 Hook 的状态槽位，没有名字可用。首次渲染顺序是 useState → useEffect，后续每次必须完全一致；某个分支少调用一次，后面所有 Hook 就整体错位，读到隔壁槽位的数据，表现为报错或串值。

```jsx
// 反例：条件调用 Hook，挂载与更新时调用个数不一致
function Bad({ show }) {
  if (show) {                 // ❌ 顺序不稳定
    const [n, setN] = useState(0);
  }
  return null;
}

// 正例：Hook 无条件执行，条件只决定是否渲染
function Good({ show }) {
  const [n, setN] = useState(0);   // ✅ 每次渲染都调用
  return show ? <p>{n}</p> : null;
}
```

工程上靠 eslint-plugin-react-hooks 的两条规则兜住：rules-of-hooks 检查调用位置，exhaustive-deps 检查依赖数组完整性。把这两条规则设为 error，能拦掉大部分 Hooks 事故。

### 8.2 useState：状态快照与函数式更新

useState(initial) 返回当前值和设置函数。要点有四条。

惰性初始化：`useState(expensive())` 每次渲染都会执行 expensive，即使值随后被丢弃；写成 `useState(() => expensive())` 才只在首次渲染执行。

更新是替换而非合并：`setObj({ a: 1 })` 会丢掉 obj 原有的 b 字段，需要 `setObj(prev => ({ ...prev, a: 1 }))`。数组同理，push 不会触发更新，要 `setList(prev => [...prev, item])`。

一次渲染里 state 是常量快照：`setCount(count + 1)` 连写两次，两次读到的都是本次渲染的 count，结果只加 1。要基于上一次值计算，必须用函数式更新 `setCount(c => c + 1)`。

React 18 起默认自动批处理（Automatic Batching，同一事件回调或同一微任务内的多次 setState 合并为一次渲染），所以连着改三次不会渲染三次，但读取到的仍是旧快照。

```jsx
import { useState } from 'react';

function Counter() {
  // 惰性初始化：只在首次渲染读一次 localStorage
  const [count, setCount] = useState(() => Number(localStorage.getItem('count')) || 0);
  const [form, setForm] = useState({ name: '', age: 18 });

  const addTwiceWrong = () => {
    setCount(count + 1);      // 两次都基于本次渲染的 count
    setCount(count + 1);      // 最终只 +1
  };

  const addTwice = () => {
    setCount(c => c + 1);     // 函数式更新，基于最新值
    setCount(c => c + 1);     // 最终 +2
  };

  // 对象状态必须展开旧值，否则字段被覆盖
  const rename = (name) => setForm(prev => ({ ...prev, name }));

  return (
    <div>
      <button onClick={addTwice}>{count}</button>
      <input value={form.name} onChange={e => rename(e.target.value)} />
    </div>
  );
}
```

### 8.3 useEffect：依赖、清理与执行时机

useEffect(setup, deps) 在组件渲染完成、DOM 更新、浏览器绘制之后执行，是异步的，不阻塞绘制。setup 可以返回一个清理函数（cleanup），React 会在下一次 effect 执行前、以及组件卸载时调用它。

依赖数组决定执行时机，三种写法含义完全不同：

| 写法 | 执行时机 | 适用场景 | 风险 |
| --- | --- | --- | --- |
| 不传第二个参数 | 每次渲染后都执行 | 几乎不用，调试用 | 内部 setState 时无限循环 |
| `[]` | 仅挂载后执行一次 | 一次性订阅、初始化 | 读了外部变量会拿到旧值 |
| `[a, b]` | 挂载后 + a 或 b 变化后 | 跟随某个数据重新同步 | 漏项导致读到旧数据 |

清理函数不是可选项。定时器、事件监听、WebSocket、未完成的请求，少了清理就是内存泄漏，外加组件卸载后 setState 的警告。请求还有另一类问题：竞态（Race Condition，先发出的慢请求后返回，把后发出的新结果覆盖掉）。用户快速切换筛选条件时，这个问题几乎必然出现。

```jsx
import { useEffect, useState } from 'react';

function List({ query }) {
  const [list, setList] = useState([]);
  const [loading, setLoading] = useState(false);
  const [tick, setTick] = useState(0);

  // 定时器：卸载或依赖变化时清掉，避免泄漏
  useEffect(() => {
    const id = setInterval(() => setTick(t => t + 1), 1000);
    return () => clearInterval(id);
  }, []);

  // 请求竞态：每次 effect 有自己的 ignore 标志
  useEffect(() => {
    let ignore = false;
    setLoading(true);
    fetch(`/api/list?q=${encodeURIComponent(query)}`)
      .then(r => r.json())
      .then(data => {
        if (ignore) return;        // 已作废的响应直接丢弃
        setList(data);
        setLoading(false);
      });
    return () => { ignore = true; };   // query 变化时把上一次标记作废
  }, [query]);

  return <ul>{list.map(x => <li key={x.id}>{x.name}</li>)}</ul>;
}
```

两条纪律。其一，不要用 effect 同步能直接算出来的派生状态：`const total = items.length` 就该写在渲染中，写进 state 再用 effect 更新会让一次变更变成两轮渲染，还可能闪一下。其二，不要用 effect 响应「用户点击」这类明确事件，逻辑放在事件处理函数里，effect 只处理「与外部系统同步」的事情。

执行时机上还有 useLayoutEffect：它在 DOM 变更后、浏览器绘制前同步执行，会阻塞绘制。测量元素尺寸、修改滚动位置、需要避免闪烁的场景用它；数据请求和订阅一律用 useEffect。

| 对比项 | useEffect | useLayoutEffect |
| --- | --- | --- |
| 执行时刻 | 浏览器绘制之后 | DOM 更新后、绘制之前 |
| 是否阻塞绘制 | 否 | 是，长任务会卡住画面 |
| 典型用途 | 请求、订阅、上报 | 测量布局、滚动定位、防止闪烁 |
| 服务端渲染 | 不执行，安全 | 会告警，需规避 |

### 8.4 useMemo 与 useCallback：缓存的是引用

useMemo(fn, deps) 记住计算结果，依赖不变就复用上次的值；useCallback(fn, deps) 记住函数引用，等价于 `useMemo(() => fn, deps)`。

两者都不是「越多越好」的优化。依赖比较本身有成本，依赖每轮渲染都变化时缓存完全失效，还多占一份内存。真正值得用的场景只有三种：计算确实昂贵；结果要作为其他 Hook 的依赖；函数引用要传给 React.memo 子组件，避免因为「每次都是新函数」而让它白 memo。

```jsx
import { memo, useCallback, useMemo, useState } from 'react';

const Child = memo(function Child({ onSelect }) {
  console.log('Child 渲染');
  return <button onClick={() => onSelect(1)}>选择</button>;
});

function Table({ list, keyword }) {
  const [selected, setSelected] = useState(null);

  // 昂贵计算：只有 list 或 keyword 变化才重新过滤
  const visible = useMemo(
    () => list.filter(x => x.name.includes(keyword)),
    [list, keyword]
  );

  // 引用稳定：依赖为空，onSelect 永远是这个函数
  const handleSelect = useCallback((id) => setSelected(id), []);

  return (
    <>
      <ul>{visible.map(x => <li key={x.id}>{x.name}</li>)}</ul>
      <Child onSelect={handleSelect} />
    </>
  );
}
```

常见反例：用 useMemo 包一个字符串拼接或一次加法，收益远小于比较依赖的开销；用 useCallback 包一个依赖里有新建对象字面量或内联箭头函数的函数，缓存每次都命中不了；把 `onClick={() => doSomething(id)}` 写成 useCallback 却没把 id 放进依赖，回调里拿到过期 id。

### 8.5 useRef：不触发渲染的那一格抽屉

useRef(initial) 返回一个 `{ current }` 对象，这个对象在组件整个生命周期内保持同一引用。两个典型用途：保存与渲染无关的可变值（定时器 id、上一次的值、渲染次数），以及获取 DOM 节点。

关键特性：修改 ref.current 不会触发重渲染。因此它不能用来驱动 UI，适合做「渲染过程之外的小账本」。另外不要在渲染期间写 ref，React 的并发渲染可能打断或重放渲染，那时写 ref 会产生不可预期的值。

| 维度 | useState | useRef |
| --- | --- | --- |
| 修改后是否重渲染 | 是 | 否 |
| 渲染中读到的值 | 该次渲染的快照 | 始终是最新值 |
| 是否要求不可变 | 是，必须新建对象 | 否，可随意赋值 |
| 典型用途 | 界面要跟着变的数据 | 定时器 id、DOM 节点、上一轮值 |

```jsx
import { useEffect, useRef } from 'react';

function SearchBox({ onSearch }) {
  const inputRef = useRef(null);     // DOM 引用
  const timerRef = useRef(null);     // 防抖定时器 id，不需要参与渲染

  useEffect(() => {
    inputRef.current?.focus();       // 挂载后自动聚焦
  }, []);

  const onChange = (e) => {
    const value = e.target.value;
    clearTimeout(timerRef.current);
    timerRef.current = setTimeout(() => onSearch(value), 300);
  };

  return <input ref={inputRef} onChange={onChange} placeholder="搜索" />;
}
```

### 8.6 useContext 与 useReducer：跨层传值与集中式状态

Context（上下文，允许数据跳过中间层直接送达子树任意深度的机制）解决的是 props 逐层透传（prop drilling）。三步：createContext 建容器，Provider 提供值，useContext 在任意后代读取。要点是 Provider 的 value 若每次渲染都新建对象，所有消费者都会跟着重渲染，所以要用 useMemo 包住 value，或按更新频率拆成多个 Context。

useReducer(reducer, initialState) 返回 `[state, dispatch]`，把状态更新收敛到一个纯函数里。适合：多个字段互相关联；下一个状态依赖上一个状态且分支多；希望状态变更可单测、可打印日志。

```jsx
import { createContext, useContext, useMemo, useReducer } from 'react';

const CartContext = createContext(null);

function cartReducer(state, action) {
  switch (action.type) {
    case 'add':
      return { ...state, items: [...state.items, action.item] };
    case 'remove':
      return { ...state, items: state.items.filter(i => i.id !== action.id) };
    case 'clear':
      return { ...state, items: [], total: 0 };
    default:
      throw new Error(`未知 action: ${action.type}`);  // 拼写错误立刻暴露
  }
}

export function CartProvider({ children }) {
  const [state, dispatch] = useReducer(cartReducer, { items: [], total: 0 });
  const value = useMemo(() => ({ state, dispatch }), [state]); // 引用稳定
  return <CartContext.Provider value={value}>{children}</CartContext.Provider>;
}

export function CartBadge() {
  const { state } = useContext(CartContext);
  return <span>已选 {state.items.length} 件</span>;
}
```

选择依据可以固化成一张表。单个布尔值、单个输入框用 useState 更直接；一个表单十几个字段、字段之间有联动和校验时，useReducer 的 reducer 更好读也更好测。

| 维度 | useState | useReducer |
| --- | --- | --- |
| 状态结构 | 单个值或少量独立值 | 多字段互相关联的对象 |
| 更新逻辑 | 一两行直接替换 | 多分支、带校验、多步骤 |
| 复用与测试 | 逻辑散落在组件里 | reducer 是纯函数，可单独单测 |
| 组合方式 | 配合 Context 或状态库 | 常与 Context 组合做局部状态管理 |
| 调试 | 打点看 setState 调用 | 打印 action 序列即可还原过程 |

### 8.7 自定义 Hook：把有状态逻辑抽出来

自定义 Hook（Custom Hook，以 use 开头、内部调用其他 Hook、返回状态与操作函数的普通函数）是 Hooks 最大的复用红利。命名必须以 use 开头，这不是约定俗成而是功能要求：lint 靠它识别，规则校验才生效。

返回形态有惯例：位置固定、需要重命名时返回数组（`[value, setValue]`）；字段多、按需取用时返回对象（`{ data, loading, error, refetch }`）。

```jsx
import { useEffect, useState } from 'react';

// 防抖：value 停止变化 ms 毫秒后，返回的副本才跟上
export function useDebounce(value, ms = 300) {
  const [debounced, setDebounced] = useState(value);
  useEffect(() => {
    const id = setTimeout(() => setDebounced(value), ms);
    return () => clearTimeout(id);   // value 再次变化时取消上一个定时器
  }, [value, ms]);
  return debounced;
}

// 使用：输入框即时响应，请求按防抖后的值发
function Search() {
  const [keyword, setKeyword] = useState('');
  const debounced = useDebounce(keyword, 500);
  useEffect(() => {
    if (!debounced) return;
    fetch(`/api/search?q=${encodeURIComponent(debounced)}`)
      .then(r => r.json()).then(console.log);
  }, [debounced]);
  return <input value={keyword} onChange={e => setKeyword(e.target.value)} />;
}
```

要清楚边界：两个组件调用同一个自定义 Hook，各自拿到独立状态，不会共享数据；共享数据要靠 Context 或状态库。自定义 Hook 每次渲染都会重新执行，里面的昂贵计算同样需要 useMemo，否则只是把复杂度从一个组件搬到另一个函数。

### 8.8 闭包陷阱、依赖缺失与 StrictMode 双调用

闭包陷阱（Stale Closure，回调捕获了某次渲染的变量，之后一直读旧值）是 Hooks 最高频的 bug，表现形式很迷惑：定时器里打印的永远是最初的数字，effect 里拿到的永远是首次的 props。

```jsx
function Demo() {
  const [count, setCount] = useState(0);
  const onClick = () => {
    // 这里的 count 是「点击那一次渲染」的 count
    setTimeout(() => alert(count), 1000);
  };
  return <button onClick={() => { onClick(); setCount(count + 1); }}>点我</button>;
}
```

三种解法，按优先级选一种，不要混用：函数式更新 `setCount(c => c + 1)`，彻底不读外部值；把 count 放进依赖数组，让回调随值重建；确实不能在依赖里列出时，用 ref 保存最新值（常抽成 useLatest）。

依赖缺失的排错顺序可以固定下来：

1. 打开 exhaustive-deps 的告警，不要用 `// eslint-disable-next-line` 糊过去，先读它提示缺哪几项；
2. 把缺的每项都加进依赖数组，观察是否出现死循环；
3. 若加了对象或函数才循环，说明它们在渲染中被新建，用 useMemo、useCallback 或移到组件外定义；
4. 若某项逻辑上不该触发重跑，改用 ref 或函数式更新，而不是删依赖；
5. 用 React DevTools 的 Profiler 看是否出现「渲染 → effect 里 setState → 再渲染」的抖动，这就是依赖写错的典型指纹。

StrictMode 双调用是另一类高频困惑。React 18 开发模式下，StrictMode 会故意执行 mount → unmount → mount，重复挂载一次组件，专门用来暴露缺失清理函数的 effect。现象是接口请求两次、日志打两遍，生产构建不会重复。

```jsx
// 开发环境：effect 会执行两次（挂载 → 清理 → 再挂载）
<React.StrictMode>
  <App />
</React.StrictMode>
```

正确态度是把 effect 写成可重复执行的：每次都能从零建立、并在清理里彻底还原。用「加一个 ref 标志跳过第二次」来掩盖，等于把真实缺陷藏起来，上线后依然会在依赖变化时暴露。

### 8.9 速查表与常见坑

| 需求 | 该用 | 不该用 |
| --- | --- | --- |
| UI 要随数据变化 | useState | useRef |
| 请求、订阅、定时器 | useEffect + 清理函数 | 渲染期间直接副作用 |
| 缓存昂贵计算 | useMemo + 稳定的依赖 | 给每次都在变的值加缓存 |
| 稳住传给 memo 子组件的回调 | useCallback + 完整依赖 | useCallback + 空依赖读外部变量 |
| 跨层提供全局值 | Context + memo 化 value | 逐层透传 props |
| 复杂关联状态 | useReducer | 十几个 useState 互相 set |
| 复用有状态逻辑 | 自定义 useXxx Hook | 复制粘贴 effect 代码 |

常见坑：

- 在 if、循环、回调里调用 Hook，报 "Rendered more hooks than during the previous render"，根因是调用顺序不稳定，必须把 Hook 提到顶层。
- 依赖数组写 `[]` 却在里面读 props 或 state，拿到的是首次渲染的旧值；把用到的每个值都列进依赖，或用函数式更新规避。
- 在渲染期间直接 setState 或写 ref，前者造成无限循环或 "Too many re-renders"，后者在并发模式下读到脏值；副作用只放在 effect 或事件处理函数里。
- effect 里发请求却不清理，快速切换条件时结果错乱；用 ignore 标志或 AbortController 丢弃过期响应，并在卸载时终止。

## 9. React 进阶与状态管理

### 9.1 重渲染不等于改 DOM：渲染、协调与提交

React 把一次更新切成两个阶段。渲染阶段（Render Phase，重新执行组件函数、算出新的元素树）是纯计算，可以被中断、丢弃、重放；提交阶段（Commit Phase，把差异一次性写入真实 DOM）不可中断。真正决定"改哪几个节点"的那步叫协调（Reconciliation，对比新旧元素树、按差异生成最小 DOM 操作的过程），React 内部用 Fiber（把每个组件与 DOM 节点表示为可遍历工作单元的对象）承载这棵可中断的树。

由此有三条必须记住的结论：

1. 组件函数被调用，不代表 DOM 变了。父组件重渲染时子组件函数也会被调用，但若产出的元素与上次一致，协调阶段不会产生任何 DOM 操作。
2. 重渲染的成本主要来自你自己的代码：函数体里的排序过滤、对象构造、以及被牵连的整棵子树，而不是"调用函数"这个动作本身。
3. 只有渲染阶段保持纯（不写 ref、不改外部变量、不发请求），并发特性才敢放心打断它，否则中断会让副作用执行一半。

React 按类型与 key 判断能否复用已有实例：

| 判断条件 | React 的行为 | 直接后果 |
| --- | --- | --- |
| 类型相同（同一个组件函数或同一个标签） | 复用实例与 DOM，只更新变化的属性 | state 保留，DOM 不重建 |
| 类型不同（div 换成 section，或换成另一个组件函数） | 卸载旧子树、挂载新子树 | state 全部丢失，DOM 重建 |
| 同层列表 key 稳定且唯一 | 按 key 匹配并移动已有实例 | state 跟着数据走 |
| 同层列表缺 key 或 key 重复 | 退化为按位置匹配 | state 错位，控制台告警 |

最常见的"组件莫名被重建"来自在渲染期间定义组件：

```jsx
// ❌ 每次渲染都创建一个新的函数类型，React 视为类型不同，输入框内容被清空
function Panel() {
  const Editor = () => <input defaultValue="草稿" />;
  return <Editor />;
}

// ✅ 组件定义在模块作用域，类型引用始终相同，输入内容可以保留
const Editor = () => <input defaultValue="草稿" />;
function Panel() {
  return <Editor />;
}
```

同理，`<Comp />` 与 `Comp()` 不是一回事：后者只是普通函数调用，得到的元素没有走协调流程，Hooks 在其中调用会直接报错。

### 9.2 key 是身份标识，不是序号

key（React 给同层兄弟节点打的标签，只在该层兄弟之间有效，不会作为 props 传入组件）决定协调时"这个元素上次是哪一位"。它一旦选错，React 会把某个实例的 state 安到另一条数据上。

用数组下标当 key 会出两类事故。第一类是数据顺序变化：删除、插入、排序之后，下标与数据的对应关系整体错位，输入框里的内容、展开状态、动画进度都"留在位置上"而不是"跟着数据走"。第二类是纯渲染性能：列表头部插入一项，所有下标的 key 全部变化，React 只能一个一个更新文本，无法移动节点。

```jsx
// ❌ 下标当 key：删掉第一项后，原来第二项的输入内容会"留在第一行"
{list.map((item, i) => <Row key={i} item={item} />)}

// ✅ 业务 id 当 key：state 跟着数据行移动
{list.map(item => <Row key={item.id} item={item} />)}
```

| key 取值 | 稳定性 | 适用场景 |
| --- | --- | --- |
| 业务主键 id | 稳定且兄弟间唯一 | 绝大多数增删排序列表 |
| 联合字段（如 `${userId}-${roleId}`） | 稳定，需保证唯一 | 关联表数据 |
| 数组下标 index | 顺序一变即失配 | 纯静态展示、无内部 state、不增删 |
| 随机数 / `Date.now()` | 每次渲染都不同 | 应禁止，等同于主动重建整棵子树 |

key 还有两个实用技巧。其一，主动改 key 就是"强制重置组件"，例如表单在切换用户时应当清空：`<ProfileForm key={userId} />`。其二，key 只需在同一父节点下唯一，不同层级可以重复，把一个列表项的 key 复用到它的兄弟坐标里才会出问题。

### 9.3 重渲染的四种来源与 React.memo

组件重渲染只有四种触发方式：自身 state 或 reducer 变化；父组件重渲染（默认会连带子组件）；订阅的 Context 值变化；外部状态库通过 useSyncExternalStore 通知变化。React.memo（高阶组件，接收组件并返回一个对 props 做浅比较、相等则跳过渲染的新组件）只能拦住第二种。

浅比较（Shallow Compare，只逐项比较对象第一层的引用是否相同）意味着 `style={{ color: 'red' }}`、`onClick={() => ...}` 这类每次渲染都新建的值会让 memo 必然失效。所以 memo 通常要配合 useCallback、useMemo，或干脆换一种结构。

比"到处加 memo"更有效的三个做法是：状态下移（State Colocation，把频繁变化的 state 放进真正使用它的子组件）、children 提升（把重型子树作为 props 传进来，父组件重渲染时它的元素引用不变）、按更新频率拆分 Context。

```jsx
// ❌ 每次按键都让 ExpensiveTree 跟着重渲染
function Page() {
  const [text, setText] = useState('');
  return (
    <>
      <input value={text} onChange={e => setText(e.target.value)} />
      <ExpensiveTree />
    </>
  );
}

// ✅ children 提升：ExpensiveTree 的元素由 Page 创建，Layout 重渲染时它引用不变
function Layout({ children }) {
  const [text, setText] = useState('');
  return (
    <>
      <input value={text} onChange={e => setText(e.target.value)} />
      {children}
    </>
  );
}
function Page() {
  return <Layout><ExpensiveTree /></Layout>;
}
```

| 手段 | 解决什么 | 代价或风险 |
| --- | --- | --- |
| React.memo | 父组件重渲染牵连子组件 | props 含新对象或新函数即失效 |
| useCallback / useMemo | 让 memo 的判断能命中 | 依赖数组要人工维护 |
| 状态下移 | 一处小改动引发大范围重渲染 | 需要重新划组件边界 |
| children 提升 | 重型子树被父级 state 拖累 | 结构变绕，可读性下降 |
| 虚拟列表 | 一次渲染上万个 DOM 节点 | 引入额外依赖，需固定行高 |

前提是先测量。React DevTools 的 Profiler 面板能记录每次提交的耗时和重渲染原因；一个普通组件重渲染常在几十微秒量级，为它加 memo 的比较开销与维护成本可能更高。只有 Flamegraph 里明显偏黄的组件、或输入框出现可感知卡顿的位置，才值得优化。

### 9.4 Context 的能力边界

Context（上下文，让数据跳过中间层直接送达任意深度后代的机制）解决的是 props 逐层透传。但它没有选择器（selector）能力：Provider 的 value 一变，所有消费者都会重渲染，哪怕组件只用到其中一个字段。把整张业务表放进一个 Context，就等于给每个消费者装上"任何字段变动都重渲染"的开关。

三条控制手法：按变化频率拆成多个 Context；用 useMemo 稳定 value 的引用；把 dispatch 与 state 分成两个 Context——dispatch 的引用恒定，写入时只有读 state 的组件重渲染。

```jsx
import { createContext, useContext, useMemo, useReducer } from 'react';

const StateCtx = createContext(null);
const DispatchCtx = createContext(null);

export function StoreProvider({ children }) {
  const [state, dispatch] = useReducer(reducer, { todos: [], filter: 'all' });
  return (
    <StateCtx.Provider value={state}>
      <DispatchCtx.Provider value={dispatch}>{children}</DispatchCtx.Provider>
    </StateCtx.Provider>
  );
}

// 只写不读的组件走这条：dispatch 引用不变，永远不会因数据更新而重渲染
export const useTodoDispatch = () => useContext(DispatchCtx);
export const useTodos = () => useContext(StateCtx);
```

判断标准可以落成一句话：读多写少、整块消费、更新频率低（主题、登录用户、国际化文案）用 Context 很合适；出现"每秒都在变"或"只关心某一个字段"时，换成带 selector 的状态库。

### 9.5 先给状态分类，再选工具

选型走偏通常源于把所有状态塞进一个 store。实际上前端状态至少分四类，各用各的工具：

| 状态类型 | 例子 | 推荐工具 |
| --- | --- | --- |
| 服务端状态 | 列表接口数据、详情、分页结果 | TanStack Query、SWR、RTK Query |
| 客户端全局状态 | 登录用户、主题、跨页面购物车 | Zustand、Redux Toolkit、Jotai |
| URL 状态 | 筛选条件、页码、Tab、弹窗深链 | React Router 的 searchParams |
| 局部 UI 状态 | 输入框值、展开收起、hover | useState / useReducer |

常用工具横向对比：

| 方案 | 形态 | 重渲染粒度 | 样板代码 | 适合场景 |
| --- | --- | --- | --- | --- |
| Context + useReducer | React 内置 | 全部消费者 | 少 | 小应用、低频全局值 |
| Redux Toolkit | 单一 store + 中间件 + DevTools | useSelector 精确选择 | 中 | 中大型团队、需要时间旅行调试与严格约束 |
| Zustand | 极简 store，函数式订阅 | 按 selector 精确 | 极少 | 中小型应用的全局状态 |
| Jotai / 原子化 | 每个字段一个 atom | 按 atom 精确 | 少 | 状态高度分散、派生关系多 |
| TanStack Query | 请求缓存层 | 按 query key | 中 | 接口多、需要缓存与失效策略 |

先问三个问题：数据来自接口且别人也会改吗？只有一个组件用它吗？需要跨路由共享吗？三个都否，useState 就够；只有最后一个为是，Context 够；接口众多，再加一层请求库。请求库能省掉手写的 loading、error、重试、缓存失效、窗口聚焦重取，这部分代码往往比状态库本身更值得消灭。

### 9.6 Redux Toolkit：把约束写进工具

Redux 的三个概念：store 持有唯一的 state 树；action 是描述"发生了什么"的纯对象；reducer 是 `(state, action) => newState` 的纯函数。Redux Toolkit（官方推荐的写法封装，简写 RTK）用 createSlice 把 action type、action creator 和 reducer 一次生成，并用 Immer 让你可以"直接改" state 而由库负责生成新对象。

```js
// store/todosSlice.js
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';

export const fetchTodos = createAsyncThunk('todos/fetch', async (_, { rejectWithValue }) => {
  const res = await fetch('/api/todos');
  if (!res.ok) return rejectWithValue(`HTTP ${res.status}`);
  return res.json();
});

const slice = createSlice({
  name: 'todos',
  initialState: { list: [], status: 'idle', error: null },
  reducers: {
    added(state, action) {
      state.list.push(action.payload);        // Immer 内部生成新对象，可以直接改
    },
    toggled(state, action) {
      const t = state.list.find(x => x.id === action.payload);
      if (t) t.done = !t.done;
    },
  },
  extraReducers: (builder) => {
    builder
      .addCase(fetchTodos.pending, (state) => { state.status = 'loading'; })
      .addCase(fetchTodos.fulfilled, (state, action) => {
        state.status = 'succeeded';
        state.list = action.payload;
      })
      .addCase(fetchTodos.rejected, (state, action) => {
        state.status = 'failed';
        state.error = action.payload ?? '请求失败';
      });
  },
});

export const { added, toggled } = slice.actions;
export default slice.reducer;
```

```js
// store/index.js
import { configureStore } from '@reduxjs/toolkit';
import todos from './todosSlice';

export const store = configureStore({ reducer: { todos } });
```

```jsx
// 组件：selector 只取需要的字段，其他字段变化不会让本组件重渲染
import { useDispatch, useSelector } from 'react-redux';
import { toggled } from './store/todosSlice';

export default function TodoList() {
  const list = useSelector(s => s.todos.list);
  const status = useSelector(s => s.todos.status);
  const dispatch = useDispatch();

  if (status === 'loading') return <p>加载中…</p>;
  return (
    <ul>
      {list.map(t => (
        <li key={t.id} onClick={() => dispatch(toggled(t.id))}>
          {t.done ? '已完成' : '待办'} {t.title}
        </li>
      ))}
    </ul>
  );
}
```

| 对比项 | 传统 Redux | Redux Toolkit |
| --- | --- | --- |
| action type 常量 | 手写字符串，易拼错 | createSlice 自动生成 |
| 不可变更新 | 手写展开运算符，嵌套深时易错 | Immer 内部代理，直接赋值 |
| 异步流程 | 自由发挥，thunk/saga 混用 | createAsyncThunk 统一 pending/fulfilled/rejected |
| 默认中间件 | 需手动加 thunk、DevTools | configureStore 默认开启并检查误用 |
| 样板代码量 | 一个功能三四个文件 | 一个 slice 文件 |

三个高频错误：reducer 里既直接改 state 又 `return { ...state }`，RTK 会报错提示"不能既修改又返回新值"；selector 里 `useSelector(s => ({ a: s.x, b: s.y }))` 每次返回新对象，触发"返回结果不一致"的无限重渲染告警，改成两次独立 useSelector 或加 shallowEqual；把所有 UI 临时状态都塞进 store，导致任何输入都产生全局 action。

### 9.7 Zustand：用订阅粒度换取简洁

Zustand（德语"状态"，一个不依赖 Provider、以 hook 形式订阅全局 store 的轻量状态库）的核心是 `create` 返回一个 hook，调用时传入 selector，只有 selector 的结果变化才重渲染。

```js
// store/useCartStore.js
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

export const useCartStore = create(
  persist(
    (set, get) => ({
      items: [],
      add: (item) => set((s) => ({ items: [...s.items, item] })),
      remove: (id) => set((s) => ({ items: s.items.filter(i => i.id !== id) })),
      total: () => get().items.reduce((sum, i) => sum + i.price * i.qty, 0),
    }),
    { name: 'cart' }        // 中间件自动把 items 存进 localStorage
  )
);
```

```jsx
export default function CartBadge() {
  // 只订阅长度：items 内容变化但长度不变时，本组件不会重渲染
  const count = useCartStore((s) => s.items.length);
  const add = useCartStore((s) => s.add);   // 方法引用恒定，取一次即可
  return <button onClick={() => add({ id: 1, price: 9.9, qty: 1 })}>购物车 {count}</button>;
}
```

| 对比项 | Redux Toolkit | Zustand |
| --- | --- | --- |
| 是否需要 Provider | 需要 react-redux 的 Provider | 不需要，直接 import hook |
| 状态存放 | 单一 root reducer | 可多个独立 store |
| 更新方式 | dispatch action，经 reducer | 直接调用 store 上的方法 |
| 调试体验 | Redux DevTools 完整时间旅行 | 有 DevTools 中间件，能力较弱 |
| 适合的团队约束 | 强，适合多人协作与规范审查 | 弱，靠约定，适合小团队 |
| 引入成本 | 中等 | 极低 |

selector 返回对象或数组时同样每次都是新引用，这一点两个库都一样：

```jsx
// ❌ 每次都返回新数组，判定为变化，可能无限重渲染
const done = useCartStore((s) => s.items.filter(i => i.done));
// ✅ 用浅比较钩子包一层，或让 store 里预存派生结果
const done = useCartStore(useShallow((s) => s.items.filter(i => i.done)));
```

### 9.8 React Router：把路由也当成状态

React Router（React 生态最常用的路由库，负责把 URL 映射到组件树并管理导航）从 6.4 起提供数据路由：loader 在进入路由之前取数据，action 处理表单提交，页面的"加载中"不再需要手写。

```jsx
import {
  createBrowserRouter, RouterProvider, Outlet, useLoaderData, useParams,
} from 'react-router-dom';

const router = createBrowserRouter([
  {
    path: '/',
    element: <RootLayout />,
    errorElement: <ErrorPage />,          // 子路由抛错时在此层兜住
    children: [
      { index: true, element: <Home /> },
      {
        path: 'todos/:id',
        loader: async ({ params }) => {   // 渲染前执行，数据先就绪
          const res = await fetch(`/api/todos/${params.id}`);
          if (res.status === 404) throw new Response('Not Found', { status: 404 });
          return res.json();
        },
        element: <TodoDetail />,
      },
    ],
  },
]);

function RootLayout() {
  return <><Header /><Outlet /><Footer /></>;   // Outlet 是子路由的出口
}

function TodoDetail() {
  const todo = useLoaderData();        // 无需 loading 分支
  return <h2>{todo.title}</h2>;
}
```

路由对象挂到应用根上：`<RouterProvider router={router} />`。数据路由与组件内取数的差异很关键：

| 对比项 | loader | useEffect 内 fetch |
| --- | --- | --- |
| 开始时机 | 导航时、渲染前 | 渲染并提交之后 |
| 加载态展示 | 需配 HydrateFallback 或导航态 | 每个页面自己写 loading |
| 出错处理 | 抛 Response，交给 errorElement | 各自 try/catch |
| 并发导航 | 自动取消上一次导航 | 需手动处理竞态 |

两点部署与用法上的硬知识：使用 BrowserRouter 时，静态服务器必须把未命中的路径重写到 index.html，否则在 `/todos/1` 刷新会 404；把路由参数当作数据来源（筛选、分页、Tab 放 searchParams）能让分享链接与后退按钮正常工作，比藏在 useState 里更可靠。

### 9.9 并发特性：useTransition、useDeferredValue 与 Suspense

并发渲染（Concurrent Rendering，把渲染切成可中断的小片、让高优先级更新先提交的渲染模式）在 React 18 起默认开启，但只有用上对应 API 才有效果。三个 API 各管一件事：

- useTransition 返回 `[isPending, startTransition]`，把一次更新标记为"非紧急"，紧急更新（打字、点击）可以先提交；
- useDeferredValue 给某个值生成一个滞后的副本，先用旧值渲染，空闲时再换成新值；
- Suspense 声明"这块内容没准备好时先显示 fallback"，配合 React.lazy 做代码分割。

```jsx
import { useDeferredValue, useMemo, useState } from 'react';

export default function Search({ items }) {
  const [keyword, setKeyword] = useState('');
  const deferred = useDeferredValue(keyword);   // 输入框即时响应，列表稍后跟上
  const stale = keyword !== deferred;           // 列表显示的还是上一轮结果

  const visible = useMemo(
    () => items.filter(i => i.title.includes(deferred)),
    [items, deferred]
  );

  return (
    <>
      <input value={keyword} onChange={e => setKeyword(e.target.value)} />
      <ul style={{ opacity: stale ? 0.6 : 1 }}>
        {visible.map(i => <li key={i.id}>{i.title}</li>)}
      </ul>
    </>
  );
}
```

```jsx
import { Suspense, lazy } from 'react';

const Detail = lazy(() => import('./Detail'));   // 打包成独立 chunk，按需下载

export default function App() {
  return (
    <Suspense fallback={<Spinner />}>
      <Detail />
    </Suspense>
  );
}
```

| API | 作用对象 | 典型场景 | 误用后果 |
| --- | --- | --- | --- |
| useTransition | 一次状态更新 | 切换筛选条件引发的重列表渲染 | 包住输入框的 setState，输入变卡 |
| useDeferredValue | 一个值 | 大数据量搜索建议 | 对廉价计算使用，反而多一次渲染 |
| Suspense | 一棵子树 | 路由级、组件级懒加载 | 缺少 fallback 导致白屏闪烁 |

要清楚它们不能让计算变快，只是改变优先级：总耗时不变，交互响应变好。复杂计算仍然要 useMemo 或搬到 Web Worker。

### 9.10 SSR、水合与 RSC 概览

渲染模式决定 HTML 在哪里生成：

| 模式 | HTML 生成位置 | 首屏 | 数据新鲜度 | 典型实现 |
| --- | --- | --- | --- | --- |
| CSR 客户端渲染 | 浏览器 | 慢，需等 JS | 实时 | Vite + React 单页应用 |
| SSR 服务端渲染 | 每次请求时服务端 | 快 | 实时 | Next.js、Remix |
| SSG 静态生成 | 构建时 | 最快，可上 CDN | 构建时 | Next.js 静态导出、Astro |
| ISR 增量再生成 | 构建后按周期后台重生成 | 快 | 有延迟 | Next.js |

服务端返回 HTML 后，浏览器还要接管它：这一步叫水合（Hydration，在已有 DOM 上绑定事件、复用节点而不是重建）。水合失败最常见的原因是首屏内容不一致——渲染期读了 `Date.now()`、`Math.random()`、`window`、`localStorage`，或服务端与客户端的分支不同。React 会报 hydration mismatch，并退化为整树客户端重渲染，性能收益清零。规则是：任何"只有浏览器才知道"的值都放到 useEffect 或 `useSyncExternalStore` 里取。

RSC（React Server Components，只在服务端执行、代码不进入客户端 bundle 的组件类型）把这一思路再推一步：组件可以直接读数据库、读文件，返回的是可序列化的描述而不是 HTML 字符串。

```jsx
// app/posts/page.jsx —— 服务端组件，默认就是，无需指令
import { db } from '@/lib/db';
import LikeButton from './LikeButton';   // 客户端组件

export default async function Page() {
  const posts = await db.post.findMany();   // 顶层 await，只在服务端执行
  return (
    <ul>
      {posts.map(p => (
        <li key={p.id}>
          {p.title}
          <LikeButton id={p.id} />   {/* 只有这个交互组件需要 'use client' */}
        </li>
      ))}
    </ul>
  );
}
```

```jsx
// app/posts/LikeButton.jsx
'use client';                     // 声明边界：本文件及它导入的模块进入客户端包

import { useState } from 'react';
export default function LikeButton({ id }) {
  const [liked, setLiked] = useState(false);
  return <button onClick={() => setLiked(!liked)}>{liked ? '已赞' : '点赞'}</button>;
}
```

三条限制必须记住：服务端组件不能用 useState、useEffect、事件处理；传给客户端组件的 props 必须可序列化，普通函数、类实例、Symbol 都不能直接传；`'use client'` 是边界而非标记，被它导入的模块也会一并打包到客户端，把整个组件库拖进去是常见的包体膨胀原因。

判断要不要上这套：内部中后台、登录后才用的页面，CSR 完全够用；内容站、营销页、需要 SEO 的商品页才值得为 SSR/SSG 付出额外复杂度；RSC 目前需要 Next.js App Router 这类框架支持，自建成本高。

### 9.11 速查表与常见坑

| 现象 | 优先检查 | 常用解法 |
| --- | --- | --- |
| 输入框内容莫名清空 | 组件是否在渲染中定义、key 是否随渲染变化 | 组件移到模块顶层，key 用稳定 id |
| 列表删一项后状态错位 | key 用了数组下标 | 换业务主键 |
| 改一个输入框整页变卡 | 大子树被父级 state 牵连 | 状态下移、children 提升、memo |
| 加了 memo 还是重渲染 | props 里有新对象或内联函数 | useCallback / useMemo，或调整结构 |
| Context 一更新全页重渲染 | 单一 Context 承载了高频数据 | 拆分 Context，或换 Zustand |
| 无限重渲染告警 | selector 每次返回新对象 | 拆成多次取值或用 shallowEqual |
| 刷新子路由 404 | 静态服务器未做 SPA 回退 | 重写所有路径到 index.html |
| 首屏闪烁后重渲染 | 水合内容不一致 | 浏览器专属值放 effect 里取 |

常见坑：

- 在渲染期间创建组件类型（`const Row = () => ...`），或把组件写在另一个组件的函数体里，导致每次渲染都卸载重建子树，输入内容与滚动位置全部丢失。
- 认为 `React.memo` 是免费的：props 里带 `style={{}}`、`onClick={() => {}}` 时它根本不生效，白白增加一层比较与一层排查成本。
- 把服务端接口数据直接塞进全局 store：缓存、去重、失效、重试全要手写，最终变成一个残缺的请求库；这类数据应交给 TanStack Query 或 RTK Query。
- 用 useEffect 做导航跳转（`if (!user) navigate('/login')`），渲染期间先闪一帧受保护内容；在数据路由里改用 loader 重定向，或在渲染中直接返回 `<Navigate />`。
- 给列表加 memo 却忽略父组件传下来的 `children` 和 `render props` 函数，每次渲染都是新引用，优化实际未生效；用 Profiler 确认"为什么重渲染"里的原因字段再动手。

## 10. 三框架对比与迁移

### 10.1 设计哲学：依赖追踪与整体重渲染

Vue 与 React 最根本的分歧不在模板还是 JSX，而在一次数据变化之后，框架如何知道该更新什么。

Vue 走响应式（Reactivity，数据被读取时自动记录谁在用，被修改时自动通知这些使用者）路线。响应式系统在运行时给数据套了一层拦截，谁读了哪个属性就建立一条依赖记录，属性变化时按记录精确通知。React 走不可变（Immutability，状态对象创建后不修改，变更靠创建新对象表达）加重渲染（Re-render，函数组件整个重新执行一遍、返回新的元素树，再由协调器比对差异）路线。组件不订阅任何属性，它只是把当前 state 映射成 UI。

```javascript
// Vue 3：依赖自动收集，只有真正读过 count 的副作用会被重新调度
import { ref, computed } from 'vue';

const count = ref(0);
const double = computed(() => count.value * 2);   // 依赖 count，自动记录，依赖不变就不重算
count.value++;                                    // 触发 count 的全部订阅者
```

```jsx
// React：状态不能原地改，setter 提交新值后整个组件函数重跑一遍
import { useState, useMemo } from 'react';

function Counter() {
  const [count, setCount] = useState(0);
  const double = useMemo(() => count * 2, [count]);  // 依赖必须手写，漏写就拿到陈旧值
  return <button onClick={() => setCount(count + 1)}>{count} / {double}</button>;
}
```

代码量差不多，代价的位置完全不同：

| 维度 | Vue | React |
| --- | --- | --- |
| 更新粒度 | 依赖级，只重跑用到了该属性的渲染副作用 | 组件级，整个函数组件重跑 |
| 依赖声明 | 自动追踪，靠运行时代理拦截 | 手动书写依赖数组，漏写产生陈旧闭包 |
| 缓存手段 | `computed` 天然缓存 | `useMemo` / `useCallback` 手动缓存 |
| 状态修改 | 直接赋值、`push`、`delete` 都生效 | 必须返回新对象、新数组，浅比较才认得变化 |
| 运行时开销 | 每个属性读写有代理拦截成本 | 重渲染范围大时成本被放大 |
| 渲染可中断 | 更新按依赖批量刷新，不易拆分 | 渲染函数是纯的，可暂停、可丢弃，这是并发特性（Concurrent Rendering）的前提 |

依赖声明这一条决定了写法的形状：Vue 的 `computed` 自己知道依赖谁，依赖不变就一直用缓存的值；React 不知道 `double` 用了 `count`，只能靠开发者写在数组里。这也解释了两边性能优化的方向不同：React 优化是「缩小重渲染范围」（`memo`、`useMemo`、拆组件、状态下移）；Vue 优化是「减少不必要的响应式代理」（`shallowRef`、`markRaw`、冻结大对象）。

更新粒度这件事在有大量静态内容的页面上差别很明显。假设一个页面由搜索框和一张一千行的表格组成，关键词状态和表格数据都放在同一个父组件里：Vue 的模板编译后，关键词只对应一个文本节点的依赖，输入时只有那个节点的渲染副作用重跑，表格的依赖没有变化，一行都不会进入比对；React 里父组件函数会整个重跑，如果传给表格的 `data` 每次都是新建的数组，子组件的 `memo` 浅比较也会失败，于是一千行一起进入协调流程。所以 React 项目里把状态下移到真正使用它的组件、或用 `useMemo` 稳住传给子组件的对象引用，不是可选优化，而是基本写法。

另一个容易被忽略的差异是批处理。React 18 之前只有事件处理函数内部的多次 `setState` 会被合并，写在 `setTimeout` 或 `Promise` 回调里的多次设置会各自触发一次渲染；React 18 起默认自动批处理（Automatic Batching），所有场景都合并成一次。Vue 的更新从设计上就是异步批量：同一轮同步代码里改十次数据，渲染副作用只进队列一次，在微任务里统一刷新。

一个误解需要澄清：Vue 的 `v-model` 只是双向绑定的语法糖，数据依然遵守单向流动，子组件不能直接改父组件的变量，只能发出更新事件。所以在数据流这个层面，Vue 与 React 遵守同一条规则，差别只在事件契约的默认形态。

### 10.2 数据流与状态管理对比

单向数据流（One-way Data Flow，数据由父组件通过 props 流向下、子组件通过事件把意图传回上）是两者的共同基础，实现手法不同：

| 能力 | Vue | React |
| --- | --- | --- |
| 父传子 | `props`，可用 `defineProps` 声明类型与默认值 | `props`，直接解构形参，用 TS 接口约束 |
| 子传父 | `emit('change', v)`，父组件写 `@change` | 父组件传 `onChange` 函数，子组件调用它 |
| 双向绑定 | `v-model` / `v-model:title` | 受控组件：`value` + `onChange` 成对出现 |
| 跨层传递 | `provide` / `inject`，无类型保障 | `createContext` + `useContext`，可带默认值 |
| 内容分发 | 具名插槽、作用域插槽（子组件把数据传给插槽） | `children` 或 props 传 ReactNode，本身是值所以更灵活 |
| 事件命名 | 模板指令，`@click`、`@input` | 驼峰属性，`onClick`、`onChange`，本质上就是普通 prop |

状态管理库的差异同样源于设计哲学：

| 维度 | Vue（Pinia / Vuex 4） | React（Redux Toolkit / Zustand） |
| --- | --- | --- |
| 状态载体 | 响应式对象，读取即订阅 | 普通对象，靠 selector 订阅 |
| 更新入口 | Pinia 在 action 内直接赋值；Vuex 必须 `commit` mutation | `dispatch` action 交给 reducer 返回新 state；Zustand 用 `set` |
| 派生值 | `computed` / getter 自动缓存 | selector 加 `useMemo`，或 reselect 做记忆化 |
| 异步 | action 写成 `async` 直接 `await` | RTK 用 `createAsyncThunk`；Zustand 在 action 内直接 async |
| 订阅粒度 | 组件模板用了什么就依赖什么 | selector 返回值经浅比较决定是否重渲染 |
| 调试 | devtools 展示 action 时间线 | Redux DevTools 支持时间旅行、状态快照对比 |

状态管理没有框架差异，只有层次差异。判断顺序是固定的：只在一个组件内用的状态留在组件内；兄弟组件共享就提升到共同父组件；跨越三层以上且与业务无关的状态（主题、当前用户、权限码）用 provide/inject 或 Context；只有需要跨页面存活、需要被多个不相关模块读写、需要持久化或时间旅行调试的，才引入全局 store。过早引入全局 store 的代价在两边一样：任何一次改动都要跨文件追数据来源，删一个字段要全仓库搜索。

生态取舍上还有一处不对称。Vue 官方只推 Pinia，社区几乎没有第二个主流选择，团队省掉了选型会议；React 侧 Redux Toolkit 与 Zustand 长期并存，RTK 适合多人协作、需要严格 action 规范与时间旅行调试的中大型项目，Zustand 适合想少写样板代码、状态边界清晰的中小型项目，两者不要在一个仓库里混用。

```javascript
// Vue 3 + Pinia：状态是响应式的，push 就能触发订阅者更新
import { defineStore } from 'pinia';
import { ref, computed } from 'vue';

export const useCart = defineStore('cart', () => {
  const items = ref([]);
  const total = computed(() => items.value.reduce((s, i) => s + i.price * i.qty, 0));
  function add(product) {
    items.value.push({ ...product, qty: 1 });   // 原地修改，不需要返回新数组
  }
  return { items, total, add };
});
```

```javascript
// React + Zustand：必须返回新对象，旧对象一个字段都不能改
import { create } from 'zustand';

export const useCart = create((set) => ({
  items: [],
  add: (product) =>
    set((state) => ({ items: [...state.items, { ...product, qty: 1 }] })),
}));

// 派生值抽成 selector，注意每次返回新数组会让浅比较永远失败
export const selectTotal = (s) => s.items.reduce((sum, i) => sum + i.price * i.qty, 0);
```

在 React 里写 `state.items.push(x)` 组件不会更新，因为数组引用没变，浅比较判定「没变化」；在 Vue 里写 `items.value.push(x)` 立刻生效。这条差异是 Vue 开发者转 React 时最高频的错误，也是 code review 里最该盯的一行。

### 10.3 生命周期与 API 对照表

| 阶段 | Vue 2 选项式 | Vue 3 组合式 | React 函数组件 |
| --- | --- | --- | --- |
| 实例创建前 | `beforeCreate` | `setup()` 函数体 | 函数体（每次渲染都执行） |
| 实例创建后 | `created` | `setup()` 函数体 | 函数体 |
| 挂载前 | `beforeMount` | `onBeforeMount` | 无对应，需 `useLayoutEffect` |
| 挂载后 | `mounted` | `onMounted` | `useEffect(fn, [])` |
| 更新前 | `beforeUpdate` | `onBeforeUpdate` | 无对应（渲染阶段不允许副作用） |
| 更新后 | `updated` | `onUpdated` | `useEffect(fn)`，无依赖数组 |
| 卸载前 | `beforeDestroy` | `onBeforeUnmount` | `useEffect` 返回的清理函数 |
| 卸载后 | `destroyed` | `onUnmounted` | 清理函数执行完毕，无额外钩子 |
| 被缓存组件激活 | `activated` | `onActivated` | 无对应，需自行实现 |
| 后代组件报错 | `errorCaptured` | `onErrorCaptured` | 用 ErrorBoundary 组件捕获 |

同一个需求「监听窗口宽度并在卸载时移除监听」，三版写法如下：

```javascript
// Vue 2：钩子按阶段声明，组件生命周期内各跑一次
export default {
  data: () => ({ width: window.innerWidth }),
  mounted() { window.addEventListener('resize', this.onResize); },
  beforeDestroy() { window.removeEventListener('resize', this.onResize); },
  methods: { onResize() { this.width = window.innerWidth; } }
};
```

```javascript
// Vue 3：钩子在 setup 内同步注册，注册一次即绑定到当前组件实例
import { ref, onMounted, onBeforeUnmount } from 'vue';

const width = ref(window.innerWidth);
const onResize = () => { width.value = window.innerWidth; };
onMounted(() => window.addEventListener('resize', onResize));
onBeforeUnmount(() => window.removeEventListener('resize', onResize));
```

```jsx
// React：没有阶段钩子，只有「副作用 + 清理函数」，依赖变化时成对重跑
import { useState, useEffect } from 'react';

const [width, setWidth] = useState(window.innerWidth);
useEffect(() => {
  const onResize = () => setWidth(window.innerWidth);
  window.addEventListener('resize', onResize);
  return () => window.removeEventListener('resize', onResize);  // 卸载与下次执行前都会调用
}, []);   // 空数组 = 只在挂载后跑一次
```

三个必须记住的差异：Vue 的钩子由框架按阶段调用，一个组件实例内只执行一次；React 的 `useEffect` 会在依赖变化时反复执行，清理函数在「下次执行前」和「卸载时」各调用一次，所以清理逻辑必须与订阅逻辑严格对称。Vue 3 的钩子必须在 `setup` 同步执行阶段注册，写在 `await` 之后会丢失当前实例的上下文，控制台直接报 warning。React 的 `StrictMode` 在开发环境会故意挂载、卸载、再挂载一次，正是用这种双调用来暴露清理不对称的副作用，生产构建不会双调用。

时机细节上还有两处常被忽略。Vue 3 的 `onMounted` 回调执行时，子组件的挂载已经完成、DOM 可读，此时读元素尺寸是安全的；但元素如果包在 `v-if` 里，挂载那一刻它并不存在，需要等条件变真后再用 `nextTick` 等一次刷新。React 的 `useEffect` 在浏览器完成绘制之后才执行，读尺寸再改样式会看到一帧闪烁，需要同步测量并立即修正布局时改用 `useLayoutEffect`，它在 DOM 变更后、绘制前同步执行。另外 React 拿 DOM 只能靠 `useRef`，`ref.current` 首次渲染时是 `null`，赋值发生在提交阶段，所以不能在渲染期间读它。

### 10.4 Vue 2 到 Vue 3 的破坏性变更

Vue 3 之所以带来这么多破坏性变更，是因为它把响应式实现从 `Object.defineProperty` 换成了 `Proxy`，并重写了渲染器。迁移前先对着表扫一遍存量代码：

| 变更点 | Vue 2 写法 | Vue 3 写法 / 处理方式 |
| --- | --- | --- |
| 应用创建 | `new Vue({ render: h => h(App) })` | `createApp(App)` |
| 全局挂载 | `Vue.prototype.$http = http` | `app.config.globalProperties.$http = http` |
| 全局 API | `Vue.use`、`Vue.component`、`Vue.mixin` | `app.use`、`app.component`、`app.mixin` |
| 响应式工具 | `Vue.observable`、`Vue.set`、`Vue.delete` | 分别换成 `reactive`、直接赋值、`delete obj.key` |
| 组件 v-model | `value` + `input` 事件 | `modelValue` + `update:modelValue` |
| `.sync` 修饰符 | `:title.sync="t"` | `v-model:title="t"` |
| 过滤器 | `{{ price \| currency }}` | 移除，改用 `computed` 或方法 |
| 事件总线 | `$on` / `$off` / `$emit` 挂在实例上 | 移除，改用 `mitt` 等独立库 |
| `$listeners` | 单独对象 | 合并进 `$attrs`，用 `v-bind="$attrs"` 透传 |
| `$children` | 直接访问子实例数组 | 移除，改用 `$refs` |
| 函数式组件 | `functional: true` | 普通函数组件，第二个参数是 `props` |
| 异步组件 | `() => import('./X.vue')` | `defineAsyncComponent(() => import('./X.vue'))` |
| `.native` 修饰符 | `@click.native` | 移除，在 `emits` 中声明事件即可 |
| 生命周期命名 | `beforeDestroy` / `destroyed` | `beforeUnmount` / `unmounted` |
| 过渡类名 | `v-enter` / `v-leave` | `v-enter-from` / `v-leave-from` |
| `v-if` 与 `v-for` 优先级 | `v-for` 先 | `v-if` 先，不应在同一节点混用 |
| `key` 位置 | 可写在子节点 | `<template v-for>` 上必须写在 `template` 标签 |
| 数组下标赋值 | `arr[0] = x` 不触发更新 | 天然触发，无需 `$set` |

推荐用官方兼容构建做渐进迁移，而不是一次性重写：

```bash
npm i vue@3 vue-router@4 pinia      # 核心依赖与生态库必须同步升到 Vue 3 版本
npm i -D @vue/compat                # 兼容层：让 Vue 2 写法在 Vue 3 运行时上跑起来
```

```javascript
// vite.config.js：vue 别名指向兼容构建，全局先开 MODE 2（完全兼容，控制台打迁移警告）
import { defineConfig } from 'vite';
import vue from '@vitejs/plugin-vue';

export default defineConfig({
  resolve: { alias: { vue: '@vue/compat' } },
  plugins: [vue({ template: { compilerOptions: { compatConfig: { MODE: 2 } } } })],
});
```

```javascript
// 单个组件就地升级到 MODE 3（关闭兼容行为，只剩 Vue 3 语义），按目录逐个推进
export default {
  compatConfig: { MODE: 3 },
  // ...
};
```

全局 API 与 v-model 是影响面最大的两处，最小对照如下：

```javascript
// Vue 2 入口
import Vue from 'vue';
Vue.prototype.$http = http;
new Vue({ router, render: (h) => h(App) }).$mount('#app');

// Vue 3 入口
import { createApp } from 'vue';
const app = createApp(App);
app.config.globalProperties.$http = http;   // 模板里仍可写 this.$http
app.use(router).mount('#app');
```

```html
<!-- Vue 2 -->
<Child v-model="msg" :title.sync="title" />
```

```html
<!-- Vue 3：一个组件可以有多条 v-model -->
<Child v-model="msg" v-model:title="title" />
```

```javascript
// Vue 3 子组件：显式声明 props 与 emits，替代 Vue 2 的 value/input 约定
const props = defineProps({ modelValue: String, title: String });
const emit = defineEmits(['update:modelValue', 'update:title']);
emit('update:modelValue', '新值');
```

迁移顺序建议固定为六步：第一，冻结新功能，先把构建链路和依赖升级到支持 Vue 3 的版本；第二，接入 `@vue/compat` 把所有警告打成清单，按警告数量排序；第三，先改全局入口与路由、状态管理这类「一个文件影响全站」的地方；第四，再按目录批量替换生命周期命名、`$listeners`、`$children` 这类机械改动；第五，逐个把组件切到 MODE 3，配合单测与关键路径回归；第六，全部通过后移除 `@vue/compat` 依赖与别名，删除过滤器和事件总线的残留。

兼容构建不是免费的。它会在运行时保留 Vue 2 的行为分支，包体积变大，控制台警告密集，个别边界行为（属性合并、`$scopedSlots`、`$listeners` 的透传顺序）仍可能与 Vue 2 有细微差异。所以 `@vue/compat` 只适合当过渡期脚手架，必须给出明确截止时间，比如两个迭代内把警告清空。另一条经验是迁移期间不要顺手重构架构：既换框架大版本、又换状态管理、又重排目录，出问题时无法判断是哪一步引入的，回归成本会翻倍。

### 10.5 React 与 Vue 的思维转换

| 概念 | Vue | React |
| --- | --- | --- |
| 视图描述 | 模板（编译期优化） | JSX（就是函数调用） |
| 条件渲染 | `v-if` / `v-else` | `cond && <A />` 或三元表达式 |
| 列表渲染 | `v-for` + `:key` | `list.map()` + `key` |
| 样式 | `:class` 对象语法、`<style scoped>` | `className` 字符串、CSS Modules、CSS-in-JS |
| 事件 | `@click="fn"` | `onClick={fn}`，注意传函数而非调用结果 |
| 输入框 | `v-model` | `value` + `onChange` 受控 |
| 派生数据 | `computed` 自动追踪 | `useMemo` 或直接在渲染中计算 |
| 副作用 | `watch` / `watchEffect` | `useEffect` + 依赖数组 |
| 逻辑复用 | 组合式函数（composable），如 `useMouse()` | 自定义 Hook，如 `useMouse()` |
| 拿到 DOM | 模板 `ref="el"` | `useRef()` + `ref={el}` |
| 内容插槽 | `<slot name="header" />` | `props.header` 或 `children` |

同一个「输入关键词过滤列表」的功能，两版代码对照非常直观：

```html
<!-- Vue 3 -->
<script setup>
import { ref, computed } from 'vue';
const keyword = ref('');
const list = ref([{ id: 1, name: '键盘' }, { id: 2, name: '鼠标' }]);
const filtered = computed(() => list.value.filter((i) => i.name.includes(keyword.value)));
</script>

<template>
  <input v-model="keyword" placeholder="搜索" />
  <ul>
    <li v-for="item in filtered" :key="item.id">{{ item.name }}</li>
  </ul>
</template>
```

```jsx
// React
import { useState, useMemo } from 'react';

function Search() {
  const [keyword, setKeyword] = useState('');
  const list = [{ id: 1, name: '键盘' }, { id: 2, name: '鼠标' }];
  const filtered = useMemo(
    () => list.filter((i) => i.name.includes(keyword)),
    [keyword]                        // list 是常量可省略；若来自 props 必须列入
  );
  return (
    <>
      <input value={keyword} onChange={(e) => setKeyword(e.target.value)} placeholder="搜索" />
      <ul>{filtered.map((item) => <li key={item.id}>{item.name}</li>)}</ul>
    </>
  );
}
```

四条思维转换可以直接背下来：状态要造新值而不是改旧值；依赖要显式写出来而不是等框架追踪；组件函数里的每个局部变量在每次渲染时都是新的，异步回调里读到的是它被创建那一刻的值；逻辑复用从「混入」变成「返回值是状态的普通函数」，两边其实已经趋同。

其中的闭包问题值得单独展开，同一段定时器逻辑在两边表现完全不同：

```jsx
// React：effect 只跑一次，回调里的 count 被闭包固定在首次渲染的值，永远是 0
useEffect(() => {
  const id = setInterval(() => setCount(count + 1), 1000);      // 错误
  return () => clearInterval(id);
}, []);

useEffect(() => {
  const id = setInterval(() => setCount((c) => c + 1), 1000);   // 正确：函数式更新拿最新值
  return () => clearInterval(id);
}, []);
```

```javascript
// Vue：ref 是可变容器，回调每次读 .value 都是当前值，不存在快照问题
import { ref } from 'vue';
const count = ref(0);
setInterval(() => { count.value++; }, 1000);
```

Vue 的响应式对象是一个稳定引用，变量始终指向同一个容器，读 `.value` 时才真正取值；React 的 `count` 只是某次渲染里的普通数字，函数执行完就固定了。由此得出一条经验：React 的 `useEffect` 里只要引用了外部变量，要么写进依赖数组，要么改成函数式更新，要么用 `useRef` 保存最新值，三条路必须选一条。

### 10.6 团队选型

| 维度 | Vue | React |
| --- | --- | --- |
| 上手成本 | 低，模板接近 HTML，官方文档成体系 | 中，需要先理解不可变、纯函数、Hooks 规则 |
| 类型支持 | 组合式 API + TS 较好，模板类型检查依赖 `vue-tsc` | 一流，JSX 就是 TS 的一部分 |
| 生态广度 | 官方方案齐全（路由、状态、构建一体） | 社区方案极多，选择成本与碎片化并存 |
| 服务端渲染 | Nuxt | Next.js、Remix |
| 跨端 | uni-app、Taro 覆盖小程序与 App | React Native 覆盖原生 App |
| 企业后台 | Element Plus、Ant Design Vue | Ant Design、MUI，组件库与模板最丰富 |
| 招聘市场 | 国内中小团队与外包项目存量多 | 大厂与海外岗位比例更高 |
| 长期维护 | 官方主导，版本节奏稳定 | 由 Meta 与社区共同推进，RFC 流程公开 |

判断规则可以简化成四条：团队前端经验普遍较浅、要快速交付中后台，选 Vue；需要跨端到原生 App 或组件库要最广，选 React；已有 Vue 2 项目且体量可控，直接迁 Vue 3 而不是换框架，重写收益远低于成本；长期项目看重人才供给与生态稳定性，两者都可行，此时更该看团队已有的技术资产，而不是框架本身的优劣。

### 10.7 迁移还是重写：先做一次量化体检

决定动作之前，先用能算出来的指标给存量项目做体检，而不是凭「代码太乱」的感觉直接重写。

| 评估项 | 取值方式 | 判断参考 |
| --- | --- | --- |
| 代码规模 | `cloc src` 统计有效行数 | 两万行以内、页面数少于五十，迁移通常比重写快 |
| 逻辑耦合 | 统计 mixins 数量、全局事件总线的使用点 | mixins 超过十个、事件总线到处用，说明逻辑耦合重，要先解耦 |
| 第三方依赖 | `npm ls` 找出不兼容 Vue 3 的包 | 有替代品的直接换；没有替代品的核心包会卡死整个计划 |
| 测试覆盖 | 单测行覆盖率、关键路径 E2E 用例数 | 覆盖率低于三成，迁移只能靠人肉回归，风险偏高 |
| 业务节奏 | 未来两个迭代是否有大改版 | 已有改版计划时，重写与改版合并做才划算 |

结论只有三种：改动集中在 API 替换、业务逻辑不用动，走 `@vue/compat` 渐进迁移；核心依赖不兼容且短期没有替代品，先做依赖替换或功能裁剪，迁移往后排；业务本身要重构、页面结构要大改，直接按 Vue 3 重写并复用数据层与工具函数，硬迁移反而要背两套历史包袱。同一套判断也适用于 React 项目换用 Vue 的场景，只是需要额外评估组件库（如 Ant Design 到 Element Plus）的迁移面积。

### 10.8 常见坑

- **把原地修改带到 React**：`state.list.push(x)`、`state.user.name = 'a'` 都不会触发更新，因为引用没变。改法是展开或 `immer` 的 `produce`，并始终检查赋值语句左边是不是旧对象。
- **以为 Vue 的 `watch` 也需要写依赖**：Vue 自动追踪，但侦听 `reactive` 对象本身是浅层监听，`watch(state, cb)` 只有整个对象被替换才触发；要监听某个字段必须写成 `() => state.count` 的 getter。
- **迁移时只改 API 不改语义**：`$listeners` 合并进 `$attrs` 后，`v-bind="$attrs"` 会把 `onXxx` 一起透传，如果组件内部又手动绑定了同名事件，会出现一次点击触发两次处理函数，需在 `emits` 里显式声明并检查透传链。
- **`v-if` 与 `v-for` 同节点混用**：Vue 3 里 `v-if` 优先级更高，会访问不到 `v-for` 的作用域变量而报错。外层套一个 `<template v-for>` 或先用 `computed` 过滤数据，是唯一稳妥的写法。

## 11. 性能优化与调试

### 11.1 性能优化在优化什么：三个核心指标

性能优化（Performance Optimization，用更少的传输、计算与重绘，让页面更快可用、更稳地响应操作）在工程实践里不是「压缩图片、加上缓存」的零散技巧，而是一个闭环：先用指标定义「快」，再用工具找出「哪里慢」，然后改代码，最后在相同条件下复测。没有指标约束的优化，典型结局是包体积减了 200KB、构建时间短了一半，用户感知却毫无变化，因为真正的瓶颈是首屏那张 1.8MB 的横幅图。

Web 性能的公共语言是核心 Web 指标（Core Web Vitals，Google 提出的一组以真实用户体验为中心的质量指标）。它只有三个数，却覆盖了「能不能打开」「点了有没有反应」「内容会不会乱跳」三件事：

| 指标 | 全称与含义 | 衡量什么 | 良好 | 需改进 | 差 |
| --- | --- | --- | --- | --- | --- |
| LCP | Largest Contentful Paint，最大内容绘制 | 视口内最大元素（大图、大标题、视频封面）绘制完成的时间 | ≤ 2.5s | 2.5–4s | > 4s |
| INP | Interaction to Next Paint，交互到下次绘制 | 用户点击、按键后，到下一帧画面更新的延迟；取代了只测首次输入的 FID | ≤ 200ms | 200–500ms | > 500ms |
| CLS | Cumulative Layout Shift，累积布局偏移 | 页面生命周期内所有意外布局位移的累计分数 | ≤ 0.1 | 0.1–0.25 | > 0.25 |

判定不看平均值，而看第 75 百分位（p75）：把一段时间内所有访问的 LCP 排序，75% 的用户达到的数值才算达标。平均值会被少数极快的光纤用户拉低，掩盖掉一半移动端用户的糟糕体验。同样是这三个指标，移动端与桌面端要分开看。

三个指标会互相牵制，必须知道它们的成本归属：LCP 主要由「首字节时间 TTFB + 关键资源下载 + 首屏渲染」构成，动它要减请求、减字节、提前加载；INP 主要由「事件回调里的同步计算 + 重渲染 + 布局」构成，动它要拆长任务、让渡主线程；CLS 则完全由「有没有给元素预留空间」决定，几乎不花性能预算就能修好。

采集现场的代码如下（JavaScript，浏览器端，基于 web-vitals 库）：

```js
// 上报真实用户指标：只在页面隐藏时发，避免占用关键请求
import { onLCP, onINP, onCLS, onTTFB } from 'web-vitals';

function send(metric) {
  const body = JSON.stringify({
    name: metric.name,        // LCP / INP / CLS
    value: metric.value,      // 毫秒或无量纲分数
    rating: metric.rating,    // good / needs-improvement / poor
    id: metric.id,            // 同一次页面访问的唯一标识，用于串联同一次会话
    page: location.pathname,
  });
  // sendBeacon 在页面卸载时也能发出，不会阻塞导航
  navigator.sendBeacon('/api/vitals', body);
}

[onLCP, onINP, onCLS, onTTFB].forEach((fn) => fn(send, { reportAllChanges: false }));
```

### 11.2 定位瓶颈：从现场数据到火焰图

指标告警只告诉你「慢了」，不告诉你「为什么慢」。定位需要一个从粗到细的漏斗，顺序不能颠倒，否则容易在错误的方向上优化一周。

**第一步，看现场数据。** 真实用户监控（Real User Monitoring，RUM，在生产环境采集真实用户访问数据的方案）能按机型、网络、页面版本切分。看两件事：哪个页面最差、哪个指标最差。若只有老版本 Chrome 差，先怀疑兼容性回退；若只有 4G 用户差，先怀疑资源体积。

**第二步，在受控环境复现。** 打开 Chrome DevTools 的 Performance 面板，把网络限速设为「Fast 4G」，CPU 降速设为 4 倍或 6 倍。开发机上跑得飞快的页面，在 6 倍降速下才会暴露真实问题。这一步的意义是把偶发的现场问题变成可重复的实验室问题。

**第三步，录制一条完整的加载与交互时间线。** 点录制、刷新页面、页面稳定后点击一次核心按钮、停止录制。看三处：

- Main 轨道上的红色三角与长任务（Long Task，主线程连续占用超过 50ms 的任务）。红色越密集，INP 越差。
- 时间线顶部的 LCP 标记，直接告诉你哪个元素是最大内容元素、它在第几毫秒完成。
- Layout Shift 标记，点开可以看到每次位移的元素、移动距离与影响分数。

**第四步，看 Self Time。** 切到底部的 Bottom-Up 或 Call Tree 视图，按 Self Time 排序。Self Time 是函数自身消耗的时间，不含它调用的子函数，因此能准确指出「谁在烧 CPU」。一屏又一屏的 `commitRoot`、`renderWithHooks`、`patch` 说明是渲染太贵；集中在某个业务函数说明是计算太贵。

常用的几类工具回答的问题完全不同，混用会得出错误结论：

| 工具 | 数据来源 | 回答的问题 | 局限 |
| --- | --- | --- | --- |
| Lighthouse | 实验室，单次审计 | 整体评分多少、有哪些通用建议 | 受本机条件影响，分数波动大，不等于用户体验 |
| Performance 面板 | 实验室，录制时间线 | 哪一帧慢、哪个函数占 CPU、长任务在哪 | 一次只覆盖录到的那条操作路径 |
| Coverage | 实验室 | 加载的 JS/CSS 有多少完全没被执行 | 只统计加载期，不含懒加载后的代码 |
| RUM / web-vitals | 生产现场 | p75 是多少、哪个页面哪类机型差 | 只给结果，不给原因 |
| Vue / React DevTools Profiler | 开发环境 | 哪个组件 render 次数多、耗时高 | 开发构建的绝对耗时不能直接用于上线对比 |

**第五步，看资源。** 切到 Network 面板按体积排序，切到 Coverage 面板看未使用的 JS/CSS 比例。单页应用常见的病是首屏只用一个页面，却加载了全站十几个路由与整套图表库的代码，未使用率 70% 以上。

**第六步，改一处、复测一次。** 每次只改一个变量，用同样的限速条件重录，对比 LCP 时间与长任务数量。改完不量化复测，等于没改。

### 11.3 加载优化：代码分割、懒加载、预加载与图片

加载阶段的优化目标是减少首屏关键路径上的字节数与往返次数。

**代码分割（Code Splitting，把一个大 bundle 拆成多个按需加载的小文件）** 是投入产出比最高的一步。语法基础是动态导入 `import()`，它返回 Promise，打包器会把它自动拆成独立 chunk。

```js
// Vue 3：路由级懒加载，访问 /report 时才下载报表页代码
const routes = [
  { path: '/', component: () => import('../views/Home.vue') },
  { path: '/report', component: () => import('../views/Report.vue') },
];

// 组件级懒加载：重量级弹窗用 defineAsyncComponent 包一层
import { defineAsyncComponent } from 'vue';
const HeavyChart = defineAsyncComponent({
  loader: () => import('../components/HeavyChart.vue'),
  loadingComponent: SkeletonBox,  // 加载中的占位，避免布局跳动
  delay: 200,                     // 200ms 内加载完就不显示骨架屏，减少闪烁
  timeout: 10000,                 // 超时进入 errorComponent
});
```

```jsx
// React 18：React.lazy 必须配合 Suspense 提供兜底 UI
import { lazy, Suspense } from 'react';
const HeavyChart = lazy(() => import('./HeavyChart'));

export default function Report() {
  return (
    <Suspense fallback={<div className="chart-skeleton" style={{ height: 320 }} />}>
      <HeavyChart />
    </Suspense>
  );
}
```

拆包策略比拆包本身更容易出错。Vite（基于 Rollup 的构建工具）里用 `manualChunks` 把长期不变的第三方库单独分组，让业务代码更新时用户不必重新下载框架：

```js
// vite.config.js：把体积大且很少变更的依赖拆成稳定 chunk
export default {
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          'vendor-vue': ['vue', 'vue-router', 'pinia'],
          'vendor-chart': ['echarts'],
        },
      },
    },
    chunkSizeWarningLimit: 600, // 单位 KB，超过就告警，提醒继续拆
  },
};
```

拆得过细同样有害：每个 chunk 都是一次额外请求，HTTP/1.1 下尤其明显，层级深的依赖瀑布会让 LCP 反而变差。经验阈值是单个 chunk 超过 500KB（未压缩）就该拆，首屏关键 chunk 控制在 3 个以内。

拆之前要看清产物里到底装了什么。用 `rollup-plugin-visualizer` 生成一份可视化报告，比盲猜有效得多：

```js
// vite.config.js：构建后生成 stats.html，用浏览器打开即可看到每块占比
import { visualizer } from 'rollup-plugin-visualizer';

export default {
  plugins: [visualizer({ filename: 'stats.html', gzipSize: true, template: 'treemap' })],
};
```

报告里最常见的三类浪费：整包引入了工具库（`import _ from 'lodash'` 会把全部函数打进包，应改成 `lodash-es` 的具名导入或 `lodash/debounce` 单函数引入）；整包引入了图表库（ECharts 可按需注册 `BarChart`、`GridComponent` 与渲染器，体积能砍掉一半以上）；同一个库存在多个版本（用 `pnpm why 库名` 查依赖树，靠 `pnpm.overrides` 收敛版本）。这三类问题都不需要改业务逻辑，属于纯粹的打包损失。

**预加载**决定「什么时候下载」。三者差别必须分清：

| 指令 | 作用 | 是否阻塞渲染 | 典型用途 |
| --- | --- | --- | --- |
| `<link rel="preload">` | 提前以高优先级下载当前页马上要用的资源 | 否，但会抢带宽 | 首屏字体、LCP 大图 |
| `<link rel="modulepreload">` | 提前下载并解析 ES 模块及其依赖 | 否 | 首屏必需的 JS chunk |
| `<link rel="prefetch">` | 空闲时以低优先级下载「下一个页面可能用」的资源 | 否 | 详情页预取、下一页路由 |

**图片**通常是 LCP 的最大变量，四个属性就能解决大半问题：

```html
<!-- 首屏大图：显式宽高防 CLS，高优先级提示，禁止懒加载 -->
<img src="/hero-800.avif"
     srcset="/hero-400.avif 400w, /hero-800.avif 800w, /hero-1600.avif 1600w"
     sizes="(max-width: 600px) 100vw, 800px"
     width="800" height="450"
     fetchpriority="high" decoding="async"
     alt="产品首屏截图">

<!-- 首屏之外的图片：懒加载，浏览器接近视口时才请求 -->
<img src="/thumb.avif" width="320" height="180" loading="lazy" alt="案例缩略图">
```

`width`/`height` 让浏览器在图片到达前就按比例预留空间，是修 CLS 最有效的一招；`srcset` + `sizes` 让手机不下载桌面尺寸；`loading="lazy"` 绝不能加在首屏大图上，否则浏览器会等布局稳定才发起请求，LCP 直接恶化一到两秒。

**传输层还有两道免费收益。** 一是文本资源用 Brotli 压缩（比 gzip 再小 15%–20%），图片、视频本身已压缩，不要重复压缩；二是给带内容哈希的文件名配长缓存（`Cache-Control: public, max-age=31536000, immutable`），入口 HTML 则用 `no-cache` 保证能拿到新版本。这一步不需要改任何业务代码，却直接影响二次访问的 LCP 与 TTFB。

### 11.4 渲染优化：虚拟列表与避免重复计算

渲染优化解决的是「主线程被占满」。浏览器主线程同时负责 JS 执行、样式计算、布局与绘制，任何一段超过 50ms 的同步工作都会让点击响应排队，INP 随之上升。

**长列表是头号杀手。** 一次渲染 5000 行 DOM，意味着 5000 次创建、样式计算与布局，滚动时还会持续触发。虚拟列表（Virtual List，只渲染视口内可见的若干行，用占位高度撑起滚动条的方案）把 DOM 数量从「数据量」降到「视口行数 + 缓冲」。

```jsx
// React：@tanstack/react-virtual 的核心用法，固定行高 40px
import { useVirtualizer } from '@tanstack/react-virtual';

function UserList({ rows }) {
  const parentRef = useRef(null);
  const virtualizer = useVirtualizer({
    count: rows.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 40,   // 行高固定时可精确计算，避免滚动抖动
    overscan: 8,              // 上下各多渲染 8 行，抵消快速滚动白屏
  });

  return (
    <div ref={parentRef} style={{ height: 480, overflow: 'auto' }}>
      <div style={{ height: virtualizer.getTotalSize(), position: 'relative' }}>
        {virtualizer.getVirtualItems().map((item) => (
          <div key={item.key}
               style={{ position: 'absolute', top: 0, height: 40,
                        transform: `translateY(${item.start}px)` }}>
            {rows[item.index].name}
          </div>
        ))}
      </div>
    </div>
  );
}
```

Vue 侧可选用 `vue-virtual-scroller` 或 `@tanstack/vue-virtual`，思路完全一致：容器固定高度、只渲染可视区、滚动时用 `transform` 位移而非改 `top`，避免触发重排。

**避免重复计算**分两类。第一类是模板里的「函数调用」：Vue 模板中 `{{ formatPrice(item) }}` 在每次重渲染时都会执行，而计算属性（computed）只在其依赖变化时重算；React 里对应的是 `useMemo`。第二类是「无谓重渲染」：父组件状态一变，整棵子树跟着 render，即使大部分子组件的 props 没变。React 需要 `React.memo` 加稳定的 props 引用，Vue 的依赖追踪更精确，但传对象字面量仍会导致子组件更新。

**请求瀑布同样属于首屏成本。** 组件挂载后才发起接口请求，等于把「下载 JS → 挂载 → 等接口 → 再渲染」串成一条链，LCP 被推后整整一个往返；列表页每行各发一次详情请求（N+1 请求）则会把并发打满，浏览器对同域 HTTP/1.1 只给 6 个并发连接。可行的改法是在路由层预取首屏数据（Vue Router 的 `beforeRouteEnter` 或 React Router 的 `loader`），把相互独立的接口并行发出，并把按行请求合并成一次批量接口。


### 11.5 运行时开销：事件、定时器与内存

除了加载与渲染，还有一类开销藏在「一直在跑的东西」里：滚动监听、`resize` 监听、轮询定时器、未退订的 WebSocket。它们不体现在首屏指标上，却持续占用主线程，最终表现为「页面用久了越来越卡」。

**滚动回调是最典型的例子。** 直接在回调里读取 `getBoundingClientRect()`、`scrollTop`、`offsetHeight` 会强制浏览器立刻完成一次同步布局（forced synchronous layout）：写样式与读几何交替进行，每次都打断渲染流水线，一帧的预算只有约 16.7ms，几个来回就掉帧。修法有三层，从便宜到精细依次是：加 `passive: true` 让滚动不被监听器阻塞；用 `requestAnimationFrame` 把每帧的执行次数压到一次；把「元素是否进入视口」的判断整体换成 `IntersectionObserver`，浏览器在合成阶段就能完成测量，完全不必监听 `scroll`。

```js
// 滚动加载：passive 不阻塞滚动，rAF 保证每帧最多执行一次
let ticking = false;
function onScroll() {
  if (ticking) return;
  ticking = true;
  requestAnimationFrame(() => {
    const top = list.getBoundingClientRect().top; // 读几何信息会触发布局，故放进 rAF
    if (top < window.innerHeight + 200) loadMore();
    ticking = false;
  });
}
window.addEventListener('scroll', onScroll, { passive: true });

// 更省的做法：滚动可见性判定交给 IntersectionObserver
const io = new IntersectionObserver((entries) => {
  if (entries[0].isIntersecting) loadMore();
}, { rootMargin: '200px' }); // 提前 200px 触发，滚动体验更顺
io.observe(sentinel);
```

**定时器与订阅必须随组件一起销毁。** 组件已卸载但 `setInterval` 还在跑，回调里再访问已销毁的响应式状态或调用 `setState`，轻则报警告，重则内存持续上涨，因为闭包把整个组件作用域钉在堆里。Vue 3 在 `onUnmounted` 里清理，React 在 `useEffect` 的清理函数里清理，异步请求则用 `AbortController` 取消。判断是否泄漏的实用办法是打开 DevTools 的 Memory 面板，操作几轮后强制垃圾回收、拍两张堆快照做对比，看 Detached HTMLDivElement 的数量是否只增不减。

**一次性的大计算要切成小任务。** 一万条数据的过滤排序若放在一次同步执行里，就是几百毫秒的长任务，期间的点击全部排队。把数据分批处理并在批间让出主线程，是成本最低的改法：

```js
// 分片处理：每批 200 条，批间让出主线程，输入与绘制才有机会插进来
async function processInChunks(items, handle, size = 200) {
  for (let i = 0; i < items.length; i += size) {
    items.slice(i, i + size).forEach(handle);
    await new Promise((r) => setTimeout(r, 0));
  }
}
```

### 11.6 Vue 的常用手段

| 手段 | 作用 | 使用条件与代价 |
| --- | --- | --- |
| `computed` | 缓存派生值，依赖不变不重算 | 依赖必须是响应式的，纯函数才安全 |
| `v-once` | 静态子树只渲染一次，跳过后续 patch | 内容确实永不变，否则不会更新 |
| `v-memo="[a, b]"` | 依赖数组不变时跳过该子树更新 | 需手写依赖，漏写会导致视图不更新 |
| `shallowRef` / `shallowReactive` | 只对第一层做响应式 | 适合大数组、地图实例、第三方对象 |
| `markRaw` | 让对象永不转为响应式 | 只读的配置、图表实例 |
| `keep-alive` | 缓存组件实例与 DOM | 列表页返回详情页时保留滚动位置，注意内存 |
| `v-show` | 用 `display` 切换，不销毁 DOM | 频繁显隐用 v-show，一次性的用 v-if |
| `defineAsyncComponent` | 组件级懒加载 | 配合骨架屏与 delay |
| `app.config.performance = true` | 在 DevTools 时间线打组件性能标记 | 仅开发环境开启 |

一个高频误用是深层侦听大对象：

```js
// 反例：deep 侦听 10 万条数据的数组，每次滚动更新都会全量遍历比对
watch(bigList, () => syncToServer(), { deep: true });

// 正例：只侦听真正关心的字段，或改用 shallowRef + 显式触发
watch(() => bigList.length, () => syncToServer());
```

Vue 3 开发环境还可以打开组件性能标记，在 Chrome Performance 面板里看到每个组件的 `render` 耗时，直接定位「哪个组件最贵」。

### 11.7 React 的常用手段

| 手段 | 作用 | 前提 |
| --- | --- | --- |
| `React.memo` | props 浅比较相等则跳过重渲染 | props 引用必须稳定，否则等于没加 |
| `useMemo` | 缓存计算结果 | 计算确实昂贵或需要稳定引用 |
| `useCallback` | 缓存函数引用，配合 memo 子组件 | 依赖数组必须完整 |
| 状态下放 | 把 state 放进最小子树 | 与兄弟组件无共享 |
| 组合（children） | 父组件重渲染时 children 引用不变，不跟着重渲染 | 把不变部分作为 props 传入 |
| `useDeferredValue` | 让昂贵更新降级为低优先级 | 用于搜索联想、大列表过滤 |
| `useTransition` | 把状态更新标记为非紧急 | 同上，需配合 loading 状态 |
| React DevTools Profiler | 火焰图看每个组件 render 与 commit 耗时 | 开发环境 |

`React.memo` 最常见的失效场景是「父组件每次渲染都新建对象或函数」。修复的关键不是到处包 `useMemo`，而是把不稳定的部分挪到组件外或 `useCallback` 里：

```jsx
// 反例：每次渲染都新建 row 对象与 handler，memo 子组件必然重渲染
<Row data={{ user }} onSelect={() => select(user.id)} />

// 正例一：直接传原始值，子组件只在 id 变化时更新
<Row userId={user.id} userName={user.name} onSelect={handleSelect} />

// 正例二：搜索输入不阻塞输入框，用低优先级更新大列表
function Search() {
  const [kw, setKw] = useState('');
  const deferredKw = useDeferredValue(kw);   // 输入框即时响应，列表稍后跟上
  const list = useMemo(() => filterAll(deferredKw), [deferredKw]);
  return (
    <>
      <input value={kw} onChange={(e) => setKw(e.target.value)} />
      <List data={list} />
    </>
  );
}
```

### 11.8 一次 LCP 4.2 秒的完整排查

某后台首页的 RUM 显示移动端 LCP p75 为 4.2s，CLS 0.19，桌面端一切正常。按定位漏斗逐步推进。

**复现。** DevTools 设为 Fast 4G + 4 倍 CPU 降速，打开首页录制。Performance 面板显示 LCP 元素是首页横幅图，完成于 3.9s。

**归因。** Network 面板按体积排序：横幅图为未压缩 PNG，1.8MB；首屏 JS 2.3MB，其中 `vendor-chart` 占 780KB，但首页没有图表；Coverage 显示首屏 JS 未使用率 68%。

**改三处。** 一，横幅图转 AVIF 并按 `srcset` 给出三档尺寸，1.8MB 降到 210KB，同时去掉误加的 `loading="lazy"`。二，把图表库从首页入口移除，改为路由级动态导入，首屏 JS 降到 640KB。三，给横幅容器写死宽高比，消除图片撑开造成的 CLS。

```css
/* 预留空间：图片未加载时容器已占据正确高度，加载完成不再撑开页面 */
.hero { width: 100%; aspect-ratio: 16 / 9; background: #f2f3f5; }
.hero img { display: block; width: 100%; height: 100%; object-fit: cover; }
```

**复测。** 同样限速下 LCP 从 3.9s 降到 1.6s，CLS 从 0.19 降到 0.02，首屏字节数减少约 70%。上线一周后 RUM 的 p75 落在 2.1s，达标。

整个过程里唯一没有做的事是「加一个骨架屏」。骨架屏能改善感知，却不会让 LCP 变好，反而常因为骨架高度与真实内容不一致成为 CLS 的来源。先修真实指标，再谈感知优化。

### 11.9 常见坑

- **在开发环境测性能。** 开发构建没有压缩、没有 tree-shaking，React StrictMode 还会在开发环境故意双调用渲染函数，Vue 也会走完整警告分支。所有结论必须基于 `build` 后的产物，用 `vite preview` 或静态服务器起本地服务再测。
- **懒加载首屏主图。** `loading="lazy"` 加在 LCP 元素上会让浏览器推迟请求，移动端往往多出一到两秒。首屏大图应该用 `fetchpriority="high"`，只有首屏之下的图片才懒加载。
- **滥用 memo 与 useMemo。** 给 props 每次都变的组件包 `React.memo`、给一次加法包 `useMemo`，只增加了浅比较与依赖数组的开销。优化前先用 Profiler 确认它是热点，优化后用同样的场景复测，没有数据支撑的「优化」通常只是把代码写复杂了。
- **忽略布局预留。** 图片不写宽高、骨架屏高度与真实内容不符、字体加载导致的文字跳动、异步插入的公告条，都会推高 CLS。规约很简单：任何可能在加载后改变尺寸的元素，都必须有稳定的占位空间；字体用 `font-display: swap` 并预加载首屏字体，能显著减少文字重排。

## 12. 测试、CI 与项目实战

### 12.1 测试分层：先想清楚每一层测什么

前端测试（Frontend Testing，用自动化手段验证界面与交互行为是否符合预期的工程实践）不是「写得多就好」，而是按成本与收益分层投入。三层结构如下：

| 层次 | 测什么 | 典型工具 | 单次耗时 | 失败定位 | 建议占比 |
| --- | --- | --- | --- | --- | --- |
| 单元测试 | 纯函数、工具方法、store 逻辑 | Vitest、Jest | 毫秒级 | 精确到函数与分支 | 约 60% |
| 组件测试 | 渲染结果、事件、props/emit | Vue Test Utils、React Testing Library | 十毫秒到百毫秒 | 定位到组件 | 约 30% |
| 端到端测试 | 真实浏览器里的完整用户流程 | Playwright、Cypress | 秒级 | 只能定位到页面 | 约 10% |

在金字塔下方还有一层成本最低的检查：静态检查。ESLint 抓未使用变量与可疑写法，TypeScript 的 `tsc --noEmit` 抓类型不匹配，两者跑一次通常只要几秒，却能在提交前拦掉大量低级错误。把静态检查、单元测试、组件测试、端到端测试按这个顺序串进流水线，失败得越早，修复成本越低。

反面模式是「只写端到端」：几十条浏览器用例跑二十分钟，任何一条挂掉都只告诉你「首页白屏」，排查还得靠人肉。另一种反面模式是「什么都测实现细节」：断言组件内部的 `data.count` 或私有方法调用次数，重构一次就全红，测试从资产变成负债。判断标准很简单——测试描述的是「用户看到什么、做了什么」，而不是「代码怎么写的」。

### 12.2 单元测试：Vitest 与 Jest 的取舍

Jest（Facebook 开源的 JavaScript 测试框架，自带断言、模拟与覆盖率）曾是事实标准；Vitest（基于 Vite 的测试运行器，复用同一套配置与转换管线）是近几年的主流替代品。两者 API 高度相似，`describe`、`it`、`expect`、`vi.fn`/`jest.fn` 基本一一对应，迁移成本主要在选择器与配置上。

| 维度 | Vitest | Jest |
| --- | --- | --- |
| 配置来源 | 直接读 `vite.config.ts`，路径别名与插件天然一致 | 需单独维护 `jest.config` 与 `moduleNameMapper` |
| 转换速度 | 借助 Vite 的 esbuild 预构建，冷启动快 | 依赖 babel/ts-jest，项目大时明显更慢 |
| ESM 支持 | 原生 | 需要额外配置或实验开关 |
| 生态成熟度 | 较新，部分老插件缺失 | 极其成熟，文档与排错资料多 |
| 适用场景 | 新建 Vite 项目、Vue 3 / React 18 项目 | 存量 Webpack / CRA 项目 |

配置示例（TypeScript）：

```ts
// vitest.config.ts
import { defineConfig } from 'vitest/config'
import vue from '@vitejs/plugin-vue'

export default defineConfig({
  plugins: [vue()],
  test: {
    environment: 'jsdom',        // 组件测试需要 DOM 环境；纯函数测试可用 node
    globals: true,               // 允许不 import 直接使用 describe/it
    setupFiles: ['./test/setup.ts'],   // 全局注入 jest-dom 等断言扩展
    coverage: {
      provider: 'v8',
      reporter: ['text', 'lcov'],
      thresholds: { lines: 80, functions: 80 },  // 低于阈值直接失败
    },
  },
})
```

关键参数说明：`environment` 决定是否提供 `window`、`document`；`globals` 打开后需同步在 `tsconfig.json` 的 `types` 里加 `vitest/globals`，否则类型报错；`thresholds` 是让覆盖率真正成为门禁的唯一方式，不设阈值时覆盖率报告只是给人看的数字。

一个带边界与定时器的用例：

```ts
// src/utils/format.spec.ts
import { describe, it, expect, vi } from 'vitest'
import { formatPrice, debounce } from './format'

describe('formatPrice', () => {
  it('保留两位小数并加千分位', () => {
    expect(formatPrice(1234.5)).toBe('¥1,234.50')
  })

  it('非法输入返回占位符', () => {
    expect(formatPrice(NaN)).toBe('--')   // 边界情况比正常路径更容易藏 bug
  })
})

describe('debounce', () => {
  it('时间窗口内只执行最后一次调用', () => {
    vi.useFakeTimers()                   // 接管定时器，测试不真实等待
    const fn = vi.fn()
    const d = debounce(fn, 300)
    d(1); d(2); d(3)
    expect(fn).not.toHaveBeenCalled()
    vi.advanceTimersByTime(300)          // 手动推进时钟
    expect(fn).toHaveBeenCalledTimes(1)
    expect(fn).toHaveBeenCalledWith(3)
    vi.useRealTimers()                   // 用完必须还原，否则污染后续用例
  })
})
```

依赖外部模块时，用替身把不确定性挡在测试边界之外。`vi.mock` 会被静态提升到文件顶部，因此工厂函数里不能引用外部变量（运行时尚未初始化），需要共享的模拟函数要用 `vi.hoisted` 声明：

```ts
const mocks = vi.hoisted(() => ({ createId: vi.fn(() => 'fixed-id') }))
vi.mock('./id', () => ({ createId: mocks.createId }))   // 让 id 生成确定，断言可预测

it('新增时使用注入的 id 生成器', () => {
  expect(addTodo([], '写测试')).toEqual([{ id: 'fixed-id', title: '写测试', done: false }])
})
```

断言只覆盖对外契约里真正重要的字段。对整对象做快照（snapshot）在 UI 结构上偶尔有用，但把快照当成主要断言手段，会让任何无关改动都变成一次「更新基线」，久而久之没人细看 diff，测试失去意义。

单元测试的命名要写清「在什么条件下、期望什么结果」，失败时日志本身就是一句需求描述。每个用例应当互相独立，不依赖执行顺序，也不共享可变状态。

### 12.3 Vue 3 组件测试：用行为断言代替结构断言

Vue Test Utils（Vue 官方组件测试库）配合 Vitest 使用，核心 API 是 `mount`（完整挂载，含子组件）与 `shallowMount`/`global.stubs`（浅挂载，子组件替换为桩）。

```ts
// components/TodoItem.spec.ts
import { mount } from '@vue/test-utils'
import TodoItem from './TodoItem.vue'

it('勾选复选框时派发 toggle 事件并带上最新状态', async () => {
  const wrapper = mount(TodoItem, {
    props: { todo: { id: 1, title: '写测试', done: false } },
  })

  await wrapper.get('[data-test="toggle"]').setValue(true)
  // 断言对外契约：事件名 + 参数，而不是组件内部 data
  expect(wrapper.emitted('toggle')?.[0]).toEqual([1, true])
})

it('展示标题并提供可访问名称', () => {
  const wrapper = mount(TodoItem, {
    props: { todo: { id: 2, title: '买牛奶', done: true } },
  })
  expect(wrapper.text()).toContain('买牛奶')
  expect(wrapper.get('[data-test="toggle"]').attributes('aria-label')).toBe('买牛奶')
})
```

四条落地要点：第一，任何修改 DOM 的操作（`setValue`、`trigger`）都要 `await`，因为它走的是异步渲染队列，不 await 会读到上一帧；第二，选择器优先用 `data-test` 属性或可访问名称，避免 `.item > span:nth-child(2)` 这种一改样式就断的写法；第三，断言 `emitted` 与外层可见 DOM，不断言子组件实例上的字段；第四，需要真实路由或全局插件时通过 `global.plugins` 注入，不要在每个用例里手写 mock。

### 12.4 React 组件测试：以用户视角查询

React Testing Library（RTL，以 DOM 与可访问性为查询入口的 React 测试库）的设计信条是「测试越接近用户使用方式，越能提供信心」。查询优先级为：`getByRole` → `getByLabelText` → `getByPlaceholderText` → `getByText` → 最后才是 `getByTestId`。

```tsx
// TodoItem.test.tsx
import { render, screen } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import { describe, it, expect, vi } from 'vitest'
import TodoItem from './TodoItem'

it('点击复选框时回调最新的完成状态', async () => {
  const onToggle = vi.fn()
  const user = userEvent.setup()
  render(<TodoItem todo={{ id: 1, title: '写测试', done: false }} onToggle={onToggle} />)

  await user.click(screen.getByRole('checkbox', { name: '写测试' }))
  expect(onToggle).toHaveBeenCalledWith(1, true)
})
```

要点：用 `userEvent` 而不是 `fireEvent`，前者会模拟指针、焦点、键盘的完整序列，更接近真实交互；异步出现的元素用 `await screen.findByText(...)`，不要用 `waitFor` 包一个必然失败的同步断言；`jest-dom` 提供的 `toBeInTheDocument`、`toHaveTextContent` 需要在 setup 文件里引入；测试里出现 `act(...)` 警告，通常意味着某处状态更新没有被正确 await。测试卸载与清理由 RTL 自动完成，不要在 `afterEach` 里手动清 DOM。

### 12.5 端到端测试：Playwright 与稳定性三原则

Playwright（微软开源的多浏览器端到端测试框架）通过浏览器协议驱动真实 Chromium/Firefox/WebKit，自带自动等待（auto-waiting）与失败追踪。

```ts
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test'

export default defineConfig({
  testDir: './e2e',
  retries: process.env.CI ? 2 : 0,     // 本地不重试，避免掩盖真实不稳定
  use: {
    baseURL: 'http://127.0.0.1:4173',
    trace: 'on-first-retry',           // 首次重试时录制完整追踪，便于复盘
    video: 'retain-on-failure',
  },
  webServer: {                         // 测试前自动拉起预览服务
    command: 'npm run preview',
    url: 'http://127.0.0.1:4173',
    reuseExistingServer: !process.env.CI,
  },
  projects: [{ name: 'chromium', use: { ...devices['Desktop Chrome'] } }],
})
```

```ts
// e2e/todo.spec.ts
import { test, expect } from '@playwright/test'

test('新增待办后出现在列表首位', async ({ page }) => {
  await page.goto('/')
  await page.getByPlaceholder('要做什么').fill('买牛奶')
  await page.getByRole('button', { name: '添加' }).click()

  await expect(page.getByRole('listitem')).toHaveCount(1)   // 自动重试直至超时
  await expect(page.getByRole('listitem').first()).toContainText('买牛奶')
})
```

稳定性三原则：其一，定位器只用角色、标签、可见文本，不用 `nth-child` 与随构建变化的类名；其二，禁止 `page.waitForTimeout` 这类硬等待，改用会轮询的 `expect(locator)` 断言；其三，用例之间隔离数据，靠接口 mock、独立测试账号或每个用例前重置存储，避免「单独跑通过、批量跑失败」。需要让页面处于特殊状态（空列表、接口报错、慢响应）时，用 `page.route('**/api/todos', route => route.fulfill({ json: [] }))` 在浏览器层拦截请求，比在后端造数据更快也更稳定。失败后打开 `npx playwright show-trace`，能看到每一步截图、DOM 快照与网络请求，这是它相对无头脚本的最大优势。

### 12.6 CI 流水线：把检查变成不可绕过的门禁

持续集成（Continuous Integration，每次提交自动执行构建与检查并给出结论的实践）的价值不在「自动跑一遍」，而在「不通过就合不进去」。下面是一份可直接使用的 GitHub Actions 配置：

```yaml
name: ci
on:
  pull_request:
  push:
    branches: [main]

jobs:
  verify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with: { version: 9 }
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: pnpm                    # 缓存 pnpm store，命中键含锁文件哈希
      - run: pnpm install --frozen-lockfile   # 锁文件与 package.json 不一致时直接失败
      - run: pnpm lint
      - run: pnpm typecheck              # 内部是 tsc --noEmit
      - run: pnpm test -- --coverage
      - run: pnpm build
      - run: pnpm exec playwright install --with-deps chromium
      - run: pnpm exec playwright test
      - uses: actions/upload-artifact@v4
        if: failure()                    # 只在上游失败时上传报告，省存储
        with:
          name: playwright-report
          path: playwright-report/
```

设计要点：步骤按「便宜且快失败」排序，lint 挂了就不必等构建；`--frozen-lockfile` 保证 CI 装的依赖与本地完全一致，缺了它就会出现「本地能跑、CI 报错」；端到端测试放在最后，因为它最慢也最贵；上传失败产物是排障的关键，没有追踪文件的 CI 只会输出一句「超时」。需要同时支持多个 Node 版本或浏览器时用 `strategy.matrix` 展开，矩阵会让任务数成倍增长，只保留真正要兼容的组合。此外应在仓库设置里开启分支保护，把 `verify` 设为必过检查，否则流水线再漂亮也是建议而非门禁。

### 12.7 部署：静态托管、SPA 回退与缓存

前端构建产物（`dist`）本质上是一堆静态文件，任何能托管文件的地方都能上线：Vercel、Netlify、Cloudflare Pages、对象存储加 CDN、一台 Nginx。区别主要在自动化程度。

Vercel（面向前端的托管平台，自动识别框架并完成构建、CDN 分发与预览部署）的几条实用规则：每个 PR 自动生成独立预览域名，方便产品提前验收；环境变量按 Production/Preview/Development 三档隔离；带 `VITE_` 前缀的变量会被编译进产物，因此只能放公开配置，密钥必须留在服务端。

单页应用（SPA）采用前端路由，直接访问 `/todos/1` 时服务器找不到对应文件会返回 404，需要把所有未命中路径回退到 `index.html`：

```json
{
  "rewrites": [{ "source": "/(.*)", "destination": "/index.html" }],
  "headers": [
    {
      "source": "/assets/(.*)",
      "headers": [{ "key": "Cache-Control", "value": "public, max-age=31536000, immutable" }]
    }
  ]
}
```

`index.html` 绝对不能长缓存，否则用户永远拿到旧版本的脚本引用；带内容哈希的 `/assets/*` 才可以 `immutable` 长缓存。用 Nginx 自建时对应的一行是 `try_files $uri $uri/ /index.html;`，原理完全相同。

上线后常见的三种异常，排查顺序可以固定下来：

| 现象 | 大概率原因 | 处理方式 |
| --- | --- | --- |
| 首页正常，刷新子路由 404 | 缺少 SPA 回退重写 | 加 rewrites 或 `try_files` |
| 页面空白、控制台报资源 404 | 部署在子路径但 `base` 未配置 | 构建时设置 `base: '/子路径/'`，改用相对资源路径 |
| 用户长时间看到旧版本 | `index.html` 被 CDN 缓存 | 单独给 HTML 设 `no-cache` 并清理 CDN 缓存 |

另外，`dist` 目录不要提交进仓库，构建产物应当由 CI 现场生成并加上版本号；产物里的 source map 是否需要公开，取决于是否希望线上报错可读，至少要在发布记录里保存一份对应版本的 source map。

### 12.8 项目实战：同一需求写 Vue 3 与 React 两版

需求固定为待办应用：输入新增、勾选完成、删除、按全部/未完成/已完成过滤、显示剩余条数、localStorage 持久化。先定义与框架无关的数据模型，这是两个版本能共用一套端到端用例的前提：

```ts
// types.ts —— 两个版本共用
export interface Todo { id: string; title: string; done: boolean }
export type Filter = 'all' | 'active' | 'done'
```

Vue 3 版本用组合式 API，响应式数据直接改，派生值交给 `computed`：

```vue
<script setup lang="ts">
import { ref, computed, watch } from 'vue'
import type { Todo, Filter } from './types'

const todos = ref<Todo[]>(JSON.parse(localStorage.getItem('todos') || '[]'))
const filter = ref<Filter>('all')
const draft = ref('')

// 派生数据不另存 state，避免两份数据不同步
const visible = computed(() => todos.value.filter(t =>
  filter.value === 'all' ? true : filter.value === 'active' ? !t.done : t.done))
const left = computed(() => todos.value.filter(t => !t.done).length)

function add() {
  const title = draft.value.trim()
  if (!title) return                       // 空输入直接返回
  todos.value.push({ id: crypto.randomUUID(), title, done: false })
  draft.value = ''
}
function remove(id: string) {
  todos.value = todos.value.filter(t => t.id !== id)   // 整体替换，引用变化触发更新
}

watch(todos, v => localStorage.setItem('todos', JSON.stringify(v)), { deep: true })
</script>

<template>
  <input v-model="draft" placeholder="要做什么" @keyup.enter="add" />
  <button @click="add">添加</button>
  <ul>
    <li v-for="t in visible" :key="t.id">
      <input type="checkbox" v-model="t.done" :aria-label="t.title" />
      <span :class="{ done: t.done }">{{ t.title }}</span>
      <button @click="remove(t.id)">删除</button>
    </li>
  </ul>
  <p>剩余 {{ left }} 项</p>
</template>
```

React 版本用不可变更新，任何数组操作都产出新引用：

```tsx
import { useEffect, useMemo, useState } from 'react'
import type { Todo, Filter } from './types'

export default function TodoApp() {
  const [todos, setTodos] = useState<Todo[]>(
    () => JSON.parse(localStorage.getItem('todos') || '[]'))   // 惰性初始化，只读一次
  const [filter, setFilter] = useState<Filter>('all')
  const [draft, setDraft] = useState('')

  const visible = useMemo(() => todos.filter(t =>
    filter === 'all' ? true : filter === 'active' ? !t.done : t.done), [todos, filter])
  const left = todos.filter(t => !t.done).length

  useEffect(() => { localStorage.setItem('todos', JSON.stringify(todos)) }, [todos])

  function add() {
    const title = draft.trim()
    if (!title) return
    setTodos(prev => [...prev, { id: crypto.randomUUID(), title, done: false }])
    setDraft('')
  }
  const toggle = (id: string, done: boolean) =>
    setTodos(prev => prev.map(t => (t.id === id ? { ...t, done } : t)))
  const remove = (id: string) => setTodos(prev => prev.filter(t => t.id !== id))

  return (
    <div>
      <input value={draft} placeholder="要做什么" onChange={e => setDraft(e.target.value)} />
      <button onClick={add}>添加</button>
      <ul>
        {visible.map(t => (
          <li key={t.id}>
            <input type="checkbox" checked={t.done} aria-label={t.title}
              onChange={e => toggle(t.id, e.target.checked)} />
            <span style={{ textDecoration: t.done ? 'line-through' : 'none' }}>{t.title}</span>
            <button onClick={() => remove(t.id)}>删除</button>
          </li>
        ))}
      </ul>
      <p>剩余 {left} 项</p>
    </div>
  )
}
```

两版的关键差异集中在一张表里：

| 关注点 | Vue 3 | React |
| --- | --- | --- |
| 状态声明 | `ref` / `reactive`，读写靠 `.value` | `useState`，只能通过设置函数改 |
| 派生数据 | `computed` 自动追踪依赖 | `useMemo` 手动写依赖数组 |
| 数组更新 | 可直接 `push`、原地修改 | 必须展开生成新数组、新对象 |
| 双向绑定 | `v-model` 一行完成 | 受控组件写 `value` + `onChange` |
| 持久化 | `watch` 监听深度变化 | `useEffect` 依赖 `todos` |
| 列表渲染 | `v-for` + `:key` | `map` + `key` |

同一个需求在两套框架下页面结构与可访问名称完全一致，所以那套基于 `getByRole`、`getByPlaceholder` 的 Playwright 用例可以原样跑在 Vue 版和 React 版上，只需改 `baseURL`。这是「按用户可见语义定位」在工程上的直接回报：测试成了跨框架的资产，而不是框架的附属品。反过来说，如果用例大量依赖 `div.app > ul > li:nth-child(2)`，换框架时就等于从零重写。

### 12.9 学习路径

前端知识密度高，按项目倒推比按书目录顺读有效得多。一条可执行的六周路径：

1. 第一至二周：HTML 语义化、CSS 盒模型与 Flex、JavaScript 作用域/闭包/事件循环/Promise，产出是能独立写一个静态响应式页面。
2. 第三周：选定一个框架（Vue 3 或 React 之一），掌握组件、props、状态、列表与条件渲染，产出是可以交互的待办应用。
3. 第四周：路由、状态管理、请求封装与错误处理，产出是接真实接口的多页面应用。
4. 第五周：TypeScript、构建工具、ESLint 与单元测试，产出是带 CI 的仓库。
5. 第六周：端到端测试、部署上线、性能自查（LCP、首屏体积），产出是一个可分享的线上地址。

判断「是否学会」的三个信号：不看文档能写出增删改查并处理加载与错误状态；看到报错能定位到是类型、异步还是渲染问题；修 bug 时会先补一条能复现的测试。框架层面的细节可以随时查文档，工程习惯却必须靠完整跑通一次「开发—测试—上线」闭环才能建立。

选资料的原则同样围绕项目：官方文档优先于二手教程，遇到版本差异以官方迁移指南为准；不要同时学 Vue 2 与 Vue 3，也不要同时学两套框架的源码，先把一门用熟到能独立交付，再横向对照另一门，差异才会变成理解而不是负担。

### 12.10 常见坑

- **只靠端到端测试兜底**：几十条浏览器用例又慢又难定位，正确的比例是大量单元测试加少量关键链路端到端测试。
- **断言实现细节**：断言内部 state 名字、私有方法调用次数、`nth-child` 结构，重构即全红，测试变成维护负担。
- **CI 里跳过 lint 或使用非冻结锁文件**：前者让风格与潜在错误进入主干，后者让「本地能跑、CI 失败」反复发生。
- **SPA 上线后刷新 404、`index.html` 被长缓存**：缺少回退重写与正确缓存头，用户会看到 404 或永远停留在旧版本。

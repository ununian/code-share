---
# try also 'default' to start simple
theme: unicorn
#colorSchema: 'light'

# random image from a curated Unsplash collection by Anthony
# like them? see https://unsplash.com/collections/94734566/slidev
# apply any windi css classes to the current slide
class: 'text-center'
# https://sli.dev/custom/highlighters.html
highlighter: shiki
# show line numbers in code blocks
lineNumbers: true

drawings:
  persist: false
# use UnoCSS
css: unocss
website: 'ligai.cn'
handle: '孙运天'
gradientColors: ['#A21CAF', '#5B21B6']
---

# 2022 年 10 月 技术分享

---

# 目录

- 工程化 - 编写同时兼容 Vue2 与 Vue3 的代码
- 编码技巧 - 利用 **代数数据类型** 精简代码
- 业务实践 - 利用 **RxJS** 简化 WebSocket 的使用

---
layout: full
---

# 编写同时兼容 Vue2 与 Vue3 的代码

现在的评论编辑器、评论的附件展示以及富文本编辑器都可以在 Vue2(Web) 与 Vue3(VSCode、Idea) 中使用

<div class="grid gap-x-4 grid-cols-3">

PC
![PC](/assets/%E8%AF%84%E8%AE%BA-web.png)

VSCode
![PC](/assets/%E8%AF%84%E8%AE%BA-vscode.png)

IDEA
![PC](/assets/%E8%AF%84%E8%AE%BA-idea.png)

</div>

好处不言而喻，一个是可以在不同 Vue 版本的工程中间共享代码，另一个为以后升级 Vue3 减少阻碍

---
layout: full
---

## 原理

兼容工作由 2 部分完成

<div class="grid gap-x-4 gap-y-4 grid-cols-2">

### 编译阶段

### 运行阶段

负责根据使用的项目环境，自动选择使用 Vue2 或 Vue3 的 API

转换 createElement 函数的参数，使得 Vue2 与 Vue3 的参数格式一致

> - 使用的时候我们只需要从 Vue-Demi 里面 import 需要使用的 Api，就会自动根据环境进行切换
> - 分为 在浏览器中运行（IIFE） 和 使用打包工具（cjs、umd、esm）的两种情况

> - Vue2 和 Vue3 Composition Api 的区别非常小
> - Vue2 和 Vue3 运行时 Api 最大的区别就在于 createElement 函数的参数格式不一致，Vue3 已经换成了 React JSX 的格式

</div>

---
layout: full
---

## 编译阶段-IIFE

先说简单的，IIFE 的情况下，在 `window` 中定义一个 VueDemi 的变量，然后检查 `window` 中的 Vue 变量的版本，根据版本 re-export 对应的Api

核心代码在 https://github.com/vueuse/vue-demi/blob/main/lib/index.iife.js 

<div class="overflow-y-auto">

```js
var VueDemi = (function (VueDemi, Vue, VueCompositionAPI) {
  // Vue 2.7 有不同，这里只列出 2.0 ~ 2.6 的版本
  if (Vue.version.slice(0, 2) === '2.') {
    for (var key in VueCompositionAPI) {
      VueDemi[key] = VueCompositionAPI[key]
    }
    VueDemi.isVue2 = true
  } else if (Vue.version.slice(0, 2) === '3.') {
    for (var key in Vue) {
      VueDemi[key] = Vue[key]
    }
    VueDemi.isVue3 = true
  }
  return VueDemi
})(this.VueDemi,this.Vue,this.VueCompositionAPI)
```
</div>

---
layout: full
---

## 编译阶段-打包工具

利用 npm postinstall 的 hook，检查本地的 Vue 版本，然后根据版本 reexport 对应的 api

核心代码在 https://github.com/vueuse/vue-demi/blob/main/scripts/postinstall.js 以及 utils.js。

```js
const Vue = loadModule('vue') // 这里是检查本地的 vue 版本
if (Vue.version.startsWith('2.')) {
  switchVersion(2)
}
else if (Vue.version.startsWith('3.')) {
  switchVersion(3)
}

function switchVersion(version, vue) {
  copy('index.cjs', version, vue)
  copy('index.mjs', version, vue)
}
// VueDemi 自己的 lib 目录下有 v2 v3 v2.7 三个文件夹，分别对应不同的 Vue 版本，Copy 函数的功能就是把需要的版本复制到 lib 目录下
// 然后在 package.json 里面指向 lib/index.cjs 和 lib/index.mjs
function copy(name, version, vue) {
  const src = path.join(dir, `v${version}`, name)
  const dest = path.join(dir, name)
  fs.write(dest, fs.read(src))
}

```
---
layout: full
---

## 运行阶段 createElement 函数的区别

<div class="grid gap-x-4 gap-y-0 grid-cols-2">

### Vue 2

### Vue 3

```ts
h(LayoutComponent, {
    staticClass: 'button',
    class: { 'is-outlined': isOutlined },
    staticStyle: { color: '#34495E' },
    style: { backgroundColor: buttonColor },
    attrs: { id: 'submit' },
    domProps: { innerHTML: '' },
    on: { click: submitForm },
    key: 'submit-button',
    // 这里只考虑 scopedSlots 的情况了
    // 之前的 slots 没必要考虑，全部用 scopedSlots 是一样的
    scopedSlots: { 
      header: () => h('div', this.header),
      content: () => h('div', this.content),
    },
  }
);
```

```ts
h(LayoutComponent, {
    class: ['button', { 'is-outlined': isOutlined }],
    style: [{ color: '#34495E' }, { backgroundColor: buttonColor }],
    id: 'submit',
    innerHTML: '',
    onClick: submitForm,
    key: 'submit-button',
  }, {
    header: () => h('div', this.header),
    content: () => h('div', this.content),
  }
);
```

attrs 需要需要写在 attrs 属性中

attrs 和 props 一样，只需要写在最外层就行

`on: { click=> {} }`

`onClick: ()=> {}`

scopedSlots 写在 scopedSlots 属性中

slot 写在 createElement 函数第三个参数中

</div>

<style>
p {
  margin-bottom: 0.5rem !important;
}
</style>

---
layout: full
---

<div class="overflow-y-auto">


```ts
import { h as hDemi, isVue2 } from 'vue-demi';

// 我们使用的时候使用的 Vue2 的写法，但是 props 还是写在最外层，为了 ts 的智能提示
export const h = (
  type: String | Record<any, any>,
  options: Options & any = {},
  children?: any,
) => {
  if (isVue2) {
    const propOut = omit(options, [
      'props',
      // ... 省略了其他 Vue 2 的默认属性如 attrs、on、domProps、class、style
    ]);
    // 这里提取出了组件的 props
    const props = defaults(propOut, options.props || {}); 
    if ((type as Record<string, any>).props) {
      // 这里省略了一些过滤 attrs 和 props 的逻辑，不是很重要
      return hDemi(type, { ...options, props }, children);
    }
    return hDemi(type, { ...options, props }, children);
  }

  const { props, attrs, domProps, on, scopedSlots, ...extraOptions } = options;

  const ons = adaptOnsV3(on); // 处理事件
  const params = { ...extraOptions, ...props, ...attrs, ...domProps, ...ons }; // 排除 scopedSlots

  const slots = adaptScopedSlotsV3(scopedSlots); // 处理 slots
  if (slots && Object.keys(slots).length) {
    return hDemi(type, params, {
      default: slots?.default || children,
      ...slots,
    });
  }
  return hDemi(type, params, children);
};

const adaptOnsV3 = (ons: Object) => {
  if (!ons) return null;
  return Object.entries(ons).reduce((ret, [key, handler]) => {
    // 修饰符的转换
    if (key[0] === '!') {
      key = key.slice(1) + 'Capture';
    } else if (key[0] === '&') {
      key = key.slice(1) + 'Passive';
    } else if (key[0] === '~') {
      key = key.slice(1) + 'Once';
    }
    key = key.charAt(0).toUpperCase() + key.slice(1);
    key = `on${key}`;

    return { ...ret, [key]: handler };
  }, {});
};

const adaptScopedSlotsV3 = (scopedSlots: any) => {
  if (!scopedSlots) return null;
  return Object.entries(scopedSlots).reduce((ret, [key, slot]) => {
    if (isFunction(slot)) {
      return { ...ret, [key]: slot };
    }
    return ret;
  }, {} as Record<string, Function>);
};

```
</div>

---


## 后续思考

如果支持 SFC 和 JSX 呢？

- 需要在编译时替换 createElement 函数
- 需要深入了解一下 vite 和 rollup 的插件机制和编译配置

---

# 利用 **代数数据类型** 精简代码

---
layout: full
---

## 说在前面
- 我认为衡量一段代码复杂度的标准就是看状态的数量
- 状态越少，代码越简单。状态数量越多，代码越复杂，越容易出错

- 在 Vue 组件内部
  - 可以简单的认为 data 里面的变量就是状态，props、计算属性都**不是状态**
  - Composition Api 中 ref 和 reactive 是状态，而 computed 都**不是状态**
  - 接口数据可以理解成异步的计算属性

---
layout: full
---

## 概念

- **状态** 可以由系统内部的行为更改的数据
- **状态大小** 状态所有可能的值的集合的大小，记作 size(State)
- **代码的复杂度** = States.reduce((acc, cur) => acc * size(cur), 1)

#
#### 一些常见的数据类型的状态大小

- unit: 1，在前端里面可以是 null、undefined，所有的常量、非状态，它的大小也是 1
- Boolean: 2
- Number 和 String，这种可以有很多甚至是无限多个值的类型，要怎么计算他们的状态大小呢？
  - 首选我们明确一点，我们只关心状态在业务逻辑中的意义，而不是他的具体值，只区分可以影响到业务逻辑的状态值就行
  - 一个接口返回的数据是一个数字，但是我们只关心这个数字是正数还是负数，那么这个数字的状态大小就是 2
  - 例如一个数字输入组件值，最大值是10，超过 10 需要报错。那这个 Number 就是 2 种状态，小于等于10和大于10，至于具体是多少，屏幕显示的是多少，对于业务来说意义不是特别大
  - Issue详情的 Action 有 3 种状态，Create Edit Preview，虽然数据类型是 String，所以 action 的大小是 3。这也就是复合类型种的枚举类型

---
layout: full
---


## 复合类型的状态大小

有 2 种情况

<div class="grid mt-4 gap-x-4 gap-y-0 grid-cols-2">

### 和类型

### 积类型

```ts
Type Speed = 
  | Slow 
  | Fast
```

```ts
Type Velocity = {
  direction: Direction
  speed: Speed 
}
```

`size(C) = size(A) + size(B)`

`size(C) = size(A) * size(B)`

枚举

键值对、对象

</div>

---
layout: full
---

# 例子 评论编辑器的显示控制

> - 每个评论都有 2 个编辑器，一个用来回复评论，一个用来编辑评论。
> - 并且同一时间最多只允许一个活动的编辑器。


<div class="grid gap-x-4 mt-8 grid-cols-2">

回复评论

编辑评论

![回复](/assets/%E8%AF%84%E8%AE%BA-%E5%9B%9E%E5%A4%8D.png)


![编辑](/assets/%E8%AF%84%E8%AE%BA-%E7%BC%96%E8%BE%91.png)

</div>

---
layout: full
---

## 优化前的做法

- 为回复组件定义两个变量 IsShowReply 和 IsShowEdit，然后通过 v-if 来控制是不是显示编辑器
- 当点击按钮的时候，以点回复为例
  - 1. 判断自己的 IsShowReply 是否为 true，如果是就直接返回不做处理
  - 2. 判断自己的 IsShowEdit， 如果是 true 则修改为 false
  - 3. 依次设置所有其他评论组件的 IsShowReply 和 IsShowEdit 为 false
    - 这个获取其他评论组件的地方就有坑，之前的代码里面用的是 `document.querySelectAll` 然后用 `__vue__` 这种比较 hack 的方法来查询其他的组件
    - 这里问题其实很多，一个是修改状态，还依赖了 dom，非常不优雅
  - 4. 修改自己的 IsShowReply 为 true

<div class="mt-2"></div>


#### 那这种写法，在评论组件有10个的情况下，状态数量会是多少呢？

<div v-click>

`size(CommentComponent) = size(Boolean) * size(Boolean) = 2 * 2 = 4`

</div>

<div v-click>

`size(total) = size(CommentComponent) ^ count(CommentComponent) = 4 ^ 10 = 1048576`

</div>

<v-click>

> - 虽然在逻辑上互斥，但是在代码层面这些组件是毫无关系的，而且也确实可以全部设置成 true，如果代码出问题（包括写错），没处理好互斥，那这种情况这完全有可能出现。
> - 因为处理互斥还涉及到查找 dom 和 组件，那出问题的几率是大大提高的。

</v-click>

---
layout: full
---

## 优化后的做法

- 在 store 中定义一个变量 activeComment，用来表示当前活动的评论组件及其类型。

```ts
type CommentId = number;
type ActiveCommentStatus = `${'Edit' | 'Reply'}${CommentId}` | 'Close';
let activeComment: ActiveCommentStatus = 'Close';
```

- 除了 'Close' 外，由 2 部分组成
  - 第一部分是说明当前是 编辑 还是 回复
  - 第二部分是评论的 id
- 在组件使用时 v-if="currentEditComment === `Edit${id}`" 和 v-if="currentEditComment === `Reply${id}`" 
- 那按钮的回调函数要怎么写呢？

<v-clicks>

  - 以点回复为例，设置 currentEditComment = `Reply${id}`
  - over，就这么简单，没有判断，没有dom，没有其他组件

</v-clicks>

<v-click>

#### 那这种写法，在评论组件有10个的情况下，状态数量会是多少呢？

这里 id 虽然是 number，但是对于前端来说就是一个常量，只可能有一种状态

`size(total) = size('Reply' | 'Edit') * count(Comment) + size('Close') = 2 * 10 + 1 = 21`

> - 在实际使用中，我们发现确实有 21 种状态。然后在代码层面，我们也精准的控制了只可能存在这21种状态。所以要出错也不大可能。

</v-click>

---
layout: full
---

# 题外话

如果有人说前面的 Edit 和 Reply 是字符串，可能会打错，例如变成了 Editttt，那怎么办呢？ 

那就需要 TypeScript 的模板字符串类型来帮助你了

<div>


```ts {monaco}
type CommentId = number;
type ActiveCommentStatus = `${'Edit' | 'Reply'}${CommentId}` | 'Close';
let activeComment: ActiveCommentStatus = 'Close';
activeComment = 'Edit123'; // ok
activeComment = 'Reply123'; // ok
activeComment = 'Close'; // ok

activeComment = 'Editttt'; // error
activeComment = 'Edit22a'; // error

```

</div>

---

# 使用 **RxJS** 简化 WebSocket 的使用

---
layout: full
---

# 概念 Observable

- 一个可观察对象，是一个未来会发出值或者事件的集合

```js
var button = document.querySelector('button');
Rx.Observable.fromEvent(button, 'click')
  .subscribe(() => console.log('Clicked!')); // 每次触发 click 事件，都会执行这个回调
```

- 可以简单的理解成一个事件源，在特定情况下会发出一些值
- 在我们的例子中，就是 WebSocket 源源不断传过来的消息的集合。

- 可以转换为计算属性
	- 对外表现都一样，都是一个在未来可能发送改变的值
	-	计算属性使用 watch 来监听
	-	Observable 使用 subscribe 来监听

---
layout: full
---

# 概念 Subject 

- 一种特殊的 Observable，在我们项目里面可以理解成可以手动触发事件的 Observable。
- 可以转换成 ref
  - 如果把最后一个触发的事件当做最新的值，那和 ref 就一模一样了，都是一个变量

```js
const subject = new Subject();

subject.subscribe({
  next: (v) => console.log('observerA: ' + v)
});
subject.subscribe({
  next: (v) => console.log('observerB: ' + v)
});

subject.next(1);
subject.next(2);
```

- 输出

```
observerA: 1
observerB: 1
observerA: 2
observerB: 2
```

---
layout: full
---

# 概念 Operator 

### Operator 操作符 

- 操作符是 Observable 类型上的方法
- 是用来组合 Observable 的基本单元，是 RxJS 函数式编程的基础，是 RxJS 的核心
  - 顺便说一句，VueUse 很多响应式相关的 api 都是抄的 RxJS 的
- 操作符是惰性求值，每个操作符都返回新的 Observable，只有最后调用 subscribe 才会真正执行

### Pipe 管道

- 管道由多个操作符组成

```js
const observer = from([1, 2, 3, 4, 5, 6, 7, 8]); // 创建一个连续发出 1-8 的 Observable

observer.pipe(
  filter(x => x % 2 === 0), // 筛选出偶数
  map(x => x + 1), // 每个值加 1
  skip(1) // 跳过第一个值
).subscribe({
  next: (v) => console.log(v)
});

```

- 输出

```
5 7 9
```
---
layout: full
---

# RxJS 与 Vue 交互

可以使用 VueUse 的 RxJS 模块，这里介绍 2 个比较常用的

#### from 

把 `Ref<T>` 转换成 `Observable<T>`

```ts
const count = ref(0)
from(count).subscribe({
  next: (v) => console.log(v)
}) 
count.value++;
// 输出 1

```

#### useObservable
从 `Observable<T>` 转成 `Readonly<Ref<T>>`

```ts
const count = useObservable(
  interval(1000) // 创建一个定时器 Observable，每秒发出一个值
)

watch(count, (val) => console.log(val)) // 每秒触发一次，val 是当前毫秒数
```

---
layout: full
---

# 例子 评论 WebSocket 处理

### 需求

- 1. 评论列表实时更新
- 2. 能接收 WebSocket 的消息，也需要从本地操作中更新数据
  - 就是说 WebSocket 失效的情况下，也不影响本地操作
- 3. 初始的评论是从接口获取的， WebSocket 只会推送更新数据

---
layout: full
---

### 整体设计
- 将本地的操作也当作一种消息，每次操作完后生成就一条 WebSocket 格式的消息，存入本地消息队列中
- 将 WebSocket 消息和本地消息合并
- 过滤掉重复、无效的消息，并 **归并** (reduce) 成一个所有消息的数组
- 将消息数组与接口数据转换成最终的评论列表
  - 这里的转换是一个 **reduce** 操作，这一步是**没有副作用**的，所以可以放在 computed 中
  - 没有副作用的意思是，不会修改原始数据，只会生成新的数据
  - 接口数据是**原始状态**(State)，消息是**操作**(Commit)，计算结果就只需要把 Commit 依次应用到 State 上就可以了
  - 其实就有限状态机，Redux、Vuex 的设计思路也是如此，也就是函数式编程的思想

#

### 消息源

- WebSocket 消息的 Observable
- 本地消息的 Subject
- 接口获取到的评论数组，其实原始数组和消息部分关系不大，最后计算所有评论的时候加上就行

#
### 输出

- 实时更新的评论列表

---
layout: full
---

### 管道设计

- 1. **merge** - 合并本地消息与 WebSocket 消息
- 2. **filter** - 过滤掉不支持的 action 和 msgType 的消息
- 3. **filter** - 过滤掉不属于当前 Issue 的消息
- 4. **distinctUntilChanged** - 过滤 Id 相同的非更新消息。因为创建、回复、删除操作都只需要一条消息就行了
- 5. **map** - 做一些转换处理，主要是数据兼容问题，回复接口和 WebSocket 的数据结构有细微的差别
- 6. **reduce** - 将消息归并成一个数组，这里面也有一个去重逻辑，如果是更新消息，那么会取时间最新的那条

---
layout: full
---

#### 消息部分

<div class="overflow-y-auto">


```ts
const localMessages = new Subject<IssueCommentWSMessageResponse>();
const receiveCommentMessages = useObservable( // 把 SwitchMap 转换成的 Observable 转换成一个 Readonly<Ref>
  // combineLatest 是组合 2 个 Observable，当任意一个 Observable 发出值时，就会触发回调
  // actionStore.action 是一个 Ref，每次打开详情弹框都会变
  // commentRefreshFlag 是一个 Ref，手动刷新评论列表时会变
  combineLatest([from(actionStore.action), from(commentRefreshFlag)]).pipe( 
    switchMap(([action, flag]) => { // switchMap 用于将 combineLatest 的 Observable，转换为另外一个 Observable
      if (action?.type === 'Edit' && !isVisitor.value) { // 如果是编辑操作才会有评论
        return socket
          // 将 global:comment 事件转换为 Observable，这和 button 的 click 事件是一样的
          .on$<IssueCommentWSMessageResponse>('global:comment') 
          // 消息处理管道开始
          .pipe(
            mergeWith(localMessages.asObservable()), // 第1步 合并 WebSocket 消息和本地消息
            filter((msg) => { // 第2步 过滤掉不支持的 action 和 msgType 的消息
              return !!(
                msg?.result &&
                ['Issue.comment'].includes(msg.msgType) &&
                ['reply', 'add', 'update', 'delete'].includes(msg.action)
              );
            }),
            filter((msg) => { // 第3步 过滤掉不属于当前 Issue 的消息
              return (
                msg.action === 'delete' || // 删除消息没有 linkId
                msg.result.linkId === action.issueId
              );
            }),
            distinctUntilChanged((a, b) => { // 第4步 过滤 Id 相同的非更新消息
              if (a.action === 'update' || b.action === 'update') {
                return false; 
              }
              if (a.action === b.action) {
                return a.result.commentId === b.result.commentId;
              }
              return false;
            }),
            map((res) => { // 第5步 做一些转换处理
              // ... 一些数据转换逻辑
              return res; 
            }),
            scan((acc, res) => { // 第6步 将消息归并成一个数组
              if (['reply', 'add', 'delete'].includes(res.action)) {
                const index = acc.findIndex(
                  (c) =>
                    res.result.commentId === c.result.commentId &&
                    res.action === c.action
                );
                if (index !== -1) {
                  // 如果是更新消息，那么会取时间最新的那条
                  if (res.result.updateTime > acc[index].result.updateTime) {
                    acc[index] = res;
                  }
                  return acc;
                }
              }

              acc.push(res);
              return acc;
            }, [] as IssueCommentWSMessageResponse[]),
            // 为了保证第一次有值，所以这里加了一个 startWith。
            // 如果没有 startWith，则刚进页面的时候，没有本地和 WS 消息，那这个 Observable 就不会发出值，下面的计算属性就不会重新计算
            startWith([] as IssueCommentWSMessageResponse[]) 
          );
      }
      return of([]); // 如果不是编辑操作，那么就返回空数组
    })
  ),
  {
    initialValue: [] as IssueCommentWSMessageResponse[]
  }
);

```

</div>

---
layout: full
---

#### 组合部分

<div class="overflow-y-auto">


```ts
const compositionComments = computed(() => {
  const addedComments = receiveCommentMessages.value  // 所有的新增评论
    .filter((msg) => msg.action === 'add')
    .map((msg) => msg.result);

  const addedReplies = receiveCommentMessages.value  // 所有的新增回复
    .filter((msg) => msg.action === 'reply')
    .map((msg) => msg.result as unknown as IssueCommentReply);

  const deleteComment = receiveCommentMessages.value // 所有的需要删除的评论、回复
    .filter((msg) => msg.action === 'delete')
    .map((msg) => msg.result.commentId);

  // 所有的需要更新的评论、回复的 Map。key 是 commentId，value 是最新的评论
  const modifiedCommentMap = receiveCommentMessages.value.reduce(
    (map, msg) => {
      if (msg.action === 'update') {
        map.set(msg.result.commentId, msg.result);
      }
      return map;
    },
    new Map<number, CommentIssue>()
  );

  return orderBy( // 排序功能
    // 这里可以得到所有的评论（包括需要删除的），开始执行归并操作
    [...addedComments, ...(originCommentResponse.value?.list || [])]
      .reduce((arr, comment) => {
        // 如果有需要更新的评论，那么就用最新的评论替换掉旧的评论
        const modifiedComment =
          modifiedCommentMap.get(comment.commentId) || comment;

        // 这里是获取该评论的所有回复（包括需要更新、删除的），排序的目的是为了保证回复的回复在回复的后面
        const allReplies = orderBy(
          addedReplies,
          (r) => r.updateTime,
          'asc'
        ).reduce((arr, reply) => {
          if (
            reply.replyId === comment.commentId ||
            arr.some((r) => r.commentId === reply.replyId)
          ) {
            arr.push(reply);
          }
          return arr;
        }, clone(comment.subReplies)); // 原始的回复

        // 获取处理完更新后的所有回复，这里逻辑和评论一样
        const subReplies = orderBy(
          allReplies.reduce((subArr, subComment) => {
            const modifiedSubComment =
              modifiedCommentMap.get(subComment.commentId) || subComment;
            if (modifiedSubComment) {
              subArr.push({
                ...modifiedSubComment
              } as IssueCommentReply);
            }
            return subArr;
          }, [] as IssueCommentReply[]),
          'createTime',
          'asc'
        );

        // 如果评论在需要删除的列表里，那么讲 deleteFlag 设置成 1
        if (deleteComment.includes(modifiedComment.commentId)) {
          modifiedComment.deleteFlag = 1;
        }

        // 回复也要判断删除
        subReplies.forEach((reply) => {
          if (deleteComment.includes(reply.commentId)) {
            reply.deleteFlag = 1;
          }
        });

        // 将修改后的评论和回复放到数组里
        arr.push({
          ...modifiedComment,
          subReplies: subReplies.filter(
            (reply) =>
              reply.deleteFlag === 0 ||
              subReplies.some((r) => r.replyId === reply.commentId)
          )
        });
        return arr;
      }, [] as CommentIssue[])
      .filter(
        // 如果评论的 deleteFlag 是 1，而且没有回复，那么就过滤掉该评论
        (comment) => comment.deleteFlag === 0 || comment.subReplies.length
      ),
    'createTime',
    order.value
  );
});

```

</div>

---
layout: full
---

# 总结

- 首先大家可以自己想一下，如果自己做这个功能，会怎么做？代码量会是多少？
- 我认为这也算是一个比较复杂的功能
- 使用 RxJS 后，代码中的状态非常少，可以说只有本地消息列表一个状态，如果把本地消息也弄成事件的机制，那么基本上就没状态了。
- 而且本地消息数组这个只有一种操作，就是往里面添加模拟消息，没有删除、替换的地方，管理非常的简单。
- 代码量也非常少，大概在 200 行左右，而且非常容易理解
  - 逻辑代码都在管道里面执行，而且每个操作符各司其职，从上到下依次执行，没有混在一起
  - 这也是函数式编程以及 RxJS 最大的优点，不会修改已有变量，只会返回新值

---
layout: center
---

# Thank you
---
# try also 'default' to start simple
theme: unicorn
# random image from a curated Unsplash collection by Anthony
# like them? see https://unsplash.com/collections/94734566/slidev
# apply any windi css classes to the current slide
class: 'text-center'
# https://sli.dev/custom/highlighters.html
highlighter: shiki
# show line numbers in code blocks
lineNumbers: false

drawings:
  persist: false
# use UnoCSS
css: unocss
---

# 2022 年 10 月 技术分享

---

# 目录

- 工程化 - 编写同时兼容 Vue2 与 Vue3 的代码
- 编码技巧 - 利用 **代数数据结构** 精简代码
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

负责根据使用的项目环境，re-export Vue2 或 Vue3 的 API

转换 createElement 函数的参数，使得 Vue2 与 Vue3 的参数格式一致

> - 使用的时候我们只需要从 Vue-Demi 里面 import 需要使用的 Api，就会自动根据环境进行切换
> - 分为在浏览器中运行（IIFE） 和使用了打包工具（cjs、umd、esm）的两种情况

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
      return hDemi(type, { ...options, props });
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

# 利用 **代数数据结构** 精简代码

---
layout: full
---

## 说在前面
- 我认为衡量一段代码复杂度的标准就是看状态机内状态的数量
- 状态越少，代码越简单。状态数量越多，代码越复杂，越容易出错
- 状态指的是可以由**代码边界内**的行为**更改**的数据
  - 首先明确一点，**代码边界**是一个相对的概念，在讨论不同组件或者项目、代码块的时候他的**代码边界**是不一样的
  - 举例来说，接口的返回数据对于我们前端来说，它不能由我们前端修改，所以它不是状态。但是如果把视角放在我们整个系统来说，它就是一个状态了
  - 详情弹框的显示状态（编辑、新增），对于详情弹框内部来说，它是一个输入的数据，是不能修改的，所以他不是一个状态。但是对于前端项目来说，它是一个状态，因为我们可以通过点击按钮来修改它的值。
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

- unit: 1，在前端里面可以是 null、undefined
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

# 例子 评论编辑器的展示控制

> - 每个评论都有 2 个编辑器，一个用来回复评论，一个用来编辑评论。
> - 并且同一时间最多只允许一个活动的编辑器。


<div class="grid gap-x-4 mt-8 grid-cols-2">

![编辑](/assets/%E8%AF%84%E8%AE%BA-%E5%9B%9E%E5%A4%8D.png)

![回复](/assets/%E8%AF%84%E8%AE%BA-%E7%BC%96%E8%BE%91.png)

</div>

---
layout: full
---

## 优化前的做法

- 为回复组件定义两个变量 IsShowReply 和 IsShowEdit，然后通过 v-if 来控制是不是显示编辑器
- 当点击按钮的时候，以点回复为例
  1. 判断自己的 IsShowReply 是否为 true，如果是就直接返回不做处理
  2. 判断自己的 IsShowEdit， 如果是 true 则修改为 false
  3. 依次设置所有其他评论组件的 IsShowReply 和 IsShowEdit 为 false
    - 这个获取其他评论组件的地方就有坑，之前的代码里面用的是 `document.querySelectAll` 然后用 `__vue__` 这种比较 hack 的方法来查询其他的组件
    - 这里问题其实很多，一个是修改状态，还依赖了 dom，非常不优雅
  4. 修改自己的 IsShowReply 为 true

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
const activeComment = ref<ActiveCommentStatus>('Close');
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

```ts {monaco}
type CommentId = number;
type ActiveCommentStatus = `${'Edit' | 'Reply'}${CommentId}` | 'Close';

```
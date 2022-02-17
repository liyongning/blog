# 微前端框架 之 qiankun 从入门到源码分析

**当学习成为了习惯，知识也就变成了常识。**感谢各位的 **点赞**、**收藏**和**评论**。

新视频和文章会第一时间在微信公众号发送，欢迎关注：[李永宁lyn](https://gitee.com/liyongning/typora-image-bed/raw/master/202202171742614.jpg)

文章已收录到 [github](https://github.com/liyongning/blog)，欢迎 Watch 和 Star。

## 封面

![](https://gitee.com/liyongning/typora-image-bed/raw/master/202202041054955.jpg)

## 简介

从 single-spa 的缺陷讲起 -> qiankun 是如何从框架层面解决 single-spa 存在的问题 -> qiankun 源码解读，带你全方位刨析 qiankun 框架。

## 介绍

qiankun 是基于 single-spa 做了二次封装的微前端框架，通过解决了 single-spa 的一些弊端和不足，来帮助大家实现更简单、无痛的构建一个生产可用的微前端架构系统。

[微前端框架 之 single-spa 从入门到精通](https://mp.weixin.qq.com/s?__biz=MzA3NTk4NjQ1OQ==&mid=2247484245&idx=1&sn=9ee91018578e6189f3b11a4d688228c5&chksm=9f696021a81ee937847c962e3135017fff9ba8fd0b61f782d7245df98582a1410aa000dc5fdc&token=1002082546&lang=zh_CN#rd) 通过从 基本使用 -> 部署 -> 框架源码分析 -> 手写框架，带你全方位刨析 single-spa 框架。

因为 qiankun 是基于 single-spa 做的二次封装，主要解决了 single-spa 的一些痛点和不足，所以最好对 single-spa 有一个全面的了解和认识，明白其原理、了解它的不足和缺陷，然后带着问题和目的去阅读 qiankun 源码，可以达到事半功倍的效果，整个阅读过程的思路也会更加清晰明了。

## 为什么不是 single-spa

如果你很了解 single-spa 或者阅读过 [微前端框架 之 single-spa 从入门到精通](https://mp.weixin.qq.com/s?__biz=MzA3NTk4NjQ1OQ==&mid=2247484245&idx=1&sn=9ee91018578e6189f3b11a4d688228c5&chksm=9f696021a81ee937847c962e3135017fff9ba8fd0b61f782d7245df98582a1410aa000dc5fdc&token=1002082546&lang=zh_CN#rd) ，你会发现 single-spa 就做了两件事，加载微应用（加载方法还是用户自己提供的）、维护微应用状态（初始化、挂载、卸载）。了解多了会发现 single-spa 虽好，但是却存在一些比较严重的问题

1. **对微应用的侵入性太强**

   single-spa 采用 JS Entry 的方式接入微应用。微应用改造一般分为三步：

   - 微应用路由改造，添加一个特定的前缀
   - 微应用入口改造，挂载点变更和生命周期函数导出
   - 打包工具配置更改

   侵入型强其实说的就是第三点，更改打包工具的配置，使用 single-spa 接入微应用需要将微应用整个打包成一个 JS 文件，发布到静态资源服务器，然后在主应用中配置该 JS 文件的地址告诉 single-spa 去这个地址加载微应用。

   不说其它的，就现在这个改动就存在很大的问题，将整个微应用打包成一个 JS 文件，常见的打包优化基本上都没了，比如：按需加载、首屏资源加载优化、css 独立打包等优化措施。

   项目发布以后出现了 bug ，修复之后需要更新上线，为了清除浏览器缓存带来的影响，一般文件名会带上 chunkcontent，微应用发布之后文件名都会发生变化，这时候还需要更新主应用中微应用配置，然后重新编译主应用然后发布，这套操作简直是不能忍受的，这也是 [微前端框架 之 single-spa 从入门到精通](https://mp.weixin.qq.com/s?__biz=MzA3NTk4NjQ1OQ==&mid=2247484245&idx=1&sn=9ee91018578e6189f3b11a4d688228c5&chksm=9f696021a81ee937847c962e3135017fff9ba8fd0b61f782d7245df98582a1410aa000dc5fdc&token=1002082546&lang=zh_CN#rd) 这篇文章中示例项目中微应用发布时的环境配置选择 development 的原因。

2. **样式隔离问题**

   single-spa 没有做这部分的工作。一个大型的系统会有很的微应用组成，怎么保证这些微应用之间的样式互不影响？微应用和主应用之间的样式互不影响？这时只能通过约定命名规范来实现，比如应用样式以自己的应用名称开头，以应用名构造一个独立的命名空间，这个方式新系统还好说，如果是一个已有的系统，这个改造工作量可不小。

3. **JS 隔离**

   这部分工作  single-spa 也没有做。 JS 全局对象污染是一个很常见的现象，比如：微应用 A 在全局对象上添加了一个自己特有的属性，`window.A`，这时候切换到微应用 B，这时候如何保证 `window` 对象是干净的呢？

4. **资源预加载**

   这部分的工作 single-spa 更没做了，毕竟将微应用整个打包成一个 js 文件。现在有个需求，比如为了提高系统的用户体验，在第一个微应用挂载完成后，需要让浏览器在后台悄悄的加载其它微应用的静态资源，这个怎么实现呢？

5. **应用间通信**

   这部分工作 single-spa 没做，它只在注册微应用时给微应用注入一些状态信息，后续就不管了，没有任何通信的手段，只能用户自己去实现

以上 5 个问题中第 2、3、5 还好说，可以通过一些方式来解决，比如采用命名空间的方式解决样式隔离问题， 通过备份全局对象，每次微应用切换时初始化全局对象的方式来解决 JS 隔离的问题，通信问题可以通过传递一些通信方法，这点依赖了 JS 对象本身的特性（传递的是引用）来实现；但是第一个和第四个就不好解决了，这是 JS Entry 方式带来的问题，要解决这个问题，难度相对就会大很多，工作量也会更大。况且这些通用的脏活累活就不应该由用户（框架使用者）来解决，而是由框架来解决。

## 为什么是 qiankun

上面说到，通用的脏活累活应该在框架层面去做，qiankun 基于 single-spa 做了二次封装，很好的解决了上面提到的几个问题。

1. **HTML Entry**

   qiankun 通过 HTML Entry 的方式来解决 JS Entry 带来的问题，让你接入微应用像使用 iframe 一样简单。

2. **样式隔离**

   qiankun 实现了两种样式隔离

   - 严格的样式隔离模式，为每个微应用的容器包裹上一个 `shadow dom` 节点，从而确保微应用的样式不会对全局造成影响
   - 实验性的方式，通过动态改写 `css` 选择器来实现，可以理解为 `css scoped` 的方式

3. **运行时沙箱**

   qiankun 的运行时沙箱分为 `JS` 沙箱和 `样式沙箱`

   `JS 沙箱` 为每个微应用生成单独的 `window proxy` 对象，配合 `HTML Entry` 提供的 JS 脚本执行器 (execScripts) 来实现 JS 隔离；

   `样式沙箱` 通过重写 `DOM` 操作方法，来劫持动态样式和 JS 脚本的添加，让样式和脚本添加到正确的地方，即主应用的插入到主应用模版内，微应用的插入到微应用模版，并且为劫持的动态样式做了 `scoped css` 的处理，为劫持的脚本做了 JS 隔离的处理，更加具体的内容可继续往下阅读或者直接阅读 [微前端专栏](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzA3NTk4NjQ1OQ==&action=getalbum&album_id=2251416802327232513&scene=173&from_msgid=2247484245&from_itemidx=1&count=3&nolastread=1#wechat_redirect) 中的 `qiankun 2.x 运行时沙箱 源码分析`

4. **资源预加载**

   qiankun 实现预加载的思路有两种，一种是当主应用执行 `start` 方法启动 qiankun 以后立即去预加载微应用的静态资源，另一种是在第一个微应用挂载以后预加载其它微应用的静态资源，这个是利用 single-spa 提供的 `single-spa:first-mount` 事件来实现的

5. **应用间通信**

   qiankun 通过发布订阅模式来实现应用间通信，状态由框架来统一维护，每个应用在初始化时由框架生成一套通信方法，应用通过这些方法来更改全局状态和注册回调函数，全局状态发生改变时触发各个应用注册的回调函数执行，将新旧状态传递到所有应用

## 说明

文章基于 `qiankun 2.0.26` 版本做了完整的源码分析，目前网上好像还没有 `qiankun 2.x` 版本的完整源码分析，简单搜了下好像都是 1.x 版本的

由于框架代码比较多的，博客有字数限制，所以将全部内容拆成了三篇文章，每一篇都可独立阅读：

- 微前端框架 之 qiankun 从入门到精通

  ，文章由以下三部分组成

  - `为什么不是 single-spa`，详细介绍了 single-spa 存在的问题
  - `为什么是 qiankun`，详细介绍了 qiankun 是怎么从框架层面解决 single-spa 存在的问题的
  - `源码解读`，完整解读了 qiankun 2.x 版本的源码

- [qiankun 2.x 运行时沙箱 源码分析](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzA3NTk4NjQ1OQ==&action=getalbum&album_id=2251416802327232513&scene=173&from_msgid=2247484245&from_itemidx=1&count=3&nolastread=1#wechat_redirect)，详细解读了 qiankun 2.x 版本的沙箱实现

- [HTML Entry 源码分析](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzA3NTk4NjQ1OQ==&action=getalbum&album_id=2251416802327232513&scene=173&from_msgid=2247484245&from_itemidx=1&count=3&nolastread=1#wechat_redirect)，详细解读了 HTML Entry 的原理以及在 qiankun 中的应用

## 源码解读

这里没有单独编写示例代码，因为 `qiankun` 源码中提供了完整的示例项目，这也是 `qiankun` 做的很好的一个地方，提供完整的示例，避免大家在使用时重复踩坑。

微前端实现和改造时面临的第一个困难就是主应用的设置、微应用的接入，`single-spa` 官方没有提供一个很好的示例项目，所以大家在使用 `single-spa` 接入微应用时还是需要踩不少坑的，甚至有些问题需要去阅读源码才能解决

### 框架目录结构

从 [github](https://github.com/liyongning/qiankun) 克隆项目以后，执行一下命令：

- 安装 `qiankun` 框架所需的包

  ```shell
  yarn install
  ```

- 安装示例项目的包

  ```shell
  yarn examples:install
  ```

以上命令执行结束以后：

![image-20220202220056482](https://gitee.com/liyongning/typora-image-bed/raw/master/202202022200737.png)

#### 有料的 package.json

- [npm-run-all](https://link.juejin.cn?target=https%3A%2F%2Fwww.npmjs.com%2Fpackage%2Fnpm-run-all)

  > 一个 CLI 工具，用于并行或顺序执行多个 npm 脚本

- [father-build](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fumijs%2Ffather)

  > 基于 rollup 的库构建工具，father 更加强大

- 多项目的目录组织以及 scripts 部分的编写

- main 和 module 字段

  标识组件库的入口，当两者同时存在时，module 字段的优先级高于 main

#### 示例项目中的主应用

这里需要更改一下示例项目中主应用的 `webpack` 配置

```javascript
{
  ...
  devServer: {
    // 从 package.json 中可以看出，启动示例项目时，主应用执行了两条命令，其实就是启动了两个主应用，但是却只配置了一个端口，浏览器打开 localhost:7099 和你预想的有一些出入，这时显示的是 loadMicroApp(手动加载微应用) 方式的主应用，基于路由配置的主应用没起来，因为端口被占用了
    // port: '7099'
		// 这样配置，手动加载微应用的主应用在 7099 端口，基于路由配置的主应用在 7088 端口
    port: process.env.MODE === 'multiple' ? '7099' : '7088'
  }
  ...
}
```

#### 启动示例项目

```shell
yarn examples:start
```

命令执行结束以后，访问 `localhost:7099` 和 `localhost:7088` 两个地址，可以看到如下内容：

![image-20220202220258551](https://gitee.com/liyongning/typora-image-bed/raw/master/202202022202760.png)

![image-20220202220401608](https://gitee.com/liyongning/typora-image-bed/raw/master/202202022204818.png)

到这一步，就证明项目正式跑起来了，所有准备工作就绪

### 示例项目

官方为我们准备了两种主应用的实现方式，五种微应用的接入示例，覆盖面可以说是比较广了，足以满足大家的普遍需要了

#### 主应用

主应用在 `examples/main` 目录下，提供了两种实现方式，基于路由配置的 `registerMicroApps` 和 手动加载微应用的 `loadMicroApp`。主应用很简单，就是一个从 0 通过 webpack 配置的一个同时支持 react 和 vue 的项目，至于为什么同时支持 react 和 vue，继续往下看

##### webpack.config.js

就是一个普通的 `webpack` 配置，配置了一个开发服务器 `devServer`、两个 `loader` (babel-loader、css loader)、一个插件 `HtmlWebpackPlugin` (告诉 webpack html 模版文件是哪个)

通过 `webpack` 配置文件的 `entry` 字段得知入口文件分别为 `index.js` 和 `multiple.js`

##### 基于路由配置

通用将微应用关联到一些 `url` 规则的方式，实现当浏览器 `url` 发生变化时，自动加载相应的微应用的功能

###### index.js

```javascript
// qiankun api 引入
import { registerMicroApps, runAfterFirstMounted, setDefaultMountApp, start, initGlobalState } from '../../es';
// 全局样式
import './index.less';

// 专门针对 angular 微应用引入的一个库
import 'zone.js';

/**
 * 主应用可以使用任何技术栈，这里提供了 react 和 vue 两种，可以随意切换
 * 最终都导出了一个 render 函数，负责渲染主应用
 */
// import render from './render/ReactRender';
import render from './render/VueRender';

// 初始化主应用，其实就是渲染主应用
render({ loading: true });

// 定义 loader 函数，切换微应用时由 qiankun 框架负责调用显示一个 loading 状态
const loader = loading => render({ loading });

// 注册微应用
registerMicroApps(
  // 微应用配置列表
  [
    {
      // 应用名称
      name: 'react16',
      // 应用的入口地址
      entry: '//localhost:7100',
      // 应用的挂载点，这个挂载点在上面渲染函数中的模版里面提供的
      container: '#subapp-viewport',
      // 微应用切换时调用的方法，显示一个 loading 状态
      loader,
      // 当路由前缀为 /react16 时激活当前应用
      activeRule: '/react16',
    },
    {
      name: 'react15',
      entry: '//localhost:7102',
      container: '#subapp-viewport',
      loader,
      activeRule: '/react15',
    },
    {
      name: 'vue',
      entry: '//localhost:7101',
      container: '#subapp-viewport',
      loader,
      activeRule: '/vue',
    },
    {
      name: 'angular9',
      entry: '//localhost:7103',
      container: '#subapp-viewport',
      loader,
      activeRule: '/angular9',
    },
    {
      name: 'purehtml',
      entry: '//localhost:7104',
      container: '#subapp-viewport',
      loader,
      activeRule: '/purehtml',
    },
  ],
  // 全局生命周期钩子，切换微应用时框架负责调用
  {
    beforeLoad: [
      app => {
        // 这个打印日志的方法可以学习一下，第三个参数会替换掉第一个参数中的 %c%s，并且第三个参数的颜色由第二个参数决定
        console.log('[LifeCycle] before load %c%s', 'color: green;', app.name);
      },
    ],
    beforeMount: [
      app => {
        console.log('[LifeCycle] before mount %c%s', 'color: green;', app.name);
      },
    ],
    afterUnmount: [
      app => {
        console.log('[LifeCycle] after unmount %c%s', 'color: green;', app.name);
      },
    ],
  },
);

// 定义全局状态，并返回两个通信方法
const { onGlobalStateChange, setGlobalState } = initGlobalState({
  user: 'qiankun',
});

// 监听全局状态的更改，当状态发生改变时执行回调函数
onGlobalStateChange((value, prev) => console.log('[onGlobalStateChange - master]:', value, prev));

// 设置新的全局状态，只能设置一级属性，微应用只能修改已存在的一级属性
setGlobalState({
  ignore: 'master',
  user: {
    name: 'master',
  },
});

// 设置默认进入的子应用，当主应用启动以后默认进入指定微应用
setDefaultMountApp('/react16');

// 启动应用
start();

// 当第一个微应用挂载以后，执行回调函数，在这里可以做一些特殊的事情，比如开启一监控或者买点脚本
runAfterFirstMounted(() => {
  console.log('[MainApp] first app mounted');
});
```

###### VueRender.js

```javascript
/**
 * 导出一个由 vue 实现的渲染函数，渲染了一个模版，模版里面包含一个 loading 状态节点和微应用容器节点
 */
import Vue from 'vue/dist/vue.esm';

// 返回一个 vue 实例
function vueRender({ loading }) {
  return new Vue({
    template: `
      <div id="subapp-container">
        <h4 v-if="loading" class="subapp-loading">Loading...</h4>
        <div id="subapp-viewport"></div>
      </div>
    `,
    el: '#subapp-container',
    data() {
      return {
        loading,
      };
    },
  });
}

// vue 实例
let app = null;

// 渲染函数
export default function render({ loading }) {
  // 单例，如果 vue 实例不存在则实例化主应用，存在则说明主应用已经渲染，需要更新主营应用的 loading 状态
  if (!app) {
    app = vueRender({ loading });
  } else {
    app.loading = loading;
  }
}
```

###### ReactRender.js

```javascript
/**
 * 同 vue 实现的渲染函数，这里通过 react 实现了一个一样的渲染函数
 */
import React from 'react';
import ReactDOM from 'react-dom';

// 渲染主应用
function Render(props) {
  const { loading } = props;

  return (
    <>
      {loading && <h4 className="subapp-loading">Loading...</h4>}
      <div id="subapp-viewport" />
    </>
  );
}

// 将主应用渲染到指定节点下
export default function render({ loading }) {
  const container = document.getElementById('subapp-container');
  ReactDOM.render(<Render loading={loading} />, container);
}
```

##### 手动加载微应用

通常这种场景下的微应用是一个不带路由的可独立运行的业务组件，这种使用方式的情况比较少见

###### multiple.js

```javascript
/**
 * 调用 loadMicroApp 方法注册了两个微应用
 */
import { loadMicroApp } from '../../es';

const app1 = loadMicroApp(
  // 应用配置，名称、入口地址、容器节点
  { name: 'react15', entry: '//localhost:7102', container: '#react15' },
  // 可以添加一些其它的配置，比如：沙箱、样式隔离等
  {
    sandbox: {
      // strictStyleIsolation: true,
    },
  },
);

const app2 = loadMicroApp(
  { name: 'vue', entry: '//localhost:7101', container: '#vue' },
  {
    sandbox: {
      // strictStyleIsolation: true,
    },
  },
);
```

#### vue

vue 微应用在 `examples/vue` 目录下，就是一个通过 vue-cli 创建的 vue demo 应用，然后对 `vue.config.js` 和 `main.js` 做了一些更改

##### vue.config.js

一个普通的 `webpack` 配置，需要注意的地方就三点

```javascript
{
  ...
  // publicPath 没在这里设置，是通过 webpack 提供的全局变量 __webpack_public_path__ 来即时设置的，webpackjs.com/guides/public-path/
  devServer: {
    ...
    // 设置跨域，因为主应用需要通过 fetch 去获取微应用引入的静态资源的，所以必须要求这些静态资源支持跨域
    headers: {
      'Access-Control-Allow-Origin': '*',
    },
  },
  output: {
    // 把子应用打包成 umd 库格式
    library: `${name}-[name]`,	// 库名称，唯一
    libraryTarget: 'umd',
    jsonpFunction: `webpackJsonp_${name}`,
  }
  ...
}
```

##### main.js

```javascript
// 动态设置 __webpack_public_path__
import './public-path';
import ElementUI from 'element-ui';
import 'element-ui/lib/theme-chalk/index.css';
import Vue from 'vue';
import VueRouter from 'vue-router';
import App from './App.vue';
// 路由配置
import routes from './router';
import store from './store';

Vue.config.productionTip = false;

Vue.use(ElementUI);

let router = null;
let instance = null;

// 应用渲染函数
function render(props = {}) {
  const { container } = props;
  // 实例化 router，根据应用运行环境设置路由前缀
  router = new VueRouter({
    // 作为微应用运行，则设置 /vue 为前缀，否则设置 /
    base: window.__POWERED_BY_QIANKUN__ ? '/vue' : '/',
    mode: 'history',
    routes,
  });

  // 实例化 vue 实例
  instance = new Vue({
    router,
    store,
    render: h => h(App),
  }).$mount(container ? container.querySelector('#app') : '#app');
}

// 支持应用独立运行
if (!window.__POWERED_BY_QIANKUN__) {
  render();
}

/**
 * 从 props 中获取通信方法，监听全局状态的更改和设置全局状态，只能操作一级属性
 * @param {*} props 
 */
function storeTest(props) {
  props.onGlobalStateChange &&
    props.onGlobalStateChange(
      (value, prev) => console.log(`[onGlobalStateChange - ${props.name}]:`, value, prev),
      true,
    );
  props.setGlobalState &&
    props.setGlobalState({
      ignore: props.name,
      user: {
        name: props.name,
      },
    });
}

/**
 * 导出的三个生命周期函数
 */
// 初始化
export async function bootstrap() {
  console.log('[vue] vue app bootstraped');
}

// 挂载微应用
export async function mount(props) {
  console.log('[vue] props from main framework', props);
  storeTest(props);
  render(props);
}

// 卸载、销毁微应用
export async function unmount() {
  instance.$destroy();
  instance.$el.innerHTML = '';
  instance = null;
  router = null;
}
```

##### public-path.js

```javascript
/**
 * 在入口文件中使用 ES6 模块导入，则在导入后对 __webpack_public_path__ 进行赋值。
 * 在这种情况下，必须将公共路径(public path)赋值移至专属模块，然后将其在最前面导入
 */

// qiankun 设置的全局变量，表示应用作为微应用在运行
if (window.__POWERED_BY_QIANKUN__) {
  // eslint-disable-next-line no-undef
  __webpack_public_path__ = window.__INJECTED_PUBLIC_PATH_BY_QIANKUN__;
}
```

#### jQuery

这是一个使用了 jQuery 的项目，在 `examples/purehtml` 目录下，展示了如何接入使用 jQuery 开发的应用

##### package.json

为了达到演示效果，使用 `http-server` 在起了一个本地服务器，并且支持跨域

```json
{
  ...
  "scripts": {
    "start": "cross-env PORT=7104 http-server . --cors",
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  ...
}
```

##### entry.js

```javascript
// 渲染函数
const render = $ => {
  $('#purehtml-container').html('Hello, render with jQuery');
  return Promise.resolve();
};

// 在全局对象上导出三个生命周期函数
(global => {
  global['purehtml'] = {
    bootstrap: () => {
      console.log('purehtml bootstrap');
      return Promise.resolve();
    },
    mount: () => {
      console.log('purehtml mount');
      // 调用渲染函数
      return render($);
    },
    unmount: () => {
      console.log('purehtml unmount');
      return Promise.resolve();
    },
  };
})(window);
```

##### index.html

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Purehtml Example</title>
  <script src="//cdn.bootcss.com/jquery/3.4.1/jquery.min.js">
  </script>
</head>
<body>
  <div style="display: flex; justify-content: center; align-items: center; height: 200px;">
    Purehtml Example
  </div>
  <div id="purehtml-container" style="text-align:center"></div>
  <!-- 引入 entry.js，相当于 vue 项目的 publicPath 配置 -->
  <script src="//localhost:7104/entry.js" entry></script>
</body>
</html>
```

#### angular 9、react 15、react 16

这三个实例项目就不一一分析了，和 vue 项目类似，都是配置打包工具将微应用打包成一个 umd 格式，然后配置应用入口文件 和 路由前缀

#### 小结

好了，读到这里，系统改造（可以开始干活了）基本上就已经可以顺利进行了，从主应用的开发到微应用接入，应该是不会有什么问题了。

当然如果你想继续深入了解，比如：

- 上面用到那些 API 的原理是什么？
- qiankun 是怎么解决我们之前提到的 single-spa 未解决的问题的？
- ...

接下来就带着我们的疑问和目的去全面深入的了解 `qiankun` 框架的内部实现

### 框架源码

整个框架的源码目录是 `src`，入口文件是 `src/index.ts`

#### 入口 src/index.ts

```typescript
/**
 * 在示例或者官网提到的所有 API 都在这里统一导出
 */
// 最关键的三个，手动加载微应用、基于路由配置、启动 qiankun
export { loadMicroApp, registerMicroApps, start } from './apis';
// 全局状态
export { initGlobalState } from './globalState';
// 全局的未捕获异常处理器
export * from './errorHandler';
// setDefaultMountApp 设置主应用启动后默认进入哪个微应用、runAfterFirstMounted 设置当第一个微应用挂载以后需要调用的一些方法
export * from './effects';
// 类型定义
export * from './interfaces';
// prefetch
export { prefetchImmediately as prefetchApps } from './prefetch';
```

#### registerMicroApps

```typescript
/**
 * 注册微应用，基于路由配置
 * @param apps = [
 *  {
 *    name: 'react16',
 *    entry: '//localhost:7100',
 *    container: '#subapp-viewport',
 *    loader,
 *    activeRule: '/react16'
 *  },
 *  ...
 * ]
 * @param lifeCycles = { ...各个生命周期方法对象 }
 */
export function registerMicroApps<T extends object = {}>(
  apps: Array<RegistrableApp<T>>,
  lifeCycles?: FrameworkLifeCycles<T>,
) {
  // 防止微应用重复注册，得到所有没有被注册的微应用列表
  const unregisteredApps = apps.filter(app => !microApps.some(registeredApp => registeredApp.name === app.name));

  // 所有的微应用 = 已注册 + 未注册的(将要被注册的)
  microApps = [...microApps, ...unregisteredApps];

  // 注册每一个微应用
  unregisteredApps.forEach(app => {
    // 注册时提供的微应用基本信息
    const { name, activeRule, loader = noop, props, ...appConfig } = app;

    // 调用 single-spa 的 registerApplication 方法注册微应用
    registerApplication({
      // 微应用名称
      name,
      // 微应用的加载方法，Promise<生命周期方法组成的对象>
      app: async () => {
        // 加载微应用时主应用显示 loading 状态
        loader(true);
        // 这句可以忽略，目的是在 single-spa 执行这个加载方法时让出线程，让其它微应用的加载方法都开始执行
        await frameworkStartedDefer.promise;

        // 核心、精髓、难点所在，负责加载微应用，然后一大堆处理，返回 bootstrap、mount、unmount、update 这个几个生命周期
        const { mount, ...otherMicroAppConfigs } = await loadApp(
          // 微应用的配置信息
          { name, props, ...appConfig },
          // start 方法执行时设置的配置对象
          frameworkConfiguration,
          // 注册微应用时提供的全局生命周期对象
          lifeCycles,
        );

        return {
          mount: [async () => loader(true), ...toArray(mount), async () => loader(false)],
          ...otherMicroAppConfigs,
        };
      },
      // 微应用的激活条件
      activeWhen: activeRule,
      // 传递给微应用的 props
      customProps: props,
    });
  });
}
```

#### start

```typescript
/**
 * 启动 qiankun
 * @param opts start 方法的配置对象 
 */
export function start(opts: FrameworkConfiguration = {}) {
  // qiankun 框架默认开启预加载、单例模式、样式沙箱
  frameworkConfiguration = { prefetch: true, singular: true, sandbox: true, ...opts };
  // 从这里可以看出 start 方法支持的参数不止官网文档说的那些，比如 urlRerouteOnly，这个是 single-spa 的 start 方法支持的
  const { prefetch, sandbox, singular, urlRerouteOnly, ...importEntryOpts } = frameworkConfiguration;

  // 预加载
  if (prefetch) {
    // 执行预加载策略，参数分别为微应用列表、预加载策略、{ fetch、getPublicPath、getTemplate }
    doPrefetchStrategy(microApps, prefetch, importEntryOpts);
  }

  // 样式沙箱
  if (sandbox) {
    if (!window.Proxy) {
      console.warn('[qiankun] Miss window.Proxy, proxySandbox will degenerate into snapshotSandbox');
      // 快照沙箱不支持非 singular 模式
      if (!singular) {
        console.error('[qiankun] singular is forced to be true when sandbox enable but proxySandbox unavailable');
        // 如果开启沙箱，会强制使用单例模式
        frameworkConfiguration.singular = true;
      }
    }
  }

  // 执行 single-spa 的 start 方法，启动 single-spa
  startSingleSpa({ urlRerouteOnly });

  frameworkStartedDefer.resolve();
}
```

#### 预加载 - doPrefetchStrategy

```typescript
/**
 * 执行预加载策略，qiankun 支持四种
 * @param apps 所有的微应用 
 * @param prefetchStrategy 预加载策略，四种 =》 
 *  1、true，第一个微应用挂载以后加载其它微应用的静态资源，利用的是 single-spa 提供的 single-spa:first-mount 事件来实现的
 *  2、string[]，微应用名称数组，在第一个微应用挂载以后加载指定的微应用的静态资源
 *  3、all，主应用执行 start 以后就直接开始预加载所有微应用的静态资源
 *  4、自定义函数，返回两个微应用组成的数组，一个是关键微应用组成的数组，需要马上就执行预加载的微应用，一个是普通的微应用组成的数组，在第一个微应用挂载以后预加载这些微应用的静态资源
 * @param importEntryOpts = { fetch, getPublicPath, getTemplate }
 */
export function doPrefetchStrategy(
  apps: AppMetadata[],
  prefetchStrategy: PrefetchStrategy,
  importEntryOpts?: ImportEntryOpts,
) {
  // 定义函数，函数接收一个微应用名称组成的数组，然后从微应用列表中返回这些名称所对应的微应用，最后得到一个数组[{name, entry}, ...]
  const appsName2Apps = (names: string[]): AppMetadata[] => apps.filter(app => names.includes(app.name));

  if (Array.isArray(prefetchStrategy)) {
    // 说明加载策略是一个数组，当第一个微应用挂载之后开始加载数组内由用户指定的微应用资源，数组内的每一项表示一个微应用的名称
    prefetchAfterFirstMounted(appsName2Apps(prefetchStrategy as string[]), importEntryOpts);
  } else if (isFunction(prefetchStrategy)) {
    // 加载策略是一个自定义的函数，可完全自定义应用资源的加载时机（首屏应用、次屏应用)
    (async () => {
      // critical rendering apps would be prefetch as earlier as possible，关键的应用程序应该尽可能早的预取
      // 执行加载策略函数，函数会返回两个数组，一个关键的应用程序数组，会立即执行预加载动作，另一个是在第一个微应用挂载以后执行微应用静态资源的预加载
      const { criticalAppNames = [], minorAppsName = [] } = await prefetchStrategy(apps);
      // 立即预加载这些关键微应用程序的静态资源
      prefetchImmediately(appsName2Apps(criticalAppNames), importEntryOpts);
      // 当第一个微应用挂载以后预加载这些微应用的静态资源
      prefetchAfterFirstMounted(appsName2Apps(minorAppsName), importEntryOpts);
    })();
  } else {
    // 加载策略是默认的 true 或者 all
    switch (prefetchStrategy) {
      case true:
        // 第一个微应用挂载之后开始加载其它微应用的静态资源
        prefetchAfterFirstMounted(apps, importEntryOpts);
        break;

      case 'all':
        // 在主应用执行 start 以后就开始加载所有微应用的静态资源
        prefetchImmediately(apps, importEntryOpts);
        break;

      default:
        break;
    }
  }
}

// 判断是否为弱网环境
const isSlowNetwork = navigator.connection
  ? navigator.connection.saveData ||
    (navigator.connection.type !== 'wifi' &&
      navigator.connection.type !== 'ethernet' &&
      /(2|3)g/.test(navigator.connection.effectiveType))
  : false;

/**
 * prefetch assets, do nothing while in mobile network
 * 预加载静态资源，在移动网络下什么都不做
 * @param entry
 * @param opts
 */
function prefetch(entry: Entry, opts?: ImportEntryOpts): void {
  // 弱网环境下不执行预加载
  if (!navigator.onLine || isSlowNetwork) {
    // Don't prefetch if in a slow network or offline
    return;
  }

  // 通过时间切片的方式去加载静态资源，在浏览器空闲时去执行回调函数，避免浏览器卡顿
  requestIdleCallback(async () => {
    // 得到加载静态资源的函数
    const { getExternalScripts, getExternalStyleSheets } = await importEntry(entry, opts);
    // 样式
    requestIdleCallback(getExternalStyleSheets);
    // js 脚本
    requestIdleCallback(getExternalScripts);
  });
}

/**
 * 在第一个微应用挂载之后开始加载 apps 中指定的微应用的静态资源
 * 通过监听 single-spa 提供的 single-spa:first-mount 事件来实现，该事件在第一个微应用挂载以后会被触发
 * @param apps 需要被预加载静态资源的微应用列表，[{ name, entry }, ...]
 * @param opts = { fetch , getPublicPath, getTemplate }
 */
function prefetchAfterFirstMounted(apps: AppMetadata[], opts?: ImportEntryOpts): void {
  // 监听 single-spa:first-mount 事件
  window.addEventListener('single-spa:first-mount', function listener() {
    // 已挂载的微应用
    const mountedApps = getMountedApps();
    // 从预加载的微应用列表中过滤出未挂载的微应用
    const notMountedApps = apps.filter(app => mountedApps.indexOf(app.name) === -1);

    // 开发环境打印日志，已挂载的微应用和未挂载的微应用分别有哪些
    if (process.env.NODE_ENV === 'development') {
      console.log(`[qiankun] prefetch starting after ${mountedApps} mounted...`, notMountedApps);
    }

    // 循环加载微应用的静态资源
    notMountedApps.forEach(({ entry }) => prefetch(entry, opts));

    // 移除 single-spa:first-mount 事件
    window.removeEventListener('single-spa:first-mount', listener);
  });
}

/**
 * 在执行 start 启动 qiankun 之后立即预加载所有微应用的静态资源
 * @param apps 需要被预加载静态资源的微应用列表，[{ name, entry }, ...]
 * @param opts = { fetch , getPublicPath, getTemplate }
 */
export function prefetchImmediately(apps: AppMetadata[], opts?: ImportEntryOpts): void {
  // 开发环境打印日志
  if (process.env.NODE_ENV === 'development') {
    console.log('[qiankun] prefetch starting for apps...', apps);
  }

  // 加载所有微应用的静态资源
  apps.forEach(({ entry }) => prefetch(entry, opts));
}
```

#### 应用间通信 initGlobalState

```typescript
// 触发全局监听，执行所有应用注册的回调函数
function emitGlobal(state: Record<string, any>, prevState: Record<string, any>) {
  // 循环遍历，执行所有应用注册的回调函数
  Object.keys(deps).forEach((id: string) => {
    if (deps[id] instanceof Function) {
      deps[id](cloneDeep(state), cloneDeep(prevState));
    }
  });
}

/**
 * 定义全局状态，并返回通信方法，一般由主应用调用，微应用通过 props 获取通信方法。 
 * @param state 全局状态，{ key: value }
 */
export function initGlobalState(state: Record<string, any> = {}) {
  if (state === globalState) {
    console.warn('[qiankun] state has not changed！');
  } else {
    // 方法有可能被重复调用，将已有的全局状态克隆一份，为空则是第一次调用 initGlobalState 方法，不为空则非第一次次调用
    const prevGlobalState = cloneDeep(globalState);
    // 将传递的状态克隆一份赋值为 globalState
    globalState = cloneDeep(state);
    // 触发全局监听，当然在这个位置调用，正常情况下没啥反应，因为现在还没有应用注册回调函数
    emitGlobal(globalState, prevGlobalState);
  }
  // 返回通信方法，参数表示应用 id，true 表示自己是主应用调用
  return getMicroAppStateActions(`global-${+new Date()}`, true);
}

/**
 * 返回通信方法 
 * @param id 应用 id
 * @param isMaster 表明调用的应用是否为主应用，在主应用初始化全局状态时，initGlobalState 内部调用该方法时会传递 true，其它都为 false
 */
export function getMicroAppStateActions(id: string, isMaster?: boolean): MicroAppStateActions {
  return {
    /**
     * 全局依赖监听，为指定应用（id = 应用id）注册回调函数
     * 依赖数据结构为：
     * {
     *   {id}: callback
     * }
     *
     * @param callback 注册的回调函数
     * @param fireImmediately 是否立即执行回调
     */
    onGlobalStateChange(callback: OnGlobalStateChangeCallback, fireImmediately?: boolean) {
      // 回调函数必须为 function
      if (!(callback instanceof Function)) {
        console.error('[qiankun] callback must be function!');
        return;
      }
      // 如果回调函数已经存在，重复注册时给出覆盖提示信息
      if (deps[id]) {
        console.warn(`[qiankun] '${id}' global listener already exists before this, new listener will overwrite it.`);
      }
      // id 为一个应用 id，一个应用对应一个回调
      deps[id] = callback;
      // 克隆全局状态
      const cloneState = cloneDeep(globalState);
      // 如果需要，立即出发回调执行
      if (fireImmediately) {
        callback(cloneState, cloneState);
      }
    },

    /**
     * setGlobalState 更新 store 数据
     *
     * 1. 对新输入 state 的第一层属性做校验，如果是主应用则可以添加新的一级属性进来，也可以更新已存在的一级属性，
     *    如果是微应用，则只能更新已存在的一级属性，不可以新增一级属性
     * 2. 触发全局监听，执行所有应用注册的回调函数，以达到应用间通信的目的
     *
     * @param state 新的全局状态
     */
    setGlobalState(state: Record<string, any> = {}) {
      if (state === globalState) {
        console.warn('[qiankun] state has not changed！');
        return false;
      }

      // 记录旧的全局状态中被改变的 key
      const changeKeys: string[] = [];
      // 旧的全局状态
      const prevGlobalState = cloneDeep(globalState);
      globalState = cloneDeep(
        // 循环遍历新状态中的所有 key
        Object.keys(state).reduce((_globalState, changeKey) => {
          if (isMaster || _globalState.hasOwnProperty(changeKey)) {
            // 主应用 或者 旧的全局状态存在该 key 时才进来，说明只有主应用才可以新增属性，微应用只可以更新已存在的属性值，且不论主应用微应用只能更新一级属性
            // 记录被改变的key
            changeKeys.push(changeKey);
            // 更新旧状态中对应的 key value
            return Object.assign(_globalState, { [changeKey]: state[changeKey] });
          }
          console.warn(`[qiankun] '${changeKey}' not declared when init state！`);
          return _globalState;
        }, globalState),
      );
      if (changeKeys.length === 0) {
        console.warn('[qiankun] state has not changed！');
        return false;
      }
      // 触发全局监听
      emitGlobal(globalState, prevGlobalState);
      return true;
    },

    // 注销该应用下的依赖
    offGlobalStateChange() {
      delete deps[id];
      return true;
    },
  };
}
```

#### 全局未捕获异常处理器

```typescript
/**
 * 整个文件的逻辑一眼明了，整个框架提供了两种全局异常捕获，一个是 single-spa 提供的，另一个是 qiankun 自己的，你只需提供相应的回调函数即可
 */

// single-spa 的异常捕获
export { addErrorHandler, removeErrorHandler } from 'single-spa';

// qiankun 的异常捕获
// 监听了 error 和 unhandlerejection 事件
export function addGlobalUncaughtErrorHandler(errorHandler: OnErrorEventHandlerNonNull): void {
  window.addEventListener('error', errorHandler);
  window.addEventListener('unhandledrejection', errorHandler);
}

// 移除 error 和 unhandlerejection 事件监听
export function removeGlobalUncaughtErrorHandler(errorHandler: (...args: any[]) => any) {
  window.removeEventListener('error', errorHandler);
  window.removeEventListener('unhandledrejection', errorHandler);
}
```

#### setDefaultMountApp

```typescript
/**
 * 设置主应用启动后默认进入的微应用，其实是规定了第一个微应用挂载完成后决定默认进入哪个微应用
 * 利用的是 single-spa 的 single-spa:no-app-change 事件，该事件在所有微应用状态改变结束后（即发生路由切换且新的微应用已经被挂载完成）触发
 * @param defaultAppLink 微应用的链接，比如 /react16
 */
export function setDefaultMountApp(defaultAppLink: string) {
  // 当事件触发时就说明微应用已经挂载完成，但这里只监听了一次，因为事件被触发以后就移除了监听，所以说是主应用启动后默认进入的微应用，且只执行了一次的原因
  window.addEventListener('single-spa:no-app-change', function listener() {
    // 说明微应用已经挂载完成，获取挂载的微应用列表，再次确认确实有微应用挂载了，其实这个确认没啥必要
    const mountedApps = getMountedApps();
    if (!mountedApps.length) {
      // 这个是 single-spa 提供的一个 api，通过触发 window.location.hash 或者 pushState 更改路由，切换微应用
      navigateToUrl(defaultAppLink);
    }

    // 触发一次以后，就移除该事件的监听函数，后续的路由切换（事件触发）时就不再响应
    window.removeEventListener('single-spa:no-app-change', listener);
  });
}

// 这个 api 和 setDefaultMountApp 作用一致，官网也提到，兼容老版本的一个 api
export function runDefaultMountEffects(defaultAppLink: string) {
  console.warn(
    '[qiankun] runDefaultMountEffects will be removed in next version, please use setDefaultMountApp instead',
  );
  setDefaultMountApp(defaultAppLink);
}
```

#### runAfterFirstMounted

```typescript
/**
 * 第一个微应用 mount 后需要调用的方法，比如开启一些监控或者埋点脚本
 * 同样利用的 single-spa 的 single-spa:first-mount 事件，当第一个微应用挂载以后会触发
 * @param effect 回调函数，当第一个微应用挂载以后要做的事情
 */
export function runAfterFirstMounted(effect: () => void) {
  // can not use addEventListener once option for ie support
  window.addEventListener('single-spa:first-mount', function listener() {
    if (process.env.NODE_ENV === 'development') {
      console.timeEnd(firstMountLogLabel);
    }

    effect();

    // 这里不移除也没事，因为这个事件后续不会再被触发了
    window.removeEventListener('single-spa:first-mount', listener);
  });
}
```

#### 手动加载微应用 loadMicroApp

```typescript
/**
 * 手动加载一个微应用，是通过 single-spa 的 mountRootParcel api 实现的，返回微应用实例
 * @param app = { name, entry, container, props }
 * @param configuration 配置对象
 * @param lifeCycles 还支持一个全局生命周期配置对象，这个参数官方文档没提到
 */
export function loadMicroApp<T extends object = {}>(
  app: LoadableApp<T>,
  configuration?: FrameworkConfiguration,
  lifeCycles?: FrameworkLifeCycles<T>,
): MicroApp {
  const { props } = app;
  // single-spa 的 mountRootParcel api
  return mountRootParcel(() => loadApp(app, configuration ?? frameworkConfiguration, lifeCycles), {
    domElement: document.createElement('div'),
    ...props,
  });
}
```

#### qiankun 的核心 loadApp

> 接下来介绍 `loadApp` 方法，个人认为 `qiankun` 的核心代码可以说大部分都在这里，当然这也是整个框架的精髓和难点所在

```typescript
/**
 * 完成了以下几件事：
 *  1、通过 HTML Entry 的方式远程加载微应用，得到微应用的 html 模版（首屏内容）、JS 脚本执行器、静态经资源路径
 *  2、样式隔离，shadow DOM 或者 scoped css 两种方式
 *  3、渲染微应用
 *  4、运行时沙箱，JS 沙箱、样式沙箱
 *  5、合并沙箱传递出来的 生命周期方法、用户传递的生命周期方法、框架内置的生命周期方法，将这些生命周期方法统一整理，导出一个生命周期对象，
 * 供 single-spa 的 registerApplication 方法使用，这个对象就相当于使用 single-spa 时你的微应用导出的那些生命周期方法，只不过 qiankun
 * 额外填了一些生命周期方法，做了一些事情
 *  6、给微应用注册通信方法并返回通信方法，然后会将通信方法通过 props 注入到微应用
 * @param app 微应用配置对象
 * @param configuration start 方法执行时设置的配置对象 
 * @param lifeCycles 注册微应用时提供的全局生命周期对象
 */
export async function loadApp<T extends object>(
  app: LoadableApp<T>,
  configuration: FrameworkConfiguration = {},
  lifeCycles?: FrameworkLifeCycles<T>,
): Promise<ParcelConfigObject> {
  // 微应用的入口和名称
  const { entry, name: appName } = app;
  // 实例 id
  const appInstanceId = `${appName}_${+new Date()}_${Math.floor(Math.random() * 1000)}`;

  // 下面这个不用管，就是生成一个标记名称，然后使用该名称在浏览器性能缓冲器中设置一个时间戳，可以用来度量程序的执行时间，performance.mark、performance.measure
  const markName = `[qiankun] App ${appInstanceId} Loading`;
  if (process.env.NODE_ENV === 'development') {
    performanceMark(markName);
  }

  // 配置信息
  const { singular = false, sandbox = true, excludeAssetFilter, ...importEntryOpts } = configuration;

  /**
   * 获取微应用的入口 html 内容和脚本执行器
   * template 是 link 替换为 style 后的 template
   * execScript 是 让 JS 代码(scripts)在指定 上下文 中运行
   * assetPublicPath 是静态资源地址
   */
  const { template, execScripts, assetPublicPath } = await importEntry(entry, importEntryOpts);

  // single-spa 的限制，加载、初始化和卸载不能同时进行，必须等卸载完成以后才可以进行加载，这个 promise 会在微应用卸载完成后被 resolve，在后面可以看到
  if (await validateSingularMode(singular, app)) {
    await (prevAppUnmountedDeferred && prevAppUnmountedDeferred.promise);
  }

  // --------------- 样式隔离 ---------------
  // 是否严格样式隔离
  const strictStyleIsolation = typeof sandbox === 'object' && !!sandbox.strictStyleIsolation;
  // 实验性的样式隔离，后面就叫 scoped css，和严格样式隔离不能同时开启，如果开启了严格样式隔离，则 scoped css 就为 false，强制关闭
  const enableScopedCSS = isEnableScopedCSS(configuration);

  // 用一个容器元素包裹微应用入口 html 模版, appContent = `<div id="__qiankun_microapp_wrapper_for_${appInstanceId}__" data-name="${appName}">${template}</div>`
  const appContent = getDefaultTplWrapper(appInstanceId, appName)(template);
  // 将 appContent 有字符串模版转换为 html dom 元素，如果需要开启样式严格隔离，则将 appContent 的子元素即微应用入口模版用 shadow dom 包裹起来，以达到样式严格隔离的目的
  let element: HTMLElement | null = createElement(appContent, strictStyleIsolation);
  // 通过 scoped css 的方式隔离样式，从这里也就能看出官方为什么说：
  // 在目前的阶段，该功能还不支持动态的、使用 <link />标签来插入外联的样式，但考虑在未来支持这部分场景
  // 在现阶段只处理 style 这种内联标签的情况 
  if (element && isEnableScopedCSS(configuration)) {
    const styleNodes = element.querySelectorAll('style') || [];
    forEach(styleNodes, (stylesheetElement: HTMLStyleElement) => {
      css.process(element!, stylesheetElement, appName);
    });
  }

  // --------------- 渲染微应用 ---------------
  // 主应用装载微应用的容器节点
  const container = 'container' in app ? app.container : undefined;
  // 这个是 1.x 版本遗留下来的实现，如果提供了 render 函数，当微应用需要被激活时就执行 render 函数渲染微应用，新版本用的 container，弃了 render
  // 而且 legacyRender 和 strictStyleIsolation、scoped css 不兼容
  const legacyRender = 'render' in app ? app.render : undefined;

  // 返回一个 render 函数，这个 render 函数要不使用用户传递的 render 函数，要不将 element 插入到 container
  const render = getRender(appName, appContent, container, legacyRender);

  // 渲染微应用到容器节点，并显示 loading 状态
  render({ element, loading: true }, 'loading');

  // 得到一个 getter 函数，通过该函数可以获取 <div id="__qiankun_microapp_wrapper_for_${appInstanceId}__" data-name="${appName}">${template}</div>
  const containerGetter = getAppWrapperGetter(
    appName,
    appInstanceId,
    !!legacyRender,
    strictStyleIsolation,
    enableScopedCSS,
    () => element,
  );

  // --------------- 运行时沙箱 ---------------
  // 保证每一个微应用运行在一个干净的环境中（JS 执行上下文独立、应用间不会发生样式污染）
  let global = window;
  let mountSandbox = () => Promise.resolve();
  let unmountSandbox = () => Promise.resolve();
  if (sandbox) {
    /**
     * 生成运行时沙箱，这个沙箱其实由两部分组成 => JS 沙箱（执行上下文）、样式沙箱
     * 
     * 沙箱返回 window 的代理对象 proxy 和 mount、unmount 两个方法
     * unmount 方法会让微应用失活，恢复被增强的原生方法，并记录一堆 rebuild 函数，这个函数是微应用卸载时希望自己被重新挂载时要做的一些事情，比如动态样式表重建（卸载时会缓存）
     * mount 方法会执行一些一些 patch 动作，恢复原生方法的增强功能，并执行 rebuild 函数，将微应用恢复到卸载时的状态，当然从初始化状态进入挂载状态就没有恢复一说了
     */
    const sandboxInstance = createSandbox(
      appName,
      containerGetter,
      Boolean(singular),
      enableScopedCSS,
      excludeAssetFilter,
    );
    // 用沙箱的代理对象作为接下来使用的全局对象
    global = sandboxInstance.proxy as typeof window;
    mountSandbox = sandboxInstance.mount;
    unmountSandbox = sandboxInstance.unmount;
  }

  // 合并用户传递的生命周期对象和 qiankun 框架内置的生命周期对象
  const { beforeUnmount = [], afterUnmount = [], afterMount = [], beforeMount = [], beforeLoad = [] } = mergeWith(
    {},
    // 返回内置生命周期对象，global.__POWERED_BY_QIANKUN__ 和 global.__INJECTED_PUBLIC_PATH_BY_QIANKUN__ 的设置就是在内置的生命周期对象中设置的
    getAddOns(global, assetPublicPath),
    lifeCycles,
    (v1, v2) => concat(v1 ?? [], v2 ?? []),
  );

  await execHooksChain(toArray(beforeLoad), app, global);

  // get the lifecycle hooks from module exports，获取微应用暴露出来的生命周期函数
  const scriptExports: any = await execScripts(global, !singular);
  const { bootstrap, mount, unmount, update } = getLifecyclesFromExports(scriptExports, appName, global);

  // 给微应用注册通信方法并返回通信方法，然后会将通信方法通过 props 注入到微应用
  const {
    onGlobalStateChange,
    setGlobalState,
    offGlobalStateChange,
  }: Record<string, Function> = getMicroAppStateActions(appInstanceId);

  const parcelConfig: ParcelConfigObject = {
    name: appInstanceId,
    bootstrap,
    // 挂载阶段需要执行的一系列方法
    mount: [
      // 性能度量，不用管
      async () => {
        if (process.env.NODE_ENV === 'development') {
          const marks = performance.getEntriesByName(markName, 'mark');
          // mark length is zero means the app is remounting
          if (!marks.length) {
            performanceMark(markName);
          }
        }
      },
      // 单例模式需要等微应用卸载完成以后才能执行挂载任务，promise 会在微应用卸载完以后 resolve
      async () => {
        if ((await validateSingularMode(singular, app)) && prevAppUnmountedDeferred) {
          return prevAppUnmountedDeferred.promise;
        }

        return undefined;
      },
      // 添加 mount hook, 确保每次应用加载前容器 dom 结构已经设置完毕
      async () => {
        // element would be destroyed after unmounted, we need to recreate it if it not exist
        // unmount 阶段会置空，这里重新生成
        element = element || createElement(appContent, strictStyleIsolation);
        // 渲染微应用到容器节点，并显示 loading 状态
        render({ element, loading: true }, 'mounting');
      },
      // 运行时沙箱导出的 mount
      mountSandbox,
      // exec the chain after rendering to keep the behavior with beforeLoad
      async () => execHooksChain(toArray(beforeMount), app, global),
      // 向微应用的 mount 生命周期函数传递参数，比如微应用中使用的 props.onGlobalStateChange 方法
      async props => mount({ ...props, container: containerGetter(), setGlobalState, onGlobalStateChange }),
      // 应用 mount 完成后结束 loading
      async () => render({ element, loading: false }, 'mounted'),
      async () => execHooksChain(toArray(afterMount), app, global),
      // initialize the unmount defer after app mounted and resolve the defer after it unmounted
      // 微应用挂载完成以后初始化这个 promise，并且在微应用卸载以后 resolve 这个 promise
      async () => {
        if (await validateSingularMode(singular, app)) {
          prevAppUnmountedDeferred = new Deferred<void>();
        }
      },
      // 性能度量，不用管
      async () => {
        if (process.env.NODE_ENV === 'development') {
          const measureName = `[qiankun] App ${appInstanceId} Loading Consuming`;
          performanceMeasure(measureName, markName);
        }
      },
    ],
    // 卸载微应用
    unmount: [
      async () => execHooksChain(toArray(beforeUnmount), app, global),
      // 执行微应用的 unmount 生命周期函数
      async props => unmount({ ...props, container: containerGetter() }),
      // 沙箱导出的 unmount 方法
      unmountSandbox,
      async () => execHooksChain(toArray(afterUnmount), app, global),
      // 显示 loading 状态、移除微应用的状态监听、置空 element
      async () => {
        render({ element: null, loading: false }, 'unmounted');
        offGlobalStateChange(appInstanceId);
        // for gc
        element = null;
      },
      // 微应用卸载以后 resolve 这个 promise，框架就可以进行后续的工作，比如加载或者挂载其它微应用
      async () => {
        if ((await validateSingularMode(singular, app)) && prevAppUnmountedDeferred) {
          prevAppUnmountedDeferred.resolve();
        }
      },
    ],
  };

  // 微应用有可能定义 update 方法
  if (typeof update === 'function') {
    parcelConfig.update = update;
  }

  return parcelConfig;
}
```

#### 样式隔离

`qiankun` 的样式隔离有两种方式，一种是严格样式隔离，通过 `shadow dom` 来实现，另一种是实验性的样式隔离，就是 `scoped css`，两种方式不可共存

##### 严格样式隔离

在 `qiankun` 中的严格样式隔离，就是在这个 `createElement` 方法中做的，通过 `shadow dom` 来实现， `shadow dom` 是浏览器原生提供的一种能力，在过去的很长一段时间里，浏览器用它来封装一些元素的内部结构。以一个有着默认播放控制按钮的 `<video>` 元素为例，实际上，在它的 Shadow DOM 中，包含来一系列的按钮和其他控制器。Shadow DOM 标准允许你为你自己的元素（custom element）维护一组 Shadow DOM。具体内容可查看 [shadow DOM](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.mozilla.org%2Fzh-CN%2Fdocs%2FWeb%2FWeb_Components%2FUsing_shadow_DOM)

```typescript
/**
 * 做了两件事
 *  1、将 appContent 由字符串模版转换成 html dom 元素
 *  2、如果需要开启严格样式隔离，则将 appContent 的子元素即微应用的入口模版用 shadow dom 包裹起来，达到样式严格隔离的目的
 * @param appContent = `<div id="__qiankun_microapp_wrapper_for_${appInstanceId}__" data-name="${appName}">${template}</div>`
 * @param strictStyleIsolation 是否开启严格样式隔离
 */
function createElement(appContent: string, strictStyleIsolation: boolean): HTMLElement {
  // 创建一个 div 元素
  const containerElement = document.createElement('div');
  // 将字符串模版 appContent 设置为 div 的子与阿苏
  containerElement.innerHTML = appContent;
  // appContent always wrapped with a singular div，appContent 由模版字符串变成了 DOM 元素
  const appElement = containerElement.firstChild as HTMLElement;
  // 如果开启了严格的样式隔离，则将 appContent 的子元素（微应用的入口模版）用 shadow dom 包裹，以达到微应用之间样式严格隔离的目的
  if (strictStyleIsolation) {
    if (!supportShadowDOM) {
      console.warn(
        '[qiankun]: As current browser not support shadow dom, your strictStyleIsolation configuration will be ignored!',
      );
    } else {
      const { innerHTML } = appElement;
      appElement.innerHTML = '';
      let shadow: ShadowRoot;

      if (appElement.attachShadow) {
        shadow = appElement.attachShadow({ mode: 'open' });
      } else {
        // createShadowRoot was proposed in initial spec, which has then been deprecated
        shadow = (appElement as any).createShadowRoot();
      }
      shadow.innerHTML = innerHTML;
    }
  }

  return appElement;
}
```

##### 实验性样式隔离

实验性样式的隔离方式其实就是 `scoped css`，`qiankun` 会通过动态改写一个特殊的选择器约束来限制 `css` 的生效范围，应用的样式会按照如下模式改写：

```css
// 假设应用名是 react16
.app-main {
  font-size: 14px;
}
div[data-qiankun-react16] .app-main {
  font-size: 14px;
}
```

###### process

```typescript
/**
 * 做了两件事：
 *  实例化 processor = new ScopedCss()，真正处理样式选择器的地方
 *  生成样式前缀 `div[data-qiankun]=${appName}`
 * @param appWrapper = <div id="__qiankun_microapp_wrapper_for_${appInstanceId}__" data-name="${appName}">${template}</div>
 * @param stylesheetElement = <style>xx</style>
 * @param appName 微应用名称
 */
export const process = (
  appWrapper: HTMLElement,
  stylesheetElement: HTMLStyleElement | HTMLLinkElement,
  appName: string,
) => {
  // lazy singleton pattern，单例模式
  if (!processor) {
    processor = new ScopedCSS();
  }

  // 目前支持 style 标签
  if (stylesheetElement.tagName === 'LINK') {
    console.warn('Feature: sandbox.experimentalStyleIsolation is not support for link element yet.');
  }

  // 微应用模版
  const mountDOM = appWrapper;
  if (!mountDOM) {
    return;
  }

  // div
  const tag = (mountDOM.tagName || '').toLowerCase();

  if (tag && stylesheetElement.tagName === 'STYLE') {
    // 生成前缀 `div[data-qiankun]=${appName}`
    const prefix = `${tag}[${QiankunCSSRewriteAttr}="${appName}"]`;
     /**
     * 实际处理样式的地方
     * 拿到样式节点中的所有样式规则，然后重写样式选择器
     *  含有根元素选择器的情况：用前缀替换掉选择器中的根元素选择器部分，
     *  普通选择器：将前缀插到第一个选择器的后面
     */
    processor.process(stylesheetElement, prefix);
  }
}

export const QiankunCSSRewriteAttr = 'data-qiankun';
```

###### ScopedCSS

```typescript
// https://developer.mozilla.org/en-US/docs/Web/API/CSSRule
enum RuleType {
  // type: rule will be rewrote
  STYLE = 1,
  MEDIA = 4,
  SUPPORTS = 12,

  // type: value will be kept
  IMPORT = 3,
  FONT_FACE = 5,
  PAGE = 6,
  KEYFRAMES = 7,
  KEYFRAME = 8,
}

const arrayify = <T>(list: CSSRuleList | any[]) => {
  return [].slice.call(list, 0) as T[];
};

export class ScopedCSS {
  private static ModifiedTag = 'Symbol(style-modified-qiankun)';

  private sheet: StyleSheet;

  private swapNode: HTMLStyleElement;

  constructor() {
    const styleNode = document.createElement('style');
    document.body.appendChild(styleNode);

    this.swapNode = styleNode;
    this.sheet = styleNode.sheet!;
    this.sheet.disabled = true;
  }

  /**
   * 拿到样式节点中的所有样式规则，然后重写样式选择器
   *  含有根元素选择器的情况：用前缀替换掉选择器中的根元素选择器部分，
   *  普通选择器：将前缀插到第一个选择器的后面
   * 
   * 如果发现一个样式节点为空，则该节点的样式内容可能会被动态插入，qiankun 监控了该动态插入的样式，并做了同样的处理
   * 
   * @param styleNode 样式节点
   * @param prefix 前缀 `div[data-qiankun]=${appName}`
   */
  process(styleNode: HTMLStyleElement, prefix: string = '') {
    // 样式节点不为空，即 <style>xx</style>
    if (styleNode.textContent !== '') {
      // 创建一个文本节点，内容为 style 节点内的样式内容
      const textNode = document.createTextNode(styleNode.textContent || '');
      // swapNode 是 ScopedCss 类实例化时创建的一个空 style 节点，将样式内容添加到这个节点下
      this.swapNode.appendChild(textNode);
      /**
       * {
       *  cssRules: CSSRuleList {0: CSSStyleRule, 1: CSSStyleRule, 2: CSSStyleRule, 3: CSSStyleRule, length: 4}
       *  disabled: false
       *  href: null
       *  media: MediaList {length: 0, mediaText: ""}
       *  ownerNode: style
       *  ownerRule: null
       *  parentStyleSheet: null
       *  rules: CSSRuleList {0: CSSStyleRule, 1: CSSStyleRule, 2: CSSStyleRule, 3: CSSStyleRule, length: 4}
       *  title: null
       *  type: "text/css"
       * }
       */
      const sheet = this.swapNode.sheet as any; // type is missing
      /**
       * 得到所有的样式规则，比如
       * [
       *  {selectorText: "body", style: CSSStyleDeclaration, styleMap: StylePropertyMap, type: 1, cssText: "body { background: rgb(255, 255, 255); margin: 0px; }", …}
       *  {selectorText: "#oneGoogleBar", style: CSSStyleDeclaration, styleMap: StylePropertyMap, type: 1, cssText: "#oneGoogleBar { height: 56px; }", …}
       *  {selectorText: "#backgroundImage", style: CSSStyleDeclaration, styleMap: StylePropertyMap, type: 1, cssText: "#backgroundImage { border: none; height: 100%; poi…xed; top: 0px; visibility: hidden; width: 100%; }", …}
       *  {selectorText: "[show-background-image] #backgroundImage {xx}"
       * ]
       */
      const rules = arrayify<CSSRule>(sheet?.cssRules ?? []);
      /**
       * 重写样式选择器
       *  含有根元素选择器的情况：用前缀替换掉选择器中的根元素选择器部分，
       *  普通选择器：将前缀插到第一个选择器的后面
       */
      const css = this.rewrite(rules, prefix);
      // 用重写后的样式替换原来的样式
      // eslint-disable-next-line no-param-reassign
      styleNode.textContent = css;

      // cleanup
      this.swapNode.removeChild(textNode);
      return;
    }

    /**
     * 
     * 走到这里说明样式节点为空
     */

    // 创建并返回一个新的 MutationObserver 它会在指定的DOM发生变化时被调用
    const mutator = new MutationObserver(mutations => {
      for (let i = 0; i < mutations.length; i += 1) {
        const mutation = mutations[i];

        // 表示该节点已经被 qiankun 处理过，后面就不会再被重复处理
        if (ScopedCSS.ModifiedTag in styleNode) {
          return;
        }

        // 如果是子节点列表发生变化
        if (mutation.type === 'childList') {
          // 拿到 styleNode 下的所有样式规则，并重写其样式选择器，然后用重写后的样式替换原有样式
          const sheet = styleNode.sheet as any;
          const rules = arrayify<CSSRule>(sheet?.cssRules ?? []);
          const css = this.rewrite(rules, prefix);

          // eslint-disable-next-line no-param-reassign
          styleNode.textContent = css;
          // 给 styleNode 添加一个 ScopedCss.ModifiedTag 属性，表示已经被 qiankun 处理过，后面就不会再被处理了
          // eslint-disable-next-line no-param-reassign
          (styleNode as any)[ScopedCSS.ModifiedTag] = true;
        }
      }
    });

    // since observer will be deleted when node be removed
    // we dont need create a cleanup function manually
    // see https://developer.mozilla.org/en-US/docs/Web/API/MutationObserver/disconnect
    // 观察 styleNode 节点，当其子节点发生变化时调用 callback 即 实例化时传递的函数
    mutator.observe(styleNode, { childList: true });
  }

  /**
   * 重写样式选择器，都是在 ruleStyle 中处理的：
   *  含有根元素选择器的情况：用前缀替换掉选择器中的根元素选择器部分，
   *  普通选择器：将前缀插到第一个选择器的后面
   * 
   * @param rules 样式规则
   * @param prefix 前缀 `div[data-qiankun]=${appName}`
   */
  private rewrite(rules: CSSRule[], prefix: string = '') {
    let css = '';

    rules.forEach(rule => {
      // 几种类型的样式规则，所有类型查看 https://developer.mozilla.org/zh-CN/docs/Web/API/CSSRule#%E7%B1%BB%E5%9E%8B%E5%B8%B8%E9%87%8F
      switch (rule.type) {
        // 最常见的 selector { prop: val }
        case RuleType.STYLE:
          /**
           * 含有根元素选择器的情况：用前缀替换掉选择器中的根元素选择器部分，
           * 普通选择器：将前缀插到第一个选择器的后面
           */
          css += this.ruleStyle(rule as CSSStyleRule, prefix);
          break;
        // 媒体 @media screen and (max-width: 300px) { prop: val }
        case RuleType.MEDIA:
          // 拿到其中的具体样式规则，然后调用 rewrite 通过 ruleStyle 去处理
          css += this.ruleMedia(rule as CSSMediaRule, prefix);
          break;
        // @supports (display: grid) {}
        case RuleType.SUPPORTS:
          // 拿到其中的具体样式规则，然后调用 rewrite 通过 ruleStyle 去处理
          css += this.ruleSupport(rule as CSSSupportsRule, prefix);
          break;
        // 其它，直接返回样式内容
        default:
          css += `${rule.cssText}`;
          break;
      }
    });

    return css;
  }

  /**
   * 普通的根选择器用前缀代替
   * 根组合选择器置空，忽略非标准形式的兄弟选择器，比如 html + body {...}
   * 针对普通选择器则是在第一个选择器后面插入前缀，比如 .xx 变成 .xxprefix
   * 
   * 总结就是：
   *  含有根元素选择器的情况：用前缀替换掉选择器中的根元素选择器部分，
   *  普通选择器：将前缀插到第一个选择器的后面
   * 
   * handle case:
   * .app-main {}
   * html, body {}
   * 
   * @param rule 比如：.app-main {} 或者 html, body {}
   * @param prefix `div[data-qiankun]=${appName}`
   */
  // eslint-disable-next-line class-methods-use-this
  private ruleStyle(rule: CSSStyleRule, prefix: string) {
    // 根选择，比如 html、body、:root
    const rootSelectorRE = /((?:[^\w\-.#]|^)(body|html|:root))/gm;
    // 根组合选择器，比如 html body {...} 、 html > body {...}
    const rootCombinationRE = /(html[^\w{[]+)/gm;

    // 选择器
    const selector = rule.selectorText.trim();

    // 样式文本
    let { cssText } = rule;

    // 如果选择器为根选择器，则直接用前缀将根选择器替换掉
    // handle html { ... }
    // handle body { ... }
    // handle :root { ... }
    if (selector === 'html' || selector === 'body' || selector === ':root') {
      return cssText.replace(rootSelectorRE, prefix);
    }

    // 根组合选择器
    // handle html body { ... }
    // handle html > body { ... }
    if (rootCombinationRE.test(rule.selectorText)) {
      // 兄弟选择器 html + body，非标准选择器，无效，转换时忽略
      const siblingSelectorRE = /(html[^\w{]+)(\+|~)/gm;

      // since html + body is a non-standard rule for html
      // transformer will ignore it
      if (!siblingSelectorRE.test(rule.selectorText)) {
        // 说明时 html + body 这种非标准形式，则将根组合器置空
        cssText = cssText.replace(rootCombinationRE, '');
      }
    }

    // 其它一般选择器，比如 类选择器、id 选择器、元素选择器、组合选择器等
    // handle grouping selector, a,span,p,div { ... }
    cssText = cssText.replace(/^[\s\S]+{/, selectors =>
      // item 是匹配的字串，p 是第一个分组匹配的内容，s 是第二个分组匹配的内容
      selectors.replace(/(^|,\n?)([^,]+)/g, (item, p, s) => {
        // handle div,body,span { ... }
        if (rootSelectorRE.test(item)) {
          // 说明选择器中含有根元素选择器
          return item.replace(rootSelectorRE, m => {
            // do not discard valid previous character, such as body,html or *:not(:root)
            const whitePrevChars = [',', '('];

            // 将其中的根元素替换为前缀
            if (m && whitePrevChars.includes(m[0])) {
              return `${m[0]}${prefix}`;
            }

            // replace root selector with prefix
            return prefix;
          });
        }

        // selector1 selector2 =》 selector1prefix selector2
        return `${p}${prefix} ${s.replace(/^ */, '')}`;
      }),
    );

    return cssText;
  }

  // 拿到其中的具体样式规则，然后调用 rewrite 通过 ruleStyle 去处理
  // handle case:
  // @media screen and (max-width: 300px) {}
  private ruleMedia(rule: CSSMediaRule, prefix: string) {
    const css = this.rewrite(arrayify(rule.cssRules), prefix);
    return `@media ${rule.conditionText} {${css}}`;
  }

  // 拿到其中的具体样式规则，然后调用 rewrite 通过 ruleStyle 去处理
  // handle case:
  // @supports (display: grid) {}
  private ruleSupport(rule: CSSSupportsRule, prefix: string) {
    const css = this.rewrite(arrayify(rule.cssRules), prefix);
    return `@supports ${rule.conditionText} {${css}}`;
  }
}
```

## 结语

以上内容就是对 qiankun 框架的完整解读了，相信你在阅读完这篇文章以后会有不错的收获，源码在 [github](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fliyongning%2Fqiankun.git)

阅读 qiankun 时的感受就是 `书读百变其义自现`，qiankun 框架有些地方实现还是比较难理解的，相信大家阅读源码时也会有这个感受，那就多读几遍吧，当然也可以来评论区交流，共同学习，共同进步！！

## 链接

* [微前端专栏](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzA3NTk4NjQ1OQ==&action=getalbum&album_id=2251416802327232513&scene=173&from_msgid=2247484245&from_itemidx=1&count=3&nolastread=1#wechat_redirect)

- [github](https://github.com/liyongning/qiankun)



感谢各位的：**点赞**、**收藏**和**评论**，我们下期见。

---

**当学习成为了习惯，知识也就变成了常识。**感谢各位的 **点赞**、**收藏**和**评论**。

新视频和文章会第一时间在微信公众号发送，欢迎关注：[李永宁lyn](https://gitee.com/liyongning/typora-image-bed/raw/master/202202171742614.jpg)

文章已收录到 [github](https://github.com/liyongning/blog)，欢迎 Watch 和 Star。


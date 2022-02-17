# qiankun 2.x 运行时沙箱 源码分析

**当学习成为了习惯，知识也就变成了常识。**感谢各位的 **点赞**、**收藏**和**评论**。

新视频和文章会第一时间在微信公众号发送，欢迎关注：[李永宁lyn](https://gitee.com/liyongning/typora-image-bed/raw/master/202202171742614.jpg)

文章已收录到 [github](https://github.com/liyongning/blog)，欢迎 Watch 和 Star。

## 封面

![](https://gitee.com/liyongning/typora-image-bed/raw/master/202202051853215.jpg)

## 简介

从源码层面详细讲解了 `qiankun` 框架中的 JS 沙箱 和 样式沙箱的实现原理。

## 序言

`沙箱` 这个词想必大家应该不陌生，即使陌生，读完这篇文章也就不那么陌生了

沙箱 (Sandboxie) ，又叫沙盘，即是一个虚拟系统程序，允许你在沙盘环境中运行浏览器或其他程序，因此运行所产生的变化可以随后删除。它创造了一个类似沙盒的独立作业环境，在其内部运行的程序并不能对硬盘产生永久性的影响。 在网络安全中，沙箱指在隔离环境中，用以测试不受信任的文件或应用程序等行为的工具

而今天要说的沙箱来自 `qiankun` 的实现，是为了解决微前端方案中的隔离问题，`qiankun` 目前可以说的最好的 `微前端` 实现方案吧，它基于 `single-spa` 做了二次封装，解决了 `single-spa` 遗留的众多问题， `运行时沙箱` 就是其中之一

## 为什么需要它

`single-spa` 虽好，但却存在一些需要在框架层面解决但却没有解决的问题，比如为每个微应用提供一个干净、独立的运行环境

JS 全局对象污染是一个很常见的现象，比如：微应用 A 在全局对象上添加了一个自己特有的属性 `window.A`，这时候切换到微应用 B，这时候如何保证 `window`对象是干净的呢？答案就是 `qiankun` 实现的 `运行时沙箱`

## 总结

先上总结，运行时沙箱分为 `JS 沙箱` 和 `样式沙箱`

### JS 沙箱

JS 沙箱，通过 proxy 代理 window 对象，记录 window 对象上属性的增删改查

   * 单例模式

     > 直接代理了原生 window 对象，记录原生 window 对象的增删改查，当 window 对象激活时恢复 window 对象到上次即将失活时的状态，失活时恢复 window 对象到初始初始状态

   * 多例模式

     > 代理了一个全新的对象，这个对象是复制的 window 对象的一部分不可配置属性，所有的更改都是基于这个 fakeWindow 对象，从而保证多个实例之间属性互不影响

将这个 proxy 作为微应用的全局对象，所有的操作都在这个 proxy 对象上，这就是 JS 沙箱的原理

### 样式沙箱

通过增强多例模式下的 createElement 方法，负责创建元素并劫持 script、link、style 三个标签的创建动作

增强 appendChild、insertBefore 方法，负责添加元素，并劫持 script、link、style 三个标签的添加动作，根据是否是主应用调用决定标签是插入到主应用还是微应用，并且将 proxy 对象传递给微应用，作为其全局对象，以达到 JS 隔离的目的

初始化完成后返回一个 free 函数，会在微应用卸载时被调用，负责清除 patch、缓存动态添加的样式（因为微应用被卸载后所有的相关DOM元素都会被删掉）

free 函数执行完成后返回 rebuild 函数，在微应用重新挂载时会被调用，负责向微应用添加刚才缓存的动态样式

其实严格来说这个样式沙箱有点名不副实，真正的样式隔离是 严格样式隔离模式 和 scoped css模式 提供的，当然如果开启了 scoped css，样式沙箱中动态添加的样式也会经过 scoped css 的处理

回到正题，样式沙箱实际做的事情其实很简单，就是将动态添加的 script、link、style 这三个元素插入到对的位置，属于主应用的插入主应用，属于微应用的插入到对应的微应用中，方便微应用卸载的时候一起删除，

当然样式沙箱还额外做了两件事：

* 在卸载之前为动态添加样式做缓存，在微应用重新挂载时再插入到微应用内
* 将 proxy 对象传递给 execScripts 函数，将其设置为微应用的执行上下文

以上内容就是对运行时沙箱的一个总结，更加详细的实现过程，可继续阅读下面的源码分析部分

## 源码分析

接下来就进入令人头昏脑胀的源码分析部分，说实话，运行时沙箱这段代码还是有一些难度的，我在阅读 `qiankun` 源码的时候，这部分内容反反复复阅读了好几遍，[github](https://github.com/liyongning/qiankun.git)

### 入口位置 - createSandbox

```typescript
/**
 * 生成运行时沙箱，这个沙箱其实由两部分组成 => JS 沙箱（执行上下文）、样式沙箱
 *
 * @param appName 微应用名称
 * @param elementGetter getter 函数，通过该函数可以获取 <div id="__qiankun_microapp_wrapper_for_${appInstanceId}__" data-name="${appName}">${template}</div>
 * @param singular 是否单例模式
 * @param scopedCSS
 * @param excludeAssetFilter 指定部分特殊的动态加载的微应用资源（css/js) 不被 qiankun 劫持处理
 */
export function createSandbox(
  appName: string,
  elementGetter: () => HTMLElement | ShadowRoot,
  singular: boolean,
  scopedCSS: boolean,
  excludeAssetFilter?: (url: string) => boolean,
) {
  /**
   * JS 沙箱，通过 proxy 代理 window 对象，记录 window 对象上属性的增删改查，区别在于：
   *  单例模式直接代理了原生 window 对象，记录原生 window 对象的增删改查，当 window 对象激活时恢复 window 对象到上次即将失活时的状态，
   * 失活时恢复 window 对象到初始初始状态
   *  多例模式代理了一个全新的对象，这个对象是复制的 window 对象的一部分不可配置属性，所有的更改都是基于这个 fakeWindow 对象，从而保证多个实例
   * 之间属性互不影响
   * 后面会将 sandbox.proxy 作为微应用的全局对象，所有的操作都在这个 proxy 对象上，这就是 JS 沙箱的原理
   */
  let sandbox: SandBox;
  if (window.Proxy) {
    sandbox = singular ? new LegacySandbox(appName) : new ProxySandbox(appName);
  } else {
    // 不支持 proxy 的浏览器，通过 diff 方式实现的沙箱
    sandbox = new SnapshotSandbox(appName);
  }

  /**
   * 样式沙箱
   * 
   * 增强多例模式下的 createElement 方法，负责创建元素并劫持 script、link、style 三个标签的创建动作
   * 增强 appendChild、insertBefore 方法，负责添加元素，并劫持 script、link、style 三个标签的添加动作，做一些特殊的处理 => 
   * 根据是否是主应用调用决定标签是插入到主应用还是微应用，并且将 proxy 对象传递给微应用，作为其全局对象，以达到 JS 隔离的目的
   * 初始化完成后返回 free 函数，会在微应用卸载时被调用，负责清除 patch、缓存动态添加的样式（因为微应用被卸载后所有的相关DOM元素都会被删掉）
   * free 函数执行完成后返回 rebuild 函数，在微应用重新挂载时会被调用，负责向微应用添加刚才缓存的动态样式
   * 
   * 其实严格来说这个样式沙箱有点名不副实，真正的样式隔离是之前说的 严格样式隔离模式 和 scoped css模式 提供的，当然如果开启了 scoped css，
   * 样式沙箱中动态添加的样式也会经过 scoped css 的处理；回到正题，样式沙箱实际做的事情其实很简单，将动态添加的 script、link、style 
   * 这三个元素插入到对的位置，属于主应用的插入主应用，属于微应用的插入到对应的微应用中，方便微应用卸载的时候一起删除，
   * 当然样式沙箱还额外做了两件事：一、在卸载之前为动态添加样式做缓存，在微应用重新挂载时再插入到微应用内，二、将 proxy 对象传递给 execScripts
   * 函数，将其设置为微应用的执行上下文
   */
  const bootstrappingFreers = patchAtBootstrapping(
    appName,
    elementGetter,
    sandbox,
    singular,
    scopedCSS,
    excludeAssetFilter,
  );
  // mounting freers are one-off and should be re-init at every mounting time
  // mounting freers 是一次性的，应该在每次挂载时重新初始化
  let mountingFreers: Freer[] = [];

  let sideEffectsRebuilders: Rebuilder[] = [];

  return {
    proxy: sandbox.proxy,

    /**
     * 沙箱被 mount
     * 可能是从 bootstrap 状态进入的 mount
     * 也可能是从 unmount 之后再次唤醒进入 mount
     * mount 时重建副作用(rebuild 函数），即微应用在被卸载时希望重新挂载时做的一些事情，比如重建缓存的动态样式
     */
    async mount() {
      /* ------------------------------------------ 因为有上下文依赖（window），以下代码执行顺序不能变 ------------------------------------------ */

      /* ------------------------------------------ 1. 启动/恢复 沙箱------------------------------------------ */
      sandbox.active();

      const sideEffectsRebuildersAtBootstrapping = sideEffectsRebuilders.slice(0, bootstrappingFreers.length);
      const sideEffectsRebuildersAtMounting = sideEffectsRebuilders.slice(bootstrappingFreers.length);

      // must rebuild the side effects which added at bootstrapping firstly to recovery to nature state
      if (sideEffectsRebuildersAtBootstrapping.length) {
        // 微应用再次挂载时重建刚才缓存的动态样式
        sideEffectsRebuildersAtBootstrapping.forEach(rebuild => rebuild());
      }

      /* ------------------------------------------ 2. 开启全局变量补丁 ------------------------------------------*/
      // render 沙箱启动时开始劫持各类全局监听，尽量不要在应用初始化阶段有 事件监听/定时器 等副作用
      mountingFreers = patchAtMounting(appName, elementGetter, sandbox, singular, scopedCSS, excludeAssetFilter);

      /* ------------------------------------------ 3. 重置一些初始化时的副作用 ------------------------------------------*/
      // 存在 rebuilder 则表明有些副作用需要重建
      // 现在只看到针对 umi 的那个 patchHistoryListener 有 rebuild 操作
      if (sideEffectsRebuildersAtMounting.length) {
        sideEffectsRebuildersAtMounting.forEach(rebuild => rebuild());
      }

      // clean up rebuilders，卸载时会再填充回来
      sideEffectsRebuilders = [];
    },

    /**
     * 恢复 global 状态，使其能回到应用加载之前的状态
     */
    // 撤销初始化和挂载阶段打的 patch；缓存微应用希望自己再次被挂载时需要做的一些事情（rebuild），比如重建动态样式表；失活微应用
    async unmount() {
      // record the rebuilders of window side effects (event listeners or timers)
      // note that the frees of mounting phase are one-off as it will be re-init at next mounting
      // 卸载时，执行 free 函数，释放初始化和挂载时打的 patch，存储所有的 rebuild 函数，在微应用再次挂载时重建通过 patch 做的事情（副作用）
      sideEffectsRebuilders = [...bootstrappingFreers, ...mountingFreers].map(free => free());

      sandbox.inactive();
    },
  };
}

```

#### JS 沙箱

##### SingularProxySandbox 单例 JS 沙箱

```typescript
/**
 * 基于 Proxy 实现的单例模式下的沙箱，直接操作原生 window 对象，并记录 window 对象的增删改查，在每次微应用切换时初始化 window 对象；
 * 激活时：将 window 对象恢复到上次即将失活时的状态
 * 失活时：将 window 对象恢复为初始状态
 * 
 * TODO: 为了兼容性 singular 模式下依旧使用该沙箱，等新沙箱稳定之后再切换
 */
export default class SingularProxySandbox implements SandBox {
  // 沙箱期间新增的全局变量
  private addedPropsMapInSandbox = new Map<PropertyKey, any>();

  // 沙箱期间更新的全局变量，key 为被更新的属性，value 为被更新的值
  private modifiedPropsOriginalValueMapInSandbox = new Map<PropertyKey, any>();

  // 持续记录更新的(新增和修改的)全局变量的 map，用于在任意时刻做 snapshot
  private currentUpdatedPropsValueMap = new Map<PropertyKey, any>();

  name: string;

  proxy: WindowProxy;

  type: SandBoxType;

  sandboxRunning = true;

  // 激活沙箱
  active() {
    // 如果沙箱由失活 -> 激活，则恢复 window 对象到上次失活时的状态
    if (!this.sandboxRunning) {
      this.currentUpdatedPropsValueMap.forEach((v, p) => setWindowProp(p, v));
    }

    // 切换沙箱状态为 激活
    this.sandboxRunning = true;
  }

  // 失活沙箱
  inactive() {
    // 开发环境，打印被改变的全局属性
    if (process.env.NODE_ENV === 'development') {
      console.info(`[qiankun:sandbox] ${this.name} modified global properties restore...`, [
        ...this.addedPropsMapInSandbox.keys(),
        ...this.modifiedPropsOriginalValueMapInSandbox.keys(),
      ]);
    }

    // restore global props to initial snapshot
    // 将被更改的全局属性再改回去
    this.modifiedPropsOriginalValueMapInSandbox.forEach((v, p) => setWindowProp(p, v));
    // 新增的属性删掉
    this.addedPropsMapInSandbox.forEach((_, p) => setWindowProp(p, undefined, true));

    // 切换沙箱状态为 失活
    this.sandboxRunning = false;
  }

  constructor(name: string) {
    this.name = name;
    this.type = SandBoxType.LegacyProxy;
    const { addedPropsMapInSandbox, modifiedPropsOriginalValueMapInSandbox, currentUpdatedPropsValueMap } = this;

    const self = this;
    const rawWindow = window;
    const fakeWindow = Object.create(null) as Window;

    const proxy = new Proxy(fakeWindow, {
      set(_: Window, p: PropertyKey, value: any): boolean {
        if (self.sandboxRunning) {
          if (!rawWindow.hasOwnProperty(p)) {
            // 属性不存在，则添加
            addedPropsMapInSandbox.set(p, value);
          } else if (!modifiedPropsOriginalValueMapInSandbox.has(p)) {
            // 如果当前 window 对象存在该属性，且 record map 中未记录过，则记录该属性初始值，说明是更改已存在的属性
            const originalValue = (rawWindow as any)[p];
            modifiedPropsOriginalValueMapInSandbox.set(p, originalValue);
          }

          currentUpdatedPropsValueMap.set(p, value);
          // 直接设置原生 window 对象，因为是单例模式，不会有其它的影响
          // eslint-disable-next-line no-param-reassign
          (rawWindow as any)[p] = value;

          return true;
        }

        if (process.env.NODE_ENV === 'development') {
          console.warn(`[qiankun] Set window.${p.toString()} while sandbox destroyed or inactive in ${name}!`);
        }

        // 在 strict-mode 下，Proxy 的 handler.set 返回 false 会抛出 TypeError，在沙箱卸载的情况下应该忽略错误
        return true;
      },

      get(_: Window, p: PropertyKey): any {
        // avoid who using window.window or window.self to escape the sandbox environment to touch the really window
        // or use window.top to check if an iframe context
        // see https://github.com/eligrey/FileSaver.js/blob/master/src/FileSaver.js#L13
        if (p === 'top' || p === 'parent' || p === 'window' || p === 'self') {
          return proxy;
        }

        // 直接从原生 window 对象拿数据
        const value = (rawWindow as any)[p];
        return getTargetValue(rawWindow, value);
      },

      // trap in operator
      // see https://github.com/styled-components/styled-components/blob/master/packages/styled-components/src/constants.js#L12
      has(_: Window, p: string | number | symbol): boolean {
        return p in rawWindow;
      },
    });

    this.proxy = proxy;
  }
}

/**
 * 在 window 对象上设置 key value 或 删除指定属性(key)
 * @param prop key
 * @param value value
 * @param toDelete 是否删除
 */
function setWindowProp(prop: PropertyKey, value: any, toDelete?: boolean) {
  if (value === undefined && toDelete) {
    // 删除 window[key]
    delete (window as any)[prop];
  } else if (isPropConfigurable(window, prop) && typeof prop !== 'symbol') {
    // window[key] = value
    Object.defineProperty(window, prop, { writable: true, configurable: true });
    (window as any)[prop] = value;
  }
}

```

##### ProxySandbox 多例 JS 沙箱

```typescript
// 记录被激活的沙箱的数量
let activeSandboxCount = 0;

/**
 * 基于 Proxy 实现的多例模式下的沙箱
 * 通过 proxy 代理 fakeWindow 对象，所有的更改都是基于 fakeWindow，这点和单例不一样（很重要），
 * 从而保证每个 ProxySandbox 实例之间属性互不影响
 */
export default class ProxySandbox implements SandBox {
  /** window 值变更记录 */
  private updatedValueSet = new Set<PropertyKey>();

  name: string;

  type: SandBoxType;

  proxy: WindowProxy;

  sandboxRunning = true;

  // 激活
  active() {
    // 被激活的沙箱数 + 1
    if (!this.sandboxRunning) activeSandboxCount++;
    this.sandboxRunning = true;
  }

  // 失活
  inactive() {
    if (process.env.NODE_ENV === 'development') {
      console.info(`[qiankun:sandbox] ${this.name} modified global properties restore...`, [
        ...this.updatedValueSet.keys(),
      ]);
    }

    // 被激活的沙箱数 - 1
    clearSystemJsProps(this.proxy, --activeSandboxCount === 0);

    this.sandboxRunning = false;
  }

  constructor(name: string) {
    this.name = name;
    this.type = SandBoxType.Proxy;
    const { updatedValueSet } = this;

    const self = this;
    const rawWindow = window;
    // 全局对象上所有不可配置属性都在 fakeWindow 中，且其中具有 getter 属性的属性还存在 propertesWithGetter map 中，value 为 true
    const { fakeWindow, propertiesWithGetter } = createFakeWindow(rawWindow);

    const descriptorTargetMap = new Map<PropertyKey, SymbolTarget>();
    // 判断全局对象是否存在指定属性
    const hasOwnProperty = (key: PropertyKey) => fakeWindow.hasOwnProperty(key) || rawWindow.hasOwnProperty(key);

    const proxy = new Proxy(fakeWindow, {
      set(target: FakeWindow, p: PropertyKey, value: any): boolean {
        // 如果沙箱在运行，则更新属性值并记录被更改的属性
        if (self.sandboxRunning) {
          // 设置属性值
          // @ts-ignore
          target[p] = value;
          // 记录被更改的属性
          updatedValueSet.add(p);

          // 不用管，和 systemJs 有关
          interceptSystemJsProps(p, value);

          return true;
        }

        if (process.env.NODE_ENV === 'development') {
          console.warn(`[qiankun] Set window.${p.toString()} while sandbox destroyed or inactive in ${name}!`);
        }

        // 在 strict-mode 下，Proxy 的 handler.set 返回 false 会抛出 TypeError，在沙箱卸载的情况下应该忽略错误
        return true;
      },

      // 获取执行属性的值
      get(target: FakeWindow, p: PropertyKey): any {
        if (p === Symbol.unscopables) return unscopables;

        // avoid who using window.window or window.self to escape the sandbox environment to touch the really window
        // see https://github.com/eligrey/FileSaver.js/blob/master/src/FileSaver.js#L13
        if (p === 'window' || p === 'self') {
          return proxy;
        }

        if (
          p === 'top' ||
          p === 'parent' ||
          (process.env.NODE_ENV === 'test' && (p === 'mockTop' || p === 'mockSafariTop'))
        ) {
          // if your master app in an iframe context, allow these props escape the sandbox
          if (rawWindow === rawWindow.parent) {
            return proxy;
          }
          return (rawWindow as any)[p];
        }

        // proxy.hasOwnProperty would invoke getter firstly, then its value represented as rawWindow.hasOwnProperty
        if (p === 'hasOwnProperty') {
          return hasOwnProperty;
        }

        // mark the symbol to document while accessing as document.createElement could know is invoked by which sandbox for dynamic append patcher
        if (p === 'document') {
          document[attachDocProxySymbol] = proxy;
          // remove the mark in next tick, thus we can identify whether it in micro app or not
          // this approach is just a workaround, it could not cover all the complex scenarios, such as the micro app runs in the same task context with master in som case
          // fixme if you have any other good ideas
          nextTick(() => delete document[attachDocProxySymbol]);
          return document;
        }
        // 以上内容都是一些特殊属性的处理

        // 获取特定属性，如果属性具有 getter，说明是原生对象的那几个属性，否则是 fakeWindow 对象上的属性（原生的或者用户设置的)
        // eslint-disable-next-line no-bitwise
        const value = propertiesWithGetter.has(p) ? (rawWindow as any)[p] : (target as any)[p] || (rawWindow as any)[p];
        return getTargetValue(rawWindow, value);
      },

      // 判断是否存在指定属性
      // see https://github.com/styled-components/styled-components/blob/master/packages/styled-components/src/constants.js#L12
      has(target: FakeWindow, p: string | number | symbol): boolean {
        return p in unscopables || p in target || p in rawWindow;
      },

      getOwnPropertyDescriptor(target: FakeWindow, p: string | number | symbol): PropertyDescriptor | undefined {
        /*
         as the descriptor of top/self/window/mockTop in raw window are configurable but not in proxy target, we need to get it from target to avoid TypeError
         see https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy/handler/getOwnPropertyDescriptor
         > A property cannot be reported as non-configurable, if it does not exists as an own property of the target object or if it exists as a configurable own property of the target object.
         */
        if (target.hasOwnProperty(p)) {
          const descriptor = Object.getOwnPropertyDescriptor(target, p);
          descriptorTargetMap.set(p, 'target');
          return descriptor;
        }

        if (rawWindow.hasOwnProperty(p)) {
          const descriptor = Object.getOwnPropertyDescriptor(rawWindow, p);
          descriptorTargetMap.set(p, 'rawWindow');
          // A property cannot be reported as non-configurable, if it does not exists as an own property of the target object
          if (descriptor && !descriptor.configurable) {
            descriptor.configurable = true;
          }
          return descriptor;
        }

        return undefined;
      },

      // trap to support iterator with sandbox
      ownKeys(target: FakeWindow): PropertyKey[] {
        return uniq(Reflect.ownKeys(rawWindow).concat(Reflect.ownKeys(target)));
      },

      defineProperty(target: Window, p: PropertyKey, attributes: PropertyDescriptor): boolean {
        const from = descriptorTargetMap.get(p);
        /*
         Descriptor must be defined to native window while it comes from native window via Object.getOwnPropertyDescriptor(window, p),
         otherwise it would cause a TypeError with illegal invocation.
         */
        switch (from) {
          case 'rawWindow':
            return Reflect.defineProperty(rawWindow, p, attributes);
          default:
            return Reflect.defineProperty(target, p, attributes);
        }
      },

      deleteProperty(target: FakeWindow, p: string | number | symbol): boolean {
        if (target.hasOwnProperty(p)) {
          // @ts-ignore
          delete target[p];
          updatedValueSet.delete(p);

          return true;
        }

        return true;
      },
    });

    this.proxy = proxy;
  }
}

```

##### createFakeWindow

```typescript
/**
 * 拷贝全局对象上所有不可配置属性到 fakeWindow 对象，并将这些属性的属性描述符改为可配置的然后冻结
 * 将启动具有 getter 属性的属性再存入 propertiesWithGetter map 中
 * @param global 全局对象 => window
 */
function createFakeWindow(global: Window) {
  // 记录 window 对象上的 getter 属性，原生的有：window、document、location、top，比如：Object.getOwnPropertyDescriptor(window, 'window') => {set: undefined, enumerable: true, configurable: false, get: ƒ}
  // propertiesWithGetter = {"window" => true, "document" => true, "location" => true, "top" => true, "__VUE_DEVTOOLS_GLOBAL_HOOK__" => true}
  const propertiesWithGetter = new Map<PropertyKey, boolean>();
  // 存储 window 对象中所有不可配置的属性和值
  const fakeWindow = {} as FakeWindow;

  /*
   copy the non-configurable property of global to fakeWindow
   see https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy/handler/getOwnPropertyDescriptor
   > A property cannot be reported as non-configurable, if it does not exists as an own property of the target object or if it exists as a configurable own property of the target object.
   */
  Object.getOwnPropertyNames(global)
    // 遍历出 window 对象所有不可配置属性
    .filter(p => {
      const descriptor = Object.getOwnPropertyDescriptor(global, p);
      return !descriptor?.configurable;
    })
    .forEach(p => {
      // 得到属性描述符
      const descriptor = Object.getOwnPropertyDescriptor(global, p);
      if (descriptor) {
        // 获取其 get 属性
        const hasGetter = Object.prototype.hasOwnProperty.call(descriptor, 'get');

        /*
         make top/self/window property configurable and writable, otherwise it will cause TypeError while get trap return.
         see https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy/handler/get
         > The value reported for a property must be the same as the value of the corresponding target object property if the target object property is a non-writable, non-configurable data property.
         */
        if (
          p === 'top' ||
          p === 'parent' ||
          p === 'self' ||
          p === 'window' ||
          (process.env.NODE_ENV === 'test' && (p === 'mockTop' || p === 'mockSafariTop'))
        ) {
          // 将 top、parent、self、window 这几个属性由不可配置改为可配置
          descriptor.configurable = true;
          /*
           The descriptor of window.window/window.top/window.self in Safari/FF are accessor descriptors, we need to avoid adding a data descriptor while it was
           Example:
            Safari/FF: Object.getOwnPropertyDescriptor(window, 'top') -> {get: function, set: undefined, enumerable: true, configurable: false}
            Chrome: Object.getOwnPropertyDescriptor(window, 'top') -> {value: Window, writable: false, enumerable: true, configurable: false}
           */
          if (!hasGetter) {
            // 如果这几个属性没有 getter，则说明由 writeable 属性，将其设置为可写
            descriptor.writable = true;
          }
        }

        // 如果存在 getter，则以该属性为 key，true 为 value 存入 propertiesWithGetter map
        if (hasGetter) propertiesWithGetter.set(p, true);

        // 将属性和描述设置到 fakeWindow 对象，并且冻结属性描述符，不然有可能会被更改，比如 zone.js
        // freeze the descriptor to avoid being modified by zone.js
        // see https://github.com/angular/zone.js/blob/a5fe09b0fac27ac5df1fa746042f96f05ccb6a00/lib/browser/define-property.ts#L71
        rawObjectDefineProperty(fakeWindow, p, Object.freeze(descriptor));
      }
    });

  return {
    fakeWindow,
    propertiesWithGetter,
  };
}
            
```

##### SnapshotSandbox

```typescript
function iter(obj: object, callbackFn: (prop: any) => void) {
  // eslint-disable-next-line guard-for-in, no-restricted-syntax
  for (const prop in obj) {
    if (obj.hasOwnProperty(prop)) {
      callbackFn(prop);
    }
  }
}

/**
 * 基于 diff 方式实现的沙箱，用于不支持 Proxy 的低版本浏览器
 */
export default class SnapshotSandbox implements SandBox {
  proxy: WindowProxy;

  name: string;

  type: SandBoxType;

  sandboxRunning = true;

  private windowSnapshot!: Window;

  private modifyPropsMap: Record<any, any> = {};

  constructor(name: string) {
    this.name = name;
    this.proxy = window;
    this.type = SandBoxType.Snapshot;
  }

  active() {
    // 记录当前快照
    this.windowSnapshot = {} as Window;
    iter(window, prop => {
      this.windowSnapshot[prop] = window[prop];
    });

    // 恢复之前的变更
    Object.keys(this.modifyPropsMap).forEach((p: any) => {
      window[p] = this.modifyPropsMap[p];
    });

    this.sandboxRunning = true;
  }

  inactive() {
    this.modifyPropsMap = {};

    iter(window, prop => {
      if (window[prop] !== this.windowSnapshot[prop]) {
        // 记录变更，恢复环境
        this.modifyPropsMap[prop] = window[prop];
        window[prop] = this.windowSnapshot[prop];
      }
    });

    if (process.env.NODE_ENV === 'development') {
      console.info(`[qiankun:sandbox] ${this.name} origin window restore...`, Object.keys(this.modifyPropsMap));
    }

    this.sandboxRunning = false;
  }
}

```

#### 样式沙箱

##### patchAtBootstrapping

```typescript
/**
 * 初始化阶段给 createElement、appendChild、insertBefore 三个方法打一个 patch
 * @param appName 
 * @param elementGetter 
 * @param sandbox 
 * @param singular 
 * @param scopedCSS 
 * @param excludeAssetFilter 
 */
export function patchAtBootstrapping(
  appName: string,
  elementGetter: () => HTMLElement | ShadowRoot,
  sandbox: SandBox,
  singular: boolean,
  scopedCSS: boolean,
  excludeAssetFilter?: Function,
): Freer[] {
  // 基础 patch，增强 createElement、appendChild、insertBefore 三个方法
  const basePatchers = [
    () => patchDynamicAppend(appName, elementGetter, sandbox.proxy, false, singular, scopedCSS, excludeAssetFilter),
  ];

  // 每一种沙箱都需要打基础 patch
  const patchersInSandbox = {
    [SandBoxType.LegacyProxy]: basePatchers,
    [SandBoxType.Proxy]: basePatchers,
    [SandBoxType.Snapshot]: basePatchers,
  };

  // 返回一个数组，数组元素是 patch 的执行结果 => free 函数
  return patchersInSandbox[sandbox.type]?.map(patch => patch());
}

```

##### patch

```typescript
/**
 * 增强多例模式下的 createElement 方法，负责创建元素并劫持 script、link、style 三个标签的创建动作
 * 增强 appendChild、insertBefore 方法，负责添加元素，并劫持 script、link、style 三个标签的添加动作，做一些特殊的处理 => 
 * 根据是否是主应用调用决定标签是插入到主应用还是微应用，并且将 proxy 对象传递给微应用，作为其全局对象，以打包 JS 隔离的目的
 * 初始化完成后返回 free 函数，负责清除 patch、缓存动态添加的样式（因为微应用被卸载后所有的相关DOM元素都会被删掉）
 * free 函数执行完成后返回 rebuild 函数，rebuild 函数在微应用重新挂载时向微应用添加刚才缓存的动态样式
 * 
 * Just hijack dynamic head append, that could avoid accidentally hijacking the insertion of elements except in head.
 * Such a case: ReactDOM.createPortal(<style>.test{color:blue}</style>, container),
 * this could made we append the style element into app wrapper but it will cause an error while the react portal unmounting, as ReactDOM could not find the style in body children list.
 * @param appName 微应用名称
 * @param appWrapperGetter getter 函数，通过该函数可以获取 <div id="__qiankun_microapp_wrapper_for_${appInstanceId}__" data-name="${appName}">${template}</div>
 * @param proxy window 代理
 * @param mounting 是否为 mounting 阶段
 * @param singular 是否为单例
 * @param scopedCSS 是否弃用 scoped css
 * @param excludeAssetFilter 指定部分特殊的动态加载的微应用资源（css/js) 不被 qiankun 劫持处理
 */
export default function patch(
  appName: string,
  appWrapperGetter: () => HTMLElement | ShadowRoot,
  proxy: Window,
  mounting = true,
  singular = true,
  scopedCSS = false,
  excludeAssetFilter?: CallableFunction,
): Freer {
  // 动态样式表，存储所有动态添加的样式
  let dynamicStyleSheetElements: Array<HTMLLinkElement | HTMLStyleElement> = [];

  // 在多例模式下增强 createElement 方法，让其除了可以创建元素，还可以了劫持创建 script、link、style 元素的情况
  const unpatchDocumentCreate = patchDocumentCreateElement(
    appName,
    appWrapperGetter,
    singular,
    proxy,
    dynamicStyleSheetElements,
  );

  // 增强 appendChild、insertBefore、removeChild 三个元素；除了本职工作之外，appendChild 和 insertBefore 还可以额外处理 script、style、link
  // 三个标签的插入，可以根据情况决定元素被插入到微应用模版空间中还是主应用模版空间，removeChild 也是可以根据情况移除主应用的元素还是移除微应用中这三个元素
  const unpatchDynamicAppendPrototypeFunctions = patchHTMLDynamicAppendPrototypeFunctions(
    appName,
    appWrapperGetter,
    proxy,
    singular,
    scopedCSS,
    dynamicStyleSheetElements,
    excludeAssetFilter,
  );

  // 记录初始化的次数
  if (!mounting) bootstrappingPatchCount++;
  // 记录挂载的次数
  if (mounting) mountingPatchCount++;

  // 初始化完成后返回 free 函数，负责清除 patch、缓存动态添加的样式、返回 rebuild 函数，rebuild 函数在微应用重新挂载时向微应用添加刚才缓存的动态样式
  return function free() {
    // bootstrap patch just called once but its freer will be called multiple times
    if (!mounting && bootstrappingPatchCount !== 0) bootstrappingPatchCount--;
    if (mounting) mountingPatchCount--;

    // 判断所有微应用是否都被卸载了
    const allMicroAppUnmounted = mountingPatchCount === 0 && bootstrappingPatchCount === 0;
    // 微应用都卸载以后移除 patch, release the overwrite prototype after all the micro apps unmounted
    unpatchDynamicAppendPrototypeFunctions(allMicroAppUnmounted);
    unpatchDocumentCreate(allMicroAppUnmounted);

    // 因为微应用被卸载的时候会删除掉刚才动态添加的样式，这里缓存了动态添加的样式内容，在微应用卸载后重新挂载时就可以用了
    dynamicStyleSheetElements.forEach(stylesheetElement => {
      if (stylesheetElement instanceof HTMLStyleElement && isStyledComponentsLike(stylesheetElement)) {
        if (stylesheetElement.sheet) {
          // record the original css rules of the style element for restore
          setCachedRules(stylesheetElement, (stylesheetElement.sheet as CSSStyleSheet).cssRules);
        }
      }
    });

    // 返回一个 rebuild 函数，微应用重新挂载时调用
    return function rebuild() {
      // 遍历动态样式表
      dynamicStyleSheetElements.forEach(stylesheetElement => {
        // 像微应用容器中添加样式节点
        document.head.appendChild.call(appWrapperGetter(), stylesheetElement);

        // 添加样式内容到样式节点，这个样式内容从刚才的缓存中找
        if (stylesheetElement instanceof HTMLStyleElement && isStyledComponentsLike(stylesheetElement)) {
          const cssRules = getCachedRules(stylesheetElement);
          if (cssRules) {
            // eslint-disable-next-line no-plusplus
            for (let i = 0; i < cssRules.length; i++) {
              const cssRule = cssRules[i];
              (stylesheetElement.sheet as CSSStyleSheet).insertRule(cssRule.cssText);
            }
          }
        }
      });

      // As the hijacker will be invoked every mounting phase, we could release the cache for gc after rebuilding
      if (mounting) {
        dynamicStyleSheetElements = [];
      }
    };
  };
}

```

##### patchDocumentCreateElement

```typescript
/**
 * 多例模式下增强 createElement 方法，让其除了具有创建元素功能之外，还可以劫持创建 script、link、style 这三个元素的情况
 * @param appName 微应用名称
 * @param appWrapperGetter 
 * @param singular 
 * @param proxy 
 * @param dynamicStyleSheetElements 
 */
function patchDocumentCreateElement(
  appName: string,
  appWrapperGetter: () => HTMLElement | ShadowRoot,
  singular: boolean,
  proxy: Window,
  dynamicStyleSheetElements: HTMLStyleElement[],
) {
  // 如果是单例模式直接 return
  if (singular) {
    return noop;
  }

  // 以微应用运行时的 proxy 为 key，存储该微应用的一些信息，比如 名称、proxy、微应用模版、自定义样式表等
  proxyContainerInfoMapper.set(proxy, { appName, proxy, appWrapperGetter, dynamicStyleSheetElements, singular });

  // 第一个微应用初始化时会执行这段，增强 createElement 方法，让其除了可以创建元素之外，还可以劫持 script、link、style 三个标签的创建动作
  if (Document.prototype.createElement === rawDocumentCreateElement) {
    Document.prototype.createElement = function createElement<K extends keyof HTMLElementTagNameMap>(
      this: Document,
      tagName: K,
      options?: ElementCreationOptions,
    ): HTMLElement {
      // 创建元素
      const element = rawDocumentCreateElement.call(this, tagName, options);
      // 劫持 script、link、style 三种标签
      if (isHijackingTag(tagName)) {
        // 下面这段似乎没啥用，因为没发现有哪个地方执行设置，proxyContainerInfoMapper.set(this[attachDocProxySysbol])
        // 获取这个东西的值，然后将该值添加到 element 对象上，以 attachElementContainerSymbol 为 key
        const proxyContainerInfo = proxyContainerInfoMapper.get(this[attachDocProxySymbol]);
        if (proxyContainerInfo) {
          Object.defineProperty(element, attachElementContainerSymbol, {
            value: proxyContainerInfo,
            enumerable: false,
          });
        }
      }

      // 返回创建的元素
      return element;
    };
  }

  // 后续的微应用初始化时直接返回该函数，负责还原 createElement 方法
  return function unpatch(recoverPrototype: boolean) {
    proxyContainerInfoMapper.delete(proxy);
    if (recoverPrototype) {
      Document.prototype.createElement = rawDocumentCreateElement;
    }
  };
}

```

##### patchTHMLDynamicAppendPrototypeFunctions

```typescript
// 增强 appendChild、insertBefore、removeChild 方法，返回 unpatch 方法，解除增强
function patchHTMLDynamicAppendPrototypeFunctions(
  appName: string,
  appWrapperGetter: () => HTMLElement | ShadowRoot,
  proxy: Window,
  singular = true,
  scopedCSS = false,
  dynamicStyleSheetElements: HTMLStyleElement[],
  excludeAssetFilter?: CallableFunction,
) {
  // Just overwrite it while it have not been overwrite
  if (
    HTMLHeadElement.prototype.appendChild === rawHeadAppendChild &&
    HTMLBodyElement.prototype.appendChild === rawBodyAppendChild &&
    HTMLHeadElement.prototype.insertBefore === rawHeadInsertBefore
  ) {
    // 增强 appendChild 方法
    HTMLHeadElement.prototype.appendChild = getOverwrittenAppendChildOrInsertBefore({
      rawDOMAppendOrInsertBefore: rawHeadAppendChild,
      appName,
      appWrapperGetter,
      proxy,
      singular,
      dynamicStyleSheetElements,
      scopedCSS,
      excludeAssetFilter,
    }) as typeof rawHeadAppendChild;
    HTMLBodyElement.prototype.appendChild = getOverwrittenAppendChildOrInsertBefore({
      rawDOMAppendOrInsertBefore: rawBodyAppendChild,
      appName,
      appWrapperGetter,
      proxy,
      singular,
      dynamicStyleSheetElements,
      scopedCSS,
      excludeAssetFilter,
    }) as typeof rawBodyAppendChild;

    HTMLHeadElement.prototype.insertBefore = getOverwrittenAppendChildOrInsertBefore({
      rawDOMAppendOrInsertBefore: rawHeadInsertBefore as any,
      appName,
      appWrapperGetter,
      proxy,
      singular,
      dynamicStyleSheetElements,
      scopedCSS,
      excludeAssetFilter,
    }) as typeof rawHeadInsertBefore;
  }

  // Just overwrite it while it have not been overwrite
  if (
    HTMLHeadElement.prototype.removeChild === rawHeadRemoveChild &&
    HTMLBodyElement.prototype.removeChild === rawBodyRemoveChild
  ) {
    HTMLHeadElement.prototype.removeChild = getNewRemoveChild({
      appWrapperGetter,
      headOrBodyRemoveChild: rawHeadRemoveChild,
    });
    HTMLBodyElement.prototype.removeChild = getNewRemoveChild({
      appWrapperGetter,
      headOrBodyRemoveChild: rawBodyRemoveChild,
    });
  }

  return function unpatch(recoverPrototype: boolean) {
    if (recoverPrototype) {
      HTMLHeadElement.prototype.appendChild = rawHeadAppendChild;
      HTMLHeadElement.prototype.removeChild = rawHeadRemoveChild;
      HTMLBodyElement.prototype.appendChild = rawBodyAppendChild;
      HTMLBodyElement.prototype.removeChild = rawBodyRemoveChild;

      HTMLHeadElement.prototype.insertBefore = rawHeadInsertBefore;
    }
  };
}

```

##### getOverwrittenAppendChildOrInsertBefore

```typescript
/**
 * 增强 appendChild 和 insertBefore 方法，让其除了具有添加元素的功能之外，还具有一些其它的逻辑，比如：
 * 根据是否是微应用或者特殊元素决定 link、style、script 元素的插入位置是在主应用还是微应用
 * 劫持 script 标签的添加，支持远程加载脚本和设置脚本的执行上下文（proxy）
 * @param opts 
 */
function getOverwrittenAppendChildOrInsertBefore(opts: {
  appName: string;
  proxy: WindowProxy;
  singular: boolean;
  dynamicStyleSheetElements: HTMLStyleElement[];
  appWrapperGetter: CallableFunction;
  rawDOMAppendOrInsertBefore: <T extends Node>(newChild: T, refChild?: Node | null) => T;
  scopedCSS: boolean;
  excludeAssetFilter?: CallableFunction;
}) {
  return function appendChildOrInsertBefore<T extends Node>(
    this: HTMLHeadElement | HTMLBodyElement,
    newChild: T,
    refChild?: Node | null,
  ) {
    // 要插入的元素
    let element = newChild as any;
    // 原始方法
    const { rawDOMAppendOrInsertBefore } = opts;
    if (element.tagName) {
      // 解析参数
      // eslint-disable-next-line prefer-const
      let { appName, appWrapperGetter, proxy, singular, dynamicStyleSheetElements } = opts;
      const { scopedCSS, excludeAssetFilter } = opts;

      // 多例模式会走的一段逻辑
      const storedContainerInfo = element[attachElementContainerSymbol];
      if (storedContainerInfo) {
        // eslint-disable-next-line prefer-destructuring
        appName = storedContainerInfo.appName;
        // eslint-disable-next-line prefer-destructuring
        singular = storedContainerInfo.singular;
        // eslint-disable-next-line prefer-destructuring
        appWrapperGetter = storedContainerInfo.appWrapperGetter;
        // eslint-disable-next-line prefer-destructuring
        dynamicStyleSheetElements = storedContainerInfo.dynamicStyleSheetElements;
        // eslint-disable-next-line prefer-destructuring
        proxy = storedContainerInfo.proxy;
      }

      const invokedByMicroApp = singular
        ? // check if the currently specified application is active
          // While we switch page from qiankun app to a normal react routing page, the normal one may load stylesheet dynamically while page rendering,
          // but the url change listener must to wait until the current call stack is flushed.
          // This scenario may cause we record the stylesheet from react routing page dynamic injection,
          // and remove them after the url change triggered and qiankun app is unmouting
          // see https://github.com/ReactTraining/history/blob/master/modules/createHashHistory.js#L222-L230
          checkActivityFunctions(window.location).some(name => name === appName)
        : // have storedContainerInfo means it invoked by a micro app in multiply mode
          !!storedContainerInfo;

      switch (element.tagName) {
        // link 和 style
        case LINK_TAG_NAME:
        case STYLE_TAG_NAME: {
          // 断言，newChild 为 style 或者 link 标签
          const stylesheetElement: HTMLLinkElement | HTMLStyleElement = newChild as any;
          // href 属性
          const { href } = stylesheetElement as HTMLLinkElement;
          if (!invokedByMicroApp || (excludeAssetFilter && href && excludeAssetFilter(href))) {
            // 进来则说明，这个创建元素的动作不是微应用调用的，或者是一个特殊指定不希望被 qiankun 劫持的 link 标签
            // 将其创建到主应用的下
            return rawDOMAppendOrInsertBefore.call(this, element, refChild) as T;
          }

          // 微应用容器 DOM
          const mountDOM = appWrapperGetter();

          // scoped css
          if (scopedCSS) {
            css.process(mountDOM, stylesheetElement, appName);
          }

          // 将该元素存到样式表中
          // eslint-disable-next-line no-shadow
          dynamicStyleSheetElements.push(stylesheetElement);
          // 参考元素
          const referenceNode = mountDOM.contains(refChild) ? refChild : null;
          // 将该元素在微应用的空间中创建，这样卸载微应用的时候就可以直接一起删除了
          return rawDOMAppendOrInsertBefore.call(mountDOM, stylesheetElement, referenceNode);
        }

        // script 标签
        case SCRIPT_TAG_NAME: {
          // 链接和文本
          const { src, text } = element as HTMLScriptElement;
          // some script like jsonp maybe not support cors which should't use execScripts
          if (!invokedByMicroApp || (excludeAssetFilter && src && excludeAssetFilter(src))) {
            // 同理，将该标签创建到主应用下
            return rawDOMAppendOrInsertBefore.call(this, element, refChild) as T;
          }

          // 微应用容器 DOM
          const mountDOM = appWrapperGetter();
          // 用户提供的 fetch 方法
          const { fetch } = frameworkConfiguration;
          // 参考节点
          const referenceNode = mountDOM.contains(refChild) ? refChild : null;

          // 如果 src 存在，则说明是一个外联脚本
          if (src) {
            // 执行远程加载，将 proxy 设置为脚本的全局对象，来达到 JS 隔离的目的
            execScripts(null, [src], proxy, {
              fetch,
              strictGlobal: !singular,
              beforeExec: () => {
                Object.defineProperty(document, 'currentScript', {
                  get(): any {
                    return element;
                  },
                  configurable: true,
                });
              },
              success: () => {
                // we need to invoke the onload event manually to notify the event listener that the script was completed
                // here are the two typical ways of dynamic script loading
                // 1. element.onload callback way, which webpack and loadjs used, see https://github.com/muicss/loadjs/blob/master/src/loadjs.js#L138
                // 2. addEventListener way, which toast-loader used, see https://github.com/pyrsmk/toast/blob/master/src/Toast.ts#L64
                const loadEvent = new CustomEvent('load');
                if (isFunction(element.onload)) {
                  element.onload(patchCustomEvent(loadEvent, () => element));
                } else {
                  element.dispatchEvent(loadEvent);
                }

                element = null;
              },
              error: () => {
                const errorEvent = new CustomEvent('error');
                if (isFunction(element.onerror)) {
                  element.onerror(patchCustomEvent(errorEvent, () => element));
                } else {
                  element.dispatchEvent(errorEvent);
                }

                element = null;
              },
            });

            // 创建一个注释元素，表示该 script 标签被 qiankun 劫持处理了
            const dynamicScriptCommentElement = document.createComment(`dynamic script ${src} replaced by qiankun`);
            return rawDOMAppendOrInsertBefore.call(mountDOM, dynamicScriptCommentElement, referenceNode);
          }

          // 说明该 script 是一个内联脚本
          execScripts(null, [`<script>${text}</script>`], proxy, {
            strictGlobal: !singular,
            success: element.onload,
            error: element.onerror,
          });
          // 创建一个注释元素，表示该 script 标签被 qiankun 劫持处理了
          const dynamicInlineScriptCommentElement = document.createComment('dynamic inline script replaced by qiankun');
          return rawDOMAppendOrInsertBefore.call(mountDOM, dynamicInlineScriptCommentElement, referenceNode);
        }

        default:
          break;
      }
    }

    // 调用原始方法，插入元素
    return rawDOMAppendOrInsertBefore.call(this, element, refChild);
  };
}

```

##### getNewRemoveChild

```typescript
/**
 * 增强 removeChild，让其可以根据情况决定是从主应用中移除指定元素，还是从微应用中移除 script、style、link 元素
 * 如果是被劫持元素，则从微应用中移除，否则从主应用中移除
 * @param opts 
 */
function getNewRemoveChild(opts: {
  appWrapperGetter: CallableFunction;
  headOrBodyRemoveChild: typeof HTMLElement.prototype.removeChild;
}) {
  return function removeChild<T extends Node>(this: HTMLHeadElement | HTMLBodyElement, child: T) {
    // 原始的 removeChild
    const { headOrBodyRemoveChild } = opts;
    try {
      const { tagName } = child as any;
      // 当移除的元素是 script、link、style 之一时特殊处理
      if (isHijackingTag(tagName)) {
        // 微应用容器空间
        let { appWrapperGetter } = opts;

        // 新建时设置的，storedContainerInfo 包含了微应用的一些信息，不过 storedContainerInfo 应该是始终为 undefeind，因为设置位置的代码似乎永远不会被执行
        const storedContainerInfo = (child as any)[attachElementContainerSymbol];
        if (storedContainerInfo) {
          // eslint-disable-next-line prefer-destructuring
          // 微应用的包裹元素，也可以说微应用模版
          appWrapperGetter = storedContainerInfo.appWrapperGetter;
        }

        // 从微应用容器空间中移除该元素
        // container may had been removed while app unmounting if the removeChild action was async
        const container = appWrapperGetter();
        if (container.contains(child)) {
          return rawRemoveChild.call(container, child) as T;
        }
      }
    } catch (e) {
      console.warn(e);
    }

    // 从主应用中移除元素
    return headOrBodyRemoveChild.call(this, child) as T;
  };
}

```

#### patchAtMounting

在微应用的挂载阶段会被调用，主要负责给各个全局变量（方法）打 patch

```typescript
export function patchAtMounting(
  appName: string,
  elementGetter: () => HTMLElement | ShadowRoot,
  sandbox: SandBox,
  singular: boolean,
  scopedCSS: boolean,
  excludeAssetFilter?: Function,
): Freer[] {
  const basePatchers = [
    // 定时器 patch
    () => patchInterval(sandbox.proxy),
    // 事件监听 patch
    () => patchWindowListener(sandbox.proxy),
    // fix umi bug
    () => patchHistoryListener(),
    // 初始化阶段时的那个 patch
    () => patchDynamicAppend(appName, elementGetter, sandbox.proxy, true, singular, scopedCSS, excludeAssetFilter),
  ];

  const patchersInSandbox = {
    [SandBoxType.LegacyProxy]: [...basePatchers],
    [SandBoxType.Proxy]: [...basePatchers],
    [SandBoxType.Snapshot]: basePatchers,
  };

  return patchersInSandbox[sandbox.type]?.map(patch => patch());
}

```

##### patch => patchInterval

```typescript
/**
 * 定时器 patch，设置定时器时自动记录定时器 id，清除定时器时自动删除已清除的定时器 id，释放 patch 时自动清除所有未被清除的定时器，并恢复定时器方法
 * @param global = windowProxy
 */
export default function patch(global: Window) {
  let intervals: number[] = [];

  // 清除定时器，并从 intervals 中清除已经清除的 定时器 id
  global.clearInterval = (intervalId: number) => {
    intervals = intervals.filter(id => id !== intervalId);
    return rawWindowClearInterval(intervalId);
  };

  // 设置定时器，并记录定时器的 id
  global.setInterval = (handler: Function, timeout?: number, ...args: any[]) => {
    const intervalId = rawWindowInterval(handler, timeout, ...args);
    intervals = [...intervals, intervalId];
    return intervalId;
  };

  // 清除所有的定时器，并恢复定时器方法
  return function free() {
    intervals.forEach(id => global.clearInterval(id));
    global.setInterval = rawWindowInterval;
    global.clearInterval = rawWindowClearInterval;

    return noop;
  };
}

```

##### patch => patchWindowListener

```typescript
/**
 * 监听器 patch，添加事件监听时自动记录事件的回调函数，移除时自动删除事件的回调函数，释放 patch 时自动删除所有的事件监听，并恢复监听函数
 * @param global windowProxy
 */
export default function patch(global: WindowProxy) {
  // 记录各个事件的回调函数
  const listenerMap = new Map<string, EventListenerOrEventListenerObject[]>();

  // 设置监听器
  global.addEventListener = (
    type: string,
    listener: EventListenerOrEventListenerObject,
    options?: boolean | AddEventListenerOptions,
  ) => {
    // 从 listenerMap 中获取已经存在的该事件的回调函数
    const listeners = listenerMap.get(type) || [];
    // 保存该事件的所有回调函数
    listenerMap.set(type, [...listeners, listener]);
    // 设置监听
    return rawAddEventListener.call(window, type, listener, options);
  };

  // 移除监听器
  global.removeEventListener = (
    type: string,
    listener: EventListenerOrEventListenerObject,
    options?: boolean | AddEventListenerOptions,
  ) => {
    // 从 listenerMap 中移除该事件的指定回调函数
    const storedTypeListeners = listenerMap.get(type);
    if (storedTypeListeners && storedTypeListeners.length && storedTypeListeners.indexOf(listener) !== -1) {
      storedTypeListeners.splice(storedTypeListeners.indexOf(listener), 1);
    }
    // 移除事件监听
    return rawRemoveEventListener.call(window, type, listener, options);
  };

  // 释放 patch，移除所有的事件监听
  return function free() {
    // 移除所有的事件监听
    listenerMap.forEach((listeners, type) =>
      [...listeners].forEach(listener => global.removeEventListener(type, listener)),
    );
    // 恢复监听函数
    global.addEventListener = rawAddEventListener;
    global.removeEventListener = rawRemoveEventListener;

    return noop;
  };
}

```

## 链接

* [微前端专栏](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzA3NTk4NjQ1OQ==&action=getalbum&album_id=2251416802327232513&scene=173&from_msgid=2247484426&from_itemidx=1&count=3&nolastread=1#wechat_redirect)

  * 微前端框架 之 qiankun 从入门到源码分析

  * HTML Entry 源码分析

  * 微前端框架 之 single-spa 从入门到精通


* [github](https://github.com/liyongning)



感谢各位的：**点赞**、**收藏**和**评论**，我们下期见。

---

**当学习成为了习惯，知识也就变成了常识。**感谢各位的 **点赞**、**收藏**和**评论**。

新视频和文章会第一时间在微信公众号发送，欢迎关注：[李永宁lyn](https://gitee.com/liyongning/typora-image-bed/raw/master/202202171742614.jpg)

文章已收录到 [github](https://github.com/liyongning/blog)，欢迎 Watch 和 Star。
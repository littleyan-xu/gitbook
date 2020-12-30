上一篇在说到single-spa使用时，其实是有一些问题的，比如：引入子应用，我们是自己写的loadScript去主动抓取指定的js，非常不灵活，而single-spa官方推荐的Systenm.js更适合模块加载，而不是应用加载。其次，主应用和子应用之间还有子应用之间样式和js并没有隔离，很容易发生冲突。而qiankun主要是帮我们解决了这些问题，高效安全的在业务环境中使用微前端框架。


### 乾坤特性
- 📦 基于 single-spa 封装，提供了更加开箱即用的 API。
- 📱 技术栈无关，任意技术栈的应用均可 使用/接入，不论是 React/Vue/Angular/JQuery 还是其他等框架。
- 💪 HTML Entry 接入方式，让你接入微应用像使用 iframe 一样简单。
- 🛡​ 样式隔离，确保微应用之间样式互相不干扰。
- 🧳 JS 沙箱，确保微应用之间 全局变量/事件 不冲突。
- ⚡️ 资源预加载，在浏览器空闲时间预加载未打开的微应用资源，加速微应用打开速度。
- 🔌 umi 插件，提供了 @umijs/plugin-qiankun 供 umi 应用一键切换成微前端架构系统。

通过以上特性可以看出qiankun帮我们解决了以上出现的问题，对应资源的引入，qiankun使用的是[import-html-entry](https://www.npmjs.com/package/import-html-entry)第三方包，这里不多做阐述；既然是源码解析，我们就重点来关注一下js沙箱的实现。

### 使用DEMO
qiankun是基于single-spa 的，所以使用和single-spa非常像，而且更简单：


```
import { registerMicroApps, start } from 'qiankun'

const basePath = __DEV__ ? '' : '/fes_micro_main' // 基础路径

registerMicroApps([
  {
    name: 'zhikuIndex', // app name registered
    entry: '//fes-test.yy.com/zhiku/index.html',
    container: '#zhikuIndex',
    activeRule: `${basePath}/zhiku`,
  },
  {
    name: 'zhikuDocs', // app name registered
    entry: '//fes-test.yy.com/zhiku/docs.html',
    container: '#zhikuDocs',
    activeRule: `${basePath}/docs`,
  },
  {
    name: 'zhikuTools', // app name registered
    entry: '//fes-test.yy.com/zhiku/tools.html',
    container: '#zhikuTools',
    activeRule: `${basePath}/tools`,
  },
])
start()

```
DEMO链接：[http://fes-test.yy.com/fes_micro_main/](http://fes-test.yy.com/fes_micro_main/)

使用qiankun常见问题：[https://qiankun.umijs.org/zh/faq](https://qiankun.umijs.org/zh/faq)


### JS 沙箱
> 前言：关于沙箱的解释，网上有很多描述，大概的理解就是在一个安全的环境中运行代码。在微前端里面我们其实主要解决的是全局对象(window)的冲突问题，不管是主应用还是子应用，同时在操作同一个全局对象，这是非常糟糕的。

qiankun里的JS沙箱主要有3种：

- LegacySandbox
- ProxySandbox
- SnapshotSandbox

对应的源码：
```
let sandbox: SandBox;
if (window.Proxy) {
  sandbox = useLooseSandbox ? new LegacySandbox(appName) : new ProxySandbox(appName);
} else {
  sandbox = new SnapshotSandbox(appName);
}
```
如果支持Proxy，使用的是基于Proxy的沙箱，如果不支持，使用的是快照沙箱；基于Proxy的沙箱，如果是单例模式则使用LegacySandbox，反之多实例模式则使用ProxySandbox。

#### LegacySandbox沙箱

```
/**
 * 基于 Proxy 实现的沙箱
 * TODO: 为了兼容性 singular 模式下依旧使用该沙箱，等新沙箱稳定之后再切换
 */
export default class SingularProxySandbox implements SandBox {
  /** 沙箱期间新增的全局变量 */
  private addedPropsMapInSandbox = new Map<PropertyKey, any>();

  /** 沙箱期间更新的全局变量 */
  private modifiedPropsOriginalValueMapInSandbox = new Map<PropertyKey, any>();

  /** 持续记录更新的(新增和修改的)全局变量的 map，用于在任意时刻做 snapshot */
  private currentUpdatedPropsValueMap = new Map<PropertyKey, any>();

  constructor(name: string) {
    const self = this;
    const rawWindow = window;
    const fakeWindow = Object.create(null) as Window;

    const proxy = new Proxy(fakeWindow, {
      set(_: Window, p: PropertyKey, value: any): boolean {
        if (self.sandboxRunning) {
          if (!rawWindow.hasOwnProperty(p)) {
            addedPropsMapInSandbox.set(p, value);
          } else if (!modifiedPropsOriginalValueMapInSandbox.has(p)) {
            // 如果当前 window 对象存在该属性，且 record map 中未记录过，则记录该属性初始值
            const originalValue = (rawWindow as any)[p];
            modifiedPropsOriginalValueMapInSandbox.set(p, originalValue);
          }

          currentUpdatedPropsValueMap.set(p, value);
          // 必须重新设置 window 对象保证下次 get 时能拿到已更新的数据
          // eslint-disable-next-line no-param-reassign
          (rawWindow as any)[p] = value;

          self.latestSetProp = p;

          return true;
        }

        // 在 strict-mode 下，Proxy 的 handler.set 返回 false 会抛出 TypeError，在沙箱卸载的情况下应该忽略错误
        return true;
      },

      get(_: Window, p: PropertyKey): any {
        const value = (rawWindow as any)[p];
        return getTargetValue(rawWindow, value);
      },

    });

    this.proxy = proxy;
  }
}
```
以上代码是删减之后的核心代码（本来也不多），可以看到其实代理的就是window对象，所以本质上还是操作的window对象。

这里定义是3个Map，对应3个状态池：
- addedPropsMapInSandbox：子应用运行期间新增的全局变量
- modifiedPropsOriginalValueMapInSandbox ：子应用运行期间更新的全局变量
- currentUpdatedPropsValueMap ：新增和更新的全局变量


```
 if (!rawWindow.hasOwnProperty(p)) {
    addedPropsMapInSandbox.set(p, value);
} else if (!modifiedPropsOriginalValueMapInSandbox.has(p)) {
    // 如果当前 window 对象存在该属性，且 record map 中未记录过，则记录该属性初始值
    const originalValue = (rawWindow as any)[p];
    modifiedPropsOriginalValueMapInSandbox.set(p, originalValue);
}

currentUpdatedPropsValueMap.set(p, value);
```
在Proxy的setter里面将这几种全局变量的变化存储起来，用于子应用卸载时恢复主应用的状态，子应用加载时又重新还原子应用上一次的状态。

说起来挺绕口，在qiankun里面被称作：激活(active)和失活(inactive)，接下来就是这两个方法的实现：

```
active() {
    if (!this.sandboxRunning) {
      this.currentUpdatedPropsValueMap.forEach((v, p) => setWindowProp(p, v));
    }

    this.sandboxRunning = true;
}

inactive() {
    this.modifiedPropsOriginalValueMapInSandbox.forEach((v, p) => setWindowProp(p, v));
    this.addedPropsMapInSandbox.forEach((_, p) => setWindowProp(p, undefined, true));
    
    this.sandboxRunning = false;
}
```
激活时，将新增的和更新的全局变量附加到全局对象上，还原子应用的状态；

失活时，将更新的全局变量还原到初始值，将新增的全家变量变为undefined，从而实现恢复主应用的状态。


#### ProxySandbox沙箱

```
/**
 * 基于 Proxy 实现的沙箱
 */
export default class ProxySandbox implements SandBox {
  /** window 值变更记录 */
  private updatedValueSet = new Set<PropertyKey>();

  name: string;

  type: SandBoxType;

  proxy: WindowProxy;

  sandboxRunning = true;

  latestSetProp: PropertyKey | null = null;


  constructor(name: string) {
    this.name = name;
    this.type = SandBoxType.Proxy;
    const { updatedValueSet } = this;

    const self = this;
    const rawWindow = window;
    const { fakeWindow, propertiesWithGetter } = createFakeWindow(rawWindow);

    const descriptorTargetMap = new Map<PropertyKey, SymbolTarget>();
    const hasOwnProperty = (key: PropertyKey) => fakeWindow.hasOwnProperty(key) || rawWindow.hasOwnProperty(key);

    const proxy = new Proxy(fakeWindow, {
        ...
    });
  }
}
```
可以看到，ProxySandbox和上面LegacySandbox最明显的不同就是ProxySandbox代理的是**fakeWindow**，而不直接是window，fakeWindow是通过createFakeWindow创建得来的，我们看一下代码：

```
function createFakeWindow(global: Window) {
  const propertiesWithGetter = new Map<PropertyKey, boolean>();
  const fakeWindow = {} as FakeWindow;
   
  Object.getOwnPropertyNames(global)
    .filter((p) => {
      const descriptor = Object.getOwnPropertyDescriptor(global, p);
      return !descriptor?.configurable;
    })
    .forEach((p) => {
      const descriptor = Object.getOwnPropertyDescriptor(global, p);
      if (descriptor) {
        const hasGetter = Object.prototype.hasOwnProperty.call(descriptor, 'get');

        if (hasGetter) propertiesWithGetter.set(p, true);

        Object.defineProperty(fakeWindow, p, Object.freeze(descriptor));
      }
    }); 
}
```
将window不可配置的属性拷贝到fakeWindow上，不可配置的属性有哪些呢，在chrome打印了一下，有42个之多，那么多常用的就是"window"、"document"、"location"这些。

上面LegacySandbox代理中将全局变量存储到map状态池中，而ProxySandbox因为代理的是一个全新的fakeWindow，所以就免去了这些操作：


```
const proxy = new Proxy(fakeWindow, {
      set(target: FakeWindow, p: PropertyKey, value: any): boolean {
        if (self.sandboxRunning) {
          // 如果设置的属性是本来就存在window上的，那么需要保留它的descriptor
          if (!target.hasOwnProperty(p) && rawWindow.hasOwnProperty(p)) {
            const descriptor = Object.getOwnPropertyDescriptor(rawWindow, p);
            const { writable, configurable, enumerable } = descriptor!;
            if (writable) {
              Object.defineProperty(target, p, {
                configurable,
                enumerable,
                writable,
                value,
              });
            }
          } else {
            target[p] = value; // 赋值操作
          }

          updatedValueSet.add(p);

          self.latestSetProp = p;

          return true;
        }

        return true;
      },
      
      get(target: FakeWindow, p: PropertyKey): any {
      
        // 这里省略了一堆判断...

        // 取值并返回值
        const value = propertiesWithGetter.has(p)
          ? (rawWindow as any)[p]
          : p in target
          ? (target as any)[p]
          : (rawWindow as any)[p];
        return getTargetValue(rawWindow, value);
      },
    }
```
对应的激活(active)和失活(inactive)操作也不需要变更状态池的更新，而只是控制激活沙箱的数量：


```
let activeSandboxCount = 0; // 激活的沙箱数量

export default class ProxySandbox implements SandBox {

  active() {
    if (!this.sandboxRunning) activeSandboxCount++;
    this.sandboxRunning = true;
  }

  inactive() {

    if (--activeSandboxCount === 0) {
      variableWhiteList.forEach((p) => {
        if (this.proxy.hasOwnProperty(p)) {
          // @ts-ignore
          delete window[p];
        }
      });
    }

    this.sandboxRunning = false;
  }
}
```

相比较而言，ProxySandbox代理全新的fackWindow，不直接操作window，更安全。正如LegacySandbox的命名，代表的就是遗留的沙箱，估计不久会就会全部被ProxySandbox所取代。

#### SnapshotSandbox
快照沙箱是在不支持Proxy情况下的降级方案。所谓的快照，其实就是记录，在激活和失活的时候通过diff 来实现状态的恢复或还原，talk is cheap, show me the code:

```
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

    for (const prop in window) {
        if (window.hasOwnProperty(prop)) {
          this.windowSnapshot[prop] = window[prop];
        }
      }

    // 恢复之前的变更
    Object.keys(this.modifyPropsMap).forEach((p: any) => {
      window[p] = this.modifyPropsMap[p];
    });

    this.sandboxRunning = true;
  }

  inactive() {
    this.modifyPropsMap = {};

    for (const prop in window) {
      if (window[prop] !== this.windowSnapshot[prop]) {
        // 记录变更，恢复环境
        this.modifyPropsMap[prop] = window[prop];
        window[prop] = this.windowSnapshot[prop];
      }
    }


    this.sandboxRunning = false;
  }
}

```
激活时（active）：

1、 遍历window的属性，将自有属性拷贝到windowSnapshot，以便失活时回到初始化状态，

2、将之前记录的修改（difyPropsMap）恢复到window上。


失活时（inactive），再次遍历window的属性，如果和windowSnapshot不相同：

1、将修改记录到modifyPropsMap以便激活时恢复状态

2、将window设置回初始状态。

到这里，3种类型的沙箱就介绍完了，源码里面这些沙箱的实例最终会赋给global,并传递给子应用加载逻辑里面去。

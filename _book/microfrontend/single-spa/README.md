本文重点是对single-spa核心源码进行解析，所以对于其介绍和应用会简单带过；因能力有限，如果有不对的地方，还请大佬不吝赐教。以下介绍文本来自于：[https://zh-hans.single-spa.js.org/](https://zh-hans.single-spa.js.org/)

## 微前端概念
微前端是指存在于浏览器中的微服务。

微前端作为用户界面的一部分，通常由许多组件组成，并使用类似于React、Vue和Angular等框架来渲染组件。每个微前端可以由不同的团队进行管理，并可以自主选择框架。虽然在迁移或测试时可以添加额外的框架，出于实用性考虑，建议只使用一种框架。

每个微前端都拥有独立的git仓库、package.json和构建工具配置。因此，每个微前端都拥有独立的构建进程和独立的部署/CI。

## single-spa
Single-spa 是一个将多个单页面应用聚合为一个整体应用的 JavaScript 微前端框架。

## single-spa之应用
#### 核心API：
- registerApplication(name, app, activeWhen, customProps) ：注册应用
- start ：启动

##### registerApplication(name, app, activeWhen, customProps)
name: 应用的名称，在源码中会被用做key用来注册和引用应用

app：异步加载函数，该函数必须返回一个加载对象，对象上必须有bootstrap、mount、unmont生命周期钩子函数

activeWhen：根据路由匹配来激活应用

customProps：自定义数据，主要是用于父应用传给生命周期钩子函数

##### start()
启动应用，开始调用bootstrap、mount、unmont生命周期函数

Question：为什么要分开2个方法来启动一个应用？

这主要是处于优化方面的考虑，我们可以先去加载应用，但是不初始化和挂载应用，等到其他时机比如点击某个按钮，或者ajax数据返回后再去初始化、挂载应用。

#### Demo：
由于业务中用vue框架项目较多，所以这次demo就以vue为主，工作流使用vue-cli。

前文提到加载子应用的加载函数必须返回一个加载对象，对象需提供一系列的生命周期函数，因为使用Vue所以可以直接使用single-spa生态系统提供的single-spa-vue来实现

当然啦，single-spa生态系统还提供其他框架的辅助库，如single-spa-react、single-spa-angular……任君选择！


子应用入口文件核心代码：

```
import Vue from 'vue'
import App from './App.vue'
import router from './router'
import singleSpaVue from 'single-spa-vue'


const appOptions = {
  el: '#vueApp', // 挂载到父应用ID为vueApp的Dom元素中
  router,
  render: h => h(App)
}

const vueLifecycles = singleSpaVue({
  Vue,
  appOptions
})

// 判断是否在微前端环境中
if (window.singleSpaNavigate) {
// 动态更改webpack的publichPath
  __webpack_public_path__= 'http://localhost:3000/' // eslint-disable-line
} else {
  new Vue({
    router,
    render: h => h(App)
  }).$mount('#app')
}

export const bootstrap = vueLifecycles.bootstrap
export const mount = vueLifecycles.mount
export const unmount = vueLifecycles.unmount

// bootstrap, mount and unmount

```

子应用修改配置文件vue.config.js，将模块打包方式改为umd，用于主应用调用：

```
module.exports = {
  configureWebpack: {
    output: {
      library: 'singleVue',
      libraryTarget: 'umd'
    },
    devServer: {
      port: 3000
    }
  }
}
```
主应用入口文件：

```
import Vue from 'vue'
import App from './App.vue'
import router from './router'
import store from './store'
import { registerApplication, start } from 'single-spa'

// 这里我们简单实现JS模块的加载，官方推荐使用syestemjs
function loadScript (src) {
  return new Promise((resolve, reject) => {
    var script = document.createElement('script')
    script.src = src
    script.onload = resolve
    script.onerror = reject
    document.head.appendChild(script)
  })
}

// 注册应用
registerApplication('singleVue', async () => {
    // 加载上面的子应用
      await loadScript('http://localhost:3000/js/chunk-vendors.js')
      await loadScript('http://localhost:3000/js/app.js')
      return window.singleVue // bootscrap mount unmount
    },
    location => {
      return location.pathname.startsWith('/vue')
    },{
      from: 'single-spa'
    }
)

start() // 启动应用

new Vue({
  router,
  store,
  render: h => h(App)
}).$mount('#app')

```
Demo运行效果：
![Demo运行效果](https://rhinosystem.bs2dl.yy.com/cont1607938989572913file)


## single-spa核心源码解析
single-spa暴露出的核心api主要是registerApplication和start，所以我们的源码解析就从这两个api入手，对核心的功能一探究竟。

#### registerApplication()

registerApplication方法源码里面被封装在了applications/app.js模块里面，为了代码更直观，会省略和简化部分代码，核心代码如下：

```
const apps = []; // 用来存储注册的应用

function registerApplication(
  appName,
  loadApp,
  activeWhen,
  customProps
) {
  // 1、首页会对传入的参数进行验证和处理，这里省略……

  // 2、push到数组中存储，源码中会跟默认属性做合并，这里省略
  apps.push({
    appName,
    loadApp,
    activeWhen,
    customProps,
    status: NOT_LOADED,
  });

  // 核心方法，根据路由处理应用
  reroute();
}
```
registerApplication看起来很简单，就做了1件事：根据参数定义app并放入数组存储起来，然鹅代码里面有2个重点：

##### 1、status: NOT_LOADED

这是app的默认状态，代码往下走就会发现其实整个single-spa就是一个状态机，所有的处理都依赖于状态值或更改状态值，状态的定义封装在了applications/app.helpers.js中，一共定义了12种状态

```
// App 的12种状态
export const NOT_LOADED = "NOT_LOADED";
export const LOADING_SOURCE_CODE = "LOADING_SOURCE_CODE";
export const NOT_BOOTSTRAPPED = "NOT_BOOTSTRAPPED";
export const BOOTSTRAPPING = "BOOTSTRAPPING";
export const NOT_MOUNTED = "NOT_MOUNTED";
export const MOUNTING = "MOUNTING";
export const MOUNTED = "MOUNTED";
export const UPDATING = "UPDATING";
export const UNMOUNTING = "UNMOUNTING";
export const UNLOADING = "UNLOADING";
export const LOAD_ERROR = "LOAD_ERROR";
export const SKIP_BECAUSE_BROKEN = "SKIP_BECAUSE_BROKEN";

export function isActive(app) {
  return app.status === MOUNTED;
}

export function shouldBeActive(app) {
  return app.activeWhen(window.location);
}
```
一图胜千言，状态图奉上：
![状态图](https://rhinosystem.bs2dl.yy.com/cont1607831470611717file)

##### 2、reroute()方法
reroute()是核心中的核心，该方法就是根据路由来处理应用，几乎所有的操作最终都是调用该方法，这里我们先按下不表，继续看另一个核心api：start()


#### start()
start封装在start.js里面，和上面一样，为了代码更直观，会省略和简化部分代码

```
let started = false;

export function start(opts) {
  started = true;
  reroute();
}

export function isStarted() {
  return started;
}
```
看到木有，start()方法也就是调用了reroute()方法，只是为了和registerApplication作区分，这里定义了一个变量started用于在reroute()作判断，接下来就进入关键的reroute()方法

#### reroute()
reroute()在源码中被封装在navigation/reroute.js中，navigation目录下还有另外一个js文件：navigation-events.js，实现重要的路由劫持，这个也先按下不表，后面再讲。

```
export function reroute() {
  const { appsToLoad, appsToMount, appsToUnMount } = getAppChanges()
  if (started) {
    return performAppChanges() // 根据路径来挂载应用
  } else {
    return loadApps() // 预加载应用
  }
}
```
##### getAppChanges（）
reroute里面首先就是要拿到根据状态所分类的各种app数组：

appsToLoad——需要去加载的app列表

appsToMount——需要去挂载的app列表

appsToUnMount——需求去解除挂载的app列表

……

getAppChanges()和registerApplication()一样，定义在app.js里面，主要功能就是对所有的app跟进状态进行区分，并返回区分之后的各个数组，这里对上上面的状态图会更一目了然

```

export function getAppChanges() {
  const appsToUnload = [],
    appsToUnmount = [],
    appsToLoad = [],
    appsToMount = [];

  const currentTime = new Date().getTime();

  apps.forEach((app) => {
    // 路由匹配上了并且程序没有报错就是需要被激活的应用
    const appShouldBeActive =
      app.status !== SKIP_BECAUSE_BROKEN && shouldBeActive(app);

    switch (app.status) {
      case LOAD_ERROR:
        // 之前加载出错，现在被激活且时间间隔超过200毫秒，那么则加入需要去加载的列表
        if (appShouldBeActive && currentTime - app.loadErrorTime >= 200) {
          appsToLoad.push(app);
        }
        break;
      case NOT_LOADED:
      case LOADING_SOURCE_CODE:
        // 没有加载或加载中的，现在被激活的也要加入需要去加载的列表
        if (appShouldBeActive) {
          appsToLoad.push(app);
        }
        break;
      case NOT_BOOTSTRAPPED:
      case NOT_MOUNTED:
       // 没有初始化或没有挂载的，现在没有激活且存在需要卸载的列表中，则需要卸载
        if (!appShouldBeActive && getAppUnloadInfo(toName(app))) {
          appsToUnload.push(app);
        } else if (appShouldBeActive) { // 反之则要加入需要去挂载的列表
          appsToMount.push(app);
        }
        break;
      case MOUNTED:
        // 已经挂载了，但是现在未处于激活状态了，则需要解除挂载
        if (!appShouldBeActive) {
          appsToUnmount.push(app);
        }
        break;
    }
  });

  return { appsToUnload, appsToUnmount, appsToLoad, appsToMount };
}
```
接下来如果是registerApplication()调用的，就会执行loadApps()方法；如果是start()调用则会执行performAppChanges()，我们先来看看loadApps()。
##### loadApps()
loadApps()的主要功能就是执行registerApplication里面传入的第2个参数loadApp()方法来对app进行预加载，并将得到的加载对象的生命周期钩子函数添加到app对象上，以便在start()方法里面调用。

loadApps()里面调用了toLoadPromise()方法来处理，这个方法在performAppChanges()里面也会调用，我们在稍后再讲；callAllEventListeners是触发所有被劫持的路由事件，这个我们在后面的路由劫持会说到。


```
async function loadApps() {
  // 源码中使用的是Promise.resolve().then的写法，这里使用async/await的写法更直观
  let apps = await Promise.all(appsToLoad.map(toLoadPromise))
  
  // 等到所有需要被加载的app加载完成后再触发所有被劫持的路由事件
  callAllEventListeners()
}

```
##### performAppChanges()
performAppChanges主要是是执行各个app的生命周期钩子函数，简化后代码如下：

```
async function performAppChanges() {
    // step1 卸载不需要的应用
    appsToUnMount.map(toUnMountPromise)
    
    // step2 加载需要的应用
    appsToLoad.map(async (app) => {
      app = await toLoadPromise(app)
      app = await toBootstrcapPromise(app)
      return toMountPromise(app)
    })
    
    appsToMount.map(async (app) =>{
      app = await toBootstrcapPromise(app)
      return toMountPromise(app)
    })
}
```
toLoadPromise()、toBootstrcapPromise()、toMountPromise()、toMountPromise()所对应都是处理生命周期钩子函数，这些方法都被封装在lifecycles目录下，接下来来一一观摩一下：


###### 加载应用：load.js

```
export async function toLoadPromise(app) {
  // 缓存机制，防止注册和start重复调用
  if (app.loadPromise) {
    return app.loadPromise
  }
  return app.loadPromise = Promise.resolve().then(
    async () => {
      app.status = LOADING_SOURCE_CODE // 状态改为加载资源
      
      // 执行加载函数，得到生命周期钩子函数
      let { bootstrap, mount, unmount } = await app.loadApp(app.customProps)
      
      app.status = NOT_BOOTSTRAPPED // 加载完成状态改为未初始化
      
      // 将生命周期钩子函数挂载到app上
      app.bootstrap = flattenFnArray(bootstrap)
      app.mount = flattenFnArray(mount)
      app.unmount = flattenFnArray(unmount)
      return app
    }
  )
}

// 将多个异步函数的数组拍平为一个异步函数
function flattenFnArray(fns) {
  fns = Array.isArray(fns) ? fns : [fns]
  return (props) => fns.reduce((p, fn) => p.then(() => fn(props)), Promise.resolve())
}
```
这里有2个重点，第1个是这里加入了缓存机制，那这是为什么呢？

上面我们提到loadApps()和performAppChanges()都会执行toLoadPromise()，首先一个疑问就为啥要调用两次？这是因为registerApplication和start调用是同步的，可以去看看上面的Demo，我们并没有等到registerApplication处理完后再去调用start，而registerApplication里面处理是异步的，所以我们在start的时候还要去load一次才安全。

所以这就带来另一个问题，toLoadPromise会被执行2次，所以我们需要将执行的过程封装为一个promise缓存起来，如果存在则直接返回。

第2个重点是flattenFnArray()用于将多个异步函数的数组拍平为一个异步函数，因为我们生命周期钩子函数是允许出现多个的，如果有多个则需要拍平为一个挂载到app上

###### 初始化应用：bootstrcap.js

```
export async function toBootstrcapPromise(app){
  if(app.status !== NOT_BOOTSTRAPPED){
    return app
  }
  app.status = BOOTSTRAPPING
  await app.bootstrap(app.customProps)
  app.status = NOT_MOUNTED
  return app
}
```
###### 挂载应用：mount.js

```
export async function toMountPromise(app){
  if(app.status !== NOT_MOUNTED){
    return app
  }
  app.status = MOUNTING
  await app.mount(app.customProps)
  app.status = MOUNTED
  return app
}
```

###### 解除挂载：unmount.js

```
export async function toUnMountPromise(app) {
  // 如果没有已挂载，则不需要卸载
  if (app.status !== MOUNTED) {
    return app
  }
  app.status = UNMOUNTING
  await app.unmount(app.customProps)
  app.status = NOT_MOUNTED
  return app
}
```
当然还有其他更多的生命周期处理js，这里不一一列举，从这些处理可以看出来，其实都是在不断的改状态，并执行生命周期钩子函数。

到这里其实核心的逻辑已经跑通了，源码中还有更新处理以及超时处理等等，这里篇幅有限就不细说了。

到这里还有一个逻辑我们还没处理——路由劫持，上面说到reroute就是根据路由的变化来处理应用。 在路由切换后，我们需要对当前失活的应用进行解除挂载并挂载激活的应用，接下来则进入路由劫持篇。

##### 路由劫持

说到路由，我们必定想到和路由相关的api:
- hashchange事件
- popstate事件
- history.pushState()
- history.replaceState()

先来看看**路由事件**的劫持，前面说到，路由的处理封装在navigator-events.js中：

```
// 捕获子应用的hashchange事件和popt
const capturedEventListeners = {
  hashchange: [],
  popstate: [],
};

const routingEventsListeningTo = ["hashchange", "popstate"];

function urlReroute() {
  reroute([], arguments);
}

window.addEventListener("hashchange", urlReroute);
window.addEventListener("popstate", urlReroute);

const originalAddEventListener = window.addEventListener;
const originalRemoveEventListener = window.removeEventListener;

window.addEventListener = function (eventName, fn) {
    if (typeof fn === "function") {
      if (
        routingEventsListeningTo.indexOf(eventName) >= 0 &&
        !find(capturedEventListeners[eventName], (listener) => listener === fn)
      ) {
        capturedEventListeners[eventName].push(fn);
        return;
      }
    }
    
    return originalAddEventListener.apply(this, arguments);
};

window.removeEventListener = function (eventName, listenerFn) {
    if (typeof listenerFn === "function") {
      if (routingEventsListeningTo.indexOf(eventName) >= 0) {
        capturedEventListeners[eventName] = capturedEventListeners[
          eventName
        ].filter((fn) => fn !== listenerFn);
        return;
      }
    }
    
    return originalRemoveEventListener.apply(this, arguments);
};

```
这段代码主要是做了3件事情：
- 监听hashchange和popstate事件，触发时调用上面的reroute方法；
- 暂存window.addEventListener和window.removeEventListener方法
- 改写这两个监听事件的方法，window.addEventListener的处理是如果事件名（eventName）是hashchange或popstate，并且所对应的处理函数没有存储起来，则加入存储队列，然后直接return;window.removeEventListener 则是反向操作；如果事件名不是hashchange或popstate，那么就会走上面暂存的原有的监听方法。


Q：为什么要劫持这2个事件并将处理函数存储起来呢？

A：一句话，主要是让主应用的监听函数先执行，然后再执行子应用的监听函数

Q：存储起来什么时候调用呢?

A：子应用加载完成或者子应用卸载时。在上面讲到的loadApps()时，里面就调用了callAllEventListeners()方法，这个方法就用来调用所存储的监听函数（listener）


```
// refoute.js
function reroute(pendingPromises = [], eventArguments){
    ……
    function callAllEventListeners() {
        callCapturedEventListeners(eventArguments);
    }
    ……
}
  
// navigation-events.js
export function callCapturedEventListeners(eventArguments) {
  if (eventArguments) {
    const eventType = eventArguments[0].type;
    if (routingEventsListeningTo.indexOf(eventType) >= 0) {
      capturedEventListeners[eventType].forEach((listener) => {
        try {
          listener.apply(this, eventArguments);
        } catch (e) {
          setTimeout(() => {
            throw e;
          });
        }
      });
    }
  }
}
  
```

说完路由事件，再来看看pushState和replaceState这2个history API，毕竟上面的事件也只能处理hash变化和前进后退的处理，这两个api也是重中之重。


```
function patchUpdateState(orginFn, type){
  return function(){
    const oldLocation = window.location.href
    orginFn.apply(this, arguments) // 执行 pushState / replaceState操作，会改变location
    
    const newLocation = window.location.href
    
    // location发送变化说明需要重现加载
    if(oldLocation !== newLocation){
      urlRouter(new PopStateEvent(type)) // 
    }
  }
}

window.history.pushState = patchedUpdateState(window.history.pushState,"pushState");

window.history.replaceState = patchedUpdateState(window.history.replaceState,"replaceState");

```
和路由事件类似，这里改写了pushState 和replaceState 方法，加入了window.location的判断并执行urlRouter()，这里new PopStateEvent主要是和上面hashchange/popstate的处理保持一致。

到这里核心逻辑就差不多理清了，最后来个流程图（能力有限，只画了部分）

![single-spa流程图](https://rhinosystem.bs2dl.yy.com/cont1607934366605528file)



## 总结

这个总结主要是我从single-spa学到知识点或者技能

- 代码封装：代码封装其实考验的就是架构能力，single-spa根据功能拆分为了applications、lifecycles、navigations这几个大的模块，然后里面又进行细分，代码的可能性高且代码解耦逻辑清晰。另外源码中还使用了各种自定义事件来代替函数调用机制，从而增强代码的语义化和将逻辑复杂的代码的解耦
- 异步处理：single-spa里面暂时没有用到async await的异步写法，但也没影响到代码的优雅性，特别是将多个异步处理函数合并为一个异步函数的处理，非常涨姿势
- 路由劫持：在spa单页应用当道的今天，如果对路由各种api还不熟悉的话，那就真该吊打了，single-spa对路由的处理也让我们对前端路由更深入了解。
- ……


> 题外话

事实上，single-sap所对应的微应用是有不同分类的，主要有这3种类型：
1. Application 微应用
2. Parcel 微组件
3. Utility 微逻辑

而这篇文章主要讲的是Application微应用，只有它是跟路由相关的。而我们接下来继续研究的qiankun也主要是Application微应用的使用。
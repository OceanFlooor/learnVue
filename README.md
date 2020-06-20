# vue 源码学习笔记

### 说明：此项目纯属个人瞎写，所以如果觉得排版难看，内容肤浅，难免有错误，请不要介意:grinning:

## vue 初次创建渲染项目历程大概如下：

![neW-vue](./img/new-vue.png)

<h5 align="center">图片来自于黄轶老师</h5>

---

## init

首先，假设我们 vue 项目的入口文件`main.js`代码如下：

```javascript
import Vue from "vue";

Vue.config.productionTip = false;

new Vue({
  render(h) {
    return h("div", {
      attrs: {
        id: "hello",
      },
    });
  },
}).$mount("#app");
```

在`new Vue()`过程中，进入文件`src\core\instance\index.js`

```javascript
// 这就是vue，是个利用构造函数创建出来的对象
function Vue(options) {
  if (process.env.NODE_ENV !== "production" && !(this instanceof Vue)) {
    warn("Vue is a constructor and should be called with the `new` keyword");
  }
  // 初始化操作
  this._init(options);
}

// 各种初始化
initMixin(Vue);
stateMixin(Vue);
eventsMixin(Vue);
lifecycleMixin(Vue);
renderMixin(Vue);

export default Vue;
```

在`function Vue`创建 Vue 实例之前，该文件对外暴露了`Vue`，通过外层，对 vue 层层处理后，最终进入此文件，所以在 Vue 实例创建之前，`function Vue`已经做过很多加工，所以这个函数原型上有很多属性和方法，为的是创建 Vue 过程中执行`this._init(options)`服务。

`this._init`方法是通过`initMixin(Vue)`挂载，来自文件`./init`：

```javascript
export function initMixin (Vue: Class<Component>) {
  Vue.prototype._init = function (options?: Object) {
    const vm: Component = this

    ...

    initLifecycle(vm)
    initEvents(vm)
    initRender(vm)
    callHook(vm, 'beforeCreate')
    initInjections(vm) // resolve injections before data/props
    initState(vm)
    initProvide(vm) // resolve provide after data/props
    callHook(vm, 'created')

    /* istanbul ignore if */
    if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
      vm._name = formatComponentName(vm, false)
      mark(endTag)
      measure(`vue ${vm._name} init`, startTag, endTag)
    }

    // 如果有el，则把项目元素挂载到el上
    if (vm.$options.el) {
      vm.$mount(vm.$options.el)
    }
  }
}
```

该方法在`initMixin(Vue)`中调用，对传入的 vue（之后称作 vm）挂载了`_init`方法，改方法接收`options`对象，即项目的`main.js`中`new Vue({...})`传入的对象,可能包含了 el 及 render 等各种东西，对 vm 进一步初始化：生命周期函数、事件、渲染涉及的函数钩子（比如`createElement`）以及数据 state 等，之后`vm.$mount(vm.$options.el)`挂载，这一步发生在生命周期`beforeMount` && `mounted`之间。

---

## \$mount 和 compile

即是上面说的`vm.$mount(vm.$options.el)`

Vue 是通过上面的方法去挂载 `vm` 的，`$mount` 方法在多个文件中都有定义，如 `src/platform/web/entry-runtime-with-compiler.js`、`src/platform/web/runtime/index.js`、`src/platform/weex/runtime/index.js`，因为`$mount`涉及平台及版本。这里只分析`compiler`版本，因为这个版本不涉及`webpack`的`vue-loader`插件（作用是把组件`<template>`的`html`语言编译成`render`函数）,也就是说`compiler`版本就是 vue 自己把`vue-loader`的工作干了。

先来看下`src/platform/web/entry-runtime-with-compiler.js`:

```javascript
const mount = Vue.prototype.$mount;
Vue.prototype.$mount = function (el?: string | Element, hydrating?: boolean): Component {
  el = el && query(el);

  /* istanbul ignore if */
  if (el === document.body || el === document.documentElement) {
    process.env.NODE_ENV !== "production" && warn(`Do not mount Vue to <html> or <body> - mount to normal elements instead.`);
    return this;
  }

  const options = this.$options;
  // resolve template/el and convert to render function
  if (!options.render) {
    let template = options.template;
    if (template) {
      if (typeof template === "string") {
        if (template.charAt(0) === "#") {
          template = idToTemplate(template);
          /* istanbul ignore if */
          if (process.env.NODE_ENV !== "production" && !template) {
            warn(`Template element not found or is empty: ${options.template}`, this);
          }
        }
      } else if (template.nodeType) {
        template = template.innerHTML;
      } else {
        if (process.env.NODE_ENV !== "production") {
          warn("invalid template option:" + template, this);
        }
        return this;
      }
    } else if (el) {
      template = getOuterHTML(el);
    }
    if (template) {
      /* istanbul ignore if */
      if (process.env.NODE_ENV !== "production" && config.performance && mark) {
        mark("compile");
      }

      // 编译出render func
      const { render, staticRenderFns } = compileToFunctions(
        template,
        {
          shouldDecodeNewlines,
          shouldDecodeNewlinesForHref,
          delimiters: options.delimiters,
          comments: options.comments,
        },
        this
      );
      options.render = render;
      options.staticRenderFns = staticRenderFns;

      /* istanbul ignore if */
      if (process.env.NODE_ENV !== "production" && config.performance && mark) {
        mark("compile end");
        measure(`vue ${this._name} compile`, "compile", "compile end");
      }
    }
  }
  return mount.call(this, el, hydrating);
};
```

从上面可以看出，思路就是`const mount = Vue.prototype.$mount`先缓存原型上的`$mount`，然后重定义`Vue.prototype.$mount`，最后再从重定义的方法中调用原本的方法`return mount.call(this, el, hydrating)`，也就是说`compiler`版本在 mount 之前做了中间处理：

首先，它对 el 做了限制，Vue 不能挂载在 body、html 这样的根节点上。接下来，vue 对`options`做了判断：如果不存在`options.render`，即用户自己并没有提供渲染函数`render`，则会把 el 或者 template 字符串转换成 `render` 方法。在 Vue 2.0 版本中，所有 Vue 的组件的渲染最终都需要 `render` 方法，无论我们是用单文件 `.vue` 方式开发组件，还是写了 el 或者 template 属性，最终都会转换成 `render` 方法，这个过程是调用 `compileToFunctions` 方法实现的。最后，调用原先原型上的 `$mount` 方法挂载。

而原先原型上的 `$mount` 方法定义在`src/platform/web/runtime/index.js`，为什么要这么设计呢？因为`runtime only`版本的 vue 不具备把`template`编译出`render`方法的功能，需要借助`webpack`的`vue-loader`，这么做也是优化了打包后的代码，而这时候它 Vue 是可以直接调用的:

```javascript
// runtime only 版本
// public mount method
Vue.prototype.$mount = function (el?: string | Element, hydrating?: boolean): Component {
  el = el && inBrowser ? query(el) : undefined;
  return mountComponent(this, el, hydrating);
};
```

`$mount` 方法支持传入 2 个参数，第一个是 `el`，它表示挂载的元素，可以是字符串，也可以是 DOM 对象，如果是字符串在浏览器环境下会调用 `query` 方法转换成 DOM 对象的。第二个参数是和服务端渲染相关，在浏览器环境下不需要传第二个参数。该方法在`return mountComponent(this, el, hydrating)`处调用了`mountComponent`，这个方法定义在 `src/core/instance/lifecycle.js` 文件中：

```javascript
export function mountComponent(vm: Component, el: ?Element, hydrating?: boolean): Component {
  vm.$el = el;
  if (!vm.$options.render) {
    vm.$options.render = createEmptyVNode;
    if (process.env.NODE_ENV !== "production") {
      /* istanbul ignore if */
      if ((vm.$options.template && vm.$options.template.charAt(0) !== "#") || vm.$options.el || el) {
        warn("You are using the runtime-only build of Vue where the template " + "compiler is not available. Either pre-compile the templates into " + "render functions, or use the compiler-included build.", vm);
      } else {
        warn("Failed to mount component: template or render function not defined.", vm);
      }
    }
  }
  callHook(vm, "beforeMount");

  let updateComponent;
  /* istanbul ignore if */
  if (process.env.NODE_ENV !== "production" && config.performance && mark) {
    updateComponent = () => {
      const name = vm._name;
      const id = vm._uid;
      const startTag = `vue-perf-start:${id}`;
      const endTag = `vue-perf-end:${id}`;

      mark(startTag);
      const vnode = vm._render();
      mark(endTag);
      measure(`vue ${name} render`, startTag, endTag);

      mark(startTag);
      vm._update(vnode, hydrating);
      mark(endTag);
      measure(`vue ${name} patch`, startTag, endTag);
    };
  } else {
    updateComponent = () => {
      vm._update(vm._render(), hydrating);
    };
  }

  // we set this to vm._watcher inside the watcher's constructor
  // since the watcher's initial patch may call $forceUpdate (e.g. inside child
  // component's mounted hook), which relies on vm._watcher being already defined
  new Watcher(
    vm,
    updateComponent,
    noop,
    {
      before() {
        if (vm._isMounted && !vm._isDestroyed) {
          callHook(vm, "beforeUpdate");
        }
      },
    },
    true /* isRenderWatcher */
  );
  hydrating = false;

  // manually mounted instance, call mounted on self
  // mounted is called for render-created child components in its inserted hook
  if (vm.$vnode == null) {
    vm._isMounted = true;
    callHook(vm, "mounted");
  }
  return vm;
}
```

`mountComponent`接收三个参数：`vm`，`el`，`hydrating`，`vm`指 vue 组件，`el`是挂载对象，同样`hydrating`是 ssr 相关参数。该方法的核心就是实例化 Watcher 对象：`new Watcher(...)`，`updateComponent`作为回调函数传入其中，而后创建的过程中通过`value = this.getter.call(vm, vm)`执行`updateComponent`，以下代码截取了涉及的关键部分（具体文件在`src/core/observer/watcher.js`）：

```javascript
export default class Watcher {
  ...

  constructor (
    vm: Component,
    // updateComponent传入其中，expOrFn接收
    expOrFn: string | Function,
    cb: Function,
    options?: ?Object,
    isRenderWatcher?: boolean
  ) {

    ...

    // parse expression for getter
    if (typeof expOrFn === 'function') {
      // this.getter实际上就等于updateComponent
      this.getter = expOrFn
    }

    ...
    // this.lazy为false,执行this.get()
    this.value = this.lazy
      ? undefined
      : this.get()
  }

  get () {
    ...

    try {
      // 执行updateComponent
      value = this.getter.call(vm, vm)
    } catch (e) {
      ...
    }
    return value
  }
  ...
}
```

而`updateComponent`在此处就是如下方法：

```javascript
let updateComponent;

updateComponent = () => {
  vm._update(vm._render(), hydrating);
};
```

调用它，内部执行`vm._update(vm._render(), hydrating)`，在此方法中调用`vm._render`方法先生成虚拟 Node，最终调用 `vm._update` 更新 DOM。函数最后判断为根节点的时候设置 `vm._isMounted` 为 `true`， 表示这个实例已经挂载了，同时执行 `mounted` 钩子函数。

然而挂载的过程中涉及两个重点：`vm._update`和`vm._render()`

---

## render

`_render()`是实例原型上挂载的一个方法，它用来把实例渲染成一个虚拟 Node。它的定义在 `src/core/instance/render.js` 文件中：

```javascript
Vue.prototype._render = function (): VNode {
  const vm: Component = this;
  const { render, _parentVnode } = vm.$options;

  // reset _rendered flag on slots for duplicate slot check
  if (process.env.NODE_ENV !== "production") {
    for (const key in vm.$slots) {
      // $flow-disable-line
      vm.$slots[key]._rendered = false;
    }
  }

  if (_parentVnode) {
    vm.$scopedSlots = _parentVnode.data.scopedSlots || emptyObject;
  }

  // set parent vnode. this allows render functions to have access
  // to the data on the placeholder node.
  vm.$vnode = _parentVnode;
  // render self
  let vnode;
  try {
    vnode = render.call(vm._renderProxy, vm.$createElement);
  } catch (e) {
    handleError(e, vm, `render`);
    // return error render result,
    // or previous vnode to prevent render error causing blank component
    /* istanbul ignore else */
    if (process.env.NODE_ENV !== "production") {
      if (vm.$options.renderError) {
        try {
          vnode = vm.$options.renderError.call(vm._renderProxy, vm.$createElement, e);
        } catch (e) {
          handleError(e, vm, `renderError`);
          vnode = vm._vnode;
        }
      } else {
        vnode = vm._vnode;
      }
    } else {
      vnode = vm._vnode;
    }
  }
  // return empty vnode in case the render function errored out
  if (!(vnode instanceof VNode)) {
    if (process.env.NODE_ENV !== "production" && Array.isArray(vnode)) {
      warn("Multiple root nodes returned from render function. Render function " + "should return a single root node.", vm);
    }
    vnode = createEmptyVNode();
  }
  // set parent
  vnode.parent = _parentVnode;
  return vnode;
};
```

上面最核心的一句代码：`vnode = render.call(vm._renderProxy, vm.$createElement)`，这里调用了`render`方法，就是之前说的渲染函数，而这里传入的`vm.$createElement`其实就是先前一系列初始化中`initRender(vm)`把此方法挂载到 vm 的原型上，具体文件在`src/core/instance/render.js`:

```javascript
export function initRender (vm: Component) {
  ...

  // bind the createElement fn to this instance
  // so that we get proper render context inside it.
  // args order: tag, data, children, normalizationType, alwaysNormalize
  // internal version is used by render functions compiled from templates
  vm._c = (a, b, c, d) => createElement(vm, a, b, c, d, false)
  // normalization is always applied for the public version, used in
  // user-written render functions.
  vm.$createElement = (a, b, c, d) => createElement(vm, a, b, c, d, true)

}
```

其中`_c`是被模板编译成的 `render` 函数使用，而`$createElement`是用户手写`render`函数中使用。也就是说，文章开头我们的`main.js`中

```javascript
import Vue from "vue";

Vue.config.productionTip = false;

new Vue({
  render(h) {
    return h("div", {
      attrs: {
        id: "hello",
      },
    });
  },
}).$mount("#app");
```

`render`接收的参数`h`就是这里所说的`$createElement`

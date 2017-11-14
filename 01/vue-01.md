# Vue源码学习01-响应式架构的实现
> 合抱之木，生于毫末；九层之台，起于累土；千里之行，始于足下。——《老子》
## 前言
由于Vue版本更新还是比较快的，github上源码也会相应变化，所以学习的源码是可以直接&lt;script&gt;引入的2.4.0开发版本。

所有的故事从下面开始。
```javascript
//github:vue/src/core/instance/index.js
function Vue (options) {
  if (process.env.NODE_ENV !== 'production' &&
    !(this instanceof Vue)
  ) {
    warn('Vue is a constructor and should be called with the `new` keyword')
  }
  this._init(options)
}

initMixin(Vue)
stateMixin(Vue)
eventsMixin(Vue)
lifecycleMixin(Vue)
renderMixin(Vue)

export default Vue
```

## [响应式原理（Object.defineProperty ）](https://cn.vuejs.org/v2/guide/reactivity.html)
官网上对原理已经有了通俗易懂的介绍，在此不再赘述。主要是学习在代码层面是如何实现的。

initMixin、stateMixin等各种Mixin函数首先在Vue.prototype上定义一系列工具函数。
其中最主要的initMixin在Vue.prototype上定义了_init函数，当我们new Vue()时就会来到这里。
```javascript
function initMixin(Vue) {
    Vue.prototype._init = function(options) {
        var vm = this;
        ...
        initState(vm);
        ...
        if (vm.$options.el) {
            vm.$mount(vm.$options.el);
        }
    };
}
```
先来看看initState做了哪些事情。

```javascript
function initState(vm) {
    vm._watchers = [];
    var opts = vm.$options;
    if (opts.props) { initProps(vm, opts.props); }
    if (opts.methods) { initMethods(vm, opts.methods); }
    if (opts.data) {
        initData(vm);
    } else {
        observe(vm._data = {}, true /* asRootData */ );
    }
    if (opts.computed) { initComputed(vm, opts.computed); }
    if (opts.watch && opts.watch !== nativeWatch) {
        initWatch(vm, opts.watch);
    }
}

function initData(vm) {
    var data = vm.$options.data;
    data = vm._data = typeof data === 'function' ?
            getData(data, vm) :
            data || {};
    ...
    var keys = Object.keys(data);
    var i = keys.length;
    while (i--) {
        var key = keys[i];
        ...
        proxy(vm, "_data", key);
        ...
    }

    observe(data, true );
}

function proxy(target, sourceKey, key) {
    sharedPropertyDefinition.get = function proxyGetter() {
        return this[sourceKey][key]
    };
    sharedPropertyDefinition.set = function proxySetter(val) {
        this[sourceKey][key] = val;
    };
    Object.defineProperty(target, key, sharedPropertyDefinition);
}

var sharedPropertyDefinition = {
    enumerable: true,
    configurable: true,
    get: noop,
    set: noop
};

```
proxy函数在Vue上直接定义了各个data属性的proxyGetter和proxySetter，而相应的value通过Vue._data进行统一管理。

以上代码的作用在["Vue实例-数据与方法"](https://cn.vuejs.org/v2/guide/instance.html#数据与方法)中有详细说明。
## data选项的处理
### 如何将data选项记录为依赖

> 当你把一个普通的 JavaScript 对象传给 Vue 实例的 data 选项，Vue 将遍历此对象所有的属性，并使用 Object.defineProperty 把这些属性全部转为 getter/setter。

proxy函数过后是observe核心函数，代码有点长，需要点耐心。
```javascript
function observe(value, asRootData) {
    ...
        ob = new Observer(value);
    ...
}

var Observer = function Observer(value) {
    ...
    def(value, '__ob__', this);
    if (Array.isArray(value)) {
        ...
    } else {
        this.walk(value);
    }
};

Observer.prototype.walk = function walk(obj) {
    var keys = Object.keys(obj);
    for (var i = 0; i < keys.length; i++) {
        defineReactive$$1(obj, keys[i], obj[keys[i]]);
    }
};

function defineReactive$$1(
    obj,
    key,
    val,
    customSetter,
    shallow
) {
    var dep = new Dep();

    var property = Object.getOwnPropertyDescriptor(obj, key);
    if (property && property.configurable === false) {
        return
    }

    var getter = property && property.get;
    var setter = property && property.set;

    ...
    Object.defineProperty(obj, key, {
        enumerable: true,
        configurable: true,
        get: function reactiveGetter() {
            var value = getter ? getter.call(obj) : val;
            if (Dep.target) {
                dep.depend();
                ...
            }
            return value
        },
        set: function reactiveSetter(newVal) {
            var value = getter ? getter.call(obj) : val;

            if (newVal === value || (newVal !== newVal && value !== value)) {
                return
            }
            ...
            if (setter) {
                setter.call(obj, newVal);
            } else {
                val = newVal;
            }
            ...
            dep.notify();
        }
    });
}

```
根据代码，Vue在Vue._data中再一次定义了各个data选项的reactiveGetter和reactiveSetter，并且通过闭包，在defineReactive$$1函数中给每个data选项定义了一个Dep类型的私有dep变量。这个dep变量目前并没有用到，但它在reactiveGetter和reactiveSetter都有存在，是个核心角色，稍后详谈。

到此，data选项已经转化成了依赖。
### data选项依赖如何工作
再次回到_init函数，随着initState工作的结束，轮到$mount上场了。
```javascript
Vue$3.prototype.$mount = function(
    el,
    hydrating
) {
    ...
    return mountComponent(this, el, hydrating)
};
function mountComponent(
    vm,
    el,
    hydrating
) {
    ...
    updateComponent = function() {
        vm._update(vm._render(), hydrating);
    };

    vm._watcher = new Watcher(vm, updateComponent, noop);
    ...
    return vm
}
var mount = Vue$3.prototype.$mount;
Vue$3.prototype.$mount = function(
    el,
    hydrating
) {
    ...
    return mount.call(this, el, hydrating)
};

var Watcher = function Watcher(
    vm,
    expOrFn,
    cb,
    options
) {
    this.vm = vm;
    vm._watchers.push(this);
    // options
    if (options) {
        this.deep = !!options.deep;
        this.user = !!options.user;
        this.lazy = !!options.lazy;
        this.sync = !!options.sync;
    } else {
        this.deep = this.user = this.lazy = this.sync = false;
    }
    this.cb = cb;
    this.id = ++uid$2; // uid for batching
    this.active = true;
    this.dirty = this.lazy; // for lazy watchers
    this.deps = []; 
    this.newDeps = [];
    this.depIds = new _Set();
    this.newDepIds = new _Set();
    this.expression = expOrFn.toString();
    // parse expression for getter
    if (typeof expOrFn === 'function') {
        this.getter = expOrFn;
    } else {
        ...
    }
    this.value = this.lazy ?
        undefined :
        this.get();
};
```
注意到$mount调用的mountComponent创建了一个Watcher类型的实例，这个Watcher类型和前面提到的Dep类型，共同构成了data选项依赖工作的核心。
new Watcher()调用的get函数做了下面几件事，这里的Watcher实例称为render watcher。
* 将watcher自身压入一个targetStack栈中，可以简单理解为一个context上下文
* 执行传入的exporFn参数，这里是updateComponent，它是页面html的渲染函数
* 将前面压入的watcher出栈
```javascript
Watcher.prototype.get = function get() {
    pushTarget(this);
    var value;
    var vm = this.vm;
    try {
        value = this.getter.call(vm, vm);
    } catch (e) {
        ...
    } finally {
        ...
        popTarget();
        this.cleanupDeps();
    }
    return value
};

Dep.target = null;
var targetStack = [];

function pushTarget(_target) {
    if (Dep.target) { targetStack.push(Dep.target); }
    Dep.target = _target;
}

function popTarget() {
    Dep.target = targetStack.pop();
}
Watcher.prototype.cleanupDeps = function cleanupDeps() {
    var this$1 = this;

    var i = this.deps.length;
    while (i--) {
        var dep = this$1.deps[i];
        if (!this$1.newDepIds.has(dep.id)) {
            dep.removeSub(this$1);
        }
    }
    var tmp = this.depIds;
    this.depIds = this.newDepIds;
    this.newDepIds = tmp;
    this.newDepIds.clear();
    tmp = this.deps;
    this.deps = this.newDeps;
    this.newDeps = tmp;
    this.newDeps.length = 0;
};
```
updateComponent渲染html，就会获取data选项的值来展示实际数据，从而触发了data选项的reactiveGetter。(updateComponent复杂且强大，以后有机会再详细看下它的实现)

回顾一下之前reactiveGetter的定义，此时的Dep.target已经是一个Watcher类型的变量了，下一步当然是dep.depend
```javascript
var Dep = function Dep() {
    this.id = uid++;
    this.subs = [];
};

Dep.prototype.depend = function depend() {
    if (Dep.target) {
        Dep.target.addDep(this);
    }
};

Watcher.prototype.addDep = function addDep(dep) {
    var id = dep.id;
    if (!this.newDepIds.has(id)) {
        this.newDepIds.add(id);
        this.newDeps.push(dep);
        if (!this.depIds.has(id)) {
            dep.addSub(this);
        }
    }
};
```
Watcher和Dep的作用清晰展现出来了

- 每一个data依赖利用dep.subs来保存依赖的watcher
- 每一个watcher利用newDeps属性来收集用到的dep
- Watcher和Dep的相互作用实现了一个观察者模式。

### 如何在data依赖发生变化时更新组件
当data依赖项的reactiveSetter被调用时,dep.notify会通知所有依赖的watcher进行update。
```javascript
Dep.prototype.notify = function notify() {
    // stabilize the subscriber list first
    var subs = this.subs.slice();
    for (var i = 0, l = subs.length; i < l; i++) {
        subs[i].update();
    }
};
Watcher.prototype.update = function update() {
    ...
    queueWatcher(this);
};
```
前面说过每个data依赖都会保存依赖的watcher，所以先收集这次update所涉及的wathcer队列并进行去重处理，这也是queueWatcher代码中has[id]和waiting所起的作用。
```javascript
var queue = [];
var activatedChildren = [];
var has = {};
var circular = {};
var waiting = false;
var flushing = false;
var index = 0;
/**
 * Push a watcher into the watcher queue.
 * Jobs with duplicate IDs will be skipped unless it's
 * pushed when the queue is being flushed.
 */
function queueWatcher(watcher) {
    var id = watcher.id;
    if (has[id] == null) {
        has[id] = true;
        if (!flushing) {
            queue.push(watcher);
        } else {
            // if already flushing, splice the watcher based on its id
            // if already past its id, it will be run next immediately.
            var i = queue.length - 1;
            while (i > index && queue[i].id > watcher.id) {
                i--;
            }
            queue.splice(i + 1, 0, watcher);
        }
        // queue the flush
        if (!waiting) {
            waiting = true;
            nextTick(flushSchedulerQueue);
        }
    }
}
```
然后对创建好的watcher队列进行了一次排序处理，具体原因在代码注释中讲的非常详细。注意其中的watcher.run，它会调用watcher创建之初传入的expOrFn，至此完整的数据依赖响应过程完成。
```javascript
/**
 * Flush both queues and run the watchers.
 */
function flushSchedulerQueue() {
    flushing = true;
    var watcher, id;

    // Sort queue before flush.
    // This ensures that:
    // 1. Components are updated from parent to child. (because parent is always
    //    created before the child)
    // 2. A component's user watchers are run before its render watcher (because
    //    user watchers are created before the render watcher)
    // 3. If a component is destroyed during a parent component's watcher run,
    //    its watchers can be skipped.
    queue.sort(function(a, b) { return a.id - b.id; });

    // do not cache length because more watchers might be pushed
    // as we run existing watchers
    for (index = 0; index < queue.length; index++) {
        watcher = queue[index];
        id = watcher.id;
        has[id] = null;
        watcher.run();
        // in dev build, check and stop circular updates.
        if ("development" !== 'production' && has[id] != null) {
            circular[id] = (circular[id] || 0) + 1;
            if (circular[id] > MAX_UPDATE_COUNT) {
                warn(
                    'You may have an infinite update loop ' + (
                        watcher.user ?
                        ("in watcher with expression \"" + (watcher.expression) + "\"") :
                        "in a component render function."
                    ),
                    watcher.vm
                );
                break
            }
        }
    }

    // keep copies of post queues before resetting state
    var activatedQueue = activatedChildren.slice();
    var updatedQueue = queue.slice();

    resetSchedulerState();

    // call component updated and activated hooks
    callActivatedHooks(activatedQueue);
    callUpdatedHooks(updatedQueue);
    ...
}

/**
 * Reset the scheduler's state.
 */
function resetSchedulerState() {
    index = queue.length = activatedChildren.length = 0;
    has = {}; {
        circular = {};
    }
    waiting = flushing = false;
}

Watcher.prototype.run = function run() {
    if (this.active) {
        var value = this.get();
        ...
    }
};
```
注意上面代码中的nextTick是一个非常精彩的异步调用，代码实现也很简洁。
> Vue在内部尝试对异步队列使用原生的 Promise.then 和 MutationObserver，如果执行环境不支持，会采用 setTimeout(fn, 0) 代替。

更详细的nextTick介绍戳这里[异步更新队列](https://cn.vuejs.org/v2/guide/reactivity.html#异步更新队列)
## computed选项的处理
跟data选项一样，computed选项的处理也分为三个步骤

### computed选项如何转化为依赖
同样是在initState中，computed选项从这里出发。
```javascript
function initState(vm) {
    ...
    if (opts.computed) { initComputed(vm, opts.computed); }
    ...
}

var computedWatcherOptions = { lazy: true };

function initComputed(vm, computed) {
    var watchers = vm._computedWatchers = Object.create(null);

    for (var key in computed) {
        var userDef = computed[key];
        var getter = typeof userDef === 'function' ? userDef : userDef.get; 
        ...
        // create internal watcher for the computed property.
        watchers[key] = new Watcher(vm, getter, noop, computedWatcherOptions);

        if (!(key in vm)) {
            defineComputed(vm, key, userDef);
        } else {
            ...
        }
    }
}

function defineComputed(target, key, userDef) {
    if (typeof userDef === 'function') {
        sharedPropertyDefinition.get = createComputedGetter(key);
        sharedPropertyDefinition.set = noop;
    } else {
        sharedPropertyDefinition.get = userDef.get ?
            userDef.cache !== false ?
            createComputedGetter(key) :
            userDef.get :
            noop;
        sharedPropertyDefinition.set = userDef.set ?
            userDef.set :
            noop;
    }
    Object.defineProperty(target, key, sharedPropertyDefinition);
}

function noop(a, b, c) {}
```
可以看到对于每个computed选项主要的工作是两个:
- 每个computed选项相应的都有一个watcher，注意到在new Watcher时传入了lazy:true选项，这会将watcher.get求值操作推迟到前面提到的updateComponent中。这里的Watcher实例称为computed watcher。
- 将每个computed选项直接在Vue上进行了defineProperty操作，而data选项是统一由_data进行管理的。

### computed选项如何工作
在updateComponent中触发computedGetter，最终对computed选项进行get求值操作，而get的作用在前面也提过了，将当前watcher加入到data选项的dep.subs中。
```javascript
function createComputedGetter(key) {
    return function computedGetter() {
        var watcher = this._computedWatchers && this._computedWatchers[key];
        if (watcher) {
            if (watcher.dirty) {
                watcher.evaluate();
            }
            if (Dep.target) {
                watcher.depend();
            }
            return watcher.value
        }
    }
}
Watcher.prototype.evaluate = function evaluate() {
    this.value = this.get();
    this.dirty = false;
};
Watcher.prototype.depend = function depend() {
    var this$1 = this;
    var i = this.deps.length;
    while (i--) {
        this$1.deps[i].depend();
    }
};
```
### computed选项如何更新
因为computedGetter操作将computed watcher加入到data选项的dep.subs中，所以data选项的notify自然会通知更新。

注意在computedGetter代码中有一个重要的地方 `if(Dep.target){ watcher.depend(); }` 。当一个data选项没有在render watcher使用,但是依赖于这个data选项的computed值在render watcher中使用时，这段代码可以将render watcher加入到data选项的watcher依赖列表中。

## watch选项的处理
watch选项同样也是在initState出发的。
```javascript
function initState(vm) {
    ...
    if (opts.watch && opts.watch !== nativeWatch) {
        initWatch(vm, opts.watch);
    }
    ...
}

function initWatch(vm, watch) {
    for (var key in watch) {
        var handler = watch[key];
        if (Array.isArray(handler)) {
            for (var i = 0; i < handler.length; i++) {
                createWatcher(vm, key, handler[i]);
            }
        } else {
            createWatcher(vm, key, handler);
        }
    }
}

function createWatcher(
    vm,
    keyOrFn,
    handler,
    options
) {
    if (isPlainObject(handler)) {
        options = handler;
        handler = handler.handler;
    }
    if (typeof handler === 'string') {
        handler = vm[handler];
    }
    return vm.$watch(keyOrFn, handler, options)
}

Vue.prototype.$watch = function(
    expOrFn,
    cb,
    options
) {
    var vm = this;
    if (isPlainObject(cb)) {
        return createWatcher(vm, expOrFn, cb, options)
    }
    options = options || {};
    options.user = true;
    var watcher = new Watcher(vm, expOrFn, cb, options);
    if (options.immediate) {
        cb.call(vm, watcher.value);
    }
    return function unwatchFn() {
        watcher.teardown();
    }
};

Watcher.prototype.teardown = function teardown() {
    var this$1 = this;

    if (this.active) {
        if (!this.vm._isBeingDestroyed) {
            remove(this.vm._watchers, this);
        }
        var i = this.deps.length;
        while (i--) {
            this$1.deps[i].removeSub(this$1);
        }
        this.active = false;
    }
};
```
可以看到，核心的机制仍然是通过new Watcher加入到data选项的dep.subs依赖列表中，这里的Watcher实例称为$watcher watcher。

## 总结
- 单个数据响应式的基础是Object.defineProperty
- 使用Watcher来封装不同类型的数据响应操作，比如render watcher、computed watcher、$watcher watcher
- Watcher和单个数据的dep闭包属性构成一个观察者模式是整个响应式工作的框架

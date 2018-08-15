# Vue源码学习02-Template HTML的解析实现(一)-渲染函数render的产生
> 他山之石，可以攻玉。——《诗经·小雅·鹤鸣》
## 前言
上一篇文章简单学习了Vue响应式架构的实现，接下来就是另一个核心，对Vue template的编译解析。
由于这块的功能非常复杂且有很多细节的地方，本人能力有限，只能大致描述一下过程，有理解不对的地方，欢迎批评指正。

实现主要分两个部分:渲染函数render是怎么生成的、VNode又是怎么绘制成HTML。

先简单回顾下$mount方法的调用过程，源码的顺序是这样的
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

// public mount method（第一个）
Vue$3.prototype.$mount = function(
    el,
    hydrating
) {
    ...
    return mountComponent(this, el, hydrating)
};

var mount = Vue$3.prototype.$mount;
//第二个
Vue$3.prototype.$mount = function(
    el,
    hydrating
) {
    ...
    if (template) {
    ...
        var ref = compileToFunctions(template, {
            shouldDecodeNewlines: shouldDecodeNewlines,
            delimiters: options.delimiters,
            comments: options.comments
        }, this);
        var render = ref.render;
        var staticRenderFns = ref.staticRenderFns;
        options.render = render;
        options.staticRenderFns = staticRenderFns;
       ...
    }
    
    return mount.call(this, el, hydrating)
};
```
源码中定义了两处 Vue$3.prototype.$mount，而根据执行顺序是第二个Vue$3.prototype.$mount先执行，它的作用是产生了一个html编译器解析template并生成了相应的render函数。

## 编译器和render函数的产生
一切从compileToFunctions开始，来到了createCompilerCreator。
```javascript
// `createCompilerCreator` allows creating compilers that use alternative
// parser/optimizer/codegen, e.g the SSR optimizing compiler.
// Here we just export a default compiler using the default parts.
var createCompiler = createCompilerCreator(function baseCompile(
    template,
    options
) {
    var ast = parse(template.trim(), options);
    optimize(ast, options);
    var code = generate(ast, options);
    return {
        ast: ast,
        render: code.render,
        staticRenderFns: code.staticRenderFns
    }
});

var ref$1 = createCompiler(baseOptions);
var compileToFunctions = ref$1.compileToFunctions;
...
Vue$3.compile = compileToFunctions;
...
```
createCompilerCreator顾名思义是编译器生成器工厂函数，它设计成一个高阶函数，可以根据执行环境有针对性的更换合适的编译器，在web端默认是baseComplie，它返回的结果是编译器生成器，注意还不是执行编译器。
```javascript
function createCompilerCreator(baseCompile) {
    return function createCompiler(baseOptions) {
        function compile(
            template,
            options
        ) {
            var finalOptions = Object.create(baseOptions);
            var errors = [];
            var tips = [];
            finalOptions.warn = function(msg, tip) {
                (tip ? tips : errors).push(msg);
            };

            if (options) {
                // merge custom modules
                if (options.modules) {
                    finalOptions.modules =
                        (baseOptions.modules || []).concat(options.modules);
                }
                // merge custom directives
                if (options.directives) {
                    finalOptions.directives = extend(
                        Object.create(baseOptions.directives),
                        options.directives
                    );
                }
                // copy other options
                for (var key in options) {
                    if (key !== 'modules' && key !== 'directives') {
                        finalOptions[key] = options[key];
                    }
                }
            }

            var compiled = baseCompile(template, finalOptions); 
            ...
            return compiled
        }

        return {
            compile: compile,
            compileToFunctions: createCompileToFunctionFn(compile)
        }
    }
}
```
createCompileToFunctionFn也是一个高阶函数，入参compile对前面提到的baseCompile进行了一些包装，返回的结果就是最终的编译器函数。
```javascript
function createCompileToFunctionFn(compile) {
    var cache = Object.create(null);

    return function compileToFunctions(
        template,
        options,
        vm
    ) {
        options = options || {};
        ...
        // check cache
        var key = options.delimiters ?
            String(options.delimiters) + template :
            template;
        if (cache[key]) {
            return cache[key]
        }
        // compile
        var compiled = compile(template, options);
        ...
        // turn code into functions
        var res = {};
        var fnGenErrors = [];
        res.render = createFunction(compiled.render, fnGenErrors);
        res.staticRenderFns = compiled.staticRenderFns.map(function(code) {
            return createFunction(code, fnGenErrors)
        });
        ...
        return (cache[key] = res)
    }
}
```
compileToFunctions的调用是在前面提到过的第二个Vue$3.prototype.$mount，执行过程大致可以分成两步：
### 编译compile
```javascript
var compiled = compile(template, options);
```
根据前面的分析，compile调用的主要是入参baseCompile函数
```javascript
var createCompiler = createCompilerCreator(function baseCompile(
    template,
    options
) {
    var ast = parse(template.trim(), options);
    optimize(ast, options);
    var code = generate(ast, options);
    return {
        ast: ast,
        render: code.render,
        staticRenderFns: code.staticRenderFns
    }
});
```
baseCompile将编译过程分为三个步骤
#### 解析器(parse)
解析函数的核心功能就是对template进行文本解析以生成抽象语法树(ast)，主要代码参考[https://johnresig.com/blog/pure-javascript-html-parser/](https://johnresig.com/blog/pure-javascript-html-parser/)
```javascript
/**
 * Convert HTML string to AST.
 */
function parse(
    template,
    options
){
    ...
    parseHTML
    ...
}
```


#### 优化器(optimize)
优化器的主要作用是对template静态的地方进行标记，后面就不需要重复渲染了。
```javascript
/**
 * Goal of the optimizer: walk the generated template AST tree
 * and detect sub-trees that are purely static, i.e. parts of
 * the DOM that never needs to change.
 *
 * Once we detect these sub-trees, we can:
 *
 * 1. Hoist them into constants, so that we no longer need to
 *    create fresh nodes for them on each re-render;
 * 2. Completely skip them in the patching process.
 */
function optimize(root, options) {
    ...
}
```
#### 生成器(generate)
```javascript
function generate(
    ast,
    options
) {
    var state = new CodegenState(options);
    var code = ast ? genElement(ast, state) : '_c("div")';
    return {
        render: ("with(this){return " + code + "}"),
        staticRenderFns: state.staticRenderFns
    }
}
```
看到其中的render属性是不是特熟悉，没错，render函数的字符串版本在这里诞生了。
同时也不要忽略了genElement函数，vue指令的解析在这里编译成可运行的代码。
```javascript
function genElement(el, state) {
    if (el.staticRoot && !el.staticProcessed) {
        return genStatic(el, state)
    } else if (el.once && !el.onceProcessed) {
        return genOnce(el, state)
    } else if (el.for && !el.forProcessed) {
        return genFor(el, state)
    } else if (el.if && !el.ifProcessed) {
        return genIf(el, state)
    } else if (el.tag === 'template' && !el.slotTarget) {
        return genChildren(el, state) || 'void 0'
    } else if (el.tag === 'slot') {
        return genSlot(el, state)
    } else {
        // component or element
        var code;
        if (el.component) {
            code = genComponent(el.component, el, state);
        } else {
            var data = el.plain ? undefined : genData$2(el, state);

            var children = el.inlineTemplate ? null : genChildren(el, state, true);
            code = "_c('" + (el.tag) + "'" + (data ? ("," + data) : '') + (children ? ("," + children) : '') + ")";
        }
        // module transforms
        for (var i = 0; i < state.transforms.length; i++) {
            code = state.transforms[i](el, code);
        }
        return code
    }
}
```
### 渲染函数render的诞生
经过上面compile函数的复杂运行，最终的结果是将template文本转换成render函数的字符串版本
在createCompileToFunctionFn中，compile生成的render字符串版本会转变成真正的函数
```javascript
function createFunction(code, errors) {
    try {
        return new Function(code)
    } catch (err) {
        errors.push({ err: err, code: code });
        return noop
    }
}
```
至此，render函数已经准备好了，接下来就是绘制html的时刻了。
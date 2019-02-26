## template模板是如何编译成render渲染函数的流程分析
在上一节分析 $mount 函数挂载的过程
```javascript
var ref = compileToFunctions(template, {
                    outputSourceRange: "development" !== 'production',
                    shouldDecodeNewlines: shouldDecodeNewlines,
                    shouldDecodeNewlinesForHref: shouldDecodeNewlinesForHref,
                    /* 改变纯文本插入分隔符。*/
                    delimiters: options.delimiters,
                    /*当设为 true 时，将会保留且渲染模板中的 HTML 注释。默认行为是舍弃它们。*/
                    comments: options.comments
                }, this);
/*
    第一个参数:template模板
    第二个参数:一些配置的选项, 不同的配置, 可以进行不同的编译
    第三个参数:vm的实例
*/
```
可以看出 compileToFunctions 函数是把 template 编译成 render 函数,在查看这个函数的
最终入口是 createCompilerCreator 函数,可以看出编译入口文件这么复杂, 原因是Vue.js在不同的
平台下的编译.
```javascript
 /* 生成编译器*/
/*baseCompile是一个函数,  进行编译  */
function createCompilerCreator(baseCompile) {
    /*返回一个createCompiler函数*/
    return function createCompiler(baseOptions) {
        function compile(template, options) {
            /*最终选项finalOptions*/
            var finalOptions = Object.create(baseOptions);
            var errors = []; /*错粗的信息*/
            var tips = []; /*提示的信息*/

            /*封装收集错误信息或者提示信息的函数*/
            var warn = function (msg, range, tip) {
                (tip ? tips : errors).push(msg);
            };
            /*用于检测用户有哪些自定义的配置, 最终合并到finalOptions上*/
            if (options) {
                if (options.outputSourceRange) {
                    // $flow-disable-line
                    var leadingSpaceLength = template.match(/^\s*/)[0].length;

                    warn = function (msg, range, tip) {
                        var data = {msg: msg};
                        if (range) {
                            if (range.start != null) {
                                data.start = range.start + leadingSpaceLength;
                            }
                            if (range.end != null) {
                                data.end = range.end + leadingSpaceLength;
                            }
                        }
                        (tip ? tips : errors).push(data);
                    };
                }
                // merge custom modules
                if (options.modules) {
                    finalOptions.modules =
                        (baseOptions.modules || []).concat(options.modules);
                }
                // merge custom directives
                if (options.directives) {
                    finalOptions.directives = extend(
                        Object.create(baseOptions.directives || null),
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
            finalOptions.warn = warn;
            /* 根据最终finalOptions的进行编译 */
            var compiled = baseCompile(template.trim(), finalOptions);
            {
                detectErrors(compiled.ast, warn);
            }
            compiled.errors = errors; /*给compiled扩展errors属性*/
            compiled.tips = tips;    /*给compiled扩展tips属性 */
            return compiled         /*最终把compiled返回*/
        }
        return {
            /*compile函数的引用*/
            compile: compile,
            /*获取createCompileToFunctionFn函数返回值的引用*/
            compileToFunctions: createCompileToFunctionFn(compile)
        }
    }
}
```
接下来分析 baseCompile 函数
```javascript
/*
 *  编译器的创建者
 *  这个函数的只要作用:
 *  1 parse函数: 把模板编程抽象语法树(AST)
 *  2 optimize: 在编译过程的优化
 *  3 generate函数: 把AST生成不同的不同平台的代码, 浏览器端需要生成字符串
 */
var createCompiler = createCompilerCreator(function baseCompile(template, options) {
    /*parse函数的作用: 把模板编译成AST, 在不同的平台的下生成AST都是一样的*/
    var ast = parse(template.trim(), options);
    if (options.optimize !== false) {
        optimize(ast, options);
    }
    /*把AST生成不同平台的代码, 浏览器端需要生成函数字符串*/
    var code = generate(ast, options);
    /*返回一个对象*/
    return {
        ast: ast, /*抽象语法树*/
        render: code.render, /*render函数体的字符串*/
        staticRenderFns: code.staticRenderFns /*渲染的优化 staticRenderFns是一个数组*/
    }
});
```
baseCompile 函数是整个编译过程的核心代码, 当做参数传递给 createCompilerCreator.
在createCompilerCreator 函数里面, 经过 options 最终的合并成 finalOptions,
传递给 baseCompile 函数,进行 parse, optimize, generate (后面的小节会分析).

createCompilerCreator 函数的返回值 createCompiler 函数,需要接受 baseOptions
```javascript
/*一些基础的配置*/
var baseOptions = {
        expectHTML: true,
        modules: modules$1,
        directives: directives$1,
        isPreTag: isPreTag,
        isUnaryTag: isUnaryTag,
        mustUseProp: mustUseProp,
        canBeLeftOpenTag: canBeLeftOpenTag,
        isReservedTag: isReservedTag,
        getTagNamespace: getTagNamespace,
        staticKeys: genStaticKeys(modules$1)
    };
```
createCompiler 函数最终的返回值是一个对象
```javascript
return {
     compile: compile, /*函数*/
     compileToFunctions: createCompileToFunctionFn(compile) /*函数*/
}
```
最终发现在 $mount 里面 compileToFunctions 函数, 最终就是在 createCompileToFunctionFn(compile),
进行编译
```javascript
 /*把函数字符串编译成fn*/
/*compile 是一个函数 */
function createCompileToFunctionFn(compile) {
    /*一个缓存对象,  同一个模板,不会被编译两次, 在次编译直接返回结果 ,进行优化*/
    var cache = Object.create(null);
    /* 返回compileToFunctions函数 */
    return function compileToFunctions(template,options,vm) {
        options = extend({}, options);
        var warn$$1 = options.warn || warn;
        delete options.warn;

        /* istanbul ignore if */
        {
            // detect possible CSP restriction
            try {
                /*判断是否支持new Function来定义一个函数,不支持会抛出异常 */
                /* 在内容安全策略比较严格的情况下, 有可能不支持new Function定义函数*/
                new Function('return 1');
            } catch (e) { /*不支持,产生异常,捕获异常, 提示模板编译器不能再这种环境下编译 */
                if (e.toString().match(/unsafe-eval|CSP/)) {
                    warn$$1(
                        'It seems you are using the standalone build of Vue.js in an ' +
                        'environment with Content Security Policy that prohibits unsafe-eval. ' +
                        'The template compiler cannot work in this environment. Consider ' +
                        'relaxing the policy to allow unsafe-eval or pre-compiling your ' +
                        'templates into render functions.'
                    );
                }
            }
        }

        // check cache
        /*根据delimiters的配置, 可以把{{}}语法改成es6的语法*/
        var key = options.delimiters
            ? String(options.delimiters) + template
            : template;
         /*进行缓存, 判断是否存在这个缓存中*/
        if (cache[key]) {
            return cache[key]
        }

        // compile
        /*真正的编译工作依赖这个函数  一个核心的方法, 作为createCompileToFunctionFn的参数传递过来*/
        var compiled = compile(template, options);

        // check compilation errors/tips
        {
            /*编译过程的的错误信息, 是一个数组*/
            if (compiled.errors && compiled.errors.length) {
                if (options.outputSourceRange) {
                    compiled.errors.forEach(function (e) {
                        warn$$1(
                            "Error compiling template:\n\n" + (e.msg) + "\n\n" +
                            generateCodeFrame(template, e.start, e.end),
                            vm
                        );
                    });
                } else { /*一些提示信息*/
                    warn$$1(
                        "Error compiling template:\n\n" + template + "\n\n" +
                        compiled.errors.map(function (e) {
                            return ("- " + e);
                        }).join('\n') + '\n',
                        vm
                    );
                }
            }
            if (compiled.tips && compiled.tips.length) {
                if (options.outputSourceRange) {
                    compiled.tips.forEach(function (e) {
                        return tip(e.msg, vm);
                    });
                } else {
                    compiled.tips.forEach(function (msg) {
                        return tip(msg, vm);
                    });
                }
            }
        }

        // turn code into functions
        var res = {};
        var fnGenErrors = []; /*错误信息的收集*/
        /*把render函数字符串生成真的render函数*/
        res.render = createFunction(compiled.render, fnGenErrors);
        /*进行渲染优化*/
        res.staticRenderFns = compiled.staticRenderFns.map(function (code) {
            return createFunction(code, fnGenErrors)
        });

        // check function generation errors.
        // this should only happen if there is a bug in the compiler itself.
        // mostly for codegen development use
        /* istanbul ignore if */
        {
            if ((!compiled.errors || !compiled.errors.length) && fnGenErrors.length) {
                warn$$1(
                    "Failed to generate render function:\n\n" +
                    fnGenErrors.map(function (ref) {
                        var err = ref.err;
                        var code = ref.code;

                        return ((err.toString()) + " in\n\n" + code + "\n");
                    }).join('\n'),
                    vm
                );
            }
        }

        return (cache[key] = res)
    }
}
```
createFunction 的代码实现
```javascript
 /*这个函数的作用: 把函数体字符串, 生成一个函数*/
function createFunction(code, errors) {
    try {
        return new Function(code)
    } catch (err) { /* 捕获异常 */
        errors.push({err: err, code: code});
        return noop /*noop 是一个空函数, 什么呀没有做*/
    }
}
```
createFunction函数通过new Function把函数字符串,定义成一个render函数


整个编译阶段是如此复杂, 原因就在于不同的平台下, 进行不同的代码编译
至此 整个的编译最核心的过程:
1. parse  把模板编译成AST, 在不同的平台的下生成AST都是一样的
2. optimize
3. generate  把AST生成不同平台的代码, 浏览器端需要生成函数字符串
在下面的小节会逐个分析





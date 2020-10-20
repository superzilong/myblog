---
title: Javascript中的异步调用
date: 2020-10-18 22:18:20
categories:
- javascript
tags:
- javascript基础
toc: true
---

一文搞懂JS中的异步调用💪
<!-- more -->
# 一. 异步与同步

Javascript只有一根线程, 所以异步调用很重要.

在javascript中调用接口时, 需要等一段时间, 这个接口才能返回, 如果在等待的时候, 去执行其它的任务, 等接口返回时再继续执行, 这就是异步调用. 

如果一直等待接口执行完毕, 不进行切换, 那就是同步执行.

异步调用的有点就是是的cpu的性能得到更充分的利用, 但是难点在于让多个异步调用按照我们指定的顺序执行.

# 二.  传统的异步调用

以异步版本的readfile接口为例, 在调用这个接口的时候需要写一个回调函数:

```js
const fs = require("fs");

fs.readFile("./test1.html", function(error, data){
    if (error) throw error;
    console.log(data);
    // read the second file
    fs.readFile("./test2.html", function(error, data){
        if (error) throw error;
   		 console.log(data);
        // if there is a third file, need use another callback function.
    })
})
```

但是如果要**按顺序**读取两个文件, 就需要在回调中嵌套另外一个回调, 这种写法非常不优雅, 被称为["回调函数噩梦"](http://callbackhell.com/)(callback hell). 

下面都以按顺序读取两个文件为例来讲解异步调用的进化.

# 三. 使用Promise进行改进

同样以readFile接口为例, 用Promise改进后的样子如下, 使用Promise的方法可以不用嵌套多层异步调用, 但是仍然用到很多then调用, 有许多的回调函数, 而**我们的目标是让异步调用跟同步调用一般简洁.**

```js
// 将readFile转换成返回Promise的版本
function proReadFile(path) {
    return new Promise((resolve, reject)=>{
        fs.readFile(path, function(error, data) {
                if (error) reject(error);
                resolve(data);
            }
        );
    });
}

proReadFile("./test1.html")
.then(function(data) {
    console.log(data.toString());
    return proReadFile("./test2.html");
})
.then(function(data) {
    console.log(data.toString());
})
.catch(function(err) {
    console.log(err)
})
```

补充一点, 可以使用promisify来改造readFile函数

```js
const promisify = require("util").promisify
const proReadFile = promisify(fs.readFile) 
```



# 四. 使用生成器函数(generator function)配合自动执行器

关于生成器函数的详细说明, 请参看MDN的[这篇说明](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Statements/function*). 这里引用一段其中的描述:

> **生成器函数**在执行时能暂停，后面又能从暂停处继续执行。
>
> 调用一个**生成器函数**并不会马上执行它里面的语句，而是返回一个这个生成器的 **迭代器对象**。当这个迭代器的 `next() `方法被首次（后续）调用时，其内的语句会执行到第一个（后续）出现[`yield`](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Operators/yield)的位置为止，[`yield`](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Operators/yield) 后紧跟迭代器要返回的值。
>
> `next()`方法返回一个对象，这个对象包含两个属性：value 和 done，value 属性表示本次 `yield `表达式的返回值，done 属性为布尔类型，表示生成器后续是否还有` yield `语句，即生成器函数是否已经执行完毕并返回。
>
> 调用 `next()`方法时，如果传入了参数，那么这个参数会传给**上一条执行的 yield语句左边的变量**，例如下面例子中的` x `：
>
> 当在生成器函数中显式 `return `时，会导致生成器立即变为完成状态，即调用 `next()` 方法返回的对象的 `done `为 `true`。如果 `return `后面跟了一个值，那么这个值会作为**当前**调用 `next()` 方法返回的 value 值。

假设有这样一个生成器函数:

```js
const fs = require("fs");
const promisify = require("util").promisify
const proReadFile = promisify(fs.readFile)

function* gen(file1, file2) {
    var res1 = yield proReadFile(file1)
    console.log(res1.toString())
    var res2 = yield proReadFile(file2)
    console.log(res2.toString())
}

```

下面这个例子中演示如何使用生成器函数来调用异步函数, **注意yield关键字的后面的表达式要返回一个Promise对象**, 但是yield关键字返回的是异步调用的返回值, 例如调用readFile, yield返回就是文件内容:

```js
// 手动执行生成器函数
var g = gen("./test1.html", "./test2.html")
g.next().value.then(
    function(data) {
        g.next(data.toString()).value.then(
            function(data) {
                g.next(data);
            }
        )
    }
)

```

这样使用生成器函数是很麻烦的, 但是可以使用递归来封装一个自动执行器, 这里只用一个简单的例子展示, 标准实现可以看[co 函数库](https://github.com/tj/co):

```js
var g = gen("./test1.html", "./test2.html")

// 递归封装的自动执行器
function autoRun(g) {
    var res = g.next();
    function next(res) {
        if(res.done) {
            return;
        };
        res.value.then(function(data) {
            next(g.next(data))
        });
    }
    next(res);
}

// 使用自动执行器来调用生成器函数
autoRun(g)
```

## 五. 使用async和await

async标记的函数, 会自动用async函数内置的自动执行器来执行生成器函数, 所以async函数是generate函数的**语法糖**.

在async函数中使用await就相当于在generate函数中使用yield关键字, 同理await后面的表达式要返回一个promise对象, 但是await返回的是异步调用的返回值.

async函数的返回值是一个promise对象.

下面的伪代码简单描述下async的原理:

```js
async function fn(args) {
    await expression1;
    await expression2;
    ...
}

// 等同于

function fn(args) {
    // 生成器函数
    function* gen(params) {
        yield expression1;
        yield expresson2;
    }
    // 自动执行器, spawn是产卵的意思...
    function spawn(gf) {
        var g = gf(args);
        // 递归调用g.next()
    }
    return spawn(gen)
}
```

下面一个例子展示了async和await的用法, 我们可以看到**使用async和await使得异步调用非常接近同步调用的形式**, 只是多了async和await关键字:

```js
const fs = require("fs");
const promisify = require("util").promisify
const proReadFile = promisify(fs.readFile)

async function read2Files(file1, file2) {
    var f1 = await proReadFile(file1)
    console.log(f1.toString())

    var f2 = await proReadFile(file2)
    console.log(f2.toString())
}

read2Files("./test1.html", "./test2.html")
```

## 六. 参考链接

[阮一峰博客]: http://www.ruanyifeng.com/blog/2015/04/generator.html


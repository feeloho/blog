# 在 JavaScript 中使用 Generator 函数实现异步变同步操作

在 ES6 中提供了一种新的异步编程解决方案，那就是 Generator 函数。这种函数又被称为生成器函数、可暂停函数。在它的方法体内可以使用 yield 关键字，配合 yield 可以实现协程。

本文主要介绍对 Generator 函数的实际应用，不过多介绍 Generator 函数的相关基础概念。



## 01. 一个普通的 HTTP 请求

```javascript
const axios = require('axios')

axios.get('https://sxyz.blog').then(res => {
    console.log(res.data)
})
```

上面代码中使用了 axios 模块发起了一个简单的 GET 请求，当这个请求得到响应时会执行 then 块中的代码，打印出响应正文。

但是如果出现连续操作（请求完一个后接着请求下一个）的时候，那么就要这么写了：

```javascript
const axios = require('axios')

axios.get('https://sxyz.blog').then(res => {
    axios.get('https://sxyz.blog').then(res => {
        console.log(res.data)
    })
})
```

从上面代码中可以发现，如果当这种连续请求的操作变得多了起来，那么就会出现 then 连环嵌套的尴尬场面。而这种需求在日常编码中却又非常常见。



## 02. 使用 Generator 函数

```javascript
const axios = require('axios')

function* request () {
    let res

    res = yield axios.get('https://sxyz.blog')
    console.log(res.data)

    res = yield axios.get('https://sxyz.blog')
    console.log(res.data)
}


let p1, p2
const r = request()

p1 = r.next().value  // Promise object
p1.then(v => {
    p2 = r.next(v).value

    p2.then(v => {
        r.next(v)

        // p3 = r.next(v).value
        // p3.then(...)
        // ...
    })
})
```

上面代码中，创建了一个名为 `request` 的 Generator 函数，然后在后面手动调用 `next` 方法接收 `axios.get` 所产生的 Promise 对象，并且调用 then 方法依次执行。



## 03. 改进 Generator 函数

虽然现在我们的关键代码（request 函数体中的请求代码）已经是同步了，这很棒。但是还有一个问题就是我们需要在代码中手动依次调用 `next` 来完成这些操作，这个过程非常繁琐且不合理。并且 then 的嵌套问题依旧存在。所以我们需要封装一个函数来帮我们自动完成这些操作：

```javascript
const axios = require('axios')

function* request () {
    let res

    res = yield axios.get('https://sxyz.blog')
    console.log(res.data)

    res = yield axios.get('https://sxyz.blog')
    console.log(res.data)
}

const co = (gen, ...args) => {
    const g = gen(...args)

    const actuator = x => {
        if (x.done) {
            return
        }

        x.value.then(value => {
            actuator(g.next(value))
        })
    }

    actuator(g.next())
}


// 现在只需
co(request)

// 如果有参数
co(request, 'aaa', 'bbbb')
```

其中 co 函数用来自动触发所有的 `next` 方法，并且解决了 then 不停嵌套的问题。

那么现在还有一些问题，比如我该如何得到 Generator 函数所产生的返回值呢，例如下面这样的代码：

```javascript
const axios = require('axios')

function* foo () {
    return yield axios.get('https://sxyz.blog')
}

function* bar () {
    const { data } = yield foo()
    console.log(data)

    return yield axios.get('https://sxyz.blog')
}

co(function* () {
    console.log((yield bar()).data)
}())
```

想要在 co 的匿名函数中得到 bar 的执行结果，同时 bar 函数也可以得到 foo 的函数结果。

如果要实现这样的功能，需要对 co 函数进行一些修改：

```javascript
const isGenerator = f =>
    f.constructor && f.constructor.name === 'GeneratorFunction'
const isGeneratorObject = o =>
    o.constructor && o.constructor.constructor && o.constructor.constructor.name === 'GeneratorFunction'

const co = g => {
    let resolve
    const p = new Promise(r => resolve = r)

    const actuator = x => {
        if (x.done) {
            return resolve(x.value)
        }

        Promise.resolve(
            isGeneratorObject(x.value) ? co(x.value) : x.value
        )
        .then(value => {
            actuator(g.next(value))
        }, reason => {
            g.throw(reason)
        })
    }

    actuator(g.next())
    return p
}
```

现在统一了每个 co 函数都返回 Promise 对象。加进了对 Generator 对象的检查，如若 yield 后面的不是 Promise 对象而是 Generator 对象时，则再次调用 co 自身来转成 Promise 对象。



## 04. 包裹层：对类的扩展

我们将上面请求的代码进一步封装成一个类：

```javascript
class Test {
    *foo () {
        return yield axios.get('https://sxyz.blog')
    }

    *bar () {
        const { data } = yield this.foo()
        console.log(data)

        return yield axios.get('https://sxyz.blog')
    }
}

co(function* () {
    const t = new Test()
    console.log((yield t.bar()).data)
}())
```

但是此时发现，所有与类有关的操作都需要写到 co 的函数体内，有点麻烦，有没有办法可以直接使用呢？有的，可以使用 ES6 中的 Proxy 对象来实现：

```javascript
const get = (target, prop, receiver) => {
    return (...args) => {
        if (!prop in target){
            return
        }

        return isGenerator(target[prop]) ?
            co(target[prop].call(target, ...args)) : target[prop].call(target, ...args)
    }
}

const te = c =>
    new Proxy(c, {
        construct (cls, args) {
            return new Proxy(new cls(...args), { get })
        }
    })


const Test = te(class {
    *foo () {
        return yield axios.get('https://sxyz.blog')
    }

    *bar () {
        const { data } = yield this.foo()
        console.log(data)

        return yield axios.get('https://sxyz.blog')
    }
})

const t = new Test()
t.bar().then(res => {
    console.log(res.data)
})
```

这样就可以不用再单独写 co 函数了，不过可能实际用途不是很大，参考只用。



## 05. 总结

本文介绍了如何使用 Generator 函数实现将异步操作转变为同步操作的方法。但在实际编码过程中可直接使用 await/async 语法糖。本文算是作为补充 Generator 同步实现的原理来看吧。


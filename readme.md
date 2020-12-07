#### 手撕koa源码
- 当前koa版本：2.13.0
- koa继承自nodejs的`events`类，因此koa实例可以使用`events`提供的事件监听 `on` 以及 触发事件 `emit`
- koa的use方法返回this，因此可以实现链式调用，app.use().use().listen()....
- koa使用getter/setter代理nodejs原生的req和res对象，并做了一些简单的修改以简化API。
- ***组合函数，这是koa异步中间件的实现原理，也是koa的源码的核心部分***
- koa中间件的context
    + koa为了能够简化API，引入上下文context概念，将原始的nodejs请求对象req和响应对象res封装到context里面

#### getter/setter
koa使用getter/setter代理了原生的req和res对象
```js
let request = {
    req: { url: 'http://localhost:8080/' },
    get url(){
        return this.req.url
    },
    set url(val){
        this.req.url = val
    }
}
```

#### 中间件机制
koa中间件机制就是函数组合的概念。将一组需要顺序执行的函数复合为一个函数，外层函数的参数实际是内层函数
的返回值。洋葱圈模型可以形象表示这种机制，是koa源码中的精髓和难点。


#### 函数组合
##### demo-01 简单的compose函数
```js
function add(x, y){
    return x + y
}

function square(z){
    return z * z
}

function double(x){
    return x * 2
}

function compose(fn1, fn2, fn3){
    return (...arg) => fn3(fn2(fn1(...arg)))
    
}
const ret = compose(add, square, double)
console.log(ret(1,2));
console.log(ret(3,4));
```
上面的复合函数compose接收三个参数，然后组合为一个函数。如果参数数量不固定，那可咋整？

##### demo-02 使用reduce复合多个函数
```js
function add(x, y){
    return x + y
}

function square(z){
    return z * z
}

function double(x){
    return x * 2
}

function compose(middlewares){
    return (...arg) => {
        let result;
        for(let i = 0; i < middlewares.length; i++){
            const mid = middlewares[i];
            if(i === 0){
                result = mid(...arg)
            } else {
                result = mid(result)
            }
        }
        return result;
    }
}
const ret = compose(add, square, double)
console.log(ret(1,2));
console.log(ret(3,4));
```

用reduce改写一下上面的compose，reduce的返回值应当与middlewares第一个函数参数格式相同，如 (...args) => return xxx
```js
// 错误的写法。。。。。
//function compose(middlewares){
//    return (...args) => middlewares.reduce((resultFn, curMid) => { return curMid(resultFn(...args))})
//}

// 正确的写法，一般以数组第一项为例参考着写
function compose(middlewares){
    return middlewares.reduce((resultMid, curMid) => {
        return (...arg) => curMid(resultMid(...arg))
    })
}
```
上面的compose此时可以接收任意多个中间件，不过，很遗憾，这些中间件都是同步的，异步该如何处理？

##### demo-03 异步中间件
同样的，先从简单的demo实现，然后归纳总结
```js
let mid1 = async (next) => {
    console.log('mid1....in')
    await next();
    console.log('mid1...out')
}

let mid2 = async (next) => {
    console.log('mid2....in')
    await next();
    console.log('mid2...out')
}

function compose(mid1, mid2){
    return mid1(function next(){
        return mid2(function next(){})
    })
}

let ret = compose(mid1, mid2)
```

让我们再来拓展一下这个复合函数，使得每一个中间件都能够接收一个ctx参数
```js
let mid1 = async (ctx, next) => {
    console.log('mid1....in')
    ctx.body = 'mid1.body'
    await next();
    console.log('mid1...out')
}

let mid2 = async (ctx, next) => {
    console.log('mid2....in', ctx.body)
    ctx.body = 'mid2.body'
    await next();
    console.log('mid2...out')
}
function compose(mid1, mid2){
    return ctx => {
        return mid1(ctx, function next(){
            mid2(ctx, function next(){})
        })
    }
}
let context = {}
let ret = compose(mid1, mid2)
ret(context).then(res => {
    console.log('res..', res)
    console.log('context..', context)
})
```
emmmm，看上去很ok，那么问题又来了，现在只能组合两个中间件，怎么组合一组数量不固定的中间件呢？即compose接受一个数组
```js
let mid1 = async (ctx, next) => {
    console.log('mid1....in')
    ctx.body = 'mid1.body'
    await next();
    console.log('mid1...out')
}

let mid2 = async (ctx, next) => {
    console.log('mid2....in', ctx.body)
    ctx.body = 'mid2.body'
    await next();
    console.log('mid2...out')
}

let mid3 = async (ctx, next) => {
    console.log('mid3....in')
    await next()
    console.log('mid3....out')
}

function compose(middlewares){
    
}
```

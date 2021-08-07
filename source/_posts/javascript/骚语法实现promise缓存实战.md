---
title: 骚语法实现promise缓存实战
date: 2021-08-07 12:51
---

![20210807125031](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c23b70e3485f40ce8a0cb6a7330bcc8a~tplv-k3u1fbpfcp-zoom-1.image)


## 前言

ES6极大的提高了我们的编程体验，除了简单的 await 等待调用链路的结果，善用某些技能还能提高编程体验

## Promise 的 sleep 技术

在es5， 我们做延迟休眠往往需要用到 `setTimeout` 并增加回调钩子，来处理延时动作， 比如, 点击按钮后2秒后弹出对话框，我们可以会这么写
``` ts
// <button onclick="handleclick()"></button>

function handleclick(){
    setTimeout(()=>{
        openDialog();
    });
}

function openDialog(){
    // 处理弹框以及弹框后面的交互逻辑
}
```

代码会有很多回调钩子，不是很直观，我们可以巧用 `promise` 来解决这个问题

``` ts
// <button onclick="handleclick()"></button>

async function sleep(ms){
    return new Promise(resolve=>{
        setTimeout(resolve, ms);
    });

    // 可以直接简写成 const sleep = ms => new Promise(resolve => setTimeout(resolve, ms));
}

async function handleclick(){
    await sleep(2000); 
    // 处理弹框以及弹框后面的交互逻辑
}

```

## 缓存HTTP结果

假定我们有一个 `api/member` 接口，我们希望接口数据能被缓存起来，这样在程序多个地方调用的时候，不会重复的发送请求，一般我们可能会这么写

``` ts
let CACHE_MEMBER_INFO = undefined;
export default async function getMemberInfo(){
    if(CACHE_MEMBER_INFO){
        return CACHE_MEMBER_INFO;
    }

    CACHE_MEMBER_INFO = await ajax.get('api/member');
    return CACHE_MEMBER_INFO;
}
```

上面代码看起来并没有多大问题。但是由于 `ajax.get` 是异步，那就意味着，在没有缓存的时候，同时发起了两个 `getMemberInfo()` 调用，仍然会发出两次 `ajax` 的请求，这明显是多余的，我们希望同一时刻，我们的 `ajax` 只会有一个，如果有多个请求，应该复用这个 `ajax` 等待结果。

    应该复用这个 `ajax` 等待结果 

仔细理解，其实我们真真的目的并不是缓存 `MEMBER_INFO` 而是 `AJAX(Promise)` 啊，所以上面的代码应该改成

``` ts
let CACHE_MEMBER_INFO = undefined;
export default async function getMemberInfo(){
    if(CACHE_MEMBER_INFO){
        return CACHE_MEMBER_INFO;
    }

    CACHE_MEMBER_INFO = ajax.get('api/member');
    return CACHE_MEMBER_INFO;
}
```
注意，这里拿掉了 `await`， 直接将 ajax.get 返回，这意味着我们在函数内部不需要等待返回值结果，而是把整个调用过程当成缓存给返回出去，可以用下面这段代码进行测试

``` ts
let count = 0;
const ajax = {
    // 模拟两秒后返回
    async get(){
        await new Promise(resolve => setTimeout(resolve, 2000));
        return ++count;
    }

}

let CACHE_MEMBER_INFO = undefined;
async function getMemberInfo(){
    if(CACHE_MEMBER_INFO){
        return CACHE_MEMBER_INFO;
    }

    CACHE_MEMBER_INFO = ajax.get('api/member');
    return CACHE_MEMBER_INFO;
}


;(async ()=>{
    console.info(new Date(), await getMemberInfo());
    
})();

;(async ()=>{
    console.info(new Date(), await getMemberInfo());
    await new Promise(resolve=> setTimeout(resolve, 5000));
    console.info(new Date(), await getMemberInfo());
})();
// Sat Aug 07 2021 12:55:09 GMT+0800 (中国标准时间) 1
// VM107:28 Sat Aug 07 2021 12:55:09 GMT+0800 (中国标准时间) 1
// VM107:30 Sat Aug 07 2021 12:55:16 GMT+0800 (中国标准时间) 1
```

两个闭包函数模拟同时调用，可以看到，打印的时间是完全一致的，打印的结果也一样，也就是说 `ajax.get` 仅被执行了一次，使用起来非常顺手！上面的代码还可以改成

``` ts
let CACHE_MEMBER_INFO = undefined;
async function getMemberInfo(){
    // 赋值语句返回本身，相当
    //      a = 1;
    //      return  a;
    // 等价于
    //      return a = 1;
    return CACHE_MEMBER_INFO = CACHE_MEMBER_INFO || ajax.get('api/member')
        .catch(err=>{
            CACHE_MEMBER_INFO = undefined; // 报错时不缓存，但是仍然需要吧错误信息给外部
            throw err;
        });
}
```

那么我们在进阶一下，将缓存包装成一个高阶方法

``` ts
type fn = (...args: any) => Promise<any>;
function cachePromiseResult(PromiseFn: fn) {
    let cache = undefined;
    return function (...args) {
        return (cache =
            cache ||
            PromiseFn.apply(this, args).catch((error) => {
                cache = undefined;
                return Promise.reject(error);
            }));
    };
}

const getMemberInfo = cachePromiseResult(async function(){
    return ajax.get('api/member');
});
```

cachePromiseResult 会将这个函数的结果缓存起来，同时对异常情况不做缓存，入参方法像平常一样写函数的调用流程就行了，so easy! 如此一来，这个方法可以应用到任何想使用的地方，注意，这里缓存的是 `Promise` ，即意味着任何一个异步动作的结果都是可以被缓存的，比如

``` ts
const init = cachePromiseResult(async function(){
    showLoading();
    await sleep(2000);
    hideLoading();

    // 产品说初始化的时候让他等个两秒钟，虽然啥也不干，滑稽
    return '初始化成功'
});
```
![20210807124803](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a9c1374e975349628a9258724abbe8d6~tplv-k3u1fbpfcp-zoom-1.image)

好了，可以开始去给小伙伴们装B了！

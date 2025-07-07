---
title: Promise
draft: false
tags:
  - 前端
date: 2024-06-12
---
> 😀 Promise是什么？他的出现能解决什么问题？

`Promise` 是 `js` 中用于处理异步操作的一种机制，他提供了一种更清晰更简洁的方式处理异步任务和回调函数，避免了传统回调函数的处理方式所产生的‘回调地狱’问题。

- 回调地狱问题复现
    
    ```jsx
    function fun1(name) {
    	setTimeout(function(name) {
    		console.log(name + '函数执行')
    		if(Math.random() < 0.3) {
    			return 'success'
    		} else {
    			return 'false'
    		}
    	},1000)
    }
    
    funtion run() {
    	const res1 = fun1('张三')
    	if(res1 === 'success') {
    		const res2 = fun1('李三')
    		if(res2 === 'success') {
    			const res3 = fun1('王五')
    			if(res3 === 'success') {
    				const res4 = fun1('赵六')
    			}
    		}
    	}
    }
    
    ```
    

# 📝 序章 Js事件循环

## 浏览器进程模型

**浏览器是一个多进程多线程的应用程序**

主要进程有：

- 浏览器进程
    - 负责页面显示（导航栏返回前进）、用户交互、子进程管理
- 网络进程
    - 负责加载网络资源
- **渲染进程（重点）**
    - 负责执行 html 、css 、 js 代码
    - 默认情况下：浏览器会为每个标签页开启一个新的渲染进程，保证不同标签页之间不受影响

## 浏览器渲染主线程是如何工作的

1. 最开始时渲染主线程会进入一个无限循环
2. 每次循环时都会检查消息队列中是否有任务存在，如果有就取出第一个任务执行，执行完进入下次循环，如果没有任务则休眠
3. 其他所有的线程都往消息队列中丢任务，任务放到队列中的末尾。在丢任务的时候如果主线程休眠则会唤醒主线程

**整个过程被称为事件循环，也被称为消息循环**

![[Promise-1751871661727.png]]





## 异步

代码执行过程中，会遇到一些无法立即处理的任务

- 定时器
- 网络通信后执行的任务
- 时间监听回调
![[Promise-1751871698597.png|674x288]]


![[Promise-1751871735488.png|653x282]]

## 任务有优先级吗？

任务没有优先级，消息队列中先进先出

但是**消息队列是有优先级的**

> w3c 做以下解释：

- 每个任务都有一个任务类型，同一类型的任务必须在一个队列，不同类型的任务可以在同一个队列
- 浏览器必须准备一个**微队列**，**微队列中的任务优先于其他所有任务执行**

> chrom 中至少包含了以下队列

- 延时队列：计时器的回调任务
- 交互队列：页面交互各种事件
- 微队列：优先级最高

# 🤗 Promise 基础

### Promise 规范

Promise 是在 [EcmScript 规范](https://tc39.es/ecma262/multipage/control-abstraction-objects.html#sec-promise-objects)下催生演变而来。规范中指出：

- 所有的异步场景，都可以看作是一个异步任务，每个异步任务在js中应该表现为一个对象，该对象称之为 `Promise` 对象，也叫做任务对象
    
- 每个对象都应该有两个阶段(未决、已决)、三个状态（等待pending、成功fulfilled、失败rejec）
    
    ![[Promise-1751871766265.png]]
    
- 挂起→完成，称之为 `resolve` 。 挂起→失败，称之为 `reject` 。任务完成时可能会有一个相关数据，任务失败时可能会有失败原因。
    
- 可以针对任务进行后续处理，针对成功的任务后续处理称为 `onFulfilled`,针对失败的后续处理称之为 `onRecjected`
    
    ![[Promise-1751871773226.png]]
    

### Promise API

根据 Promise 标准 es6 提供了一套实现 api

```jsx
const p1 = new Promise(function(resolve, reject) {
	// todo
	const success= 
	if (success) {
		resolve('成功结果')
	} else {
		reject('失败原因')
	}
	
})

// 使用
p1.then(function(data) {
	console.log('成功处理函数')
}, function(err) {
	console.log('失败处理函数')
})

p1.catch(function(msg) {
	console.log('失败处理函数')
})
```

# 📎 Promise 链式调用

日常工作开发中会遇到这么一种情况，一个异步任务需要根据另一个异步任务的执行结果来运行，举个例子：当用户上传文件成功后保存操作日志。

Promise 的 then 方法会返回一个新的 Promise，可以理解为一个新的任务。

![[Promise-1751871792918.png]]

- 新任务的状态取决于上一个任务的状态
    
    - 上一个任务成功，新任务执行then方法，then方法成功则成功
    - 上一个任务失败，新任务执行catch方法，新任务失败 rejected
    - 上一个任务 pendding 状态（还没处理完），新任务不会执行
- 练习题 1
    
    ```jsx
    const p1 = new Promise((resolve, reject) => {
      setTimeout(() => {
        resolve(1)
      }, 1000)
    })
    
    const p2 = p1.then((data) => {
      console.log(data)
      return data + 1
    })
    
    const p3 = p2.then((data) => {
      console.log(data)
    })
    
    console.log(p1, p2, p3)  // pending,pending,pending
    
    setTimeout(() => {
      console.log(p1, p2, p3) // fulfilled(1) fulfilled(2) fulfilled(undefined) 
    }, 2000)
    
    ```
    
- 练习题 2
    
    ```jsx
    const p1 = new Promise((resolve, reject) => {
      setTimeout(() => {
        resolve(1)
      }, 1000)
    })
    
    const p2 = p1.catch((data) => {
      console.log(data)
      return data + 1
    })
    
    const p3 = p2.then((data) => {
      console.log(data)
    })
    
    console.log(p1, p2, p3) // pending,pending,pending
    
    setTimeout(() => {
      console.log(p1, p2, p3) // fulfilled(1) fulfilled(1) fulfilled(undefind)
    }, 2000)
    
    ```
    
- 练习题 3
    
    ```jsx
    const p1 = new Promise((resolve, reject) => {
      setTimeout(() => {
        resolve(1)
      }, 1000)
    })
    
    const p2 = p1.then((data) => {
      throw 3
    })
    
    const p3 = p2.then((data) => {
      console.log(data)
    })
    
    console.log(p1, p2, p3)  // pending,pending,pending
    
    setTimeout(() => {
      console.log(p1, p2, p3) //  fulfilled(1)  reject(3)  reject(3)
    }, 2000)
    
    ```
    
- 练习题 4
    
    ```jsx
    new Promise((resolve, reject) => {
      resolve()
    })
      .then((res) => {
        console.log(res.toString()) // undefined.toString() 报错了 rejected
        return 2
      })
      .catch((err) => {
        return 3 // 导致成功了 fulfilled  3
      })
      .then((res) => {
        console.log(res) // 3
      })
    
    ```
    

# 📐 Promise 静态方法

解决的问题：同步执行多个 promise 最终获得结果

|静态方法|作用|
|---|---|
|`Promise.resolve()`|返回一个成功任务|
|`Promise.rejcet()`|返回一个失败任务|
|`Promise.all(任务数组)`|1. 返回一个新的任务 2.所有的任务全部成功后才会成功，否则任务失败reject 3.返回的成功返回的数组|
|`Promise.any(任务数组)`|1. 返回一个新的任务 2.只要有一个成功，该任务就会成功（返回第一个成功的信息），全部任务失败才会失败（返回全部的失败原因）|
|`Promise.allSettled(任务数组)`|1. 返回一个新的任务 2.任务数组中全部已决（成功或失败）则新任务就成功。 3.如果任务数组中有挂起任务，则新任务为挂起|
|`Promise.race(任务数组)`|1.返回一个新任务。 2.任务数组中谁现有结果则新任务就是谁（无论成功或失败）|

# 🔬 async 和 await

es7 新语法，用于简化 promise 的复杂写法

## async

必须标注在函数前, 被他修饰的函数，返回值一定是个 promise

```jsx
async function test1() {
	return 1; // 返回值是promise(1)
}

console.log(test1())
```

## await

用于被 `async` 修饰的方法内部，且必须用于修饰 `promise` 方法，表明等待 promise 的返回结果.

举个例子，与回调方式的对比。

```jsx
async function test1() {
    console.log('123')
}

async function run1() {
    try {
        const result = await test1() // 调用
        console.log('成功')
    } catch (error) {
        console.log('失败')
    }
    
}

function oldRun() {
    test1().then(data =>{
        console.log('成功')
    }).catch(err=>{
        console.log('错误')
    })
}
```

# 🧪 练习

## 面试题1：如何理解 js 中的异步？

js 是一门单线程语言，这是因为它运行在浏览器的渲染主线程中，渲染主线程只有一个。

渲染主线程承担着诸多的任务，如：渲染页面、执行 js

如果采用同步的方式，则渲染主线程很可能卡死，影响页面渲染造成卡顿现象。

所以浏览器采用异步的方式来避免。具体做法就是当某些任务时，比如计时器、网络、事件监听、计时器等主线程会将该任务交给其他线程处理，自身结束处理该任务去执行其他任务。当其他线程完成时，会将事件的回调包装成一个任务加入到消息队列中，等待主线程消费。

这种异步模式下，浏览器不会阻塞，最大程度保障单线程的流畅运行。
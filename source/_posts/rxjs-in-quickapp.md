---
title: 在快应用中使用RxJS
date: 2020-05-09 14:52:18
tags: RxJS quickapp throttle debounce timeout
---

## RxJS介绍

Rx（ReactiveX）是一种用来管理事件序列的理想方法，提供了一套完整的API，它的设计思想组合了观察者模式，迭代器模式和函数式编程。响应式编程在各个编程语言中都有对应的实现，应用较为广泛的是RxJava以及RxJS。

RxJS 是基于ReactiveX实现的JavaScript版本的库，它使编写异步或基于回调的代码更容易。你可以把它看成是一个用于处理事件的`Lodash`。RxJS也是Angular强烈推荐的事件处理库。

要使用RxJS，先要了解其中的几个核心概念：

- **Observable (可观察对象):** 表示一个概念，这个概念是一个可调用的未来值或事件的集合。
- **Observer (观察者):** 一个回调函数的集合，它知道如何去监听由 Observable 提供的值。
- **Subscription (订阅):** 表示 Observable 的执行，主要用于取消 Observable 的执行。
- **Operators (操作符):** 采用函数式编程风格的纯函数 (pure function)，使用像 `map`、`filter`、`concat`、`flatMap` 等这样的操作符来处理集合。
- **Subject (主体):** 相当于 EventEmitter，并且是将值或事件多路推送给多个 Observer 的唯一方式。
- **Schedulers (调度器):** 用来控制并发并且是中央集权的调度员，允许我们在发生计算时进行协调，例如 `setTimeout` 或 `requestAnimationFrame` 或其他。

上面的描述可能比较抽象，举一个类比现实生活的例子来帮助理解这几个概念：购房者一直在密切的关注房价，而房价随时间波动，购房者可能会根据波动的房价而采取一系列的行动，比如购入或者继续观望。购房者与房价的这样一种关系其实就构成了一种观察者关系。这里，购房者担任观察者的角色，房价是被观察的角色，当房价信息发生变化，则自动推送信息给购房者。

- 房价即为 Observable 对象；
- 购房者即为 Observer 对象；
- 而购房者观察房价即为 Subscribe（订阅）关系；

如果理解了这个场景，那么就大概理解了RxJS的基础概念，如果你没接触过需要更详细了解，可以看看这篇文章[响应式编程入门](https://hijiangtao.github.io/2020/01/13/RxJS-Introduction-and-Actions/)。这里就不做过多展开了，文章后面会列举一些RxJS的相关文档和工具，有兴趣的可以自行探索和学习。下面就直接进入结合快应用的使用方法了。

注意，本文示例均使用RxJS6.5版本编写。

## 简单示例

### 安装

```shell
npm install rxjs --save # npm安装
yarn add rxjs  # yarn安装
```

### 导入

```js
import { Observable } from 'rxjs'; 
```

### 使用

```js
const foo = new Observable(subscriber => {
                console.log('Hello');
                subscriber.next(42);
            });
foo.subscribe(x => {
    console.log(x);
});
// output:  Hello 42
```

## 实践示例

### 节流（throttle）和防抖（debounce）

#### 节流的处理

我们开发快应用时会遇到一些情况，比如点击一个按钮或，请求一个网络接口（或者一些其他异步操作），由于有些网络接口对请求频率有限制（或者有些异步操作很消耗性能），如果用户快速多次点击按钮，会短时间触发多个请求，很可能导致接口拒绝返回数据（或者降低设备运行效率），这不是我们期望的行为，这时我们就需要对按钮的点击做限流或是防抖处理。

下面是没有做处理的代码：

```html
<template>
  <div class="demo-page">
    <text class="title">按钮点击次数{{count}}</text>
    <input class="btn" type="button" value="按钮" onclick="clickHandler" />
  </div>
</template>
<script>
import fetch from '@system.fetch'
export default {
  data() {
    return {
      count: 0
    }
  },
  requestData() {
    return fetch.fetch({url:'https://api.github.com/users?per_page=5'})
  },
  clickHandler () {
    this.requestData().then(res=> {
      if (res) {
        this.count++;
      }
    })
  }
}
</script>
<style>
  .demo-page {
    flex-direction: column;
    justify-content: center;
    align-items: center;
  }

  .title {
    font-size: 40px;
    text-align: center;
  }

  .btn {
    width: 550px;
    height: 86px;
    margin-top: 75px;
    border-radius: 43px;
    background-color: #09ba07;
    font-size: 30px;
    color: #ffffff;
  }
</style>
```

很显然，由于没有对点击事件做限制，每次点击都会触发一次请求，这不是我预期的效果，通常我们的做法一般是增加一个参数用于保存上次点击时间，再根据这个参数来判断当前点击事件时间是否小于一定间隔来判断对应的逻辑是否执行。这种方式增加了额外的判断逻辑，也不是那么优雅，如果采用RxJS的方式，我们可以怎么做呢？下面是修改后的代码。

```html
<template>
  <div class="demo-page">
    <text class="title">按钮点击次数{{count}}</text>
    <input id="button" class="btn" type="button" value="按钮"/>
  </div>
</template>
<script>
  import fetch from '@system.fetch'
  import {fromEvent} from 'rxjs'
  import {throttleTime} from 'rxjs/operators'
  export default {
    data() {
      return {
        count: 0
      }
    },
    onShow() {
      const button = this.$element('button') // 获取按钮的DOM
      const observable = fromEvent(button, 'click') // 根据按钮点击事件创建可订阅流
      const throttleButton = observable.pipe(throttleTime(1000)) // 为可订阅流增加限制1秒的触发间隔
      const subscribe = throttleButton.subscribe(e => { // 订阅流执行对应的逻辑
        console.log(`button clicked: ${JSON.stringify(e)}`)
        this.requestData().then(res => {
          if (res) {
            this.count++
          }
        })
      })
    },
    requestData() {
      return fetch.fetch({url: 'https://api.github.com/users?per_page=5'})
    },
  }
</script>
```

可以看到，不管我们以多快的速度点击按钮，现在按钮点击事件被节流到每秒只能触发一次了。

![节流效果](https://quickapp.vivo.com.cn/content/images/2020/05/rxjs-quickapp-d.gif)

#### 防抖的处理

我们在开发应用的时候会遇到搜索框联想的需求，一般来说，我们会监听输入框的change事件来执行请求接口等逻辑，但是如果每次change都触发一次请求，会出现用户还没输入完成就开始提示，请求一般都是异步，会出现联想提示频繁变更，不是用户想要得情况，最好处理方式就是在一段时间内，用户的输入不再继续了，我们就触发对应的数据请求及联想更新逻辑。这里就需要用到事件的防抖了。下面是没有做防抖处理的代码：

```html
<template>
  <div class="wrap">
    <input id="input" class="input" placeholder="请输入搜索词" type="text" onchange="changeHandler"/>
    <text for="text">{{$item}}</text>
  </div>
</template>

<script>
  export default {
    data() {
      return  {
        text: []
      }
    },
    changeHandler(e) {
      this.text.push(e.value)
    },
  }
</script>

<style>
  .wrap {
    flex-direction: column;
    align-items: center;
  }
  .input {
    height: 80px;
    width: 700px;
    border: 1px solid #8d8d8d;
    border-radius: 80px;
    padding: 0 40px;
    margin-top: 40px;
    background-color: #f0f0f0;
  }
</style>
```

运行的效果是这样的：

![无防抖效果](https://quickapp.vivo.com.cn/content/images/2020/05/rxjs-quickapp-b.gif)

显然效果是不符合我们预期的，下面用RxJS的方式为它加上防抖：

```html
<template>
  <div class="wrap">
    <input id="input" class="input" placeholder="请输入搜索词" type="text"/>
    <text for="text">{{$item}}</text>
  </div>
</template>

<script>
  import {fromEvent} from 'rxjs'
  import {debounceTime} from 'rxjs/operators'
  export default {
    data() {
      return  {
        text: []
      }
    },
    onShow() {
      const input = this.$element('input') // 获取input的DOM
      const observable = fromEvent(input, 'change') // 根据输入框的change事件创建可订阅流
      const debouncedInput = observable.pipe(debounceTime(2000)) // 为可订阅流增加防抖2秒的时间间隔，2秒后没有变化则触发对应了处理逻辑
      const subscribe = debouncedInput.subscribe(e => { // 订阅流执行对应的逻辑
        console.log(`input changed: ${JSON.stringify(e)}`)
        this.text.push(e.value);
      })
    },
  }
</script>
```

现在的效果是这样的：

![防抖效果](https://quickapp.vivo.com.cn/content/images/2020/05/rxjs-quickapp-c.gif)

可以看到，防抖的效果已经达到了。

### 请求失败自动重试

我们在开发快应用的时候，发送请求是通过fetch接口，这个接口并没有提供超时和重试的机制，往往需要我们自行开发适配，这里我们采用RxJS来实现封装fetch接口，使其能够支持自动重试。

```js
import fetch from '@system.fetch'
import {throwError, of, defer} from 'rxjs'
import {retry, mergeMap} from 'rxjs/operators'

export function myFetch(params) {
  const retryNum = params.retry || 1 // 出错后重试的次数
  return new Promise((resolve) => { // 用promise封装使其支持常规async/await调用
    defer(() => fetch.fetch({...params})) // 使用defer操作符，确保每次重试都是新的请求
      .pipe(
        mergeMap((res) => {
          if (res.data.code !== 200) { // 判断接口状态码，不为200时重试，这里可以根据业务自定义
            return throwError(res.data)
          }
          return of(res)
        }),
        retry(retryNum),
      ).subscribe({
        next: val => resolve(val),
        error: val => resolve(val)
      })
  })
}
```

通过上面的封装，快应用的原生接口就实现了失败重试的能力。

#### 请求超时

通常，我们处理请求超时会采用settimeout的方式来实现，这里我们来试试如何用RxJS的方式来封装一个支持超时机制的请求接口。

```js
import fetch from '@system.fetch'
import {timeout} from 'rxjs/operators'

export function myFetch(params) {
  const TIMEOUT = params.timeout || 2000
  return new Promise((resolve) => {
    defer(() => fetch.fetch({...params}))
      .pipe(
        timeout(TIMEOUT), // 超过设定时间未返回值抛出超时错误
      ).subscribe({
        next: val => resolve(val),
        error: val => resolve(val)
      })
  })
}
```

可以看出，超时机制的封装十分简洁，代码也更加优雅。

## 技术总结

RxJS作为一个擅长处理事件的库，函数式编程使得代码更加优雅，在需要处理多个事件并发的时候，能够显现出其强大的优势，本文中只使用了少部分的操作符，就能将繁琐的操作变得更加简洁。

## 参考文档

- [ReactiveX官网](http://reactivex.io/)
- [RxJS文档](https://rxjs-dev.firebaseapp.com/)
- [学习RxJS操作符](https://www.learnrxjs.io/)
- [响应式编程入门](https://hijiangtao.github.io/2020/01/13/RxJS-Introduction-and-Actions/)
- [响应式编程介绍](https://gist.github.com/staltz/868e7e9bc2a7b8c1f754)--André Staltz
- [学习RxJS的超直观交互图](https://medium.com/angular-in-depth/learn-to-combine-rxjs-sequences-with-super-intuitive-interactive-diagrams-20fce8e6511)--Max Koretskyi
- [RxJS珠宝图在线演示](https://rxmarbles.com/)

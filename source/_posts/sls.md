---
title: 搭建以 serverless 为后台服务的疫情热搜快应用
date: 2020-03-20 17:52:18
tags: serverless quickapp backend scf tencent
---
## 起源

今年疫情的影响越来越大，已经成为一个世界性的问题，疫情的发展时刻牵动每个人的心，正好也是因为疫情，今年让作为加班狗的我突然重温“放寒假”的感觉。宅在家里太久就想搞点事情做，于是就萌发了搞个疫情热搜应用的念头。说干就干，经过两天构思，两天开发，踩了不少坑之后，一个疫情热搜快应用就诞生了。

## 构思

### 先说技术点

后端：nodejs  puppeteer cheerio

前端：快应用（当然小程序也没问题）

### 再说说采用这几个技术的原因

- nodejs：本身呢，我作为一个前端，用这个写服务端是很合情合理的吧！
- puppeteer：为什么选择这个库呢？首先当然是为了爬取数据，那么有的小朋友就要说了，爬取数据还有其他的库呀?为什么非要用他呢？没错，一开始我用的是[crawler](https://www.npmjs.com/package/crawler)，然而这个库并不能爬取单页应用，这是踩的第一个坑，后面会详细讲。然后就选择了[puppeteer](https://www.npmjs.com/package/puppeteer)，他是谷歌官方出品的一个通过 DevTools 协议控制 headless Chrome 的 Node 库，浏览器可以的，他都可以，爬取单页应用自然不在话下。
- cheerio：为服务端设计的轻量级 jQuery 核心实现，用来过滤选取爬取到的页面数据的。
- 快应用：作为小程序家族里的一个特例，唯一个用原生组件渲染的框架，性能自然比其他小程序不知高到哪里去了，只要是在国内厂商生产的安卓机器，基本都能运行。

### 然后说说 serverless

serverless 技术的诞生，让开发者可以更加专注于业务，而不必考虑系统的运维和系统性能的伸缩，以往我们要开发部署一个应用，一般需要准备一台服务器，配置好对应的项目环境，部署好对应的项目。这个过程中，需要注意的环节很多，一个地方出问题，就会导致整个应用不可用。而通过 serverless 架构，我们只需要把核心代码上传到服务提供商，然后就啥都不用管了，应用遵循运行才计费的原则，还可以自动拓展，不用担心流量突然增大导致服务不可用。很适合不熟悉运维操作的开发人员部署自己的项目。（~~当然我肯定不会说是因为国内函数计算提供商现在都有免费的额度可以白嫖的~~）

### 最后说说整个项目的架构和实现方法

- 通过 nodejs 加 puppeteer 抓取解析百度疫情热搜数据
- 把项目部署到函数计算服务提供商平台（这里我采用的是腾讯云的 SCF，免费额度和阿里的函数计算一样）
- 通过配置 API 网关，把服务暴露出来
- 开发一个快应用调用服务展示数据

## 实践

说完了技术架构和构思，下面正式开始介绍开发实践的过程：

### 准备开发环境

这里以腾讯云的 SCF 服务为例，其他云平台其实也都大同小异。

1. 安装腾讯云 SCF-CLI 命令行工具

```bash
pip install scf   # 这个工具是python写的，所以需要开发机器有python环境，且版本需要在python2.7以上
```

2. 安装腾讯云 serverless 的 vscode 插件

   如果你懒得配置 python 环境，比较喜欢可视化的操作，可以选择安装 vscode 的插件

   插件市场搜索安装 Tencent Serverless Toolkit for VS Code

以上两个工具任选一种安装好了就行。

### 初始化项目

用 SCF 命令行方式初始化一个项目，vscode 插件的方式就不说了，可视化操作按提示操作就行。

```bash
scf init -r nodejs8.9 --name virus-search  # 初始化一个项目名为virus-search的项目，运行环境是nodejs8.9
```

初始化好了项目，项目结构是下面这个样子的：

```
└── virus-search
    ├── README.md
    ├── index.js // 入口文件
    └── template.yaml // 项目配置文件
```

截止到我写这篇文章为止，不管是 SCF 命令行还是 vscode 插件，创建项目的 nodejs 版本最高均只支持到了 8.9。这里特别拎出来说是因为腾讯云实际上已经支持了 node10.15 的运行环境，不过开发工具还没开放。

### 安装项目依赖

接下来安装要用到的项目依赖

```shell
npm install puppeteer cheerio --save
```

pupeteer 会安装 chromium，这个包有 130+MB，建议把 npm 换成 cnpm 或者替换淘宝源，这样会快很多。

安装好了依赖，项目结构变成了这样

```bash
└── virus-search
    ├── README.md
    ├── node_modules/
    ├── package.json
    ├── index.js // 入口文件
    └── template.yaml // 函数配置文件
```

### 编写爬虫逻辑

这里数据来源我选择了百度的[疫情全民热搜](https://voice.baidu.com/act/virussearch/virussearch/?from=osari_map&tab=0)，这个页面长这个样子：

![baidu-virus-hot-search](https://quickapp.vivo.com.cn/content/images/2020/03/baidu-virus-hot-search.png)

我想要的数据就是这个页面热搜榜单了，在 Chrome 中打开，用 devtools 查看页面的结构:

![baidu-virus-hot-search-element](https://quickapp.vivo.com.cn/content/images/2020/03/baidu-virus-hot-search-element.png)

简单分析一下页面元素再结合 network 里面请求的情况，可以看出这是个 react 写的单页应用。这里再说回为什么用了 puppeteer 这个库，一开始用了 crawler，爬下来发现页面是一堆 js，没法解析里面的元素和数据，所以换了 puppeteer。用 puppeteer 爬取页面的代码长这样：

```js
const puppeteer = require('puppeteer');

async function getPage() {
  const browser = await puppeteer.launch({args: ['--no-sandbox']});
  const page = await browser.newPage();
  await page.goto('https://voice.baidu.com/act/virussearch/virussearch?from=osari_map&tab=0&infomore=1');
  const content = await page.content();
  console.log('page content', content);
  await browser.close();
};
```

方法执行之后，可以看到 content 输出的就是我们在 devtools 的 element 里面看到的一致的内容了。接下来我们需要解析过滤页面的数据。这里我使用的是[cheerio](https://github.com/cheeriojs/cheerio)，这个库是 Fast, flexible, and lean implementation of core jQuery designed specifically for the server.结合 puppeteer 的使用代码如下：

```js
const puppeteer = require('puppeteer');
const cheerio = require('cheerio');

async function getPage() {
  const browser = await puppeteer.launch({args: ['--no-sandbox']});
  const page = await browser.newPage();
  await page.goto('https://voice.baidu.com/act/virussearch/virussearch?from=osari_map&tab=0&infomore=1');
  const content = await page.content(); // 获取页面的HTML
  const $ = cheerio.load(content); // 把获取到的页面HTML加载进cheerio

  const list = []; // 保存过滤出来的数据
  $('#ptab-0 .VirusHot_1-5-5_32AY4F').each((idx, elem) => { // 遍历过滤数据
    const arr = [];
    $(elem).find('a').each((idx, item) => {
      const title = $(item).find('span.VirusHot_1-5-4_24HB43').contents().filter((idx, content) => {return content.nodeType === 3;}).text();
      const rank = $(item).find('span.VirusHot_1-5-4_3BslNU').text();
      arr.push({url: $(item).attr('href'), title: title, rank: rank})
    });
    list.push({ category: $($(elem).children('header')[0]).text(), data: arr });
  })
  await browser.close();
  return list;
};
```

以上就是筛选过滤数据的全部代码，现在我们再接入 serverless 的代码。完整的 index.js 是这样的：

```javascript
const puppeteer = require('puppeteer');
const cheerio = require('cheerio');

async function getPage() {
  const browser = await puppeteer.launch({args: ['--no-sandbox']});
  const page = await browser.newPage();
  await page.goto('https://voice.baidu.com/act/virussearch/virussearch?from=osari_map&tab=0&infomore=1');
  const content = await page.content();
  const $ = cheerio.load(content);

  const list = []
  $('#ptab-0 .VirusHot_1-5-5_32AY4F').each((idx, elem) => {
    const arr = [];
    $(elem).find('a').each((idx, item) => {
      const title = $(item).find('span.VirusHot_1-5-5_24HB43').contents().filter((idx, content) => {return content.nodeType === 3;}).text();
      const rank = $(item).find('span.VirusHot_1-5-5_3BslNU').text();
      arr.push({url: $(item).attr('href'), title: title, rank: rank})
    });
    list.push({ category: $($(elem).children('header')[0]).text(), data: arr });
  })
  await browser.close();
  return list;
};

exports.main_handler = async (event, context, callback) => {
    console.log("%j", event);
    const list = await getData();
    return list;
};
```

现在可以试着在本地调试一下代码运行的情况了。

```shell
scf native invoke --no-event  // 本地测试函数运行
```

发现控制台输出了错误：

![scf-native-error](https://quickapp.vivo.com.cn/content/images/2020/03/scf-native-error.png)

看来是执行超时了，需要调整一下函数的相关配置，这个配置在`template.yaml`文件中。我们修改一下默认的配置：

```diff
Resources:
  default:
    Type: TencentCloud::Serverless::Namespace
    virus-search:
      Type: TencentCloud::Serverless::Function
      Properties:
        CodeUri: ./
        Type: Event
        Description: This is a template function
        Environment:
          Variables:
            ENV_FIRST: env1
            ENV_SECOND: env2
        Handler: index.main_handler
        MemorySize: 128
        Runtime: Nodejs8.9
        Timeout: 10 # 把函数超时时间修改为10秒
Globals:
  Function:
    Timeout: 10
```

现在再运行一遍：

![scf-native-success](https://quickapp.vivo.com.cn/content/images/2020/03/scf-native-success.png)

好了，本地函数跑通了，数据也正常返回了。现在可以把函数部署到远程了。

### 部署函数

配置腾讯云账号

![scf-config-set](https://quickapp.vivo.com.cn/content/images/2020/03/scf-config-set.png)

上传部署到远程

```bash
$ scf deploy 
Package name: default-virus-search-latest.zip, package size: 130 mb
...
[o] Deploy function 'virus-search' success
[o] Deploy trigger 'api' success
[+] Function Base Information: 
  Name: virus-search
  ...
[+] Trigger Information: 
 > APIGW - virus-search_apigw:
    ModTime: 2020-03-01 12:01:13
    Type: apigw
   ...
      service:
        serviceId: service-qnwxxxxxx
        serviceName: SCF_API_SERVICE
        subDomain: https://service-qnw3irqg-xxxxxxxxxxx.gz.apigw.tencentcs.com/release/virus-search
    ...
[o] Deploy success
```

这里我们会发现 SCF 会打包函数和相关依赖，然后帮你上传。可以看到这个函数包括依赖有 130+mb 大小，上传会花费很长时间，你可以开启 COS 上传来加速这个过程，但是实际体验还是让我等待了很长时间，腾讯云目前在内测在线安装依赖的能力，后面应该会开放出来，这样可以极大提升上传部署的体验。

然后我们测试一下线上的函数运行情况，这里我踩了一堆坑，花费了几倍代码开发的时间才爬出来，就不具体描述过程了，把上传之后的坑列在下面，并给出解决的方案：

1. 第一坑就是上传之后，运行发现内存不够的情况导致执行失败。这个问题在我本地测试是没有发现的，SCF 本地运行显示使用内存才 50+MB，解决办法是修改函数执行的运行环境配置，上配置：

![scf-runtime-config](https://quickapp.vivo.com.cn/content/images/2020/03/scf-runtime-config.png)

1. 第二坑就是发现我们 `template.yaml` 里面的配置的 nodejs 运行版本是 8.9，这个会导致 puppeteer 跑不起来，需要很多额外的配置，具体可以参考这个文章[在 SCF 中运行 Puppeteer](https://cloud.tencent.com/developer/article/1410471)，但是这个配置实在是太蛋疼了，且不说各种安装依赖，安装完了还会导致函数包变得更大，每次上传等待时间都让人很无语，而且腾讯的这个上传函数包还没进度条，这里要吐槽一下，只能傻等。所以我查了 puppeteer 的文档，puppeteer 在 node10 以上版本，可以不需要安装这些依赖，所以决定修改 node 运行环境来解决，但是发现腾讯的 SCF 和 vscode 插件都不支持 nodejs10.15 版本的项目上传，会直接报错，不过可以在网页直接创建 nodejs10.15 的项目，这里也是要吐槽的。

   - 网页创建 nodejs10.15 的函数项目

     ![scf-web-create](https://quickapp.vivo.com.cn/content/images/2020/03/scf-web-create.png)
   - 选择在网页本地上传代码包

     ![scf-web-upload](https://quickapp.vivo.com.cn/content/images/2020/03/scf-web-upload.png)
   - 重新打包本地的函数项目包，这一步很重要，由于腾讯云 nodejs10.15 环境自带了 puppeteer 的环境，所以我们本地项目 node_modules 里面不需要再安装了，这样使项目包大小极大减小，实测从 130+MB 减小到不到 1Mb 了，我也是服了，删除 node_modules 的 puppeteer 依赖后打包，然后重新上传。速度飞快！┗ |｀ O′|┛ 嗷 ~~

   以上一波坑踩完，按之前的环境配置修改好新建的函数配置环境，在线测试运行函数：

   ![scf-web-success](https://quickapp.vivo.com.cn/content/images/2020/03/scf-web-success.png)

   终于成功了，简直喜极而泣！！！

### 配置 API 服务

函数在线测试成功了之后，我们要把服务通过 API 暴露出来让其他端侧调用。这个的配置就简单了许多，直接在网页上点点点，配置就好了。

![scf-web-create-api](https://quickapp.vivo.com.cn/content/images/2020/03/scf-web-create-api.png)

然后到腾讯云的 API 网关管理页面就可以看到上面创建的 API 服务了

![scf-api-manage](https://quickapp.vivo.com.cn/content/images/2020/03/scf-api-manage.png)

现在我们开发的这个函数，从外网访问地址就是 `API服务默认域名+函数名`

以上，我们后端的服务算是配置完成了，如果你有自己的域名，也可以通过自定义域名绑定来实现公网域名的修改。

## 开发快应用

有了服务端的数据，现在可以考虑快应用中的展示了。如果你不熟悉快应用的开发可以先看下[快应用官方文档](https://doc.quickapp.cn)来了解一下，如果你对快应用的开发感兴趣，可以试试[apex-ui](https://github.com/vivoquickapp/apex-ui)这个快应用组件库，帮你快速开发一个快应用，这里我就不对开发做细节的展示了，直接上页面代码：

```vue
<template>
  <div class="wrap">
    <div class="cover">
      <text>疫情全民热搜</text>
    </div>
    <div class="tabs">
      <text for="{{list}}" class="{{active === $idx? 'active' : ''}}" onclick="gotoIndex($idx)">{{$item.category}}</text>
    </div>
    <list class="list" id="list">
      <list-item class="module" for="{{(index, item) in list}}" type="module">
        <text class="category" onappear="appearHandler(index)">{{item.category}}</text>
        <div class="content {{$idx? 'bt': ''}}" for="{{title in item.data}}" onclick="{{routeDetail(title.url)}}">
          <div>
            <text class="index top-{{$idx+1}}">{{$idx+1}}</text>
            <text class="rumor" show="{{index === 3}}">谣言</text>
            <text class="title">{{title.title}}</text>
            <text class="rank">{{title.rank}}</text>
          </div>
          <text class="hot" show="{{!$idx}}">热</text>
        </div>
      </list-item>
    </list>
  </div>
</template>

<script>
  import router from '@system.router'
  import fetch from '@system.fetch'

  export default {
    data() {
      return {
        active: 0,
        list: []
      }
    },
    async onInit() {
      this.list = await this.getListData();
    },
    getListData() {
      return new Promise((resolve, reject) => {
        fetch.fetch({
          url: 'http://your.domain.name/puppeteer'
        }).then((res)=> {
          console.log(res);
          resolve(JSON.parse(res.data.data));
        }).catch((err)=> {reject(err)})
      })
    },
    routeDetail(url) {
      router.push({
        uri: url
      })
    },
    gotoIndex(index) {
      this.$element('list').scrollTo({index: index})
      this.active = index
    },
    appearHandler(index) {
      this.active = index
    }
  }
</script>

<style lang="less">
  .wrap {
    flex-direction: column;

    .cover {
      height: 100px;
      width: 750px;
      background-color: #0ba6af;
      color: #ffffff;
      padding: 0 20px;

      text {
        color: #FFFFFF;
        font-size: 40px;
      }
    }

    .tabs {
      height: 100px;
      justify-content: space-around;
      align-items: center;

      text {
        font-weight: bold;
        height: 80px;
      }
    }

    .list {
      padding: 0 20px;

      .module {
        background: linear-gradient('#b5f2f3 0%', '#ffffff 20%');
        padding: 20px;
        border: 1px solid #DCDCDC;
        border-radius: 30px;
        margin-bottom: 20px;
        margin-top: 20px;
        flex-direction: column;

        .category {
          align-self: center;
          font-size: 40px;
          color: #000000;
          font-weight: bold;
          line-height: 80px;
        }

        .content {
          height: 80px;
          justify-content: space-between;

          .rumor {
            height: 24px;
            padding: 0 10px;
            margin-right: 10px;
            background-color: #ff1845;
            color: #FFFFFF;
            border-radius: 12px;
            font-size: 16px;
            align-self: center;
          }

          .hot {
            background-color: #ff792f;
            font-size: 20px;
            padding: 0 10px;
            height: 28px;
            line-height: 28px;
            border-radius: 14px;
            text-align: center;
            align-self: center;
            color: #FFFFFF;
          }

          .index {
            width: 50px;
            font-weight: bold;
          }

          .top-1 {
            color: red;
          }

          .top-2 {
            color: coral;
          }

          .top-3 {
            color: sandybrown;
          }

          .title {
            margin-right: 20px;
            color: #000000;
            font-weight: bold;
          }

          .rank {
            color: #9c9c9c;
            font-size: 20px;
          }
        }

        .content:active {
          background-color: rgba(11, 168, 175, 0.1);
        }
      }
    }
  }

  .bt {
    border-top: 1px solid #DCDCDC;
  }

  .active {
    border-bottom: 4px solid #0ba6af;
    color: #0ba6af;
  }
</style>
```

运行之后，项目的效果如下：

![serverless-quickapp](https://quickapp.vivo.com.cn/content/images/2020/03/serverless-quickapp.gif)

以上，在踩了诸多坑之后，终于完成了这样一个疫情热搜的快应用，下面开始技术总结。

## 技术总结

1. serverless 的 nodejs 运行环境需要选择 nodejs10 以上的版本，否则会有一堆依赖缺失导致在线函数跑不起来。
2. 上传函数时候需要去掉 puppeteer 的依赖，不然会导致函数包过大，上传时间太久。
3. 开发机器如果没有 python 环境，尽量选择使用 vscode 插件开发，可以避免很多环境配置的问题，节约不少时间。
4. serverless 作为一个新技术，需要谨慎使用，现在也还存在一些问题，比如冷启动响应时间较长，不同服务提供商有各自的特性标准，不便于项目迁移等问题。

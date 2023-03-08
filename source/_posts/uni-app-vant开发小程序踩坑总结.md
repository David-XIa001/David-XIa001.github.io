---
title: uni-app+vant开发小程序踩坑总结
date: 2023-03-07 14:55:30
tags:
- 原创
categories:
- 小程序
---

# uni-app + vant 开发小程序踩坑总结

## 开发微信小程序技术选型

### 原生开发

**优势**

- 小程序原生语言适配微信,专注开发即可
- 只用阅读微信开发文档即可
- 性能会得到系统开发优化

**劣势**

- 原生语言比较小众,离开了微信生态没有太多生存空间
- 组件库及其他插件生态不太好
- 无法支持一套代码多端运行

### uni-app

**优势**

- 语法基本与 vue 一致,前端人员上手成本较低
- 一套代码多端运行
- DCloud的生态插件相对完善
- 可选择的组件库及插件较多
- 支持 vscode 编辑器开发,npm 构建
- 开发效率更高

**劣势**

- 小程序文档及 uni-app 文档都需要查看,还有一些兼容性问题

综合考虑,推荐使用 uni-app 开发小程序
<br />

## 项目框架搭建

- 官方推荐使用 HBuildX 去搭建项目 (因无法使用vscode,难以接收)
- 个人推荐使用 VSCode 编辑器开发,通过 npm 命令引入脚手架

### 安装 vue-cli (已安装跳过)

`npm install -g @vue/cli`

### 创建 uni-app 项目

`vue create -p dcloudio/uni-preset-vue my-project`

- 选择 hello uni-app 模板即可
- 下载依赖包 (强烈建议使用 **yarn**,之前使用 npm 下载依赖一直有问题,排查很久也没找到问题,后面换了 yarn 就好了)
- 跑一下项目, `npm run dev:mp-weixin` 会在当前目录下有一个 dist 文件,里面有一个 **mp-weixin** 的文件夹
- 使用微信开发者工具将它导入就可以看到编译好的页面

### 微信小程序开发前置工作

- 小程序账号申请 (分为个人,企业,政府等类型,不同的类型会有功能限制)
- 小程序认证 (微信中一些 api 必须要认证后的小程序才能使用,比如获取用户手机号,业务场景涉及的话需提前申请认证 费用 300)
- 接口设置 (小程序对于部分 api 需要审核后才能使用,比如获取当前地理位置,需要提供完整的业务场景,通过之后才能使用)
- 微信支付 (需要开通商户服务才行,如果涉及到平台还会有二级商户,服务商等概念)
- ......

### 项目开发系统配置

#### 配置 appid

找到 `manifest.json` 文件,在 `mp-weixin` 属性下面配置你小程序的 appid
{% asset_img 1.png %}
**好处**
这样每次使用微信开发者工具打开项目都能识别你的小程序,方便后续的开发及版本上传

#### 配置 alias
根目录下新建 `vue.config.js` 文件

``` js
const defaultSettings = require('./src/config') // 公共配置文件
const path = require('path')
function resolve(dir) {
  return path.join(__dirname, dir)
}
const name = defaultSettings.name
module.exports = {
  lintOnSave: process.env.NODE_ENV === 'development',
  configureWebpack: {
    name: name,
    resolve: {
      alias: {
        '@': resolve('src')
      }
    }
  }
}
```
**好处**
方便文件引入,提高文件查找效率

#### 配置公共请求

- 小程序里无法使用 axios 这个库,所以只能自己根据 uni.request 进行封装,或者在 dcloud 里面找支持的第三方库

- 下面为个人在项目中的使用,仅供参考

```js
import { loginToast } from '@/utils'

const baseUrl = process.env.VUE_APP_BASE_API
export function request(options = {}) {
  options.header = {
    'openid': uni.getStorageSync('openid') || ''
  }
  return new Promise((resolve, reject) => {
    uni
      .request({
        url: baseUrl + options.url || '',
        method: options.method || 'GET',
        data: options.data || {},
        header: options.header || {}
      })
      .then((data) => {
        const [err, res] = data
        if (res.statusCode === 200) {
          resolve(res.data)
        } else {
          reject(res.data)
          switch (res.statusCode) {
            case 401:
              loginToast()
              break
            case 403:
              loginToast()
              break
              // 404请求不存在
            case 404:
              uni.showToast({
                title: '网络请求不存在',
                icon: 'error',
                duration: 1500
              })
              break
              // 其他错误，直接抛出错误提示
            case 500:
              uni.showToast({
                title: res.data.msg,
                icon: 'none',
                duration: 2000
              })
              break
            default:
              return
          }
        }
      })
      .catch((error) => {
        reject(error)
      })
  })
}

```

#### 引入第三方组件库

第三方组件库根据个人喜好选择

推荐使用 [vant-weapp](https://youzan.github.io/vant-weapp/#/home) 组件库

安装方案:
- 使用 npm 引入
- 下载组件库直接在项目中使用 (推荐第二种,有一些特性化的场景,可能需要对组件库进行二次开发)

使用:
- `pages.json` 文件下找到 `globalStyle` 对象
- 新增属性 `usingComponents`,然后引入需要的组件
- 下面是常用的组件,基本都会用到

```json
		"usingComponents": {
			"van-button": "/wxcomponents/vant/button/index",
			"van-card": "/wxcomponents/vant/card/index",
			"van-cell": "/wxcomponents/vant/cell/index",
			"van-cell-group": "/wxcomponents/vant/cell-group/index",
			"van-checkbox": "/wxcomponents/vant/checkbox/index",
			"van-checkbox-group": "/wxcomponents/vant/checkbox-group/index",
			"van-col": "/wxcomponents/vant/col/index",
			"van-dialog": "/wxcomponents/vant/dialog/index",
			"van-field": "/wxcomponents/vant/field/index",
			"van-goods-action": "/wxcomponents/vant/goods-action/index",
			"van-goods-action-icon": "/wxcomponents/vant/goods-action-icon/index",
			"van-goods-action-button": "/wxcomponents/vant/goods-action-button/index",
			"van-icon": "/wxcomponents/vant/icon/index",
			"van-loading": "/wxcomponents/vant/loading/index",
			"van-nav-bar": "/wxcomponents/vant/nav-bar/index",
			"van-notice-bar": "/wxcomponents/vant/notice-bar/index",
			"van-notify": "/wxcomponents/vant/notify/index",
			"van-panel": "/wxcomponents/vant/panel/index",
			"van-popup": "/wxcomponents/vant/popup/index",
			"van-progress": "/wxcomponents/vant/progress/index",
			"van-radio": "/wxcomponents/vant/radio/index",
			"van-radio-group": "/wxcomponents/vant/radio-group/index",
			"van-row": "/wxcomponents/vant/row/index",
			"van-search": "/wxcomponents/vant/search/index",
			"van-slider": "/wxcomponents/vant/slider/index",
			"van-stepper": "/wxcomponents/vant/stepper/index",
			"van-steps": "/wxcomponents/vant/steps/index",
			"van-submit-bar": "/wxcomponents/vant/submit-bar/index",
			"van-swipe-cell": "/wxcomponents/vant/swipe-cell/index",
			"van-switch": "/wxcomponents/vant/switch/index",
			"van-tab": "/wxcomponents/vant/tab/index",
			"van-tabs": "/wxcomponents/vant/tabs/index",
			"van-tabbar": "/wxcomponents/vant/tabbar/index",
			"van-tabbar-item": "/wxcomponents/vant/tabbar-item/index",
			"van-tag": "/wxcomponents/vant/tag/index",
			"van-toast": "/wxcomponents/vant/toast/index",
			"van-transition": "/wxcomponents/vant/transition/index",
			"van-tree-select": "/wxcomponents/vant/tree-select/index",
			"van-datetime-picker": "/wxcomponents/vant/datetime-picker/index",
			"van-rate": "/wxcomponents/vant/rate/index",
			"van-collapse": "/wxcomponents/vant/collapse/index",
			"van-collapse-item": "/wxcomponents/vant/collapse-item/index",
			"van-picker": "/wxcomponents/vant/picker/index"
		}
```


#### 引入 iconfont 图标

[uni-app中引入iconfont字体图标使用教程](https://blog.51cto.com/yataigp/3242664)

#### 使用 sass 样式预处理器

安装 sass 和 sass-loader 即可

#### 开发生产环境变量配置

...

#### 使用 vuex 状态管理

...

#### 配置 eslint

...

## 业务场景

### 微信登录及获取手机号

> 小程序中使用微信获取用户手机号一键登录是最常见的场景
**直接上代码**

```js
<template>
  <view class="container">
    <button
      type="primary"
      open-type="getPhoneNumber"
      @getphonenumber="onGetPhoneNumber"
    >
      微信用户一键登录
    </button>
    <van-toast id="van-toast" />
  </view>
</template>

<script>
import { login } from '@/api/user'
import Toast from '@/wxcomponents/vant/toast/toast'
export default {
  data() {
    return {}
  },
  methods: {
    // 获取用户手机号的前提是小程序已经认证通过了
    onGetPhoneNumber(e) {
      if (e.detail.errMsg === 'getPhoneNumber:fail user deny') {
        // 用户拒绝授权
        Toast.fail('请授权手机号!')
      } else {
        // 允许授权
        const phoneCode = e.detail.code
        uni.login({
          provider: 'weixin',
          success: (result) => {
            if (result && result.code) {
              login({ code: result.code, phoneCode: phoneCode }).then((res) => {
                uni.setStorageSync('openid', res.data.openid)
                uni.setStorageSync('userInfo', {
                  nickname: res.data.nickname,
                  avatar: res.data.avatar
                })
                Toast({
                  type: 'success',
                  message: '登录成功',
                  duration: 800,
                  onClose: () => {
                    uni.navigateBack()
                  }
                })
              })
            }
          },
          fail: (result) => {
            Toast.fail('出错了,请重试!')
          }
        })
      }
    }
  }
}
</script>
```

### 在小程序中使用 websocket

**小程序中无法使用 new webSocket() 的方式去创建**
```js
export default {
  data() {
    return {
      timer1: null,
      timer2: null, // 心跳
      flag: false,
      wsurl: '',
    }
  },
  onLoad() {
    this.initWebSocket()
  },
  destroyed() {
    this.clearTimer()
    this.flag = false
    uni.closeSocket() // 离开路由之后断开websocket连接
  },
  methods: {
    // 初始化weosocket
    initWebSocket() {
      this.wsurl = 'xxxx'
      uni.connectSocket({
        url: this.wsurl,
        success: () => {
          this.clearTimer()
          uni.onSocketOpen((res) => {
            console.log('打开连接')
          })
          // 方式断开,持续给服务端发送信息
          this.timer2 = setInterval(() => {
            this.websocketsend('')
          }, 60 * 1000)
        },
        fail: () => {
          // 失败重连机制
          this.clearTimer()
          this.timer1 = setTimeout(() => {
            this.initWebSocket()
          }, 5000)
        }
      })
      uni.onSocketMessage((e) => {
        // 监听服务端推送过来的数据
        const resdata = JSON.parse(e.data)
      })
      // 断开重连机制
      uni.onSocketClose(() => {
        if (this.flag) {
          this.clearTimer()
          this.timer1 = setTimeout(() => {
            this.initWebSocket()
          }, 5000)
        }
      })
    },
    // 数据发送
    websocketsend(Data) {
      uni.sendSocketMessage({
        data: Data
      })
    },
    // 清空定时器
    clearTimer() {
      clearInterval(this.timer1)
      clearInterval(this.timer2)
      this.timer1 = null
      this.timer2 = null
    }
  }
}
```

### 获取地理位置及拉起地图导航

**获取地理位置**
- 前提是已经在小程序申请通过了 `getLocation` api
- 使用,需要用户授权,授权通过并打开了定位
```js
      uni.getLocation({
        type: 'gcj02',
        success: (res) => {
          console.log('当前位置的经度：' + res.longitude)
          console.log('当前位置的纬度：' + res.latitude)
        },
        fail: (err) => {
          console.log('err =', err)
        },
        complete: () => {
          this.getList()
        }
      })
```
**拉起地图导航**
- 经纬度一定需要是 number 类型,否则会报错
```js
      uni.openLocation({
        latitude: parseFloat(info.lat), // 目的地的纬度
        longitude: parseFloat(info.lng), // 目的地的经度
        name: info.stationName // 打开后显示的地址名称
      })
```

### 普通二维码跳转页面

- 需要先下载校验文件,放在域名服务器的根目录下
- 配置二维码规则
- 上线之后需要发布
- [配置普通链接二维码规则官方文档](https://developers.weixin.qq.com/miniprogram/introduction/qrcode.html#%E4%BA%8C%E7%BB%B4%E7%A0%81%E8%B7%B3%E8%BD%AC%E8%A7%84%E5%88%99)

### 自定义 tabbar

[uni-app 之 自定义 tabbar](https://juejin.cn/post/7125985273598443528)

## 坑点

- 启动小程序的时候不要挂 VPN 接口会一直 502
- 使用 vue-cli 搭建项目框架,引入依赖的时候,不要使用 npm,使用 yarn
- 业务场景需要提前申请小程序的权限,否则会阻塞业务开发
  - 微信支付申请企业商户 3-7 天
  - 微信认证一般 1-2 天
  - 申请使用定位信息 api, 非必填资料一定要填,否则会被拒绝
  - ... (尽量多看文档,很多 api 微信团队也一直在修改)
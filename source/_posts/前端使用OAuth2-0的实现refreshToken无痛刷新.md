---
title: 前端使用OAuth2.0的实现refreshToken无痛刷新
date: 2022-12-29 11:17:47
tags:
- 原创
categories:
- 工程化
---

# 前端使用OAuth2.0的实现refreshToken无痛刷新

## 业务背景

> 现在项目中登录鉴权一般使用的是 OAuth 2.0, 登录完成后会返回两个token,分别是 **accessToken** 和 **refreshToken**
    > **accessToken** 我们在调接口的时候会带给服务端,作为用户凭证在, **refreshToken** 是用来刷新用户凭证的一个参数
    > 目的是避免用户经常因为还在操作页面的时候 **accessToken** 过期,需要重新登录的较差体验

## 原理

- 登录完成后会返回两个token,分别是 **accessToken** 和 **refreshToken** 以及一个 **expiresTime** 过期时间
- **accessToken** 作为用户凭,证调接口的时候会带给服务端
- 前端会在拦截器中判断 当前时间是否超过了 **expiresTime** 过期时间
    - 超过了,就会调用刷新 **token** 的接口, **refreshToken** 将作为请求参数带到服务端,接口会返回新的 **accessToken** 和 **refreshToken** 和 **expiresTime**,后面的请求将使用新的 **accessToken** 作为用户凭证带给服务端
    - 未超过,直接使用 **accessToken** 作为用户凭证
- 只要用户在一直操作,那么就算他的 **accessToken** 过期也不需要重新登录
- 直到超过了 **refreshToken** 的过期时间(一般这个时间会比 **accessToken** 过期时间长很多,说明用户在这段时间内都没有进行操作),才会真正的过期

## 实现

> 项目将使用axios的库进行请求的发送和拦截

### 目录结构

`src └─libs ├─axios │ ├─config.js │ ├─index.js │ ├─instance.js`

### config.js

```js
/**
 * axios的公共配置
 */
const config = {
  baseURL: process.env.VUE_APP_BASE_API, // base_api
  // 公共请求头
  headers: {
    'Content-Type': 'application/json; charset=UTF-8',
  },
  timeout: 5000, // 超时时间
  // 默认的响应方式
  responseType: 'json'
}

export default config

```

### instance.js

``` js  
import axios from 'axios'
import config from './config'
import { Message } from 'element-ui'
import {
  getAccessToken,
  removeAccessToken,
  removeRefreshToken,
  removeExpiresTime
} from '@/utils/auth' //这里的函数主要是使用 localStorage 进行数据持久化
import router from '@/router'
import i18n from '@/lang'  // 国际化

/**
 * 创建一个独立的axios实例
 * 把常用的公共请求配置放这里添加
 */
const instance = axios.create(config)

/**
 * 请求拦截
 * 添加一些全局要带上的东西
 */
instance.interceptors.request.use(
  // 正常拦截
  (config) => {
    // 添加token
    const LOCAL_TOKEN = getAccessToken()
    if (LOCAL_TOKEN) {
      config.headers['Authorization'] = LOCAL_TOKEN
    }
    // 返回处理后的配置
    return Promise.resolve(config)
  },

  // 拦截失败
  (err) => Promise.reject(err)
)

/**
 * 返回拦截
 * 在这里解决数据返回的异常问题
 */
instance.interceptors.response.use(
  // 正常响应
  (res) => {
    if (res.status === 200) {
      return Promise.resolve(res.data)
    } else {
      return Promise.reject(res)
    }
  },
  // 异常响应（统一返回一个msg提示）
  (err) => {
    if (err.response && err.response.status) {
      switch (err.response.status) {
        // ...
        case 500:
          Message({
            message: i18n.t(`error.${err.response.data.code}`),
            duration: 1500,
            type: 'error'
          })
          if (err.response.data.code === 'TOKEN_INVALID') { //token 失效后的操作
            removeAccessToken()
            removeExpiresTime()
            removeRefreshToken()
            router.replace({
              path: '/login'
            })
          }
          break
        default:
          return
      }
      return Promise.reject(err.response)
    } else {
      try {
        Message({
          message: '服务器异常',
          duration: 1500,
          type: 'error'
        })
      } catch (e) {
        console.log(e)
      }
    }
  }
)

export default instance

```

### index.js

``` js
import axios from './instance'
import {
  getAccessToken,
  getExpiresTime,
  setAccessToken,
  getRefreshToken,
  setRefreshToken,
  setExpiresTime
} from '@/utils/auth' //这里的函数主要是使用 localStorage 进行数据持久化
import { refresh } from '@/api/user' // refresh 的api

// 防止重复刷新的状态开关
let isRefreshing = false
// 被拦截的请求列表
let requests = []
/**
 * 请求拦截
 * 在这里要判断是否需要刷新token
 * 帮助用户自动延长登录有效期
 */
axios.interceptors.request.use((config) => {
  /**
   * 刷新token
   */
  // 计算token的剩余有效时间
  const OLD_TOKEN_EXP = getExpiresTime() || 0
  const NOW_TIMESTAMP = Date.now()
  const TIME_DIFF = OLD_TOKEN_EXP - NOW_TIMESTAMP
  // 判断本地是否有记录
  const HAS_LOCAL_TOKEN = !!getAccessToken()
  const HAS_LOCAL_TOKEN_EXP = !!OLD_TOKEN_EXP
  // 获取接口url
  const API_URL = config.url.split('?')[0] || ''
  // 非刷新请求、有本地记录、已过期，同时满足，才会进入刷新流程
  if (
    API_URL !== '/admin-users/refreshToken' && // 这个地方判断接口是否是刷新接口, api 地址需要一致
    HAS_LOCAL_TOKEN &&
    HAS_LOCAL_TOKEN_EXP &&
    TIME_DIFF <= 2000 // 这里的 2000 ms 是防止发送请求的时候没过期,但是接口到达服务器的时候过期了,根据实际情况进行调整
  ) {
    // 如果没有在刷新，则执行刷新
    if (!isRefreshing) {
      // 打开状态
      isRefreshing = true
      // 获取新的token
      const REFRESH_TOKEN = getRefreshToken() || ''
      // 请求刷新
      refresh(REFRESH_TOKEN)
        .then((res) => {
          // 存储token信息
          const data = res.data
          setAccessToken(data.accessToken)
          setRefreshToken(data.refreshToken)
          setExpiresTime(data.expiresTime)
          // 如果新的token存在，用新token继续之前的请求，然后重置队列
          if (data.accessToken) {
            config.headers['Authorization'] = data.accessToken
            requests.forEach((cb) => cb(config))
            requests = []
          }
          // 关闭状态，允许下次继续刷新
          isRefreshing = false
        })
        .catch(() => {
          isRefreshing = false
          requests = []
        })
    }
    // 并把刷新完成之前的请求都存储为请求队列
    return new Promise((resolve) => {
      requests.push(() => {
        resolve(config)
      })
    })
  }
  return Promise.resolve(config)
})

export default axios

```

## 参考资料

[基于OAuth2.0的refreshToken前端刷新方案与演示demo](https://chengpeiquan.com/article/refresh-token.html)


---
title: webpack迁移到vite踩坑实战
date: 2023-06-19 09:43:41
tags:
- 原创
categories:
- 工程化
---

# webpack迁移到vite踩坑实战

## 项目背景

- 项目是 vue2 的老项目，打包时间大概一分钟左右，不算太慢也不算太快
- 不需要兼容 IE，所以不存在兼容性问题
- 项目是我从0到1开发的，十分熟悉，所以想拿来改造下 vite 试手

## 注意事项及原则

- 想要把项目从 webpack 改造成 vite 不是一个简单的工程，其中改动点有点多
- 一定要单独拉一个分支不影响正常业务推进，同时保留历史分支，给自己留一条退路
- 尽量不要改动业务代码

## 开始踩坑

> 本篇尽量详细的讲述"坑点"及踩坑后的思路及解决方案

### vite 和 vue 版本对应不上

- 改造第一步，先下依赖 `npm install vite -D `，安装就直接报错。
- 原因是 vite 支持的最低 vue 版本为 vue2.7
- 所以需要将项目升级至 vue2.7，这里我直接升级到 vue2.7.14，然后再下依赖
- 然后再执行 npm i @vitejs/plugin-vue2
- 如果项目中有使用过 jsx 语法的话，还需要安装 `npm i @vitejs/plugin-vue2-jsx`
- 至此，依赖版本问题解决

### vite 打包vue代码中引入的文件必须要加后缀

- 依赖引入之后需要再根目录下建立一文件叫 vite.config.js 
- 同时需要在根目录下建一个 html 文件，vite 官方文档的要求
- 在 vite.config.js 去写一些打包的配置

```js
import { defineConfig } from 'vite'
import createVuePlugin from '@vitejs/plugin-vue2'
import vueJsx from '@vitejs/plugin-vue2-jsx'
import path from 'path'

function resolve(dir) {
  return path.join(__dirname, dir)
}
export default ({ mode }) => {
  // const isProduction = mode === 'production';
  return defineConfig({
    base: '/',
    logLevel: 'info',
    plugins: [
      // vue2 和 jsx
      createVuePlugin({
        jsx: true,
        jsxOptions: {
          compositionAPI: true
        }
      }),
      vueJsx()
    ],
    devServer 设置
    server: {
      proxy: {
        '/api': {
          target: 'http://192.168.150.92:30344/p',
          logLevel: 'debug',
          changeOrigin: true
        }
      }
    },
    // 依赖解析规则等
    resolve: {
      extensions: [
        '.vue',
        '.js',
        '.jsx',
        '.json',
        '.css',
        '.scss'
      ],
      alias: [
        { find: '@', replacement: resolve('src') }
      ]
    }
  })
}


```
- package.json 中增加启动命令 `"vite": "vite  --mode development"`
- 运行后还是报错，报错的是业务代码，原因是引入的文件没有加后缀
- 整体项目中去加下后缀，体力活

### vite 打包中不能使用 require 引入模块

- 全局找一下，改为 import 引入

### svg 引入文件的方式与 webpack 不同

1. webapck 中引入 svg 文件的写法
```js
const req = require.context('./svg', false, /\.svg$/)
const requireAll = requireContext => requireContext.keys().map(requireContext)
requireAll(req)
```
2. vite 中引入svg需要使用插件解决
- `npm install vite-plugin-svg-icons -D`
- main.js 文件中 增加`import 'virtual:svg-icons-register'`
- vite.config.js 中增加 svg 的 plugin 配置
```js
import { createSvgIconsPlugin } from 'vite-plugin-svg-icons'
 plugins: [
    ...
      createSvgIconsPlugin({
        // 指定要缓存的图标文件夹
        iconDirs: [path.resolve(process.cwd(), 'src/icons/svg')],
        // 执行icon name的格式
        symbolId: 'icon-[name]'
      })
    ]
```
- 至此 svg 图标可以在页面中正常显示

### vite的环境变量配置

前人已经写的很好了,可以参考 [vite环境变量配置](https://cn.vitejs.dev/guide/env-and-mode.html)

### path 模块找不到报错

- 路由配置中有使用到 node 中的 path 模块,导致报错
- vite 中使用 `import path from 'path-browserify'`即可

### sass公共变量模块引入报错

- webpack 中引入 scss 模板代码 `~@import "@/styles/mixin.scss";`
- 在 vite 中会报错, 去掉前面 '~'即可
- 在页面中使用 scss 变量在 js 中使用会报错
- 因为引入的 scss变量 并没有模块化,成为了字符串
- 解决就是复制一份scss变量文件,然后后缀加上 module
- 例: variables.scss => variables.module.scss
- [参考解决方案](https://github.com/vitejs/vite/discussions/9601)

### 禁用 CSS 注入页面 

- CSS 内容的行为可以通过 ?inline 参数来关闭。在关闭时，被处理过的 CSS 字符串将会作为该模块的默认导出，但样式并没有被注入到页面中。
- CSS 文件的默认导入和按名导入（例如 import style from './foo.css'）将弃用。请使用 ?inline 参数代替。

### el-table 组件渲染空白

[解决方案](https://github.com/ElemeFE/element/issues/21968)


## vite 体验

- 改造完毕之后,开发的时候确实流畅很多,响应很快
- 但是同时担心改造后直接用于生产环境,可能会出问题
- 建议开发的时候使用 vite,然后生产上还是使用 webpack 打包
- 有一些业务中的代码 无法兼容 vite 和 webpack 两套,需要用一些方法去解决这个问题
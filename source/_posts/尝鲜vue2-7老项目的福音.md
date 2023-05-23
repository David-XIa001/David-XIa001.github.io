---
title: 尝鲜vue2.7 老项目的福音!
date: 2023-05-23 14:03:45
tags:
- 原创
categories:
- 工程化
---

# 尝鲜vue2.7 老项目的福音!

## 业务背景

> 现在依然有很多项目使用的是 **vue2**,因为各种各样的原因无法直接升级到 **vue3**,无法体验更高效的开发方式和更优雅的写法,也让很多前端开发者困于老项目，无法使用最新的技术，但是时代是一直向前的。
    > 拥抱最新的技术是成长最快的方式,**vue2.7**的到来,让我们可以直接在老的项目里面使用***vue3**中的组合式写法.

### 老项目如何升级到 vue2.7

[官方升级到2.7文档](https://v2.cn.vuejs.org/v2/guide/migration-vue-2-7.html)

### 个人操作

- 我的项目原版本为 vue2.6.10,直接在 **package.json**中升级至 2.7.0
- 同时对应的 vue-cli-plugin 的版本也需要升级至对应版本,官方文档有详细叙述
- 主要变更如下
- {% asset_img 1.png %}
- 重新下载依赖后,运行不报错即可(我这个一切顺利)
- 老项目中尝试组合式写法, 运行正常无报错

```vue

<script setup>
import { ref } from 'vue'

const count = ref(0)
</script>

<template>
  <div class="card">
    <button type="button" @click="count++">count is {{ count }}</button>
  </div>
</template>
```

### 几个坑点

- 如果项目中使用了 TS,那么相应的依赖可能需要升级,否则会爆红
- eslint-plugin-vue 需要升级,老版本只适应了老版本的语法检测
- "eslint-plugin-vue": "^8.7.1"  可以升级至这个版本, 下载依赖后重启编辑器
---
title: 前端实现RBAC菜单按钮权限管理
date: 2023-02-08 20:01:41
tags:
- 原创
categories:
- 工程化
---

# 前端实现RBAC菜单按钮权限管理

## 业务背景

> 现在后台管理系统一般都会涉及到角色权限的分类,目的是为了使用一套系统多个角色,每个角色分配不同的菜单和按钮权限,进而对不同角色实现权限管理

## 原理

- 建立数据源
    - 项目菜单数据源
    - 功能按钮数据源
    - 角色数据源
    - 人员数据源
- 把菜单和按钮相应的数据源分配给对应的角色
- 创建人员的时候选择对应的角色
- 用户登录之后即可获取对应的权限

### 技术原理

- 动态路由(实现菜单权限管理)
- v-permission指令(权限管理颗粒度达到到按钮的级别)

#### 动态路由

```js
// 页面静态路由(所有权限都一定会有的路由)
// 直接挂载
export const constantRoutes = [
  {
    path: '/login',
    component: () => import('@/views/login/index'),
    hidden: true
  },
  {
    path: '/404',
    component: () => import('@/views/404'),
    hidden: true
  }
]
// 需要动态加载的路由
export const asyncRoutes = [
    ...
]

```

##### 登录之后根据接口返回的数据权限动态添加路由

```js
    // accessRoutes  根据接口返回的数据权限拼接过后的路由
    router.addRoutes(accessRoutes) // 动态添加
```
### 坑点

- 动态添加的路由是无法在 **options.routes** 中查看的
    - 所以需要记录自己添加的动态路由,放在 store 中供后面 menu 组件使用
- 需要保证登录之后匹配第一个路由(附解决方案)
```js
// 解决方案
router.beforeEach(async(to, from, next) => {
  NProgress.start()
  document.title = getPageTitle(to.meta.title)
  const hasToken = getAccessToken()
  if (hasToken) {
    if (to.path === '/login') {
      next({ path: '/' })
      NProgress.done()
    } else {
      const hasRoles = store.getters.roles && store.getters.roles.length > 0
      if (hasRoles) {
        next()
      } else {
        try {
          const role = await store.dispatch('user/getInfo')
          const accessRoutes = await store.dispatch('permission/generateRoutes',role)
          router.addRoutes(accessRoutes)
          let path = '/' // 登录后默认跳转路由
          if (accessRoutes && accessRoutes[0].path !== path) {
            // 如果默认第一个路由不为默认值,则匹配第一个路由地址
            if (accessRoutes[0].children) {
              path = accessRoutes[0].path + '/' + accessRoutes[0].children[0].path
            } else {
              path = accessRoutes[0].path
            }
          }
        //  解决刷新后默认跳转到首页问题
          if (to.fullPath === '/') {
            next({ path, replace: true })
          } else {
            next({ ...to, replace: true })
          }
        } catch (error) {
          await store.dispatch('user/resetToken')
          Message.error(error || 'Has Error')
          next(`/login?redirect=${to.path}`)
          NProgress.done()
        }
      }
    }
  } else {
    if (whiteList.indexOf(to.path) !== -1) {
      next()
    } else {
      next(`/login?redirect=${to.path}`)
      NProgress.done()
    }
  }
})
```

- 重置动态路由

    vue-router 只提供了 addRouter 方法去添加路由,没有提供删除路由的方法,所以遇到切换角色或者退出登陆后换角色登录,需要重置路由后再次动态添加

- 解决方案
  - 简单粗暴,退出登录之后直接刷新浏览器( `window.location.reload();` )
  - 重置 matcher 还原动态路由
  ```js
  import Vue from 'vue'
  import Router from 'vue-router'

  const createRouter = () => {
    return new Router({
      scrollBehavior: () => ({ y: 0 }),
      routes: constantRoutes  // 静态路由
    })
  }

  const router = createRouter()
  export function resetRouter() {
    const newRouter = createRouter()
    router.matcher = newRouter.matcher // 重置路由
  }
  ```

### 权限颗粒度达到按钮级别实现

**思路**
> 给每一个按钮唯一标识，获取所有按钮权限进行逐个比对，有这个标识则显示，否则就不显示

**示例**
```js
    <el-button
        v-permission="'base:soft:add'" // 唯一标识为 'base:soft:add'
        type="primary"
        size="mini"
        plain
        icon="el-icon-plus"
        @click="handleAdd"
      >
        {{ $t("common.add") }}
    </el-button>
```

- 实现 **v-permission** 指令

v-permission 的作用就是动态添加或者移除按钮

**目录结构**

`src └─directives ├─permission │ ├─index.js │ ├─permission.js`

#### permission.js

```js
import store from '@/store'

function checkPermission(el, binding) {
  const { value } = binding
  // btns 为一个对象，利用 hasOwnProperty 判断是按钮是否存在的时间复杂度为 O1，使用数组的效率会低很多
  const bnts = store.getters && store.getters.permission_btntKeyList
  const bntKey = value
  const hasPermission = bnts.hasOwnProperty(bntKey) 
  if (!hasPermission) {
    el.parentNode && el.parentNode.removeChild(el)
  }
}

export default {
  inserted(el, binding) {
    checkPermission(el, binding)
  },
  update(el, binding) {
    checkPermission(el, binding)
  }
}

```

#### index.js

```js
import permission from './permission'

const install = function(Vue) {
  Vue.directive('permission', permission)
}

if (window.Vue) {
  window['permission'] = permission
  Vue.use(install); // eslint-disable-line
}

permission.install = install
export default permission

```

#### main.js 中全局挂载

```js
  import permission from '@/directives/permission'
  Vue.use(permission)
```


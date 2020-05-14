---
tags:
  - 其他
categories:
  - 前端
title: 一次a[href='#']带来的问题
date:
updated:
---

## 起因
接手的一个老项目中发现一个问题，当从某个页面退出登录的时候，有可能退出失败，即仍然是登录状态。
<!-- more -->

A 和 B由webpack打包的两个入口。它们（js文件）被引入的域名如下：
```
A: abc.com
B: abc.com/app/#hash1
```
A是首页，登录后通过`window.location.href`跳转到B页面，并且带上hash值。B页面可以点击退出登录按钮后回到首页，并清除登录状态。


## 流程
简化版的流程如下：  
![](https://store-g1.seewo.com/fc339686a6d34c75afe7257fc8a5eede)

流程上很普通，没有什么问题。
1. 用户首次打开页面，页面发起登录请求，由于cookie中没有token以及sessionId，node中获取不到用户信息，用户处于未登录状态。
2. 用户手动登录，node session中保存了用户的信息，并将sessionId以及用户token存入客户端cookie中。
3. 用户下次打开页面的时候，无需再次手动点击登录，页面自动发送登录请求，携带cookie，node根据sessionId获取到用户信息，再进行接下来的操作。
4. 用户退出登录时，node中会清除session中该用户的信息，并且响应头中set-cookie字段的值中，token置为空。

## 复现
在B页面hash路由中，a标签是这样写的：

```
  handleLogoutClick(){
    request('/app/logout')
    .then(function() {
      window.location.href = '/';
    }, function() {
      // ...
    })
  }

  <a href='#' onClick={ this.handleLogoutClick }>{ Lang.LOGOUT }</a>
```

关键就在于`href='#'`导致了问题，这里a标签点击的时候实际上有两个操作：

- href='#'   
在jquery的年代，给a标签href='#'是一个很正常的，表示点击它什么都不做。然而在使用了现代框架并配置hash路由功能的情况下（不少项目仍然是配置hash路由的），它表示回退到根路由，即`abc.com/app/#`。

- 触发handleLogoutClick事件  
logout事件中，会删除掉用户session中的信息，并且响应头中set-cookie字段中token会置为空，这样用户就是登出状态。

问题在于第一个操作中，回到根路由，页面会立马发送请求去拉取一些初始化数据。此时请求状态如下：
```
1. 根路由X请求
2. /app/logout
```

在X请求的时候，/app/logout还没有执行完毕，所以这两个请求存在并发情况，根据请求完成先后的不同会导致不同的结果。

**1. 根路由X请求先完成，logout后完成**  
X请求完成后，设置token到cookie；然后logout请求完成后，node清除用户session信息，响应头设置cookie中的token为空。即这时候用户cookie中没有token，回到根页面`location.href='/'`后，走上述登录流程的第一步，登录失败，处于未登录状态。此时是符合预期的。

**2. logout先完成，根路由X请求后完成**  
logout请求完成后清除了用户信息，然而当X请求完成后还会写用户信息到cookie。然后回到根页面，此时cookie中是有用户信息的，会走到上述登录流程的第三部，登录成功，此时仍然是登录状态。

给用户的感觉就是点击了退出登录按钮，但是仍然是登录状态。这种情况是不符合预期的。

## 解决
解决方法很简单，就是去除`href='#'`属性。在使用了hash路由的情况下，这种原始操作还是要慎用，优先通过路由API该改变hash值，这样比较统一。

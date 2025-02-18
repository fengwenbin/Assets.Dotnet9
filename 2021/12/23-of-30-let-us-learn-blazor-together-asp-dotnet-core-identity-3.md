---
title: (23/30)大家一起学Blazor：ASP.NET Core Identity(3)
slug: 23-of-30-let-us-learn-blazor-together-asp-dotnet-core-identity-3
description: 前面有说到`UserAuthentication()`跟`UserAuthorization()`，这两个的差别在于：前者用于验证登录者是谁，后者则决定登录者可以做什么。
date: 2021-12-23 22:19:53
copyright: Reprinted
author: StrayaWorker
originalTitle: (23/30)大家一起学Blazor：ASP.NET Core Identity(3)
originalLink: https://ithelp.ithome.com.tw/articles/10270593
draft: False
cover: https://img1.dotnet9.com/2021/12/cover_05.png
categories: 
    - .NET
tags: 
    - Blazor
    - ASP.NET Core
    - 学Blazor
---

前面有说到`UserAuthentication()`跟`UserAuthorization()`，这两个的差别在于：前者用于验证登录者是谁，后者则决定登录者可以做什么。

举例来说，一个员工要登录员工系统，他必须输入帐号(如员工 ID、姓名或是 email)、密码，系统才能知道是谁登录了，这就是`Authentication(验证)`，处理`Authentication` 的方式有`Cookie`、`Token`、`第三方验证(OAuth 或API-token)`、`OpenId`及`SAML`。

而当员工登录系统后，一般员工通常不会有跨部门或是管理权限，他只能看到他自己或所属部门的信息，例如生产部员工看不到会计部的财务，不过会计部为了计算成本却看得到生产部的原料价格，这就是`Authorization (授权)`。

在决定登录者可以做什么前，必须先知道登录者是谁，所以`UserAuthentication()`必须放在`UserAuthorization()`前面。

`ASP.NET Core Identity` 使用的是基于`Claim` 的验证，要了解`Claim`，必须先了解`Claim`、`ClaimsIdentity` 跟`ClaimsPrincipal` 是什么。

`Claim` 就是关于使用者的一些信息，`Claim Type` 跟`Claim Value`(可以不用给)就组成一个`Claim`，`Claim` 可以是`姓名`、`电话`、`角色`、`Email`甚至是`角色`等等。`Authorization` 就是用`Claim` 判断使用者有无授权。

```C#
new System.Security.Claims.Claim(ClaimTypes.Role, role.Name)
```

`ClaimsIdentity` 则是多个`Claim` 的集合，像是`驾照`上面记录了`姓名`、`生日`、`电话`，`驾照`就是一个`ClaimsIdentity`。

`ClaimsPrincipal` 是多个`ClaimsIdentity` 的集合，台湾人都有身分证跟健保卡，有些人还有驾照，身分证、健保卡跟驾照都是`ClaimsIdentity`，持有它们的人就是`ClaimsPrincipal`。

而每一个`HTTP request` 都会产生`HttpContext` 对象，该对象就存有目前`request` 的信息，其下可以找到一个类型为`ClaimsPrincipal` 的 Property 名为`User`，这个 Property 就是由`UseAuthentication()`引入的`Authentication` Middleware 产生的。

登录机制可以用`Cookie` 或是`JWT` 实现，但`Authentication` Middleware 怎么知道要用哪个方法产生`User` Property？那就要看`Authentication` Scheme 跟`Authentication` Handlers。

## Authentication Handlers

`Authentication Handlers` 就是处理验证的方式，`ASP.NET Core Identity` 可以调用`AuthenticateAsync()`API 去验证使用者已登录，如验证失败就调用`ChallengeAsync()`将使用者导回`登录页面`，如授权失败则用`ForbidAsync()`禁止使用者访问，当然也可以自己实现这些行为。下面例子中用了`JWT` 跟`Cookie` 的验证方式，如果用了前者，就必须验证`JWT token` 并产生`ClaimsPrincipal` 回传到`HttpContext.User` 中；使用后者则会检查当前`request` 的`cookie`并产生`ClaimsPrincipal`。

```C#
builder.Services.AddAuthentication()
	.AddJwtBearer()
	.AddCookie();
```

## Authentication Scheme

用了任何一种方式注册`Authentication Handlers` 就称为`Authentication Scheme`，每个`Authentication Scheme` 都有一个独特的名字以识别，且可以自己设定`Authentication Handlers`，下面的程序结果跟上面会是一样，因为它们都有预设的`Scheme` Name。

```C#
builder.Services.AddAuthentication()
    .AddJwtBearer("Bearer")
    .AddCookie("Cookies");
```

## Blazor Authentication

`Blazor` 用的验证方式跟`ASP.NET Core` 一样，不过`Blazor WebAssembly` 跟`Blazor Server` 又有不同，前者的验证就像任何前端网站一样可以被绕过，因为使用者一端的程序可以被使用者改动，因此发送数据的`API` 端一定也需要验证；后者则可用内建的`AuthenticationStateProvider` 取得前面说的`HttpContext.User`，笔者此前就是自己继承并重写这项`Service` 实现`JWT` 验证的。

`AuthenticationStateProvider` 就是昨天说到的`<AuthorizeView>`及`<CascadingAuthenticationState>`可以取得当前验证状态的原因，但如果没有要重写预设验证机制的话，建议不要自己在`Component` 注入一个`AuthenticationStateProvider` 出来，直接使用的缺点很明显，若当前`request` 的验证状态有改动，因为你改动了这个`Component` 的验证机制，该`Component` 就不会被告知。

可以看到下图，`ApiAuthenticationStateProvider`继承了`AuthenticationStateProvider`，并重写了`Task<AuthenticationState>`，这个 Property 可以取得当前的验证状态，`MarkUserAsAuthenticated()`跟`MarkUserAsLoggedOut()`则是笔者自己写的方法用以标示使用者通过验证及注销系统，`NotifyAuthenticationStateChanged()`顾名思义会通知各个 Component 当前验证状态，这就是自己继承并重写的案例。

![](https://img1.dotnet9.com/2021/12/3501.png)

下图则是在 Component 取得当前`request` 的`HttpContext.User` 及`Claims` 的作法，不过这里是先利用服务取得`User` 再取得其下`Claims`，其实是多此一举了。

![](https://img1.dotnet9.com/2021/12/3502.png)

如果只是要取得`HttpContext.User`，只要如下图般就可以了，因为 `Task<AuthenticationState>`会以`[CascadingParameter]`的方式层层传递下去。

![](https://img1.dotnet9.com/2021/12/3503.png)

**引用：**

1. [Introduction to Authentication in ASP.NET Core](https://www.tektutorialshub.com/asp-net-core/authentication-in-asp-net-core/#claim)
2. [ASP.NET Core Blazor authentication and authorization](https://docs.microsoft.com/en-us/aspnet/core/blazor/security/?view=aspnetcore-5.0)

**注：本文代码通过 .NET 6 + Visual Studio 2022 重构，可点击原文链接与重构后代码比较学习，谢谢阅读，支持原作者**

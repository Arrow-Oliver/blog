---
title: "Oauth"
date: 2024-06-01T10:49:23+08:00
draft: false
categories: [笔记,单点登录]
tags: [oauth]
card: false
weight: 0
---

# 初学OAuth 2.0

介绍：OAuth是一个关于授权（authorization）的开放网络标准。

>  **参考[理解OAuth 2.0](https://www.ruanyifeng.com/blog/2014/05/oauth_2_0.html)**

## 背景

**OAuth的适用场合，提出问题**

有一个"云冲印"的网站，可以将用户储存在Google的照片，冲印出来。用户为了使用该服务，必须让"云冲印"读取自己储存在Google上的照片。

传统方法是，用户将自己的Google用户名和密码，告诉"云冲印"，后者就可以读取用户的照片了。这样的做法有以下几个严重的缺点。

> （1）"云冲印"为了后续的服务，会保存用户的密码，这样很不安全。
>
> （2）Google不得不部署密码登录，而我们知道，单纯的密码登录并不安全。
>
> （3）"云冲印"拥有了获取用户储存在Google所有资料的权力，用户没法限制"云冲印"获得授权的范围和有效期。
>
> （4）用户只有修改密码，才能收回赋予"云冲印"的权力。但是这样做，会使得其他所有获得用户授权的第三方应用程序全部失效。
>
> （5）只要有一个第三方应用程序被破解，就会导致用户密码泄漏，以及所有被密码保护的数据泄漏。

<u>OAuth就是为了解决上面这些问题而诞生的。</u>

## 专有名词

（1） **Third-party application**：第三方应用程序，本文中又称"客户端"（client），即上一节例子中的"云冲印"。

（2）**HTTP service**：HTTP服务提供商，本文中简称"服务提供商"，即上一节例子中的Google。

（3）**Resource Owner**：资源所有者，本文中又称"用户"（user）。

（4）**User Agent**：用户代理，本文中就是指浏览器。

（5）**Authorization server**：认证服务器，即服务提供商专门用来处理认证的服务器。

（6）**Resource server**：资源服务器，即服务提供商存放用户生成的资源的服务器。它与认证服务器，可以是同一台服务器，也可以是不同的服务器。

>  <u>OAuth的作用就是让"客户端"安全可控地获取"用户"的授权，与"服务商提供商"进行互动。</u>

## OAuth的理解	

客户端无法直接访问服务提供商，需要通过认证授权服务器进行用户认证授权，成功认证后客户端即可访问服务提供商。

其中几个关键几点：

1. 传统方法是用户将用户名密码交给客户端，让客户端进行登录访问服务提供商；但对于OAuth方法即用户直接访问服务提供商，实现客户端与用户的分离。

2. 用户登录成功后生成token作为用户成功登录的标识发送给客户端，服务提供商根据token的权限范围和有效期，向"客户端"开放用户储存的资料。

   > token不同于用户名密码登录，可以限制过期时间和授权范围，更加的灵活。

<u>可以把token（用户信息）当作门禁卡，客户端拿着这张门禁卡即可出入服务提供商的大门，但是客户端并不知道这张门禁卡如何解锁打开大门，即不需知道用户的隐私信息。</u>

## OAuth运作流程

![OAuth运行流程](https://www.ruanyifeng.com/blogimg/asset/2014/bg2014051203.png)

（A）用户打开客户端以后，客户端要求用户给予授权。

**（B）用户同意给予客户端授权。**

（C）客户端使用上一步获得的授权，向认证服务器申请令牌。

（D）认证服务器对客户端进行认证以后，确认无误，同意发放令牌。

（E）客户端使用令牌，向资源服务器申请获取资源。

（F）资源服务器确认令牌无误，同意向客户端开放资源

​	<u>B步骤是关键，即用户怎样才能给于客户端授权。有了这个授权以后，客户端就可以获取令牌，进而凭令牌获取资源。</u>

## OAuth四种授权模式

客户端必须得到用户的授权（authorization grant），才能获得令牌（access token）。OAuth 2.0定义了四种授权方式。

- 授权码模式（authorization code）
- 简化模式（implicit）
- 密码模式（resource owner password credentials）
- 客户端模式（client credentials）

### 授权码模式

授权码模式（authorization code）是功能最完整、流程最严密的授权模式。它的特点就是通过客户端的后台服务器，与"服务提供商"的认证服务器进行互动。

![授权码模式](https://www.ruanyifeng.com/blogimg/asset/2014/bg2014051204.png)

（A）用户访问客户端，后者将前者导向认证服务器（客户端:login-redirect）。

（B）用户选择是否给予客户端授权。

（C）假设用户给予授权，认证服务器将用户导向客户端事先指定的"重定向URI"（客户端:login-redirect-callback），同时附上一个授权码。

（D）客户端收到授权码，附上第一次请求的"重定向URI"，向认证服务器申请令牌。这一步是在客户端的后台的服务器上完成的，对用户不可见。

（E）认证服务器核对了授权码和重定向URI，确认无误后，向客户端发送访问令牌（access token）和更新令牌（refresh token）。

下面是上面这些步骤所需要的参数。

A步骤中，客户端申请认证的URI，包含以下参数：

- response_type：表示授权类型，必选项，此处的值固定为"code"
- client_id：表示客户端的ID，必选项
- redirect_uri：表示重定向URI，可选项
- state：表示客户端的当前状态，可以指定任意值，认证服务器会原封不动地返回这个值。

```http
GET https://example.com/authorization/oauth/authorize?
response_type=code&
client_id=clientId&
state=Hd3B09Ya&
redirect_uri=https://example.com/client/callbackRedirectUri
```

C步骤中，认证服务器回应客户端的URI，包含以下参数：

- code：表示授权码，必选项。该码的有效期应该很短，通常设为10分钟，客户端只能使用该码一次，否则会被授权服务器拒绝。该码与客户端ID和重定向URI，是一一对应关系。
- state：如果客户端的请求中包含这个参数，认证服务器的回应也必须一模一样包含这个参数。

```http
GET https://example.com/client/callbackRedirectUri?
code=zB1ytDEHUwYu67uu2dJlqBl3T-CEERWVYwDRToJVOUC2O3bFjnl0RjtiFkpOnHODYepZZ9eWcG_IjkLs4obh2f&
state=Hd3B09Ya
```

客户端获取到关键的信`code`后，即可拿着`code`去认证服务器获取`token`以及用户信息。

> 这步可以都在`callbackRedirectUri`方法中实现

### 简化模式

简化模式（implicit grant type）不通过第三方应用程序的服务器（客户端后台服务器），直接在浏览器中向认证服务器申请令牌，跳过了"授权码"这个步骤，因此得名。所有步骤在浏览器中完成，令牌对访问者是可见的，且客户端不需要认证。

![简化模式](https://www.ruanyifeng.com/blogimg/asset/2014/bg2014051205.png)

（A）客户端将用户导向认证服务器。

（B）用户决定是否给于客户端授权。

（C）假设用户给予授权，认证服务器将用户导向客户端指定的"重定向URI"，并在URI的Hash部分包含了访问令牌。

（D）浏览器向资源服务器发出请求，其中不包括上一步收到的Hash值。(<u>获取提取token的脚本</u>)

（E）资源服务器返回一个网页，其中包含的代码可以获取Hash值中的令牌。

（F）浏览器执行上一步获得的脚本，提取出令牌。

（G）浏览器将令牌发给客户端。

重点说明C、D、E、F步骤：

C步骤中，认证服务器回应客户端的URI，包含以下参数：

- access_token：表示访问令牌，必选项。
- token_type：表示令牌类型，该值大小写不敏感，必选项。
- expires_in：表示过期时间，单位为秒。如果省略该参数，必须其他方式设置过期时间。
- scope：表示权限范围，如果与客户端申请的范围一致，此项可省略。
- state：如果客户端的请求中包含这个参数，认证服务器的回应也必须一模一样包含这个参数。

```http
HTTP/1.1 302 Found
Location: http://example.com/cb#access_token=2YotnFZFEjr1zCsicMWpAA
          &state=xyz&token_type=example&expires_in=3600
```

在上面的例子中，认证服务器用HTTP头信息的Location栏，指定浏览器重定向的网址。注意，在这个网址的Hash部分包含了令牌。

根据上面的D步骤，下一步浏览器会访问Location指定的网址，但是Hash部分不会发送。接下来的E步骤，服务提供商的资源服务器发送过来的代码，会提取出Hash中的令牌。

### 密码模式

密码模式（Resource Owner Password Credentials Grant）中，用户向客户端提供自己的用户名和密码。客户端使用这些信息，向"服务商提供商"索要授权。

在这种模式中，用户必须把自己的密码给客户端，但是客户端不得储存密码。这通常用在用户对客户端高度信任的情况下，比如客户端是操作系统的一部分，或者由一个著名公司出品。而认证服务器只有在其他授权模式无法执行的情况下，才能考虑使用这种模式。

流程：

（A）用户向客户端提供用户名和密码。

（B）客户端将用户名和密码发给认证服务器，向后者请求令牌。

（C）认证服务器确认无误后，向客户端提供访问令牌。

>  个人理解，这种模式已经脱离了OAuth的理念，属于正常的客户端服务端认证模式。

### 客户端模式

客户端模式（Client Credentials Grant）指客户端以自己的名义，而不是以用户的名义，向"服务提供商"进行认证。严格地说，客户端模式并不属于OAuth框架所要解决的问题。在这种模式中，用户直接向客户端注册，客户端以自己的名义要求"服务提供商"提供服务，其实不存在授权问题。

流程：

（A）客户端向认证服务器进行身份认证，并要求一个访问令牌。

（B）认证服务器确认无误后，向客户端提供访问令牌。	

## 更新令牌

token作为认证令牌，其中一项体现灵活性的就是存在过期时间，而更新令牌则是作为刷新过期时间的关键点。

如果用户访问的时候，客户端的"访问令牌"已经过期，则需要使用"更新令牌"申请一个新的访问令牌。

客户端发出更新令牌的HTTP请求，包含以下参数：

- grant*type：表示使用的授权模式，此处的值固定为"refresh*token"，必选项。
- refresh_token：表示早前收到的更新令牌，必选项。
- scope：表示申请的授权范围，不可以超出上一次申请的范围，如果省略该参数，则表示与上一次一致。

```http
POST /token HTTP/1.1
Host: server.example.com
Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token&refresh_token=tGzv3JOkF0XG5Qx2TlKWIA
```








































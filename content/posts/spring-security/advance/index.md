---
title: "进阶SpringSecurity"
date: 2024-04-11T21:57:07+08:00
draft: false
categories: [Java,笔记,Spring,框架]
tags: [SpringSecurity]
card: false
weight: 0
---

# 用户认证流程源码

**1. `UsernamePasswordAuthenticationFilter-实现类`**   

<u>简单来说，两步。</u>

① 封装对象；

② 管理认证器进行认证。

**详细。**
调用`authenticate`认证方法 -->
1.1  将传入的`username`和`password`封装到`UsernamePasswordAuthenticationToken-实现类`对象中。

1.2  调用`ProviderManager-实现类`的方法`authentic`进行后续认证。



**2. `ProviderManager-实现类`** 

<u>简单来说，选取合适认证器进行认证，并且封装认证结果</u>

**详细。**
调用`AbstractUserDetailsAuthenticationProvider-抽象类`的`authenticate`模板方法进行用户认证，并且返回认证结果。模板方法中具体调用的方法实现是`DaoAuthenticationProvider-实现类`



**3. `AbstractUserDetailsAuthenticationProvider` 和 `DaoAuthenticationProvider`**
<u>简单说，三步。</u>

① 根据用户名获取数据库中用户信息，并且封装成`UserDetails-接口`对象； 
② 根据`UserDetails`对象的密码和前文封装的`UsernamePasswordAuthenticationToken`对象的密码进行匹配；　
③ 以上操作都成功的话，将`UserDetails`对象封装到`UsernamePasswordAuthenticationToken`对象中，然后返回。

<u>模板方法：</u>

`UserDetails user = retrieveUser(username, (UsernamePasswordAuthenticationToken) authentication);`
`void additionalAuthenticationChecks(user, (UsernamePasswordAuthenticationToken) authentication);`
`Authentication result = createSuccessAuthentication(principalToReturn, authentication, user);`

**详细。**
3.1  调用`retrieveUser` 方法，此方法中`UserDetails loadedUser = this.userDetailsService.loadUserByUsername(username);`的方法根据用户名查询数据库用户信息。
（这步是需要开发人员自己实现认证业务，即实现`userDetailsService`的`loadUserByUsername(String username)`方法；
并且封装`UserDetails`对象，保存角色权限信息到此对象中。同样需要开发人员定义实现类）；

3.2  将通过用户名认证成功的`UserDetails`对象的密码和`UsernamePasswordAuthenticationToken`对象的密码进行匹配。即用户输入的密码和库中存的密码进行匹配。

3.3  帐号密码认证成功后，将`UserDetails`对象的用户名，密码，角色权限信息等封装到`UsernamePasswordAuthenticationToken`对象中。（`UsernamePasswordAuthenticationToken`对象将保存到上下文中）


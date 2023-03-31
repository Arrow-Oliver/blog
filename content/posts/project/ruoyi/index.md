---
title: "Ruoyi后台管理系统笔记"
date: 2023-03-31T23:46:02+08:00
draft: false
categories: [项目,笔记]
tags: [Ruoyi]
card: false
weight: 0
---

# 1.项目总体功能

## 项目模块

**系统管理**

* 用户管理
* 角色管理
* 菜单管理
* 部门管理
* 岗位管理
* 字典管理
* 参数设置
* 通知公告
* 日志管理
  * 操作日志
  * 登录日志

**系统监控**

* 在线用户
* 定时任务
* 数据监控
* 服务监控
* 缓存监控

**系统工具**

* 表单构建
* 代码生成
* 系统接口

## 项目技术

后端：

`Spring Boot`,`Shiro`,`Mybatis`,`Druid`

前端：

`BootStrap`,`Thymeleaf`

## 包结构

~~~java
com.ruoyi     
├── common            // 工具类
│       └── annotation                    // 自定义注解
│       └── config                        // 全局配置
│       └── constant                      // 通用常量
│       └── core                          // 核心控制
│       └── enums                         // 通用枚举
│       └── exception                     // 通用异常
│       └── json                          // JSON数据处理
│       └── utils                         // 通用类处理
│       └── xss                           // XSS过滤处理
├── framework         // 框架核心
│       └── aspectj                       // 注解实现
│       └── config                        // 系统配置
│       └── datasource                    // 数据权限
│       └── interceptor                   // 拦截器
│       └── manager                       // 异步处理
│       └── shiro                         // 权限控制
│       └── web                           // 前端控制
├── ruoyi-generator   // 代码生成（不用可移除）
├── ruoyi-quartz      // 定时任务（不用可移除）
├── ruoyi-system      // 系统代码
├── ruoyi-admin       // 后台服务
├── ruoyi-xxxxxx      // 其他模块
~~~

# 2.基本结构

## `Controller结构`

`ruoyi-admin`模块存储`Mvc`层代码；调用service层方法，返回数据。

~~~java
@Controller
@RequestMapping("/system/user")
public class SysUserController extends BaseController{
    /**
     * 跳转页面
     */
    @RequiresPermissions("system:user:view")
    @GetMapping()
    public String user()
    {
        return prefix + "/user";
    }
    /**
     * 返回JSON数据
     */
    @RequiresPermissions("system:user:list")
    @PostMapping("/list")
    @ResponseBody
    public TableDataInfo list(SysUser user)
    {
        startPage();
        List<SysUser> list = userService.selectUserList(user);
        return getDataTable(list);
    }
    
}
~~~

## `Service`结构

`ruoyi-system`模块调用业务逻辑层，以及数据层；标准的调用服务层代码，进行数据访问。

```java
@Service
public class SysUserServiceImpl implements ISysUserService{
    /**
     * 根据条件分页查询用户列表
     * 
     * @param user 用户信息
     * @return 用户信息集合信息
     */
    @Override
    @DataScope(deptAlias = "d", userAlias = "u")
    public List<SysUser> selectUserList(SysUser user){
        return userMapper.selectUserList(user);
    }
}
```

# 3.重点技术

## `shiro`

### 认证登录

**流程**

①访问`ControllerAPI`

~~~java
@PostMapping("/login")
@ResponseBody
public AjaxResult ajaxLogin(String username, String password, Boolean rememberMe)
{
    //封装token
    UsernamePasswordToken token = new UsernamePasswordToken(username, password, rememberMe);
    //获取主体
    Subject subject = SecurityUtils.getSubject();
    try
    {
        //进行shiro认证登录
        subject.login(token);
        return success();
    }
    catch (AuthenticationException e)
    {
        String msg = "用户或密码错误";
        if (StringUtils.isNotEmpty(e.getMessage()))
        {
            msg = e.getMessage();
        }
        return error(msg);
    }
}
~~~

②访问自定义`UserRealm`进行认证登录

```java
/**
 * 登录认证
 */
@Override
protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken token) throws AuthenticationException
{
    UsernamePasswordToken upToken = (UsernamePasswordToken) token;
    String username = upToken.getUsername();
    String password = "";
    if (upToken.getPassword() != null)
    {
        password = new String(upToken.getPassword());
    }

    SysUser user = null;
    try
    {
        //调用业务逻辑层进行用户名密码验证
        user = loginService.login(username, password);
    }
	//...省略各种异常问题
    catch (Exception e)
    {
        log.info("对用户[" + username + "]进行登录验证..验证未通过{}", e.getMessage());
        throw new AuthenticationException(e.getMessage(), e);
    }
    //封装shiro提供的对象并且返回，交给shiro进行保护
    SimpleAuthenticationInfo info = new SimpleAuthenticationInfo(user, password, getName());
    return info;
}
```

③登录认证（用户名密码校验）

```java
/**
 * 登录
 */
public SysUser login(String username, String password)
{
    // 验证码校验
    if (ShiroConstants.CAPTCHA_ERROR.equals(ServletUtils.getRequest().getAttribute(ShiroConstants.CURRENT_CAPTCHA)))
    {
        AsyncManager.me().execute(AsyncFactory.recordLogininfor(username, Constants.LOGIN_FAIL, MessageUtils.message("user.jcaptcha.error")));
        throw new CaptchaException();
    }
    // 用户名或密码为空 错误
    if (StringUtils.isEmpty(username) || StringUtils.isEmpty(password))
    {
        AsyncManager.me().execute(AsyncFactory.recordLogininfor(username, Constants.LOGIN_FAIL, MessageUtils.message("not.null")));
        throw new UserNotExistsException();
    }
    // 密码如果不在指定范围内 错误
    if (password.length() < UserConstants.PASSWORD_MIN_LENGTH
            || password.length() > UserConstants.PASSWORD_MAX_LENGTH)
    {
        AsyncManager.me().execute(AsyncFactory.recordLogininfor(username, Constants.LOGIN_FAIL, MessageUtils.message("user.password.not.match")));
        throw new UserPasswordNotMatchException();
    }

    // 用户名不在指定范围内 错误
    if (username.length() < UserConstants.USERNAME_MIN_LENGTH
            || username.length() > UserConstants.USERNAME_MAX_LENGTH)
    {
        AsyncManager.me().execute(AsyncFactory.recordLogininfor(username, Constants.LOGIN_FAIL, MessageUtils.message("user.password.not.match")));
        throw new UserPasswordNotMatchException();
    }

    // 查询用户信息
    SysUser user = userService.selectUserByLoginName(username);
	
    //用户名错误
    if (user == null)
    {
        AsyncManager.me().execute(AsyncFactory.recordLogininfor(username, Constants.LOGIN_FAIL, MessageUtils.message("user.not.exists")));
        throw new UserNotExistsException();
    }
    
    //用户被删除
    if (UserStatus.DELETED.getCode().equals(user.getDelFlag()))
    {
        AsyncManager.me().execute(AsyncFactory.recordLogininfor(username, Constants.LOGIN_FAIL, MessageUtils.message("user.password.delete")));
        throw new UserDeleteException();
    }
    
    //用户被禁用
    if (UserStatus.DISABLE.getCode().equals(user.getStatus()))
    {
        AsyncManager.me().execute(AsyncFactory.recordLogininfor(username, Constants.LOGIN_FAIL, MessageUtils.message("user.blocked", user.getRemark())));
        throw new UserBlockedException();
    }
	
    //密码验证
    passwordService.validate(user, password);

    AsyncManager.me().execute(AsyncFactory.recordLogininfor(username, Constants.LOGIN_SUCCESS, MessageUtils.message("user.login.success")));
    recordLoginInfo(user.getUserId());
    return user;
}

    /**
     * 密码验证
     */
    public void validate(SysUser user, String password)
    {
        String loginName = user.getLoginName();
        //登录异常次数
        AtomicInteger retryCount = loginRecordCache.get(loginName);

        if (retryCount == null)
        {
            retryCount = new AtomicInteger(0);
            loginRecordCache.put(loginName, retryCount);
        }
        //超过最大登录异常禁止登录
        if (retryCount.incrementAndGet() > Integer.valueOf(maxRetryCount).intValue())
        {
            AsyncManager.me().execute(AsyncFactory.recordLogininfor(loginName, Constants.LOGIN_FAIL, MessageUtils.message("user.password.retry.limit.exceed", maxRetryCount)));
            throw new UserPasswordRetryLimitExceedException(Integer.valueOf(maxRetryCount).intValue());
        }
        //用户名密码认证失败
        if (!matches(user, password))
        {
            AsyncManager.me().execute(AsyncFactory.recordLogininfor(loginName, Constants.LOGIN_FAIL, MessageUtils.message("user.password.retry.limit.count", retryCount)));
            //错误数加一
            loginRecordCache.put(loginName, retryCount);
            throw new UserPasswordNotMatchException();
        }
        else
        {
            //认证成功清除缓存
            clearLoginRecordCache(loginName);
        }
    }
```

**配置**

①安全管理器（中枢）

```java
@Bean
public SecurityManager securityManager(UserRealm userRealm)
{
    DefaultWebSecurityManager securityManager = new DefaultWebSecurityManager();
    // 设置realm.
    securityManager.setRealm(userRealm);
    // 记住我
    securityManager.setRememberMeManager(rememberMe ? rememberMeManager() : null);
    // 注入缓存管理器;
    securityManager.setCacheManager(getEhCacheManager());
    // session管理器
    securityManager.setSessionManager(sessionManager());
    return securityManager;
}
```

②自定义`Realm`，通过Realm来读取数据，进行用户登录权限认证

```java
@Bean
public UserRealm userRealm(EhCacheManager cacheManager)
{
    UserRealm userRealm = new UserRealm();
    userRealm.setAuthorizationCacheName(Constants.SYS_AUTH_CACHE);
    userRealm.setCacheManager(cacheManager);
    return userRealm;
}
```

③`Shiro`过滤器配置

```java
@Bean
public ShiroFilterFactoryBean shiroFilterFactoryBean(SecurityManager securityManager)
{
    CustomShiroFilterFactoryBean shiroFilterFactoryBean = new CustomShiroFilterFactoryBean();
    // Shiro的核心安全接口,这个属性是必须的
    shiroFilterFactoryBean.setSecurityManager(securityManager);
    // 身份认证失败，则跳转到登录页面的配置
    shiroFilterFactoryBean.setLoginUrl(loginUrl);
    // 权限认证失败，则跳转到指定页面
    shiroFilterFactoryBean.setUnauthorizedUrl(unauthorizedUrl);
    // Shiro连接约束配置，即过滤链的定义
    LinkedHashMap<String, String> filterChainDefinitionMap = new LinkedHashMap<>();
    // 对静态资源设置匿名访问
    // <!-- authc:所有url都必须认证通过才可以访问; anon:所有url都都可以匿名访问-->
    filterChainDefinitionMap.put("/favicon.ico**", "anon");
    filterChainDefinitionMap.put("/ruoyi.png**", "anon");
    filterChainDefinitionMap.put("/html/**", "anon");
    filterChainDefinitionMap.put("/css/**", "anon");
    filterChainDefinitionMap.put("/docs/**", "anon");
    filterChainDefinitionMap.put("/fonts/**", "anon");
    filterChainDefinitionMap.put("/img/**", "anon");
    filterChainDefinitionMap.put("/ajax/**", "anon");
    filterChainDefinitionMap.put("/js/**", "anon");
    filterChainDefinitionMap.put("/ruoyi/**", "anon");
    filterChainDefinitionMap.put("/captcha/captchaImage**", "anon");
    // 退出 logout地址，shiro去清除session
    filterChainDefinitionMap.put("/logout", "logout");
    // 不需要拦截的访问
    filterChainDefinitionMap.put("/login", "anon,captchaValidate");
    // 注册相关
    filterChainDefinitionMap.put("/register", "anon,captchaValidate");

    Map<String, Filter> filters = new LinkedHashMap<String, Filter>();
    filters.put("onlineSession", onlineSessionFilter());
    filters.put("syncOnlineSession", syncOnlineSessionFilter());
    filters.put("captchaValidate", captchaValidateFilter());
    filters.put("kickout", kickoutSessionFilter());
    // 注销成功，则跳转到指定页面
    filters.put("logout", logoutFilter());
    shiroFilterFactoryBean.setFilters(filters);

    // 所有请求需要认证
    filterChainDefinitionMap.put("/**", "user,kickout,onlineSession,syncOnlineSession");
    shiroFilterFactoryBean.setFilterChainDefinitionMap(filterChainDefinitionMap);

    return shiroFilterFactoryBean;
}
```

### 认证授权

**流程**

①实现`UserRealm`方法`doGetAuthorizationInfo`

```java
/**
 * 授权
 */
@Override
protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection arg0)
{
    SysUser user = ShiroUtils.getSysUser();
    // 角色列表
    Set<String> roles = new HashSet<String>();
    // 功能列表
    Set<String> menus = new HashSet<String>();
    //创建shiro权限管理对象
    SimpleAuthorizationInfo info = new SimpleAuthorizationInfo();
    // 管理员拥有所有权限
    if (user.isAdmin())
    {
        info.addRole("admin");
        info.addStringPermission("*:*:*");
    }
    else
    {
        //授予角色以及权限
        roles = roleService.selectRoleKeys(user.getUserId());
        menus = menuService.selectPermsByUserId(user.getUserId());
        // 角色加入AuthorizationInfo认证对象
        info.setRoles(roles);
        // 权限加入AuthorizationInfo认证对象
        info.setStringPermissions(menus);
    }
    return info;
}
```

②使用注解进行接口访问权限设置

```java
@RequiresPermissions("system:user:view")
@GetMapping()
public String user()
{
    return prefix + "/user";
}
```

**配置**

权限注解扫描开启

```java
@Bean
public AuthorizationAttributeSourceAdvisor authorizationAttributeSourceAdvisor(
        @Qualifier("securityManager") SecurityManager securityManager)
{
    AuthorizationAttributeSourceAdvisor authorizationAttributeSourceAdvisor = new AuthorizationAttributeSourceAdvisor();
    authorizationAttributeSourceAdvisor.setSecurityManager(securityManager);
    return authorizationAttributeSourceAdvisor;
}
```

## `AspectJ`

### 日志记录`@Log`

①`@Log`详情属性

```java
/**
 * 自定义操作日志记录注解
 * 
 * @author ruoyi
 */
@Target({ ElementType.PARAMETER, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Log
{
    /**
     * 模块 
     */
    public String title() default "";

    /**
     * 功能
     */
    public BusinessType businessType() default BusinessType.OTHER;

    /**
     * 操作人类别
     */
    public OperatorType operatorType() default OperatorType.MANAGE;

    /**
     * 是否保存请求的参数
     */
    public boolean isSaveRequestData() default true;

    /**
     * 是否保存响应的参数
     */
    public boolean isSaveResponseData() default true;
}
```

②Controller层调用修改数据操作时加入`@Log`注解进行切面增强

```java
/**
 * 新增保存用户
 */
@RequiresPermissions("system:user:add")
@Log(title = "用户管理", businessType = BusinessType.INSERT)
@PostMapping("/add")
@ResponseBody
public AjaxResult addSave(@Validated SysUser user)
{
    if (UserConstants.USER_NAME_NOT_UNIQUE.equals(userService.checkLoginNameUnique(user.getLoginName())))
    {
        return error("新增用户'" + user.getLoginName() + "'失败，登录账号已存在");
    }
    else if (StringUtils.isNotEmpty(user.getPhonenumber())
            && UserConstants.USER_PHONE_NOT_UNIQUE.equals(userService.checkPhoneUnique(user)))
    {
        return error("新增用户'" + user.getLoginName() + "'失败，手机号码已存在");
    }
    else if (StringUtils.isNotEmpty(user.getEmail())
            && UserConstants.USER_EMAIL_NOT_UNIQUE.equals(userService.checkEmailUnique(user)))
    {
        return error("新增用户'" + user.getLoginName() + "'失败，邮箱账号已存在");
    }
    user.setSalt(ShiroUtils.randomSalt());
    user.setPassword(passwordService.encryptPassword(user.getLoginName(), user.getPassword(), user.getSalt()));
    user.setCreateBy(getLoginName());
    return toAjax(userService.insertUser(user));
}
```

③`aspectJ`应用，对`operLog`表进行日志记录

```java
/**
 * 处理完请求后执行
 *
 * @param joinPoint 切点
 */
//                          注解为@Log在完成目标方法后进行增强
@AfterReturning(pointcut = "@annotation(controllerLog)", returning = "jsonResult")
public void doAfterReturning(JoinPoint joinPoint, Log controllerLog, Object jsonResult)
{
    handleLog(joinPoint, controllerLog, null, jsonResult);
}

protected void handleLog(final JoinPoint joinPoint, Log controllerLog, final Exception e, Object jsonResult)
    {
        try
        {
            // 获取当前的用户
            SysUser currentUser = ShiroUtils.getSysUser();

            // *========数据库日志=========*//
            SysOperLog operLog = new SysOperLog();
            operLog.setStatus(BusinessStatus.SUCCESS.ordinal());
            // 请求的地址
            String ip = ShiroUtils.getIp();
            operLog.setOperIp(ip);
            operLog.setOperUrl(ServletUtils.getRequest().getRequestURI());
            //设置当前操作用户信息
            if (currentUser != null)
            {
                operLog.setOperName(currentUser.getLoginName());
                if (StringUtils.isNotNull(currentUser.getDept())
                        && StringUtils.isNotEmpty(currentUser.getDept().getDeptName()))
                {
                    operLog.setDeptName(currentUser.getDept().getDeptName());
                }
            }

            if (e != null)
            {
                operLog.setStatus(BusinessStatus.FAIL.ordinal());
                operLog.setErrorMsg(StringUtils.substring(e.getMessage(), 0, 2000));
            }
            // 设置方法名称
            String className = joinPoint.getTarget().getClass().getName();
            String methodName = joinPoint.getSignature().getName();
            operLog.setMethod(className + "." + methodName + "()");
            // 设置请求方式
            operLog.setRequestMethod(ServletUtils.getRequest().getMethod());
            // 处理设置注解上的参数
            getControllerMethodDescription(joinPoint, controllerLog, operLog, jsonResult);
            // 保存数据库
            AsyncManager.me().execute(AsyncFactory.recordOper(operLog));
        }
        catch (Exception exp)
        {
            // 记录本地异常日志
            log.error("==前置通知异常==");
            log.error("异常信息:{}", exp.getMessage());
            exp.printStackTrace();
        }
    }
```

④异步保存数据库的操作

```java
/**
 * 操作日志记录
 * 
 * @param operLog 操作日志信息
 * @return 任务task
 */
public static TimerTask recordOper(final SysOperLog operLog)
{
    return new TimerTask()
    {
        @Override
        public void run()
        {
            // 远程查询操作地点
            operLog.setOperLocation(AddressUtils.getRealAddressByIP(operLog.getOperIp()));
            SpringUtils.getBean(ISysOperLogService.class).insertOperlog(operLog);
        }
    };
}
```

### 数据过滤`@DataScope`

①`@DataScope`数据类型

```java
public @interface DataScope
{
    /**
     * 部门表的别名
     */
    public String deptAlias() default "";

    /**
     * 用户表的别名
     */
    public String userAlias() default "";
}
```

②不同权限用户访问不同的部门（不同用户添加查询不同的部门）

```java
/**
 * 数据权限过滤关键字
 */
public static final String DATA_SCOPE = "dataScope";

@Before("@annotation(controllerDataScope)")
public void doBefore(JoinPoint point, DataScope controllerDataScope) throws Throwable
{
    clearDataScope(point);
    handleDataScope(point, controllerDataScope);
}

protected void handleDataScope(final JoinPoint joinPoint, DataScope controllerDataScope)
{
    // 获取当前的用户
    SysUser currentUser = ShiroUtils.getSysUser();
    if (currentUser != null)
    {
        // 如果是超级管理员，则不过滤数据
        if (!currentUser.isAdmin())
        {	//数据范围过滤
            dataScopeFilter(joinPoint, currentUser, controllerDataScope.deptAlias(),
                    controllerDataScope.userAlias());
        }
    }

```

③具体应用，`dept`的`dataScope`属性的值为`SQL`语句，进行当前用户部门过滤

```java
@Override
@DataScope(deptAlias = "d")
public List<SysDept> selectDeptList(SysDept dept)
{
    return deptMapper.selectDeptList(dept);
}
```

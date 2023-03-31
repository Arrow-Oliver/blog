---
title: "Novel小说项目笔记"
date: 2023-03-31T23:43:06+08:00
draft: false
categories: [项目,笔记]
tags: [Novel]
card: false
weight: 0
---

# 1.项目总体功能

## 项目模块

* 作家模块
* 用户模块
* 首页模块
* 新闻模块
* 资源模块（图片/视频/文档）
* 搜索模块
* 小说模块

## 项目技术

后端：

`Spring Boot`、`MyBatis-Plus`、`JJWT`、`Caffeine`、`Redis`

前端：

`Vue`、`element-plus`

## 包结构

```java
io
 +- github
     +- xxyopen   
        +- novel
            +- NovelApplication.java -- 项目启动类
            |
            +- core -- 项目核心模块，包括各种工具、配置和常量等
            |   +- common -- 业务无关的通用模块
            |   |   +- exception -- 通用异常处理
            |   |   +- constant -- 通用常量   
            |   |   +- req -- 通用请求数据格式封装，例如分页请求数据  
            |   |   +- resp -- 接口响应工具及响应数据格式封装 
            |   |   +- util -- 通用工具   
            |   | 
            |   +- auth -- 用户认证授权相关
            |   +- config -- 业务相关配置
            |   +- constant -- 业务相关常量         
            |   +- filter -- 过滤器 
            |   +- interceptor -- 拦截器
            |   +- json -- JSON 相关的包，包括序列化器和反序列化器
            |   +- task -- 定时任务
            |   +- util -- 业务相关工具 
            |   +- wrapper -- 装饰器
            |
            +- dto -- 数据传输对象，包括对各种 Http 请求和响应数据的封装
            |   +- req -- Http 请求数据封装
            |   +- resp -- Http 响应数据封装
            |
            +- dao -- 数据访问层，与底层 MySQL 进行数据交互
            +- manager -- 通用业务处理层，对第三方平台封装、对 Service 层通用能力的下沉以及对多个 DAO 的组合复用 
            +- service -- 相对具体的业务逻辑服务层  
            +- controller -- 主要是处理各种 Http 请求，各类基本参数校验，或者不复用的业务简单处理，返回 JSON 数据等
            |   +- front -- 小说门户相关接口
            |   +- author -- 作家管理后台相关接口
            |   +- admin -- 平台管理后台相关接口
            |   +- app -- app 接口
            |   +- applet -- 小程序接口
            |   +- open -- 开放接口，供第三方调用 
```

# 2.基本结构

1. `Controller`调用`Service`；

2. `Service`编写业务逻辑：大部分访问缓存（`Manager`），小部分直接访问数据库（`Mapper`）

## `Controller`结构

```java
//以AuthorController为例
@RestController
@RequestMapping(ApiRouterConsts.API_AUTHOR_URL_PREFIX)
@RequiredArgsConstructor
public class AuthorController {

    private final AuthorService authorService;
    
    /**
     * 作家注册接口
     */
    @PostMapping("register")
    public RestResp<Void> register(@Valid @RequestBody AuthorRegisterReqDto dto) {
        dto.setUserId(UserHolder.getUserId());
        return authorService.register(dto);
    }

}
```

* 请求信息（`Req`）：`AuthorRegisterReqDto` 
  * 进行封装的请求对象，而不是直接用数据库对象（`AuthorInfo`）接收信息

* 响应的信息（`Resp`）:`RestResp<T>`
  * 统一响应信息处理类；泛型为响应前端格式的对象，而不是数据库对象

## `Service`结构

```java
//AuthorService实现类
@Service
@RequiredArgsConstructor
@Slf4j
public class AuthorServiceImpl implements AuthorService {
	
    private final AuthorInfoCacheManager authorInfoCacheManager;

    private final AuthorInfoMapper authorInfoMapper;

    @Override
    public RestResp<Void> register(AuthorRegisterReqDto dto) {
        //判断当前作家是否存在
        AuthorInfoDto authorInfo = authorInfoCacheManager.getAuthorInfo(dto.getUserId());
        if(!Objects.isNull(authorInfo)){
            return RestResp.ok();
        }
        //注册信息
        AuthorInfo authorRegister = new AuthorInfo();
        BeanUtils.copyProperties(dto,authorRegister);
        authorRegister.setInviteCode("0");
        authorRegister.setCreateTime(LocalDateTime.now());
        authorRegister.setUpdateTime(LocalDateTime.now());
        authorInfoMapper.insert(authorRegister);
        return RestResp.ok();
    }
}
```

## `Manager`结构

```java
@Component
@RequiredArgsConstructor
public class AuthorInfoCacheManager {

    private final AuthorInfoMapper authorInfoMapper;

    @Cacheable(cacheManager = CacheConsts.REDIS_CACHE_MANAGER,
            value = CacheConsts.AUTHOR_INFO_CACHE_NAME,
            unless = "#result == null")
    public AuthorInfoDto getAuthorInfo(Long userId) {
        //查询当前用户是否是作者
        LambdaQueryWrapper<AuthorInfo> queryWrapper = new LambdaQueryWrapper<>();
        queryWrapper.eq(AuthorInfo::getUserId,userId)
                .last(DatabaseConsts.SqlEnum.LIMIT_1.getSql());
        AuthorInfo authorInfo = authorInfoMapper.selectOne(queryWrapper);
        if(Objects.isNull(authorInfo)){
            return null;
        }
        return AuthorInfoDto.builder()
                .id(authorInfo.getId())
                .penName(authorInfo.getPenName())
                .status(authorInfo.getStatus())
                .build();
    }
}
```

## `Mapper`结构

```java
public interface AuthorInfoMapper extends BaseMapper<AuthorInfo> {
}
```

# 3.重点技术

## 用户单点登录（SSO）

1. 进入login接口，验证信息，将用户信息进行`jwt`加密存入`Token`并且返回
2. 除了`login`接口，请求进入`AuthInterceptor`认证登录信息
3. 未登录进行登录认证实现类，即`FrontAuthStrategy`进行用户信息认证
4. 认证失败抛出异常，认证成功用户信息存入缓存以及ThreadLocal

**`login`接口实现类**

```java
public RestResp<UserLoginRespDto> login(UserLoginReqDto userDto) {
    //判断用户是否存在
    LambdaQueryWrapper<UserInfo> queryWrapper = new LambdaQueryWrapper<>();
    queryWrapper.eq(UserInfo::getUsername,userDto.getUsername());
    UserInfo userInfo = getOne(queryWrapper);
    if(Objects.isNull(userInfo)){
        throw new BusinessException(ErrorCodeEnum.USER_ACCOUNT_NOT_EXIST);
    }
    //判断加密密码是否正确
   if(!Objects.equals(userInfo.getPassword(),
           DigestUtils.md5DigestAsHex(userDto.getPassword().getBytes(StandardCharsets.UTF_8)))){
       throw new BusinessException(ErrorCodeEnum.USER_PASSWORD_ERROR);
   }
    //进行JWT加密并且生成Token返回respDTO
    return RestResp.ok(UserLoginRespDto.builder()
            .token(jwtUtils.generateToken(userInfo.getId(), SystemConfigConsts.NOVEL_FRONT_KEY))
            .uid(userInfo.getId())
            .nickName(userInfo.getNickName())
            .build());

}
```

**`AuthInterceptor`登录认证拦截器**

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class AuthInterceptor implements HandlerInterceptor {
	//如果key为String，value为接口；Map中存入接口的实现类 单例实体，key为beanId，Value为实体
    private final Map<String, AuthStrategy> authStrategy;

    private final ObjectMapper objectMapper;

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        //获取JWT
        String token = request.getHeader(SystemConfigConsts.HTTP_AUTH_HEADER_NAME);
        //获取URI
        String uri = request.getRequestURI();

        log.debug("拦截路径：{}",uri);
        //解析URI
        String subUri = uri.substring(ApiRouterConsts.API_URL_PREFIX.length() + 1);
        String systemName = subUri.substring(0, subUri.indexOf("/"));
        String authStrategyName = String.format("%sAuthStrategy", systemName);

        //选择认证器
        try {
            authStrategy.get(authStrategyName).auth(token,uri);
            //放行
            return HandlerInterceptor.super.preHandle(request, response, handler);
        }catch (BusinessException exception){
            // 认证失败
            response.setCharacterEncoding(StandardCharsets.UTF_8.name());
            response.setContentType(MediaType.APPLICATION_JSON_VALUE);
            response.getWriter().write(objectMapper.writeValueAsString(RestResp.fail(exception.getErrorCodeEnum())));
            return false;
        }
    }
}
```

**`FrontAuthStrategy`用户登录认证器**

```java
@Component
@RequiredArgsConstructor
public class FrontAuthStrategy implements AuthStrategy {

    private final JwtUtils jwtUtils;

    private final UserInfoCacheManager userInfoCacheManager;


    @Override
    public void auth(String token, String requestUri) throws BusinessException {
        authSSO(jwtUtils,userInfoCacheManager,token);
    }
}
```

**`AuthStrategy`认证器接口**

```java
public interface AuthStrategy {
    /**
     * 请求用户认证
     *
     * @param token 登录 token
     * @param requestUri 请求的 URI
     * @throws BusinessException 认证失败则抛出业务异常
     */
    void auth(String token, String requestUri)  throws BusinessException;

    /**
     * 前台多系统单点登录统一账号认证（门户系统、作家系统以及后面会扩展的漫画系统和视频系统等）
     *
     * @param jwtUtils             jwt 工具
     * @param userInfoCacheManager 用户缓存管理对象
     * @param token                token 登录 token
     * @return 用户ID
     */
    default Long authSSO(JwtUtils jwtUtils, UserInfoCacheManager userInfoCacheManager, String token){
        //判断token是否过期
        if(!StringUtils.hasText(token)){
            throw new BusinessException(ErrorCodeEnum.USER_LOGIN_EXPIRED);
        }
        //解析token
        Long userId = jwtUtils.parseToken(token, SystemConfigConsts.NOVEL_FRONT_KEY);
        if(Objects.isNull(userId)){
            throw  new BusinessException(ErrorCodeEnum.USER_LOGIN_EXPIRED);
        }
        //查询用户数据
        UserInfoDto userInfo = userInfoCacheManager.getUserInfo(userId);
        if(Objects.isNull(userInfo)){
            throw new BusinessException(ErrorCodeEnum.USER_ACCOUNT_NOT_EXIST);
        }
        //存入UserHolder
        UserHolder.setUserId(userId);
        //返回用户Id
        return userId;
    }

}
```

## 验证码生成

1. 生成会话Id作为验证码的标识
2. 生成验证码图片并且存入Redis返回
3. 用户注册中进行验证

**生成验证码实现类**

```java
@Override
public RestResp<ImgVerifyCodeRespDto> imgVerifyCode() throws IOException {
    //生成会话ID
    String uuid = IdWorker.get32UUID();

    //生成图片
    String img = verifyCodeManager.generateCode(uuid);
    //封装对象
    return RestResp.ok(ImgVerifyCodeRespDto.builder()
            .sessionId(uuid)
            .img(img)
            .build());
}
```

**`generateCode`方法**

```java
public String generateCode(String sessionId) throws IOException {
    //生成验证码
    String code = ImgVerifyCodeUtils.getRandomVerifyCode(4);
        //生成图片
        String img = ImgVerifyCodeUtils.genVerifyCodeImg(code);
        //会话Id为Key，code为Value存入redis
        stringRedisTemplate.opsForValue().set(CacheConsts.IMG_VERIFY_CODE_CACHE_KEY + sessionId,
                code,5,TimeUnit.MINUTES);
        return img;
}
```

**`verifyCodeOk`方法**

```java
/**
 * 校验验证码
 * @param sessionId
 * @return
 */
public boolean verifyCodeOk(String sessionId,String code){
    // 根据会话Id查询验证码信息
   return Objects.equals(code,
           stringRedisTemplate.opsForValue().get(CacheConsts.IMG_VERIFY_CODE_CACHE_KEY + sessionId));
}
```

**`delVerifyCode`方法**

```java
/**
 * 删除验证码
 * @param sessionId
 */
public void delVerifyCode(String sessionId){
    //验证完成后进行删除
    stringRedisTemplate.delete(CacheConsts.IMG_VERIFY_CODE_CACHE_KEY + sessionId);
}
```

**`UserRegister`方法**

```java
@Override
public RestResp<UserRegisterRespDto> register(UserRegisterReqDto registerReqDto) {
    //校验验证码
    @NotBlank @Length(min = 32, max = 32) String sessionId = registerReqDto.getSessionId();
	
    if (!verifyCodeManager.verifyCodeOk(sessionId,registerReqDto.getVelCode())) {
        throw new BusinessException(ErrorCodeEnum.USER_VERIFY_CODE_ERROR);
    }
    //用户名是否存在
    LambdaQueryWrapper<UserInfo> queryWrapper = new LambdaQueryWrapper<>();
    queryWrapper.eq(UserInfo::getUsername,registerReqDto.getUsername());
    if(count(queryWrapper) > 0){
        throw new BusinessException(ErrorCodeEnum.USER_NAME_EXIST);
    }
    //存入数据库
    UserInfo userInfo = new UserInfo();
    userInfo.setUsername(registerReqDto.getUsername());
	//密码MD5加密     
userInfo.setPassword(DigestUtils.md5DigestAsHex(registerReqDto.getPassword().getBytes(StandardCharsets.UTF_8)));
    userInfo.setNickName(registerReqDto.getUsername());
    userInfo.setSalt("0");
    userInfo.setCreateTime(LocalDateTime.now());
    userInfo.setUpdateTime(LocalDateTime.now());
    save(userInfo);
    // 删除验证码
    verifyCodeManager.delVerifyCode(sessionId);
    // 生成JWT 并返回，直接登录
    Long id = userInfo.getId();
    return RestResp.ok(UserRegisterRespDto.builder()
            .uid(id)
            .token(jwtUtils.generateToken(id,SystemConfigConsts.NOVEL_FRONT_KEY))
            .build());
}
```

## 文件上传下载

1. 上传图片请求`api/front/resource/image`对文件进行重命名并且存入本地，返回重命名后的文件名
2. 前端再次发送请求`/image/**`，`FileInterceptor`进行拦截，解析请求路径，获取本地图片上传到浏览器

**图片下载实现类**

```java
@SneakyThrows
@Override
public RestResp<String> uploadImg(MultipartFile file) {
    //修改当前图片信息
    LocalDateTime now = LocalDateTime.now();
    String savePath =
            SystemConfigConsts.IMAGE_UPLOAD_DIRECTORY
            + now.format(DateTimeFormatter.ofPattern("yyyy")) + File.separator
            + now.format(DateTimeFormatter.ofPattern("MM")) + File.separator
            + now.format(DateTimeFormatter.ofPattern("dd"));
    String originalFilename = file.getOriginalFilename();
    assert originalFilename != null;
    //重命名文件名
    String saveFileName = IdWorker.get32UUID() + originalFilename.substring(originalFilename.lastIndexOf("."));
    File saveFile = new File(fileUploadPath + savePath, saveFileName);
    if (!saveFile.getParentFile().exists()) {
        boolean isSuccess = saveFile.getParentFile().mkdirs();
        if (!isSuccess) {
            throw new BusinessException(ErrorCodeEnum.USER_UPLOAD_FILE_ERROR);
        }
    }
    //转存图片路径
    file.transferTo(saveFile);
    if (Objects.isNull(ImageIO.read(saveFile))) {
        // 上传的文件不是图片
        Files.delete(saveFile.toPath());
        throw new BusinessException(ErrorCodeEnum.USER_UPLOAD_FILE_TYPE_NOT_MATCH);
    }
    //返回地址信息
    return RestResp.ok(savePath + File.separator + saveFileName);


}
```

`FileInterceptor`文件拦截器

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class FileInterceptor implements HandlerInterceptor {
	//加载本地图片路径
    @Value("${novel.file.upload.path}")
    private String fileUploadPath;

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        // 获取请求的 URI
        String requestUri = request.getRequestURI();
        log.debug("请求拦截：{}",requestUri);
        // 缓存10天，并将图片显示在浏览器
        response.setDateHeader("expires", System.currentTimeMillis() + 60 * 60 * 24 * 10 * 1000);
        try (OutputStream out = response.getOutputStream(); InputStream input = new FileInputStream(fileUploadPath + requestUri)) {
            byte[] b = new byte[4096];
            for (int n; (n = input.read(b)) != -1; ) {
                out.write(b, 0, n);
            }
        }
        return false;
    }
}
```

## 首页数据缓存

1. 使用SpringCache将数据进行缓存Redis或者Caffeine缓存
2. 返回响应信息

```java
@Service
@RequiredArgsConstructor
public class HomeServiceImpl implements HomeService {
	//查询缓存
    private final HomeBookCacheManager homeBookCacheManager;

    @Override
    public RestResp<List<HomeBookRespDto>> homeBooks() {
        return RestResp.ok(homeBookCacheManager.homeBooks());
    }
}
```

`HomeBookCacheManager`首页信息缓存

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class HomeBookCacheManager {

    private final HomeBookMapper homeBookMapper;

//    private final HomeFriendLinkMapper homeFriendLinkMapper;

    private final BookInfoMapper bookInfoMapper;

    @Cacheable(cacheManager = CacheConsts.CAFFEINE_CACHE_MANAGER,
            value = CacheConsts.HOME_BOOK_CACHE_NAME)
    public List<HomeBookRespDto> homeBooks() {
        //升序查询bookID
        LambdaQueryWrapper<HomeBook> queryWrapper = new LambdaQueryWrapper<>();
        queryWrapper.orderByAsc(HomeBook::getSort);
        List<HomeBook> homeBooks = homeBookMapper.selectList(queryWrapper);
        List<Long> bookIds = homeBooks.stream().map(HomeBook::getBookId).collect(Collectors.toList());
        //查询首页书籍
        if (!CollectionUtils.isEmpty(bookIds)) {
            LambdaQueryWrapper<BookInfo> bookInfoLambdaQueryWrapper = new LambdaQueryWrapper<>();
            bookInfoLambdaQueryWrapper.in(BookInfo::getId, bookIds);
            //小说信息可能无序
            List<BookInfo> bookInfos = bookInfoMapper.selectList(bookInfoLambdaQueryWrapper);
            //封装数据
            if (!CollectionUtils.isEmpty(bookInfos)){
                //将bookId为Key，BookInfo为Value进行Map封装
                Map<Long, BookInfo> bookInfoMap = bookInfos.stream().
                        collect(Collectors.toMap(BookInfo::getId, Function.identity()));
                return homeBooks.stream().map(v ->{
                    HomeBookRespDto bookRespDto = new HomeBookRespDto();
                    bookRespDto.setType(v.getType());
                    bookRespDto.setBookId(v.getBookId());
                    //根据有序bookId进行封装
                    BookInfo bookInfo = bookInfoMap.get(v.getBookId());
                    bookRespDto.setAuthorName(bookInfo.getAuthorName());
                    bookRespDto.setBookDesc(bookInfo.getBookDesc());
                    bookRespDto.setPicUrl(bookInfo.getPicUrl());
                    bookRespDto.setBookName(bookInfo.getBookName());
                    //返回升序小说信息
                    return bookRespDto;
                }).collect(Collectors.toList());

            }
        }
        //如果未查询到数据，返回空集合
        return Collections.emptyList();


//        return homeBookMapper.findHomeBooks();
    }
}
```

> 该数据库查询中未使用Mybatis进行连表查询：
>
> 单表查询简单、单表查询解耦、单表查询提高查询效率

## 随机查询数据（推荐列表）

1. 查询当前分类的所有小说信息
2. 生成随机数，进行随机选取四个小说信息进行显示

**推荐小说列表实现类**

```java
	private static final int REC_BOOK_COUNT = 4;

@Override
public RestResp<List<BookInfoRespDto>> recList(Long bookId) throws NoSuchAlgorithmException {
    //获取当前分类ID
    Long categoryId = bookInfoCacheManager.getBookInfoById(bookId.toString()).getCategoryId();
    //查询当前分类的所有bookID
    List<Long> lastUpdateIds = bookInfoCacheManager.getBookCategories(categoryId);
    //随机选出四个返回给前端显示
    ArrayList<BookInfoRespDto> respDtoList = new ArrayList<>();
    //随机数列表
    ArrayList<Integer> recIdIndexList = new ArrayList<>();
    //进行有效的四次添加
    int count = 0;
    //生成随机数的对象
    SecureRandom rand = SecureRandom.getInstanceStrong();
    while (count < REC_BOOK_COUNT) {
        
        int recIdIndex = rand.nextInt(lastUpdateIds.size());
        //随机数不同的话封装，相同的话继续生成随即数
        if (!recIdIndexList.contains(recIdIndex)) {
            recIdIndexList.add(recIdIndex);
            bookId = lastUpdateIds.get(recIdIndex);
            BookInfoRespDto bookInfoRespDto = bookInfoCacheManager.getBookInfoById(bookId.toString());
            respDtoList.add(bookInfoRespDto);
            count++;
        }
    }
    return RestResp.ok(respDtoList);
}
```

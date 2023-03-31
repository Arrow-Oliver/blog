---
title: "Novel Cloud小说项目笔记"
date: 2023-03-31T23:35:03+08:00
draft: false
categories: [项目,笔记]
tags: [Novel]
card: false
weight: 0
---

# 1.项目总体功能

## 项目模块

* 作家模块
* 小说模块
* 通用模块
* 资源模块
* 网关模块
* 主页模块
* 监控模块
* 新闻模块
* 支付模块
* 搜索模块
* 用户模块

## 项目技术

后端：

`Srping Boot`, `Spring Boot Admin`,`Lombok`

`Spring Cloud`, `Nacos`, `Spring Cloud Gateway`, `Elasticsearch`, `RabbitMQ`,`Docker`

`MySQL`,`MyBatis`, `Mybatis Dynamic SQL`,  `PageHelper`, `Sharding-JDBC`, `Redis`,`Redission`

## 包结构

~~~java
novel-cloud
├── novel-common -- 通用模块，供其他业务微服务模块依赖
├── novel-gen -- 持久层代码生成器，集成 Swagger
├── novel-gateway -- 基于 Spring Cloud Gateway 构建的网关服务
├── novel-monitor -- 基于 Spring Boot Admin 构建的监控中心
├── novel-search -- 基于 Elasticsearch 构建的搜索微服务
├── novel-file -- 基于 Aliyun OSS 构建的文件微服务
├── novel-home -- 门户首页微服务
├── novel-news -- 新闻中心微服务
    ├── news-api -- 实体类以及feginApi
    ├── news-service -- 内部和外部接口实现
├── novel-user -- 用户中心微服务
    ├── user-api -- 实体类以及feginApi
    ├── user-service -- 内部和外部接口实现
├── novel-author -- 作家中心微服务
├── novel-book -- 小说微服务
    ├── book-api -- 实体类以及feginApi
    ├── book-service -- 内部和外部接口实现
└── novel-pay -- 支付微服务
~~~

# 2.基本结构

## `Gateway`结构（重点）

~~~yml
spring:
  cloud:
    nacos:
      discovery:
        server-addr: 192.168.50.152:8848
        namespace: 49e71a76-832c-4cf3-8b60-6f89f7a46191
    gateway:
      routes:
        - id: home-route
          uri: lb://novel-home 
          predicates:
            - Path=/api/home/** 
          filters:
            #注意过滤器按顺序执行，下面的顺序不能打乱
            - SwaggerFilter
            - RewritePath=/api/(?<segment>.*), /$\{segment}                       
        - id: news-route
          uri: lb://news-service 
          predicates:
            - Path=/api/news/** 
          filters:
            #注意过滤器按顺序执行，下面的顺序不能打乱
            - SwaggerFilter
            - RewritePath=/api/(?<segment>.*), /$\{segment}
        - id: user-route
          uri: lb://user-service 
          predicates:
            - Path=/api/user/** 
          filters:
            #注意过滤器按顺序执行，下面的顺序不能打乱
            - SwaggerFilter
            - RewritePath=/api/(?<segment>.*), /$\{segment} 
        - id: book-route
          uri: lb://book-service 
          predicates:
            - Path=/api/book/** 
          filters:
            #注意过滤器按顺序执行，下面的顺序不能打乱
            - SwaggerFilter
            - RewritePath=/api/(?<segment>.*), /$\{segment}
        - id: search-route
          uri: lb://novel-search 
          predicates:
            - Path=/api/search/** 
          filters:
            #注意过滤器按顺序执行，下面的顺序不能打乱
            - SwaggerFilter
            - RewritePath=/api/(?<segment>.*), /$\{segment} 
        - id: monitor-route
          uri: lb://novel-monitor  
          predicates:
            - Path=/monitor/**  
          filters:
            - RewritePath=/monitor/(?<segment>.*), /$\{segment}
~~~

> 访问`/api/home/**` 接口，通过`RewritePath`过滤器将接口转发到`/home/**`

## `Controller`结构

```java
//外部通过gateway调用访问的接口
@RestController
@RequiredArgsConstructor
@RequestMapping("book")
@Slf4j
public class BookController {

    private final BookService bookService;

    private final RabbitTemplate rabbitTemplate;

    /**
     * 小说分类列表查询接口
     */
    @GetMapping("listBookCategory")
    public ResultBean<List<BookCategory>> listBookCategory(){
        return ResultBean.ok(bookService.listBookCategory());
    }
```

## `Service`结构

```java
@Service
@RequiredArgsConstructor
public class BookServiceImpl implements BookService {

    private final BookMapper bookMapper;

    private final BookIndexMapper bookIndexMapper;

    private final BookContentMapper bookContentMapper;

    private final BookCategoryMapper bookCategoryMapper;

    private final BookCommentMapper bookCommentMapper;

    private final UserFeignClient userFeignClient;

    @Override
    public List<BookCategory> listBookCategory() {
        //查询语句
        SelectStatementProvider selectStatementProvider = select(BookCategoryDynamicSqlSupport.id,
                BookCategoryDynamicSqlSupport.workDirection,
                BookCategoryDynamicSqlSupport.name)
                .from(BookCategoryDynamicSqlSupport.bookCategory)
                .orderBy(BookCategoryDynamicSqlSupport.sort)
                .build()
                .render(RenderingStrategies.MYBATIS3);
        //查询数据
        return bookCategoryMapper.selectMany(selectStatementProvider);
    }
}
```

## `Fegin`结构(重点)

```java
//首页小说信息显示ServiceImpl
@SneakyThrows
@Override
public List<HomeBookVO> listHomeBook() {
    String result = cacheService.get(CacheKey.INDEX_BOOK_SETTINGS_KEY);
    if (result == null || result.length() < Constants.OBJECT_JSON_CACHE_EXIST_LENGTH) {
        //查询首页数据库
        List<HomeBook> homeBooks = homeBookMapper.selectMany(
                select(HomeBookDynamicSqlSupport.homeBook.bookId, HomeBookDynamicSqlSupport.type)
                        .from(HomeBookDynamicSqlSupport.homeBook)
                        .orderBy(HomeBookDynamicSqlSupport.sort)
                        .build()
                        .render(RenderingStrategies.MYBATIS3));
        //查询小说详情信息
        List<Long> bookIds = homeBooks.stream().map(HomeBook::getBookId).collect(Collectors.toList());
        //---------------Fegin调用-------------------//
        List<Book> books = bookFeignClient.queryBookByIds(bookIds);
        Map<Long, Book> bookMap = books.stream()
                .collect(Collectors.toMap(Book::getId, Function.identity(), (v1, v2) -> v2));
        //封装vo
        List<HomeBookVO> list = homeBooks.stream().map(v -> {
            HomeBookVO homeBookVO = new HomeBookVO();
            BeanUtils.copyProperties(v, homeBookVO);
            Book book = bookMap.get(v.getBookId());
            if (book != null) {
                BeanUtils.copyProperties(book, homeBookVO);
            }
            return homeBookVO;
        }).collect(Collectors.toList());
        result = new ObjectMapper().writeValueAsString(list);
        cacheService.setObject(CacheKey.INDEX_BOOK_SETTINGS_KEY, result);
    }
    return new ObjectMapper().readValue(result, List.class);
}
```

**当前`Home`服务下`fegin`包**

```java
@FeignClient("book-service")
public interface BookFeignClient extends BookApi {
}
```

**`Book`服务下的`BookApi`**

```java
public interface BookApi {

    /**
     * 查询多个小说详情
     * @param ids
     * @return
     */
    @GetMapping("api/book/queryBookByIds")
    List<Book> queryBookByIds(@RequestParam("ids") List<Long> ids);
}
```

**`BookApi`实现类接口**

```java
//内部通过Fegin调用的接口
@RestController
@RequestMapping(("api/book"))
@RequiredArgsConstructor
public class BookApi {

    private final BookService bookService;

    /**
     * 查询多个小说详情
     */
    @GetMapping("queryBookByIds")
    List<Book> queryBookByIds(@RequestParam("ids") List<Long> ids){
        return bookService.queryBookByIds(ids);
    }
}
```

`BookApiFallback`

```java
//远程服务调用失败，反馈到此实现类
public class BookApiFallback implements BookApi {
    @Override
    public List<Book> queryBookByIds(List<Long> ids) {
        return new ArrayList<>();
    }
}
```

# 3.重点技术

## `rabbitMQ`应用

### 1.队列和交换机注册

```java
@Configuration
@ConditionalOnProperty(prefix = "spring.rabbitmq", name = "host", matchIfMissing = false)
public class RabbitConfig {


    /**
     * 更新数据库队列
     */
    @Bean
    public Queue updateDbQueue() {
        return new Queue("UPDATE-DB-QUEUE", true);
    }

    /**
     * 更新数据库队列
     */
    @Bean
    public Queue updateEsQueue() {
        return new Queue("UPDATE-ES-QUEUE", true);
    }


    /**
     * 增加点击量交换机
     */
    @Bean
    public FanoutExchange addVisitExchange() {
        return new FanoutExchange("ADD-BOOK-VISIT-EXCHANGE");
    }

    /**
     * 更新搜索引擎队列绑定到增加点击量交换机中
     */
    @Bean
    public Binding updateEsBinding() {

        return BindingBuilder.bind(updateEsQueue()).to(addVisitExchange());
    }

    /**
     * 更新数据库绑定到增加点击量交换机中
     */
    @Bean
    public Binding updateDbBinding() {
        return BindingBuilder.bind(updateDbQueue()).to(addVisitExchange());
    }


}
```

### 2.发送消息

```java
/**
 * 点击量新增接口
 */
@PostMapping("addVisitCount")
public ResultBean addVisitCount(@ApiParam("小说ID") @RequestParam("bookId") Long bookId) {
    rabbitTemplate.convertAndSend("ADD-BOOK-VISIT-EXCHANGE", null, bookId);
    return ResultBean.ok();
}
```

`ADD-BOOK-VISIT-EXCHANGE`交换机为**全局转发**交换机

### 3.消息监听并且处理

#### DB Queue

```java
/**
 * 更新数据库
 * 流量削峰，每本小说累积10个点击更新一次
 */
@SneakyThrows
@RabbitListener(queues = {"UPDATE-DB-QUEUE"})
public void updateDb(Long bookId, Channel channel, Message message) {
    //加锁（分布式）
    log.debug("收到的消息：{}", bookId);
    RLock lock = redissonClient.getLock("addVisitCountToDb");
    lock.lock();
    try {
        //查询缓存点击量
        Integer visitCount = (Integer) cacheService.getObject(CacheKey.BOOK_ADD_VISIT_COUNT + bookId);
        if (visitCount == null) {
            visitCount = 0;
        }
        cacheService.setObject(CacheKey.BOOK_ADD_VISIT_COUNT + bookId, ++visitCount);
        //达到10次更新数据库
        if (visitCount >= Constants.ADD_MAX_VISIT_COUNT) {
            bookService.addVisitCount(bookId, visitCount);
            cacheService.del(CacheKey.BOOK_ADD_VISIT_COUNT + bookId);
        }
    } catch (Exception e) {
        log.error("更新数据库失败：{}", e);
    }
    lock.unlock();

    Thread.sleep(2 * 1000);

}
```

#### ES Queue

```java
@RabbitListener(queues = {"UPDATE-ES-QUEUE"})
public void updateEs(Long bookId, Channel channel, Message message) {

    RLock lock = redissonClient.getLock("addVisitCountToEs");
    lock.lock();
    //查询缓存是否已经修改
    try {
        //如未修改则添加文档到es
        if (cacheService.get(CacheKey.ES_IS_UPDATE_VISIT + bookId) == null) {
            cacheService.set(CacheKey.ES_IS_UPDATE_VISIT + bookId, "1", 60 * 60);
            Thread.sleep(1000 * 5);
            Book book = bookFeignClient.queryBookByIds(Collections.singletonList(bookId)).get(0);
            //如果当前book存在则更新book
            searchService.importToEs(book);
        }
    } catch (Exception e) {
        cacheService.del(CacheKey.ES_IS_UPDATE_VISIT + bookId );
        log.error("更新搜索引擎失败", e);

    }
    //修改过则直接返回
    lock.unlock();
}
```

## `Elasticsearch`应用

### 1. 查询结果业务

```java
    /**
     * 结果处理
     * @param params
     * @param page
     * @param pageSize
     * @return
     */
    @SneakyThrows
    @Override
    public PageBean<EsBookVO> searchBook(SearchParamVO params, int page, int pageSize) {
        if (params.getUpdatePeriod() != null) {
            long cur = System.currentTimeMillis();
            long period = params.getUpdatePeriod() * 24 * 3600 * 1000;
            long time = cur - period;
            params.setUpdateTimeMin(new Date(time));
        }

        ArrayList<EsBookVO> bookList = new ArrayList<>();
        SearchSourceBuilder searchSourceBuilder = buildQuery(params, page, pageSize);
        //查询结果
        Search search = new Search.Builder(searchSourceBuilder.toString()).addIndex(INDEX).build();
        log.debug(search.toString());
        SearchResult result;
        result = jestClient.execute(search);
        if(result.isSucceeded()){
            log.debug(result.getJsonString());
            Map resultMap = new ObjectMapper().readValue(result.getJsonString(), Map.class);
            if(resultMap.get("hits") != null){
                Map hitsMap = (Map) resultMap.get("hits");
                if (hitsMap.size() > 0 && hitsMap.get("hits") != null) {
                    //查询结果封装
                    List hitsList = (List) hitsMap.get("hits");
                    if(hitsList.size() > 0 && result.getSourceAsString() != null){
                        JavaType jt = new ObjectMapper().getTypeFactory().constructParametricType(ArrayList.class, EsBookVO.class);
                        bookList = new ObjectMapper().readValue("[" + result.getSourceAsString() + "]", jt);
                    //查询高亮数据
                        if(bookList != null){
                            for (int i = 0; i < hitsList.size(); i++) {
                                hitsMap = (Map) hitsList.get(i);
                                Map highlightMap = (Map) hitsMap.get("highlight");
                                if(highlightMap != null && highlightMap.size() > 0){
                                    List<String> authorNameList = (List<String>) highlightMap.get("authorName");
                                    if (!CollectionUtils.isEmpty(authorNameList)) {
                                        bookList.get(i).setAuthorName(authorNameList.get(0));
                                    }
                                    List<String> bookNameList = (List<String>) highlightMap.get("bookName");
                                    if (!CollectionUtils.isEmpty(bookNameList)) {
                                        bookList.get(i).setBookName(bookNameList.get(0));
                                    }
                                    List<String> bookDescList = (List<String>) highlightMap.get("bookDesc");
                                    if (!CollectionUtils.isEmpty(bookDescList)) {
                                        bookList.get(i).setBookDesc(bookDescList.get(0));
                                    }
                                    List<String> lastIndexNameList = (List<String>) highlightMap.get("lastIndexName");
                                    if (!CollectionUtils.isEmpty(lastIndexNameList)) {
                                        bookList.get(i).setLastIndexName(lastIndexNameList.get(0));
                                    }
                                    List<String> catNameList = (List<String>) highlightMap.get("catName");
                                    if (!CollectionUtils.isEmpty(catNameList)) {
                                        bookList.get(i).setCatName(catNameList.get(0));
                                    }
                                }

                            }

                        }


                    }

                }
            }

            return new PageBean<>(page,pageSize,total.longValue(),bookList);
        }
        throw new BusinessException(ResponseStatus.ES_SEARCH_FAIL);

}

    /**
     * 查询条件
     * @param params
     * @param page
     * @param pageSize
     * @return
     * @throws java.io.IOException
     */
    private SearchSourceBuilder buildQuery(SearchParamVO params, int page, int pageSize) throws java.io.IOException {
        //bool多条件
        BoolQueryBuilder boolQuery = QueryBuilders.boolQuery();
        if (StringUtils.hasText(params.getKeyword())) {
            boolQuery.must(QueryBuilders.queryStringQuery(params.getKeyword()));
        }
        //filter过滤条件
        if (params.getWorkDirection() != null) {
            boolQuery.filter(QueryBuilders.termQuery("workDirection", params.getWorkDirection()));
        }
        if (params.getCatId() != null) {
            boolQuery.filter(QueryBuilders.termQuery("catId", params.getCatId()));
        }
        if (params.getBookStatus() != null) {
            boolQuery.filter(QueryBuilders.termQuery("bookStatus", params.getBookStatus()));
        }
        if (params.getWordCountMin() == null) {
            params.setWordCountMin(0);
        }
        if (params.getWordCountMax() == null) {
            params.setWordCountMax(Integer.MAX_VALUE);
        }
        boolQuery.filter(QueryBuilders.rangeQuery("wordCount")
                .gte(params.getWordCountMin())
                .lte(params.getWordCountMax()));
        if (params.getUpdateTimeMin() != null) {
            boolQuery.filter(QueryBuilders.rangeQuery("lastIndexUpdateTime").gte(params.getUpdateTimeMin()));
        }
        //查询结果总数
        SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();
        searchSourceBuilder.query(boolQuery);


        Count count = new Count.Builder().addIndex(INDEX)
                .query(searchSourceBuilder.toString()).build();
        CountResult results = jestClient.execute(count);
        total = results.getCount();

        //分页条件
        searchSourceBuilder.from((page - 1) * pageSize);
        searchSourceBuilder.size(pageSize);
        //排序
        if (params.getSort() != null) {
            searchSourceBuilder.sort(StringUtil.camelName(params.getSort()), SortOrder.DESC);
        }
        //高亮条件
        HighlightBuilder highlightBuilder = new HighlightBuilder();
        highlightBuilder.field("authorName");
        highlightBuilder.field("bookName");
        highlightBuilder.field("bookDesc");
        highlightBuilder.field("lastIndexName");
        highlightBuilder.field("catName");
        highlightBuilder.preTags("<span style='color:red'>").postTags("</span>");
        highlightBuilder.fragmentSize(20000);
        searchSourceBuilder.highlighter(highlightBuilder);
        return searchSourceBuilder;
    }
```

> 1. `DML`语句编写
> 2. 查询结果处理

### 2. 导入文档业务

```java
@SneakyThrows
@Override
public void importToEs(Book book) {
    EsBookVO esBookVO = new EsBookVO();
    BeanUtils.copyProperties(book, esBookVO, "lastIndexUpdateTime");
    IndexRequest request = new IndexRequest(INDEX);
    request.id(book.getId().toString());
    request.source(new ObjectMapper().writeValueAsString(esBookVO), XContentType.JSON);
    IndexResponse index = restHighLevelClient.index(request, RequestOptions.DEFAULT);
    log.debug(index.getResult().toString());
}
```

> 1. 文档数据处理
> 2. 新增DML语句编写
> 3. 进行新增

### 3. 循环导入业务

```java
/**
 * 1分钟导入一次
 */
@Scheduled(fixedRate = 1000 * 60)
public void saveToEs() {
    //加锁
    RLock lock = redissonClient.getLock("saveToEs");
    lock.lock();
    //查询小说详情
    try {
        Date lastDate = null;
        String date = (String) cacheService.getObject(CacheKey.ES_LAST_UPDATE_TIME);
        if (date != null) {
            lastDate = new SimpleDateFormat("yyyy-MM-dd").parse(date);
        }

        if (lastDate == null) {
            lastDate = new SimpleDateFormat("yyyy-MM-dd").parse("2020-01-01");
        }
        List<Book> books = bookFeignClient.queryBookByMinUpdateTime(lastDate, 100);

        for (Book book : books) {
            //存入es
            searchService.importToEs(book);
            lastDate = book.getUpdateTime();
            Thread.sleep(5 * 1000);
        }

        cacheService.setObject(CacheKey.ES_LAST_UPDATE_TIME,
                new SimpleDateFormat("yyyy-MM-dd").format(lastDate));
    } catch (Exception e) {
        log.error(e.getMessage(), e);
    }
    //解锁
    lock.unlock();


}
```

> 1. 获取当前更新时间；如果没有得到当前时间，将获取默认更新时间
> 2. 更新当前时间以后的文档信息

## `Cache`结构

~~~java
cache --缓存
├── impl
    ├── RedisServiceImpl --缓存实现类
├── CacheKey --常量值
├── CacheService --缓存接口
~~~

**`CacheService`**

```java
public interface CacheService {

   /**
    * 根据key获取缓存的String类型数据
    */
   String get(String key);

   /**
    * 设置String类型的缓存
    */
   void set(String key, String value);

   /**
    * 设置一个有过期时间的String类型的缓存,单位秒
    */
   void set(String key, String value, long timeout);
   
   /**
    * 根据key获取缓存的Object类型数据
    */
   Object getObject(String key);
   
   /**
    * 设置Object类型的缓存
    */
   void setObject(String key, Object value);
   
   /**
    * 设置一个有过期时间的Object类型的缓存,单位秒
    */
    void setObject(String key, Object value, long timeout);

   /**
    * 根据key删除缓存的数据
    */
   void del(String key);

   
   /**
    * 判断是否存在一个key
    * */
   boolean contains(String key);
   
   /**
    * 设置key过期时间
    * */
   void expire(String key, long timeout);


}
```

**`CacheKey`**

```java
public interface CacheKey {

    /**
     * 首页小说设置
     * */
    String INDEX_BOOK_SETTINGS_KEY = "indexBookSettingsKey";

    /**
     * 首页新闻
     * */
    String INDEX_NEWS_KEY = "indexNewsKey";

    /**
     * 首页点击榜单
     * */
    String INDEX_CLICK_BANK_BOOK_KEY = "indexClickBankBookKey";

    /**
     * 首页友情链接
     * */
    String INDEX_LINK_KEY = "indexLinkKey";

    /**
     * 首页新书榜单
     * */
    String INDEX_NEW_BOOK_KEY = "indexNewBookKey";


    /**
     * 首页更新榜单
     * */
    String INDEX_UPDATE_BOOK_KEY = "indexUpdateBookKey";

    /**
     * 模板目录保存key
     * */
    String TEMPLATE_DIR_KEY =  "templateDirKey";;

    /**
     * 正在运行的爬虫线程存储KEY前缀
     * */
    String RUNNING_CRAWL_THREAD_KEY_PREFIX = "runningCrawlTreadDataKeyPrefix";

    /**
     * 上一次搜索引擎更新的时间
     * */
    String ES_LAST_UPDATE_TIME = "esLastUpdateTime";

    /**
     * 搜索引擎转换锁
     * */
    String ES_TRANS_LOCK = "esTransLock";

    /**
     * 上一次搜索引擎是否更新过小说点击量
     * */
    String ES_IS_UPDATE_VISIT = "esIsUpdateVisit";

    /**
     * 累积的小说点击量
     * */
    String BOOK_ADD_VISIT_COUNT = "bookAddVisitCount";
}
```

**`RedisServiceImpl`**

~~~java
@ConditionalOnProperty(prefix = "spring.redis", name = "host", matchIfMissing = false)
@RequiredArgsConstructor
@Service
public class RedisServiceImpl implements CacheService {

	private final StringRedisTemplate stringRedisTemplate;

	private final RedisTemplate<Object,  Object>  redisTemplate;


	@Override
	public String get(String key) {
		return stringRedisTemplate.opsForValue().get(key);
	}

	@Override
	public void set(String key, String value) {
		stringRedisTemplate.opsForValue().set(key,value);
	}

	@Override
	public void set(String key, String value, long timeout) {
		stringRedisTemplate.opsForValue().set(key,value,timeout, TimeUnit.SECONDS);

	}

	@Override
	public Object getObject(String key) {
		return redisTemplate.opsForValue().get(key);
	}

	@Override
	public void setObject(String key, Object value) {
		redisTemplate.opsForValue().set(key,value);
	}

	@Override
	public void setObject(String key, Object value, long timeout) {
		redisTemplate.opsForValue().set(key,value,timeout, TimeUnit.SECONDS);
	}

	@Override
	public void del(String key) {
		redisTemplate.delete(key);
		stringRedisTemplate.delete(key);
	}

	@Override
	public boolean contains(String key) {
		return redisTemplate.hasKey(key) || stringRedisTemplate.hasKey(key);
	}

	@Override
	public void expire(String key, long timeout) {
		redisTemplate.expire(key,timeout, TimeUnit.SECONDS);
		stringRedisTemplate.expire(key,timeout, TimeUnit.SECONDS);
	}

}
~~~

## 基础`Controller`(User)

用于`UserController`中`token`的解析以及获取

```java
public class BaseController {

    protected JwtTokenUtil jwtTokenUtil;


    /**
     * 获取登陆token
     * */
    protected String getToken(HttpServletRequest request){
        String token = CookieUtil.getCookie(request,"Authorization");
        if(token != null){
            return token;
        }
        return request.getHeader("Authorization");
    }

    /**
     * 获取登陆用户信息
     * */
    protected UserDetails getUserDetails(HttpServletRequest request) {
        String token = getToken(request);
        if(StringUtils.isBlank(token)){
            return null;
        }else{
            return jwtTokenUtil.getUserDetailsFromToken(token);
        }
    }

    @Autowired
    public void setJwtTokenUtil(JwtTokenUtil jwtTokenUtil) {
        this.jwtTokenUtil = jwtTokenUtil;
    }
}
```

**`UserController`**

```java
public class UserController extends BaseController {
    
}
```



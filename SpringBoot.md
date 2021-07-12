

# spring boot 中的@ConfigurationProperties

1. 首先，这个注解可以用于类型安全的属性注入。用在某一个类的上面。

```java
@Component
@ConfigurationProperties(prefix = "url")
public class MicroServiceUrl implements ReadProperties{
    private String orderUrl;
    private String userUrl;
    private String shoppingUrl;

    public String getOrderUrl() {
        return orderUrl;
    }

    public void setOrderUrl(String orderUrl) {
        this.orderUrl = orderUrl;
    }

    public String getUserUrl() {
        return userUrl;
    }

    public void setUserUrl(String userUrl) {
        this.userUrl = userUrl;
    }

    public String getShoppingUrl() {
        return shoppingUrl;
    }

    public void setShoppingUrl(String shoppingUrl) {
        this.shoppingUrl = shoppingUrl;
    }
}
```

2. 可以在@bean进行对象注入时使用。此时会按照前缀为url去配置文件中查找。然后注入到新创建的对象中。

   ```java
   @Bean
   @ConfigurationProperties(prefix = "url")
   public MicroServiceUrl geren1(){
       MicroServiceUrl microServiceUrl = new MicroServiceUrl();
       return microServiceUrl;
   }
   ```

   

# spring boot配置路径映射

有的时候，我们有一个静态页面，不需要动态的渲染，到了某一个controller之后就直接return视图名字或者直接返回modelandview的话，那么我们就不需要再去写一个controller接口来接收这个请求。可以使用路径映射的方式来做，当然，这个不是spring boot特有的。

具体代码

```java
// 首先继承MVC的自动配置接口 WebMvcConfigurer接口

@override 
public void addViewControllers(ViewControllerRegistry registry){
    registry.addViewController("/javaboy").setViewName("hello");
    							//请求接口地址		// 视图名
}
```

# spring boot 参数类型转换

假设你要从前端传入一个日期到后端，那么后端要怎样接收这个日期并且转换为对象呢？

我们需要定义一个日期类型转化器就行了。

```java
@Component
public class DateConverter implements Converter<String, Date> {
    
    SimpleDateFormat sdf = new SimpleDateFormat();
    @Override
    public Date convert(String s) {
        if (null != s && !"".equals(s)){
            try {
                return sdf.parse(s);
            } catch (ParseException e) {
                e.printStackTrace();
            }
        }
        return null;
    }
}
```

# spring boot 如何整合aop

1. 导入aop依赖
2. 提供一个切面类

代码示例：

```java

// 另外几个方法没写，其实有五个一共
@Component
@Aspect
public class AopTest {
    /**
     * 这个方法里面不需要写任何东西。那么，这个方法的作用是来定义切面的，具体去切那个地方的类。
     */
    @Pointcut("execution(* com.jie.controller.*.*(..) )")
    public void pc1(){}

    @Before(value = "pc1()")
    public void before(JoinPoint jp){
        String name = jp.getSignature().getName();
        System.out.println("before -- "+ name);
    }
    
	//......省略其他几个切面方法 
    
    @Around("pc1()")
    public Object around(ProceedingJoinPoint pjp) throws Throwable {
        
        // 当调用了这个之后，被切面所切的方法就会被执行。在这句之前的就跟上面的前置通知一样 。
        Object proceed = pjp.proceed();
        // 如果在这里写的代码就等于是后置通知
        
        // 这个就是方法的返回值，同样的，我们如果在这里篡改了返回值，那么原来方法的返回值也会被修改。
        return proceed;
    }

}

```

# spring boot 自定义欢迎页和欢迎图标

欢迎页查找规则：

​			在static目录下查找index.html页面，找不到就去模板目录下查找。

将favicon.ico 图标文件放在static目录或者classpath:/目录下就行。

# spring boot 如何去除自定义配置

```java
@SpringBootApplication(exclude = WebMvcAutoConfiguration.class)
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

 # 文件上传

核心代码如下：
```java
public static final String BASE_PATH = "/test/";
 
    // 主要就是用这个类，如果多文件上传的话，可以将这个改为multipartfile 的数组的形式。当然也和前端的写法有关。
     // 如果前端有多个input框，那么，就可以写多个MultipartFile参数来接收就行。不同的name，对应就行。
    public static String upload(MultipartFile imageFile) {
 
        if (imageFile.isEmpty()) {
            return null;
        }
        String filename = imageFile.getOriginalFilename();
        
        String ext= null;
        if(filename.contains(".")){
            ext = filename.substring(filename.lastIndexOf("."));
        }else{
            ext = "";
        }
        
        String uuid =  UUID.randomUUID().toString().replaceAll("-", "");
        String nfileName = uuid + ext;
        String dirPath = DateFormatUtils.format(new Date(), "yyyyMMdd");
        String filepath = BASE_PATH.endsWith("/") ? BASE_PATH+dirPath : BASE_PATH+"/"+dirPath;
        File targetFile = new File(filepath, nfileName);
        if (!targetFile.exists()) {
            targetFile.mkdirs();
        } else {
            targetFile.delete();
        }
        try {
            // 核心代码
            imageFile.transferTo(targetFile);
        } catch (IllegalStateException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }
 
        String accessUrl =  "/"+nfileName;
        logger.debug("上传文件成功 URL：" + nfileName);
        return accessUrl;
    }

```

# springboot异常处理：

异常处理都需要自定义exceptionHandler(Exception e) ，并且返回给页面定制的结果。ResponseEntity<ErrorResponse>在下面。就是要返回的类
 新建异常处理类
我们只需要在类上加上@ControllerAdvice注解这个类就成为了全局异常处理类，当然你也可以通过 assignableTypes 指定特定的 Controller 类，让异常处理类只处理特定类抛出的异常。

```java
//src/main/java/com/twuc/webApp/exception/GlobalExceptionHandler.java
/**
 * @author shuang.kou
 */
@ControllerAdvice(assignableTypes = {ExceptionController.class})
@ResponseBody
public class GlobalExceptionHandler {

    ErrorResponse illegalArgumentResponse = new ErrorResponse(new IllegalArgumentException("参数错误!"));
    ErrorResponse resourseNotFoundResponse = new ErrorResponse(new ResourceNotFoundException("Sorry, the resourse not found!"));

    @ExceptionHandler(value = Exception.class)// 拦截所有异常, 这里只是为了演示，一般情况下一个方法特定处理一种异常
    public ResponseEntity<ErrorResponse> exceptionHandler(Exception e) {

        if (e instanceof IllegalArgumentException) {
            return ResponseEntity.status(400).body(illegalArgumentResponse);
        } else if (e instanceof ResourceNotFoundException) {
            return ResponseEntity.status(404).body(resourseNotFoundResponse);
        }
        return null;
    }
}

```

```java
 @GetMapping("/resourceNotFoundException")
    public void throwException2() {
//        throw new ResourceNotFoundException();
        throw new ResponseStatusException(HttpStatus.NOT_FOUND, "Sorry, the resourse not found!", new ResourceNotFoundException());
    }

```

# spring boot 返回数据

```java
 // 重写这个方法，在下面。用新的返回方式。
    @GetMapping("/dept/{id}")
    public Department getDepartment(@PathVariable("id") Integer id){
        return departmentMapper.getDeptById(id);
    }

    @GetMapping("/dept/{id}")
    public ResponseEntity<Department> getDepartment(@PathVariable("id") Integer id){
        Department deptById = departmentMapper.getDeptById(id);
        return ResponseEntity.status(HttpStatus.OK).body(deptById);
    }
```

这两种方式都可以返回一个json数据。
```json
{"id":1,"departmentName":"AA"}
```

# 请求参数中有多个同名参数，不知道封装给谁

当前端的请求有多个相同名字的参数的时候，后端接收会发生错误。此时可以用@controlleradvise注解来进行处理。

> 问题情境

有一个book类，一个author类。两个类都有name属性，前端post请求，body中有两个相同的name参数，那么，后端的controller接口的参数应该怎么写？

> 处理方法

在被@controlleradvise标注的类中，用@InitBinder("a") 前置处理器，对参数进行处理。并且在controller接口的方法的参数前面标注@modelattribute注解，跟那个前置处理器进行绑定。

当然，最好不要让这种情况发生，一般这种情况发生就是接口或者类的设计有问题。

# spring boot整合 jdbc template

1. 在pom.xml中添加相关依赖

   ```xml
   <dependency>
       <groupId>com.alibaba</groupId>
       <artifactId>druid</artifactId>
       <version>1.0.16</version>
   </dependency>
   
   <dependency>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-starter-jdbc</artifactId>
   </dependency>
   ```

2. 配置数据库相关信息，数据库连接池，用户名，密码
3. 直接将JdbcTemlate注入到要使用的地方就行。spring已经将它注入到IOC容器中了。
4. 直接使用，具体可以查看api。有一点需要注意的是，增删改都是用update方法（重载）。而查询可以用query，同样可以传入参数。那么，在将查询结果映射到实体类的时候，需要手动写一个rowmapper。当数据库字段和实体类字段完全相同的时候，可以直接new一个BeanPropertyRowMapper<>(实体类.class)

# spring boot 整合多数据源

>  多数据源概念

当有多个数据库，然后多个jdbc template来连接多个数据库的时候，那么就需要配置多个数据源。此时注意，原来的自动配置会失效。需要自己去将JdbcTemlate注入到IOC容器中。并且手动配置数据源。那么这里就涉及到一个问题。

当有多个JdbcTemlate的时候，我们要使用的时候如果直接用@Autowired进行注入的话，因为这个注解时按照类型进行查找的。那么就会找到多个，那么会报错。此时，我们可以用@resource（name = "注入到IOC容器中的名字"），这个注解来注入。

此时又有一个问题。那么，我们在用@bean进行注入IOC容器的时候，名字是什么呢？

**默认情况下，使用 @Bean声明一个bean，bean的名称由方法名决定。此外，可以通过@Bean注解里面的name属性主动设置bean的名称。**

注入指定名称的bean

```java
@Autowired                         // 或者用@resource注解
@Qualifier("queue-test")
private Queue queue;
```



# spring boot整合 mybatis

1. 导入依赖，写配置

2. 写接口，写接口对应的xml文件。

   **注意这里有个小问题。就是xml文件的放置问题。这个主要是maven在打包的时候的问题。**

   **如果放在上面的类包中，默认情况下是会忽略掉的。需要放到resource目录下去，而且，要包名的层级和interface的层级一样。而且，此时在resource的目录下创建目录的时候需要注意，不能直接org.jie.data.mapper这样一次性创建好。这个和在类包那里创建不一样。需要自己一次次的按照层级去创建。**

   **当然，如果你非要放在和java类包同级的目录下也是可以的。![image-20201128151358712](C:\Users\Jieqiyue\AppData\Roaming\Typora\typora-user-images\image-20201128151358712.png)**

   

   就是上面那样。此时需要告诉maven打包的时候需要去找到这个位置。在pom.xml文件中配置一下就行

```xml
<build>
    <resources>
        <resource>
            <directory>src/main/java</directory>
            <includes>
                <include>**/*.xml</include>
            </includes>
        </resource>
        
        // 下面这个的话，就是把resources目录一起加到打包的的target里面去
        <resource>
            <directory>src/main/resources</directory>
        </resource>
    </resources>
</build>
```

其实还有第三种，比较推荐。

放在resources目录下创建一个mapper目录，然后在application.properties下配置一下，mybatis.mapper-location=classpath:/mapper/*.xml



# spring boot 整合 mybatis多数据源

和上面差不多。只要配置不同的sqlsessionfactory，然后扫描不同的包就行了。关键就是，哪个包被哪个数据源去扫描。



# spring boot 整合jpa

主要是学习hibernate整合spring boot，因为jpa是一个规范。提供了一些通用的api接口。而hibernate是去实现它的。详细的可以看

# springboot整合redis

需要注意的一个点是，新版本的redis要远程连接必须要配置密码。2.1.5及以上的springboot版本必须结合springsecurity才能进行redis 的连接。

- 首先进行以下基础配置

```properties
spring.redis.host=127.0.0.1
spring.redis.database=0
spring.redis.port=6379
spring.redis.password=123
```

现在就可以使用了，其实很简单。然后springboot帮我们自动配置了。RedisAutoConfiguration.java类是自动配置的。

这里提供了两个可以使用的类。将它们自动注入到需要使用的地方就行了。

```java
    @ConditionalOnSingleCandidate(RedisConnectionFactory.class)
    public RedisTemplate<Object, Object> redisTemplate(RedisConnectionFactory 				redisConnectionFactory) {
        RedisTemplate<Object, Object> template = new RedisTemplate();
        template.setConnectionFactory(redisConnectionFactory);
        return template;
    }

    @Bean
    @ConditionalOnMissingBean
    @ConditionalOnSingleCandidate(RedisConnectionFactory.class)
    public StringRedisTemplate stringRedisTemplate(RedisConnectionFactory 					redisConnectionFactory) {
        StringRedisTemplate template = new StringRedisTemplate();
        template.setConnectionFactory(redisConnectionFactory);
        return template;
    }
```

# 项目工具

devtools

添加依赖就行。pom文件里面的options选项是指是否传播到其它模块 。热部署。重新编译，并且重新运行。

可以自己配置让idea自动编译。及时编译，并重启。并且，默认情况下静态资源文件的修改，并不会导致项目的重启，此时我们可以在properties文件中来进行配置。将通过spring.devtools.restart.addition-paths=/src/main/resource/**来进行指定。	但是，即时重启并没有什么意义，并且消耗电能资源。

# spring  boot 单元测试

json测试

# spring cache

spring cache 是spring 的一种规范。（有具体的实现，比如说redis，ehcache）

- 将spring cache的依赖添加进去，添加依赖并不能使用cache。spring cache只是一个规范。
- 然后配置一个spring cache的名字，在properties配置文件中，写spring.cache.cache-names=xxx
- 在启动类上加上@EnableCaching注解，表示启动缓存
- 在需要缓存的方法上加上@Cacheable(cacheNames = "xxx"),默认情况下方法的参数是key，返回值是value。那个cachenames指定就是先前在properties配置文件里面设置的值。
- 当执行过一次方法后，在redis中就能够看到有key/value存入。
- 被缓存的对象需要实现序列化接口。
- 当然，不需要在每一个方法上面都加指定缓存名字等的重复操作。如果有可以抽取出来的公共操作，能用一个@CacheConfig注解标在类上，去指定一些公共的东西，这样在这个类中的每一个方法（标注了缓存注解）都可以使用在这个@CacheConfig所配置的东西。

> 细节

当参数有多个的时候，多个参数都会被当做key进行存储。只有当所有参数都相同的时候，才会被认为是命中缓存。

当然，也可以直接指定以哪个参数作为key。在@Cacheable注解上指定key。形式如下：

```java
@Cacheable(key="#id")
// 表示指定以参数id，作为key
// 上面是自定义的，其实还有一些内置的对象。一共有六个。比如说args ... 下面这个是指被调用的方法的名字，注意这个method是内置的。
@Cacheable(key="#method.name")
```

当key比较复杂时可以自定义key生成器。实现KeyGenerator接口就行。然后在注解里面指定哪个key生成器。默认bean的名字就类名首字母小写。

当有删除请求的需求，并且已经把数据库中的数据删除之后，可能在redis中还是存在刚刚缓存的。所以会查出一个已经删除的数据。此时可以使用@CacheEvict(cacheNames="xxx"),将这个注解放在某一个方法上面。表示会按照方法的参数作为key，去数据库中删除对应的key/value。

更新也是一样，可能会查出脏数据。此时可以使用@CachePut();加上这个注解之后，就会自动更新缓存中的数据。

# spring security 配置多数据源

在spring security的多种数据源都是被封装成UserDetailsService接口。这个接口有如下的实现类。

[![DhSJ3D.png](https://s3.ax1x.com/2020/12/01/DhSJ3D.png)](https://imgchr.com/i/DhSJ3D)

对应不同的数据源。也可以自己去实现这个接口。

如果使用了JdbcUserDetailsManager那么必须要用它提供的数据库模型。因为是预定义好的SQL语句。

- 基于内存的数据库

  ![image-20201201173818015](C:\Users\Jieqiyue\AppData\Roaming\Typora\typora-user-images\image-20201201173818015.png)

- mysql的（非自定义，不够灵活）

![image-20201201174050011](C:\Users\Jieqiyue\AppData\Roaming\Typora\typora-user-images\image-20201201174050011.png)

- mysql（自定义，重要）

我们自定义数据源主要是要去实现UserDetailsService接口。对于用户实体类，springsecurity允许我们自己去定义，但是必须要去实现UserDetails接口，这个接口就是一个规范。



首先，你的用户实体类需要实现UserDetails接口。

```java
@Entity(name = "t_user")
public class User implements UserDetails {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String username;
    private String password;
    private boolean accountNonExpired;
    private boolean accountNonLocked;
    private boolean credentialsNonExpired;
    private boolean enabled;
    @ManyToMany(fetch = FetchType.EAGER,cascade = CascadeType.PERSIST)
    private List<Role> roles;
    
    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        List<SimpleGrantedAuthority> authorities = new ArrayList<>();
        for (Role role : getRoles()) {
            authorities.add(new SimpleGrantedAuthority(role.getName()));
        }
        return authorities;
    }
    
    // 注意上面这个方法，用户有角色的时候，需要自己转换，我们一般在数据库中定义的是三张表，用户表，角色表，用户角色关联表。而且，角色在springsecurity中必须要以ROLE_admin这种形式存放，就是说要以ROLE_开头。那么我们在创建new SimpleGrantedAuthority(role.getName())的时候就要注意，是不是传入的参数是这种新式的。springsecurity就是通过这个角色来判断登陆的这个用户是否有权利去访问某一个接口（当然，这个权利也是自己配置的）。就是要满足那个形式。可以看看下面这个表的设计
```

![image-20201202192846804](C:\Users\Jieqiyue\AppData\Roaming\Typora\typora-user-images\image-20201202192846804.png)

第二，需要一个实现了UserDetailsService接口的类

```java
@Service
public class UserService implements UserDetailsService {
    @Autowired
    UserDao userDao;

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        User user = userDao.findUserByUsername(username);
        if (user == null) {
            throw new UsernameNotFoundException("用户不存在");
        }
        return user;
    }
}
```

第三，将它添加到认证中

```java
@Autowired
UserService userService;
@Override
protected void configure(AuthenticationManagerBuilder auth) throws Exception {
    auth.userDetailsService(userService);
}
```

此时，这个就是自定义的认证。	

# springsecurity 自动登录

使用起来很简单，只需要在配置的时候添加如下就行了。

```java
     http.authorizeRequests()
                .anyRequest().authenticated()
                .and()
                .rememberMe()
```

当用户访问的时候，会带上一个cookie值，新的cookie变成下面这样

```tex
Cookie:
JSESSIONID=A54B61B18A43685DA5C3863982B8CD14; 
remember-me=amllOjE2MDgxMTU5NTc0Mzg6MDhkODk0NTI0OTRjZTI0YWZhOWE2ODA0ZTc5YTM5ODM
```

多了一个cookie，这个是一个由三段组成的cookie。经过base64编码之后形成的。

自动登录也会带来一些安全问题。主要解决思路有以下两种：

- 令牌持久化操作

```java
    @Autowired
    DataSource dataSource;

    JdbcTokenRepositoryImpl jdbcTokenRepository(){
        JdbcTokenRepositoryImpl repository = new JdbcTokenRepositoryImpl();;
        repository.setDataSource(dataSource);
        return repository;
    }
```

再添加到配置中

```java
        http.authorizeRequests()
                .antMatchers("/sta/**").hasRole("admin")
                .antMatchers("/sta/**").hasRole("user")
                .anyRequest().authenticated()
                .and()
                .rememberMe()
                .key("123")
                .tokenRepository(jdbcTokenRepository())
```

这个需要在数据库中添加一个rememberme的数据库。

- 二次校验

对于敏感操作，需要二次校验。对于普通操作，如果是rememberme登陆的就让它操作。

```java
       http.authorizeRequests()
                .antMatchers("/sta/**").fullyAuthenticated()
```

# springsecurity 表单登陆源码

我们可以在springsecurity的配置类里面配置登陆的登陆页，请求地址等，如下：

```java
 http.authorizeRequests()
                .antMatchers("/sta/**").fullyAuthenticated()
                .antMatchers("/sta/**").hasRole("user")
                .anyRequest().authenticated()
                .and()
                .rememberMe()
                .key("123")
                .tokenRepository(jdbcTokenRepository())
                .and()
                .formLogin()
                .loginProcessingUrl("/doLogin")
                .successHandler((req, resp, authentication) -> {
                    Object principal = authentication.getPrincipal();
                    resp.setContentType("application/json;charset=utf-8");
                    PrintWriter out = resp.getWriter();
                    out.write(new ObjectMapper().writeValueAsString(principal));
                    out.flush();
                    out.close();
                })
```

那么，这些设置都是去设置UsernamePasswordAuthenticationFilter这个类。那么，我们要自定义认证逻辑当然就可以替换掉这个类去。我们替换掉这个类当然，上面的那些配置就没用了，需要在自己的类中配置。那么这个类是在哪里配置的呢?

![image-20201202194705118](C:\Users\Jieqiyue\AppData\Roaming\Typora\typora-user-images\image-20201202194705118.png)

就是这个FormLoginConfigurer类中配置的。这个UsernamePasswordAuthenticationFilter中有一些set方法。那么这些set方法是在FormLoginConfigurer中调用的。

那么，这个UsernamePasswordAuthenticationFilter默认是支持表单登陆，传递的不是json格式的数据。如果我们想用json格式的数据去登陆的话，我们的一种思路就是写一个类去继承sernamePasswordAuthenticationFilter那么，改变里面进行获取用户名和密码的方法。再去校验，就行了。这个UsernamePasswordAuthenticationFilter进过一系列的校验成功后，会跑到自动登录的那里去校验，如果配置了自动登录的话。

# 自定义 AuthenticationProvider

可以自定义AuthenticationProvider，不要去添加一个filter。添加一个filter的话，会每一次都进到这个filter中。那么，比如说我们需要一个短信验证码登陆的功能的话，我们有一些请求是不需要进入这个filter中的。所以，添加filter不是最佳选择。

Authentication有很多个实现类，每一个实现类对应了一个登陆方式的验证。那么，有很多AuthenticationProvider与上面的实现类一一对应。一个provider处理一个Authentication的实现类。这些AuthenticationProvider由providermanager来管理。

在usernamepasswordauthenticationfilter会调用providermanager来获取多个AuthenticationProvider来进行验证。

> 添加验证码的功能等

新建立一个filter并不是最佳的方案。以后每一次请求都会到这个filter。但是有一些接口是不需要到这个filter的。

不要去破坏原有的过滤器链。执行效率高一些。

# springsecurity 中UsernamePasswordAuthenticationToken

这个UsernamePasswordAuthenticationToken是用户名密码登陆的时候由usernamepasswordauthenticationfilter创建的。不同的认证方式会有不同的token。UsernamePasswordAuthenticationToken是继承自一个接口，这个接口的实现类有以下这些：![image-20201203151620231](C:\Users\Jieqiyue\AppData\Roaming\Typora\typora-user-images\image-20201203151620231.png)

如果我们自己写一个类继承DaoAuthenticationProvider，并且重写里面验证用户名密码的那个方法additionalAuthenticationChecks。此时还需要自定义一个ProviderManager。因为系统默认的ProviderManager此时不会讲我们上面定义的继承DaoAuthenticationProvider的类给添加进去。在验证之前添加一些判断验证码是否正确的逻辑。当验证码错误时直接抛出异常。就达到目的了。当然，验证码的生成可以去定义一个接口，在这个接口被访问的时候生成验证码，并且将这个验证码放到session中。在我们判断的时候就可以从session中取得验证码。

# springsecurity获取ip地址

Authentication中定义了getDetails方法可以获取一些额外的信息。这里面就有ip地址。这个getDetails方法默认是返回一个WebAuthenticationDetails。这个WebAuthenticationDetails里面就是ip和sessionid（默认情况下）。我们也可以通过request请求来获取ip地址。

> 拓展：WebAuthenticationDetails默认只存放了两项，那么我们可以多存几项，我们需要自己写一个类去继承这个类，并且添加一些属性，比如说验证码校验等等，只要在request中可以获取的都可以存。

# Spring Security 自动踢掉前一个登录用户，一个配置搞定

> 基本配置

```java
@Override
protected void configure(HttpSecurity http) throws Exception {
    http.authorizeRequests()
            .anyRequest().authenticated()
            .and()
            .formLogin()
            .loginPage("/login.html")
            .permitAll()
            .and()
            .csrf().disable()
            .sessionManagement()
            .maximumSessions(1);
}
```

不过还没完，我们还需要再提供一个 Bean：

```
@Bean
HttpSessionEventPublisher httpSessionEventPublisher() {
    return new HttpSessionEventPublisher();
}
```

为什么要加这个 Bean 呢？因为在 Spring Security 中，它是通过监听 session 的销毁事件，来及时的清理 session 的记录。用户从不同的浏览器登录后，都会有对应的 session，当用户注销登录之后，session 就会失效，但是默认的失效是通过调用 StandardSession#invalidate 方法来实现的，这一个失效事件无法被 Spring 容器感知到，进而导致当用户注销登录之后，Spring Security 没有及时清理会话信息表，以为用户还在线，进而导致用户无法重新登录进来（小伙伴们可以自行尝试不添加上面的 Bean，然后让用户注销登录之后再重新登录）。

# csrf 攻击与防御

在springsecurity中默认就有对于csrf攻击的防御措施。不同的网站架构略有不同。主要有两种，前后端分离，前后端不分离。但是防御的思想和手段是相同的。只是说不同的架构，springsecurity的配置略有不同。

> 防御的措施

防御的措施，或者说是思想都是一样的。csrf主要是因为利用了认证后cookie不变，并且浏览器在访问某个网站的时候会自动将这个网站的cookie发送给服务端。那么，攻击者伪造一个网站，放入一个form表单等，在被攻击者的电脑浏览器没有关闭被攻击网站的时候，在这个form表单的地址中填入被攻击网站。那么，**浏览器会自动将cookie发送给服务端** ，此时在用户不知道的情况下发送的请求。那么，要解决的思路就是，服务端要求客户端每次不仅仅是发送给cookie，还需要有一个随机数。这个随机数可以放在请求头中，也可以放在参数中。那么，攻击者虽然能够模拟一个请求，但是不会添加上这个随机数。这个随机数就是一个uuid。

当然配置的话，需要前后端都需要配置一些东西。主要是因为我们需要在前端发送请求的时候，加上这个随机数。前端的配置主要是页面的书写，form表单的书写，js的书写。后端的话，主要是configure方法中。

- 前后端分离

这种情况下，springsecurity将这个随机数存放在cookie中。我们在发送请求的时候，需要用js将cookies中的这个cookie读取到，并且放到请求参数中。其实，服务端并没有存放任何关于这个cookie的任何信息，只负责生成，负责比对。比对的时候，也是通过将request中获取到cookie，与客户端发送过来的参数里面的_csrf进行比较(**就是说，我们在发送请求的时候，在参数里面需要有一个key是__csrf,值是用js从cookie中获取的，cookie中的key是XSRF-TOKEN**)。具体可以看看CookieCsrfTokenRepository.java类。那么具体校验的类当然就是在filter中进行的。可以看看CsrfFilter.java类。

```java
.and()
.csrf().csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse());
```

# springsecurity密码的加密

>  前置知识

加密算法分为可逆和不可逆。不可逆就是不能解密。比如说密码。不可逆加密更加安全。

可逆分为对称加密和非对称加密。 

我们在数据库中存的密码，都是被加密过的。同一个密码明文加密过后并不会生成相同的密文。因为在加密的过程中还有加盐的过程。

# springsecurity 中两种资源放行策略

在controller中想获取已经登录的用户信息，有两种方式。

```java
 @GetMapping("/hello")
    public void hello(Authentication authentication){
        // 直接在方法上添加一个参数
        System.out.println("authentication : " + authentication);
        System.out.println("====================");
        //在controller中还可以这样获取
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        System.out.println("auth : "+auth);
    }
```

在控制台打印如下：

```java
authentication : org.springframework.security.authentication.UsernamePasswordAuthenticationToken@3ed28714: Principal: org.springframework.security.core.userdetails.User@19b06: Username: jie; Password: [PROTECTED]; Enabled: true; AccountNonExpired: true; credentialsNonExpired: true; AccountNonLocked: true; Granted Authorities: ROLE_admin; Credentials: [PROTECTED]; Authenticated: true; Details: org.springframework.security.web.authentication.WebAuthenticationDetails@2cd90: RemoteIpAddress: 0:0:0:0:0:0:0:1; SessionId: 32818A2ED1A1977591A6F4C650B107DE; Granted Authorities: ROLE_admin
====================
auth : org.springframework.security.authentication.UsernamePasswordAuthenticationToken@3ed28714: Principal: org.springframework.security.core.userdetails.User@19b06: Username: jie; Password: [PROTECTED]; Enabled: true; AccountNonExpired: true; credentialsNonExpired: true; AccountNonLocked: true; Granted Authorities: ROLE_admin; Credentials: [PROTECTED]; Authenticated: true; Details: org.springframework.security.web.authentication.WebAuthenticationDetails@2cd90: RemoteIpAddress: 0:0:0:0:0:0:0:1; SessionId: 32818A2ED1A1977591A6F4C650B107DE; Granted Authorities: ROLE_admin

```

如果在一个子线程中，就会获取不到。

```java
 @GetMapping("/hello")
    public void hello(Authentication authentication){
        // 直接在方法上添加一个参数
        System.out.println("authentication : " + authentication);
        System.out.println("====================");
        //在controller中还可以这样获取
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        System.out.println("auth : "+auth);
        
        // 下面这个无法获取
        new Thread(()->{
            Authentication auth1 = 					SecurityContextHolder.getContext().getAuthentication();
            System.out.println("auth : "+auth1);
        }).start();
    }
```

这个用户登录的信息是在放在session中的，然后每一个请求来，有一个线程。放到threadlocal里面去的。子线程所以就获取不到。

这个重载方法配置放行的资源，一般只有静态资源才会在这里配置，如果配置了一个接口在这下面，那么访问这个接口的时候，是不会走springsecurity的那个过滤器链的。这样的话，这个接口就不能获得到登陆的用户信息。因为登陆的用户信息是在过滤器链中的SecurityContextPersistenceFilter保存的。

```java
 	@Override
    public void configure(WebSecurity web) throws Exception {
        web.ignoring().antMatchers("/*.jpg","/js/**", "/css/**", "/images/**");
    }
```

但是，下面这个重载方法会走过滤器链。

```java
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
                .antMatchers("/hello").permitAll();
    }
```



# spring boot 与 websocket

服务器需要像客户端推送消息。没有websocket之前，有一种最简单的方法就是客户端轮询。或者长轮询。

但是这种方式也会有资源浪费等问题。

# spring boot发送邮件

添加依赖，application.properties中配置。将JavaMailSender自动注入进来。创建一个MimeMessage。

> thyemleaf 做邮件模板

跟写html页面一样。

# spring boot 中定时任务

在启动类上加上@EnableScheduling。这个能开启定时任务。

在方法上添加上@Scheduled,并且可以在注解的参数上指定任务的执行时间间隔等。

> @Scheduled注解的一些属性

- fixedDelay 

表示定时任务，前面定时任务结束时间，和后面任务的开始时间之间间隔多少时间。这个之间一定是在前面任务结束之后才开始计算的。

- fixedRate

表示两次任务开始之间的事件间隔。不管第一次任务有没有结束。

- cron表达式

另外一种解决方案是quartz

配置bean，触发器，启动器。并且需要自己去定义一个类来做具体的事情。



# swagger

swagger需要两个依赖。ui和swagger的依赖。（maven仓库复制地址）有springsecurity时，记得给swagger的资源放行。

# springboot 监控

添加OPS里面的Actuator依赖。

端点开启并不等于端点暴露。默认情况下只有一个端点没有开启。但是只有两个暴露了。可以在application.properties里面配置哪些端点暴露。

在浏览器中输入各个端点去访问的时候，在根路径后面要加上actuator。http://localhost:8080/actuator/beans 

如果不想要加，也可以配置。base-path可以来进行改变。默认情况下，要访问哪个端点，就叫端点的名字。但是也可以配置。



> 健康信息

```java
management.endpoint.health.show-details=when_authorized
```

开启显示所有健康信息。

> info

- 自定义信息

```
info.app.java.source=@java.version@
```

- git

要项目被git管理。

```
management.info.git.mode=full
```

- 构建信息

# 监控信息的可视化

添加依赖。但不是上面那个的依赖。ops中的admin server。这个server就是让client去连接的。在server端，可以可视化的查看。client通过spring.boot.admin.client.url=http://localhost:8080连接到server端。

# spring boot 部署

打成jar包。使用插件。这里打成的jar包，以jar结尾的那个jar包不能被其它应用依赖。



------

# 新版springboot

# @Configuration

@Configuration和@bean注解相互配合可以给容器中添加组件。@Configuration注解中有一个属性。

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface Configuration {
    @AliasFor(
        annotation = Component.class
    )
    String value() default "";

    boolean proxyBeanMethods() default true;
}
```

默认是true。表示在用@bean注入组件的时候，首先会去IOC容器中查看是否有这个组件。而且，被@Configuration标注的类是代理类。当这个属性为false的时候，只是一个普通类。在@Bean注入组件的时候，也不会去容器中查看是否已经有这个类。而是直接创建一个新类。

![image-20201224095328191](D:\Desktop\笔记\images\image-20201224095328191.png)

##  

springboot在启动的时候，会创建一些后置处理器。@Configuration注解注释的类，就会被ConfigurationClassPostProcessor类给拦截到。并且被@Configuration注解的类，会创建一个代理类。不是普通new出来的类。

那@Configuration和@Component有什么区别呢？

@Component注解标记的类就是一个普通的组件，并不会创建代理类。并且，@Component里面的方法上有@Bean注解的方法每次都会创建一个新的类。而@Configuration中的@Bean方法标注的类，创建一个类的时候，会先看看IOC容器中是否已经有这个类了，如果有的话，就不会继续创建。

```java
@Configuration
public class BeanConfig {
    @Bean
    public Person getPerson(){
        return new Person(getBook());
    }

    @Bean
    public Book getBook(){
        return new Book();
    }
}
```

```java
@Test
public void test1() throws IOException {
    System.out.println("beans = " + beans);
    System.out.println("book = " + book);
    System.out.println("person = " + person);
    System.out.println(person.getBook() == book);
}

beans = com.jie.config.BeanConfig$$EnhancerBySpringCGLIB$$8f365ac1@5abf6a99
book = com.jie.bean.Book@dc59ec2
person = Person{name='null', age=0}
true
```

可以看到被@Configuration标注的类，最终生成的是cglib的代理类。

而当把@Configuration换成@Component时

```java
@Component
public class BeanConfig {
    @Bean
    public Person getPerson(){
        return new Person(getBook());
    }

    @Bean
    public Book getBook(){
        return new Book();
    }
}
```

```java
beans = com.jie.config.BeanConfig@2e3f324e
book = com.jie.bean.Book@46cf8c07
person = Person{name='null', age=0}
false
```

这个就是@Configuration和@Component的区别。

那么，springboot启动类上的那个@SpringBootApplication注解呢？

@SpringBootApplication其实是一个组合注解，点开可以看到由多个注解组成。有开启自动配置的注解，也有包扫描的注解，默认情况下，包扫描路径为启动类所在包，及其子包下的类。

# springboot 自动配置

![image-20201213114829351](D:\Desktop\笔记\images\image-20201213114829351.png)

```java
@EnableConfigurationProperties
```

上面这个注解能够使，properties配置类生效。

# @Import()注解

用import注解，结合@enablexxx注解的方式能够批量的注入bean。有四种方式可以来将这两个结合，并且注册bean。

# maven打包

如果要创建一个有父子关系的多模块系统的话，各个子模块之间有互相联系的话，那么可以在parent模块中的pom.xml中的<modules></modules>中写出各个子模块。如果在父模块的<modules></modules>中写了子模块的话，打包的时候，父模块打包会将各个子模块都一起打包进去。那么这个<modules></modules>要不要写子模块呢，这个取决于各个子模块之间有无联系。如果没有联系的话，那么是不用写在<modules></modules>里面的。

版本管理方面。可以在父模块中定义

```
<properties>
<properties/>
<dependencyManagement>
</dependencyManagement>
```

来定义版本。在子模块中就不要指定版本号。

## xxx.properties读取pom.xml

### 1.xxx.properties中

以pom.xml中的version标签为例。[@xx](https://my.oschina.net/qhe9)@代表读取pom.xml中的值

这里为什么是用@呢：

由于${}方式会被maven处理。如果你pom继承了spring-boot-starter-parent，Spring Boot已经将maven-resources-plugins默认的${}方式改为了@[@方式](https://my.oschina.net/u/923227)，如[@name](https://my.oschina.net/appl)@

如果要使用原来的大括号在pom.xml中添加如下代码

```xml
<pluginManagement>
    <plugins>
        <plugin>
            <artifactId>maven-resources-plugin</artifactId>
            <configuration>
                <encoding>utf-8</encoding>
                <useDefaultDelimiters>true</useDefaultDelimiters>
            </configuration>
        </plugin>
    </plugins>
</pluginManagement>
```

# WebMvcConfigurer详解

## 1. 简介

WebMvcConfigurer配置类其实是`Spring`内部的一种配置方式，采用`JavaBean`的形式来代替传统的`xml`配置文件形式进行针对框架个性化定制，可以自定义一些Handler，Interceptor，ViewResolver，MessageConverter。基于java-based方式的spring mvc配置，需要创建一个**配置**类并实现**`WebMvcConfigurer`** 接口；

在Spring Boot 1.5版本都是靠重写**WebMvcConfigurerAdapter**的方法来添加自定义拦截器，消息转换器等。SpringBoot 2.0 后，该类被标记为@Deprecated（弃用）。官方推荐直接实现WebMvcConfigurer或者直接继承WebMvcConfigurationSupport，方式一实现WebMvcConfigurer接口（推荐）

**常用的方法**

```java
 /* 拦截器配置 */
void addInterceptors(InterceptorRegistry var1);
/* 视图跳转控制器 */
void addViewControllers(ViewControllerRegistry registry);
/**
     *静态资源处理
**/
void addResourceHandlers(ResourceHandlerRegistry registry);
/* 默认静态资源处理器 */
void configureDefaultServletHandling(DefaultServletHandlerConfigurer configurer);
/**
     * 这里配置视图解析器
 **/
void configureViewResolvers(ViewResolverRegistry registry);
/* 配置内容裁决的一些选项*/
void configureContentNegotiation(ContentNegotiationConfigurer configurer);
/** 解决跨域问题 **/
public void addCorsMappings(CorsRegistry registry) ;
```
































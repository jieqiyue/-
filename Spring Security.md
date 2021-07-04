# spring security 登陆流程

在ProviderManager中寻找可以验证的验证器。

在**DaoAuthenticationProvider和AbstractUserDetailsAuthenticationProvider中进行验证。**

usernamepasswordAuthenticationToken是什么?

这个是根据前端传过来的username和password生成的一个东西。可以用于后面获取用户名，然后根据这个获取到的用户名去数据库中获取到用户。就是根据用户名获取用户的那个，自己配置的那个。

**根据前端传来的用户名和密码，生成一个usernamepasswordAuthenticationToken，然后这个token可以获取到用户名。在获取到用户名之后，在根据用户名从数据库中，或者是内存中。反正就是实现了UserDetailsService接口，然后这个接口中有那个loadUserByUsername。从这里面获取到UserDetails。在会去判断这个数据库中获取到的UserDetails是否过期，或者是账户是否被锁定等，然后如果这一步没有异常，那么就说明这个用户是正常的。此时就可以来和前端传来的密码进行比较了。怎么比较呢？就是前面，生成的那个 usernamepasswordAuthenticationToken和数据库中查出来的UserDetails在additionalAuthenticationChecks这个方法中比较，主要就是比较密码是否正确。会去调在securityconfiger里面配置的密码加密器的matcher方法。来比较密码是否相等。如果不相等就会抛出BadCredentialsException异常。**

# 登陆配置

https://mp.weixin.qq.com/s?__biz=MzI1NDY0MTkzNQ==&mid=2247488138&idx=1&sn=25d18a61a14e4e6316537b6d45e43dd4&scene=21#wechat_redirect

#### 1.在 SecurityConfig 中自定定义了登录页面地址，如下：

```java
.and()
.formLogin()
.loginPage("/login.html")
.permitAll()
.and()
```

当我们配置了 loginPage 为 `/login.html` 之后，这个配置从字面上理解，就是设置登录页面的地址为 `/login.html`。

实际上它还有一个隐藏的操作，就是登录接口地址也设置成 `/login.html` 了。新的登录页面和登录接口地址都是 `/login.html`，现在存在如下两个请求：

- GET http://localhost:8080/login.html
- POST http://localhost:8080/login.html

前面的 GET 请求用来获取登录页面，后面的 POST 请求用来提交登录数据。

为什么登录页面和登录接口不能分开配置呢？

其实是可以分开配置的！

在 SecurityConfig 中，我们可以通过 loginProcessingUrl 方法来指定登录接口地址，如下：

```java
.and()
.formLogin()
.loginPage("/login.html")
.loginProcessingUrl("/doLogin")
.permitAll()
.and()
```

这样配置之后，登录页面地址和登录接口地址就分开了，各是各的。

此时我们还需要修改登录页面里边的 action 属性，改为 `/doLogin`，如下：

```html
<form action="/doLogin" method="post">
<!--省略-->
</form>
```

此时，启动项目重新进行登录，我们发现依然可以登录成功。

#### 2.登陆回调

##### 前后端不分离

登陆回调有两种配置的方式。

- defaultSuccessUrl  有重载方法。第二个参数为true的时候，和successForwardUrl一样。当我们登陆成功的时候，默认是会跳转到先前的那个地址。

- successForwardUrl  

  登陆成功后一定是跳转到指定的地址。

```java
.and()
.formLogin()
.loginPage("/login.html")
.loginProcessingUrl("/doLogin")
.usernameParameter("name")
.passwordParameter("passwd")
.defaultSuccessUrl("/index")
.successForwardUrl("/index")
.permitAll()
.and()
```

##### 前后端分离

successHandler的参数是一个对象。对象中要实现的方法就是onAuthenticationSuccess。那么这个方法有三个参数。拿到这三个参数，想跳转还是想返回json数据就都非常简单了。

```java
.successHandler((req, resp, authentication) -> {
    Object principal = authentication.getPrincipal();
    resp.setContentType("application/json;charset=utf-8");
    PrintWriter out = resp.getWriter();
    out.write(new ObjectMapper().writeValueAsString(principal));
    out.flush();
    out.close();
})
```

失败回调

注意这个失败回调，失败的时候，总是有一个异常给抛出来。当有异常抛出的时候，我们可以在这里面判断异常的类型，进而可以决定给前端返回的字符串是什么。在vhr项目中就是这样的。注意，这里不能想到去使用全局异常处理。因为这个是一个lambda表达式去实现那个方法。那个方法的方法签名已经写死了，不能在这个方法中抛出异常。

```java
.failureHandler((req, resp, e) -> {
    resp.setContentType("application/json;charset=utf-8");
    PrintWriter out = resp.getWriter();
    out.write(e.getMessage());
    out.flush();
    out.close();
})
```

未认证的处理

直接返回未登录的json给前端。让前端自行决定应该怎样跳转。

#### 3.注销登陆

注销登录的默认接口是 `/logout`，我们也可以配置。

```java
.and()
.logout()
.logoutUrl("/logout")
.logoutRequestMatcher(new AntPathRequestMatcher("/logout","POST"))
.logoutSuccessUrl("/index")
.deleteCookies()
.clearAuthentication(true)
.invalidateHttpSession(true)
.permitAll()
.and()
```

1. 默认注销的 URL 是 `/logout`，是一个 GET 请求，我们可以通过 logoutUrl 方法来修改默认的注销 URL。
2. logoutRequestMatcher 方法不仅可以修改注销 URL，还可以修改请求方式，实际项目中，这个方法和 logoutUrl 任意设置一个即可。
3. logoutSuccessUrl 表示注销成功后要跳转的页面。
4. deleteCookies 用来清除 cookie。
5. clearAuthentication 和 invalidateHttpSession 分别表示清除认证信息和使 HttpSession 失效，默认可以不用配置，默认就会清除。

# 授权操作

spring security可以支持多数据源。多个数据源都被抽取成了一个接口（UserDetailsService）。你只需要在websecurityconfigureradapter中配置一下就行。

核心配置

![image-20210111185207455](D:\Desktop\笔记\images\image-20210111185207455.png)

用户角色配置

![image-20210111185247084](D:\Desktop\笔记\images\image-20210111185247084.png)

权限继承

![image-20210111185336397](D:\Desktop\笔记\images\image-20210111185336397.png)

# 如何将用户信息存入数据库


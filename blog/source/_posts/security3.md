---
title: 认证鉴权与API权限控制在微服务架构中的设计与实现（三） 
date: 2017-10-24
categories: Security
tags:
- Spring Security
- OAuth2
---
引言： 本文系《认证鉴权与API权限控制在微服务架构中的设计与实现》系列的第三篇，本文重点讲解token以及API级别的鉴权。本文对涉及到的大部分代码进行了分析，欢迎订阅本系列文章。

## 1. 前文回顾
在开始讲解这一篇文章之前，先对之前两篇文章进行回忆下。在第一篇 [认证鉴权与API权限控制在微服务架构中的设计与实现（一）](http://blueskykong.com/2017/10/19/security1/)介绍了该项目的背景以及技术调研与最后选型。第二篇[认证鉴权与API权限控制在微服务架构中的设计与实现（二）](http://blueskykong.com/2017/10/22/security2/)画出了简要的登录和校验的流程图，并重点讲解了用户身份的认证与token发放的具体实现。   

![check](http://ovcjgn2x0.bkt.clouddn.com/check.png "身份及API权限校验的流程图")

本文重点讲解鉴权，包括两个方面：token合法性以及API级别的操作权限。首先token合法性很容易理解，第二篇文章讲解了获取授权token的一系列流程，token是否是认证服务器颁发的，必然是需要验证的。其次对于API级别的操作权限，将上下文信息不具备操作权限的请求直接拒绝，当然此处是设计token合法性校验在先，其次再对操作权限进行验证，如果前一个验证直接拒绝，通过则进入操作权限验证。

## 2.资源服务器配置

`ResourceServer`配置在第一篇就列出了，在进入鉴权之前，把这边的配置搞清，即使有些配置在本项目中没有用到，大家在自己的项目有可能用到。

```java
@Configuration
@EnableResourceServer
public class ResourceServerConfig extends ResourceServerConfigurerAdapter {

	//http安全配置
    @Override
    public void configure(HttpSecurity http) throws Exception {
    	//禁掉csrf，设置session策略
        http.csrf().disable()
                .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS)
                .and()//默认允许访问
                .requestMatchers().antMatchers("/**")
                .and().authorizeRequests()
                .antMatchers("/**").permitAll()
                .anyRequest().authenticated()
                .and().logout() //logout注销端点配置
                .logoutUrl("/logout")
                .clearAuthentication(true)
                .logoutSuccessHandler(new HttpStatusReturningLogoutSuccessHandler())
                .addLogoutHandler(customLogoutHandler());

    }

	//添加自定义的CustomLogoutHandler
    @Bean
    public CustomLogoutHandler customLogoutHandler() {
        return new CustomLogoutHandler();
    }

	//资源安全配置相关
    @Override
    public void configure(ResourceServerSecurityConfigurer resources) throws Exception {
        super.configure(resources);
    }
}
```

(1). `@EnableResourceServer`这个注解很重要，OAuth2资源服务器的简便注解。其使得Spring Security filter通过请求中的OAuth2 token来验证请求。通常与`EnableWebSecurity`配合使用，该注解还创建了硬编码的`@Order(3) WebSecurityConfigurerAdapter`，由于当前spring的技术，order的顺序不易修改，所以在项目中避免还有其他order=3的配置。

(2). 关联的`HttpSecurity`，与之前的 Spring Security XML中的 "http"元素配置类似，它允许配置基于web安全以针对特定http请求。默认是应用到所有的请求，通过`requestMatcher`可以限定具体URL范围。HttpSecurity类图如下。

![HttpSecurity](http://ovcjgn2x0.bkt.clouddn.com/httpsecurity.png "HttpSecurity类图")

总的来说：HttpSecurity是SecurityBuilder接口的一个实现类，从名字上我们就可以看出这是一个HTTP安全相关的构建器。当然我们在构建的时候可能需要一些配置，当我们调用HttpSecurity对象的方法时，实际上就是在进行配置。   

authorizeRequests()，formLogin()、httpBasic()这三个方法返回的分别是`ExpressionUrlAuthorizationConfigurer`、`FormLoginConfigurer`、`HttpBasicConfigurer`，他们都是SecurityConfigurer接口的实现类，分别代表的是不同类型的安全配置器。   
因此，从总的流程上来说，当我们在进行配置的时候，需要一个安全构建器SecurityBuilder(例如我们这里的HttpSecurity)，SecurityBuilder实例的创建需要有若干安全配置器SecurityConfigurer实例的配合。

(3).关联的`ResourceServerSecurityConfigurer`，为资源服务器添加特殊的配置，默认的适用于很多应用，但是这边的修改至少以resourceId为单位。类图如下。

![ResourceServerSecurityConfigurer](http://ovcjgn2x0.bkt.clouddn.com/resource.png "ResourceServerSecurityConfigurer类图")

`ResourceServerSecurityConfigurer`创建了OAuth2核心过滤器`OAuth2AuthenticationProcessingFilter`，并为其提供固定了`OAuth2AuthenticationManager`。只有被`OAuth2AuthenticationProcessingFilter`拦截到的oauth2相关请求才被特殊的身份认证器处理。同时设置了TokenExtractor、异常处理实现。
 
`OAuth2AuthenticationProcessingFilter`是OAuth2保护资源的预先认证过滤器。配合`OAuth2AuthenticationManager`使用，根据请求获取到OAuth2 token，之后就会使用`OAuth2Authentication`来填充Spring Security上下文。   
`OAuth2AuthenticationManager`在前面的文章给出的`AuthenticationManager`类图就出现了，与token认证相关。这边略过贴出源码进行讲解，读者可以自行阅读。

## 3. 鉴权endpoint
鉴权主要是使用内置的endpoint `/oauth/check_token`，笔者将对端点的分析放在前面，因为这是鉴权的唯一入口。下面我们来看下该API接口中的主要代码。

```java
	@RequestMapping(value = "/oauth/check_token")
    @ResponseBody
    public Map<String, ?> checkToken(CheckTokenEntity checkTokenEntity) {
		//CheckTokenEntity为自定义的dto
        Assert.notNull(checkTokenEntity, "invalid token entity!");
        //识别token
        OAuth2AccessToken token = resourceServerTokenServices.readAccessToken(checkTokenEntity.getToken());
		//判断token是否为空
        if (token == null) {
            throw new InvalidTokenException("Token was not recognised");
        }
		//未过期
        if (token.isExpired()) {
            throw new InvalidTokenException("Token has expired");
        }
        //加载OAuth2Authentication
        OAuth2Authentication authentication = resourceServerTokenServices.loadAuthentication(token.getValue());
		//获取response，token合法性验证完毕
        Map<String, Object> response = (Map<String, Object>) accessTokenConverter.convertAccessToken(token, authentication);

        //check for api permission
        if (response.containsKey("jti")) {
        	//上下文操作权限校验
            Assert.isTrue(checkPermissions.checkPermission(checkTokenEntity));
        }
        
        response.put("active", true);    // Always true if token exists and not expired

        return response;
    }
```

看过security-oauth源码的同学可能立马就看出上述代码与源码不同，熟悉`/oauth/check_token`校验流程的也会看出来，这边笔者对`security-oauth` jar进行了重新编译，修改了部分源码用于该项目需求的场景。主要是加入了前置的API级别的权限校验。

## 4. token 合法性验证
从上面的`CheckTokenEndpoint`中可以看出，对于token合法性验证首先是识别请求体中的token。用到的主要方法是`ResourceServerTokenServices`提供的`readAccessToken()`方法。该接口的实现类为`DefaultTokenServices`，在之前的配置中有讲过这边配置了jdbc的TokenStore。

```java
public class JdbcTokenStore implements TokenStore {
	...
	public OAuth2AccessToken readAccessToken(String tokenValue) {
		OAuth2AccessToken accessToken = null;

		try {
			//使用selectAccessTokenSql语句，调用了私有的extractTokenKey()方法
			accessToken = jdbcTemplate.queryForObject(selectAccessTokenSql, new RowMapper<OAuth2AccessToken>() {
				public OAuth2AccessToken mapRow(ResultSet rs, int rowNum) throws SQLException {
					return deserializeAccessToken(rs.getBytes(2));
				}
			}, extractTokenKey(tokenValue));
		}
		//异常情况
		catch (EmptyResultDataAccessException e) {
			if (LOG.isInfoEnabled()) {
				LOG.info("Failed to find access token for token " + tokenValue);
			}
		}
		catch (IllegalArgumentException e) {
			LOG.warn("Failed to deserialize access token for " + tokenValue, e);
			//不合法则移除
			removeAccessToken(tokenValue);
		}

		return accessToken;
	}
	...
	//提取TokenKey方法
	protected String extractTokenKey(String value) {
		if (value == null) {
			return null;
		}
		MessageDigest digest;
		try {
			//MD5
			digest = MessageDigest.getInstance("MD5");
		}
		catch (NoSuchAlgorithmException e) {
			throw new IllegalStateException("MD5 algorithm not available.  Fatal (should be in the JDK).");
		}

		try {
			byte[] bytes = digest.digest(value.getBytes("UTF-8"));
			return String.format("%032x", new BigInteger(1, bytes));
		}
		catch (UnsupportedEncodingException e) {
			throw new IllegalStateException("UTF-8 encoding not available.  Fatal (should be in the JDK).");
		}
	}
}
```

`readAccessToken()`检索出该token值的完整信息。上述代码比较简单，涉及到的逻辑也不复杂，此处简单讲解。下图为debug token校验的变量信息，读者可以自己动手操作下，截图仅供参考。

![token](http://ovcjgn2x0.bkt.clouddn.com/checktoken.png "debug token校验")

至于后面的步骤，`loadAuthentication()`为特定的access token 加载credentials。得到的credentials 与token作为`convertAccessToken()`参数，得到校验token的response。

## 5. API级别权限校验

笔者项目目前都是基于Web的权限验证，之前遗留的一个巨大的单体应用系统正在逐渐拆分，然而当前又不能完全拆分完善。为了同时兼容新旧服务，尽量减少对业务系统的入侵，实现微服务的统一性和独立性。笔者根据业务业务场景，尝试在Auth处做操作权限校验。
首先想到的是资源服务器配置ResourceServer，如：

```
http.authorizeRequests()
.antMatchers("/order/**").access("#oauth2.hasScope('select') and hasRole('ROLE_USER')")
```
这样做需要将每个操作接口的API权限控制放在各个不同的业务服务，每个服务在接收到请求后，需要先从Auth服务取出该token 对应的role和scope等权限信息。这个方法肯定是可行的，但是由于项目鉴权的粒度更细，而且暂时不想大动原有系统，在加上之前网关设计，网关调用Auth服务校验token合法性，所以最后决定在Auth系统调用中，把这些校验一起解决完。   

文章开头资源服务器的配置代码可以看出，对于所有的资源并没有做拦截，因为网关处是调用Auth系统的相关endpoint，并不是所有的请求url都会经过一遍Auth系统，所以对于所有的资源，在Auth系统中，定义需要鉴权接口所需要的API权限，然后根据上下文进行匹配。这是采用的第二种方式，也是笔者目前采用的方法。当然这种方式的弊端也很明显，一旦并发量大，网关还要耗时在调用Auth系统的鉴权上，TPS势必要下降很多，对于一些不需要鉴权的服务接口也会引起不可用。另外一点是，对于某些特殊权限的接口，需要的上下文信息很多，可能并不能完全覆盖，对于此，笔者的解决是分两方面：一是尽量将这些特殊情况进行分类，某一类的情况统一解决；二是将严苛的校验降低，对于上下文校验失败的直接拒绝，而通过的，对于某些接口，在接口内进行操作之前，对特殊的地方还要再次进行校验。

上面在讲endpoint有提到这边对源码进行了改写。`CheckTokenEntity`是自定义的DTO，这这个类中定义了鉴权需要的上下文，这里是指能校验操作权限的最小集合，如URI、roleId、affairId等等。另外定义了`CheckPermissions`接口，其方法`checkPermission(CheckTokenEntity checkTokenEntity)`返回了check的结果。而其具体实现类则定义在Auth系统中。笔者项目中调用的实例如下：

```java
@Component
public class CustomCheckPermission implements CheckPermissions {

    @Autowired
    private PermissionService permissionService;

    @Override
    public boolean checkPermission(CheckTokenEntity checkTokenEntity) {
        String url = checkTokenEntity.getUri();
        Long affairId = checkTokenEntity.getAffairId();
        Long roleId = checkTokenEntity.getRoleId();
        //校验
        if (StringUtils.isEmpty(url) || affairId <= 0 || roleId <= 0) {
            return true;
        } else {
            return permissionService.checkPermission(url, affairId, roleId);
        }
    }
}
```
   
关于jar包`spring-cloud-starter-oauth2`中的具体修改内容，大家可以看下文末笔者的GitHub项目。通过自定义`CustomCheckPermission`，覆写`checkPermission()`方法，大家也可以对自己业务的操作权限进行校验，非常灵活。这边涉及到具体业务，笔者在项目中只提供接口，具体的实现需要读者自行完成。

## 6. 总结
本文相对来说比较简单，主要讲解了token以及API级别的鉴权。token的合法性认证很常规，Auth系统对于API级别的鉴权是结合自身业务需要和现状进行的设计。这两块的校验都前置到Auth系统中，优缺点在上面的小节也有讲述。最后，架构设计根据自己的需求和现状，笔者的解决思路仅供参考。


**本文的源码地址：   
GitHub：https://github.com/keets2012/Auth-service   
码云： https://gitee.com/keets/Auth-Service**

---
### 参考
1. [微服务API级权限的技术架构](http://blog.csdn.net/OmniStack/article/details/77881185?locationNum=10&fps=1)
2. [spring-security-oauth](http://projects.spring.io/spring-security-oauth/docs/oauth2.html)
3. [Spring-Security Docs](http://projects.spring.io/spring-security/)

### 相关阅读
[认证鉴权与API权限控制在微服务架构中的设计与实现（一）](http://blueskykong.com/2017/10/19/security1/)   
[认证鉴权与API权限控制在微服务架构中的设计与实现（二）](http://blueskykong.com/2017/10/22/security2/)



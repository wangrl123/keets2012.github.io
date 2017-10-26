---
title: 认证鉴权与API权限控制在微服务架构中的设计与实现（四） 
date: 2017-10-26
categories: Security
tags:
- Spring Security
- OAuth2
---
引言： 本文系《认证鉴权与API权限控制在微服务架构中的设计与实现》系列的完结篇，前面三篇已经将认证鉴权与API权限控制的流程和主要细节讲解完。本文比较长，对这个系列进行收尾，主要内容包括对授权和鉴权流程之外的endpoint以及`Spring Security`过滤器部分踩坑的经历。欢迎阅读本系列文章。

## 1. 前文回顾
首先还是照例对前文进行回顾。在第一篇 [认证鉴权与API权限控制在微服务架构中的设计与实现（一）](http://blueskykong.com/2017/10/19/security1/)介绍了该项目的背景以及技术调研与最后选型。第二篇[认证鉴权与API权限控制在微服务架构中的设计与实现（二）](http://blueskykong.com/2017/10/22/security2/)画出了简要的登录和校验的流程图，并重点讲解了用户身份的认证与token发放的具体实现。第三篇[认证鉴权与API权限控制在微服务架构中的设计与实现（三）](http://blueskykong.com/2017/10/24/security3/)先介绍了资源服务器配置，以及其中涉及的配置类，后面重点讲解了token以及API级别的鉴权。   

本文将会讲解剩余的两个内置端点：注销和刷新token。注销token端点的处理与`Spring Security`默认提供的有些'/logout'有些区别，不仅清空SpringSecurityContextHolder中的信息，还要增加对存储token的清空。另一个刷新token端点其实和之前的请求授权是一样的API，只是参数中的grant_type不一样。   

除了以上两个内置端点，后面将会重点讲下几种`Spring Security`过滤器。API级别的操作权限校验本来设想是通过`Spring Security`的过滤器实现，特地把这边学习了一遍，踩了一遍坑。 

最后是本系列的总结，并对于存在的不足和后续工作进行论述。

## 2. 其他端点
### 2.1 注销端点
在第一篇中提到了Auth系统内置的注销端点 `/logout`，如果还记得第三篇资源服务器的配置，下面的关于`/logout`配置一定不陌生。

```java
			//...
				.and().logout()
                .logoutUrl("/logout")
                .clearAuthentication(true)
                .logoutSuccessHandler(new HttpStatusReturningLogoutSuccessHandler())
                .addLogoutHandler(customLogoutHandler());
```

上面配置的主要作用是：   

- 设置注销的URL   
- 清空Authentication信息   
- 设置注销成功的处理方式   
- 设置自定义的注销处理方式   

当然在`LogoutConfigurer`中还有更多的设置选项，笔者此处列出项目所需要的配置项。这些配置项围绕着`LogoutFilter`过滤器。顺带讲一下`Spring Security`的过滤器。其使用了`springSecurityFillterChian`作为了安全过滤的入口，各种过滤器按顺序具体如下：

- SecurityContextPersistenceFilter：与SecurityContext安全上下文信息有关
- HeaderWriterFilter：给http响应添加一些Header
- CsrfFilter：防止csrf攻击，默认开启
- LogoutFilter：处理注销的过滤器
- UsernamePasswordAuthenticationFilter：表单认证过滤器
- RequestCacheAwareFilter：缓存request请求
- SecurityContextHolderAwareRequestFilter：此过滤器对ServletRequest进行了一次包装，使得request具有更加丰富的API
- AnonymousAuthenticationFilter：匿名身份过滤器
- SessionManagementFilter：session相关的过滤器，常用来防止session-fixation protection attack，以及限制同一用户开启多个会话的数量
- ExceptionTranslationFilter：异常处理过滤器
- FilterSecurityInterceptor：web应用安全的关键Filter

各种过滤器简单标注了作用，在下一节重点讲其中的几个过滤器。注销过滤器排在靠前的位置，我们一起看下`LogoutFilter`的UML类图。

![logoutFilter](http://ovcjgn2x0.bkt.clouddn.com/logout.png "logoutFilter类图")

类图和我们之前配置时的思路是一致的，`HttpSecurity`创建了`LogoutConfigurer`，我们在这边配置了`LogoutConfigurer`的一些属性。同时`LogoutConfigurer`根据这些属性创建了`LogoutFilter`。

`LogoutConfigurer`的配置，第一和第二点就不用再详细解释了，一个是设置端点，另一个是清空认证信息。   
对于第三点，配置注销成功的处理方式。由于项目是前后端分离，客户端只需要知道执行成功该API接口的状态，并不用返回具体的页面或者继续向下传递请求。因此，这边配置了默认的`HttpStatusReturningLogoutSuccessHandler`，成功直接返回状态码200。   
对于第四点配置，自定义注销处理的方法。这边需要借助`TokenStore`，对token进行操作。`TokenStore`在之前文章的配置中已经讲过，使用的是JdbcTokenStore。首先校验请求的合法性，如果合法则对其进行操作，先后移除`refreshToken`和`existingAccessToken`。

```java
public class CustomLogoutHandler implements LogoutHandler {

	//...

    @Override
    public void logout(HttpServletRequest request, HttpServletResponse response, Authentication authentication) {
    	//确定注入了tokenStore
        Assert.notNull(tokenStore, "tokenStore must be set");
       //获取头部的认证信息
        String token = request.getHeader("Authorization");
        Assert.hasText(token, "token must be set");
        //校验token是否符合JwtBearer格式
        if (isJwtBearerToken(token)) {
            token = token.substring(6);
            OAuth2AccessToken existingAccessToken = tokenStore.readAccessToken(token);
            OAuth2RefreshToken refreshToken;
            if (existingAccessToken != null) {
                if (existingAccessToken.getRefreshToken() != null) {
                    LOGGER.info("remove refreshToken!", existingAccessToken.getRefreshToken());
                    refreshToken = existingAccessToken.getRefreshToken();
                    tokenStore.removeRefreshToken(refreshToken);
                }
                LOGGER.info("remove existingAccessToken!", existingAccessToken);
                tokenStore.removeAccessToken(existingAccessToken);
            }
            return;
        } else {
            throw new BadClientCredentialsException();
        }

    }

	//...
}
```
执行如下请求： 

```
method: get
url: http://localhost:9000/logout
header:
{
	Authorization: Basic ZnJvbnRlbmQ6ZnJvbnRlbmQ=
}
```
注销成功则会返回200，将token和SecurityContextHolder进行清空。

### 2.2 刷新端点

在第一篇就已经讲过，由于token的时效一般不会很长，而refresh_ token一般周期会很长，为了不影响用户的体验，可以使用refresh_ token去动态的刷新token。刷新token主要与`RefreshTokenGranter`有关，`CompositeTokenGranter`管理一个List列表，每一种grantType对应一个具体的真正授权者，refresh_ token对应的granter就是`RefreshTokenGranter`，而granter内部则是通过grantType来区分是否是各自的授权类型。执行如下请求：

```
method: post 
url: http://localhost:12000/oauth/token?grant_type=refresh_token&refresh_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJYLUtFRVRTLVVzZXJJZCI6ImQ2NDQ4YzI0LTNjNGMtNGI4MC04MzcyLWMyZDYxODY4ZjhjNiIsInVzZXJfbmFtZSI6ImtlZXRzIiwic2NvcGUiOlsiYWxsIl0sImF0aSI6ImJhZDcyYjE5LWQ5ZjMtNDkwMi1hZmZhLTA0MzBlN2RiNzllZCIsImV4cCI6MTUxMDk5NjU1NiwianRpIjoiYWE0MWY1MjctODE3YS00N2UyLWFhOTgtZjNlMDZmNmY0NTZlIiwiY2xpZW50X2lkIjoiZnJvbnRlbmQifQ.mICT1-lxOAqOU9M-Ud7wZBb4tTux6OQWouQJ2nn1DeE
header:
{
	Authorization: Basic ZnJvbnRlbmQ6ZnJvbnRlbmQ=
}
```
在refresh_ token正确的情况下，其返回的response和/oauth/token得到正常的响应是一样的。具体的代码可以参阅第二篇的讲解。

## 3. `Spring Security`过滤器

在上一节我们介绍了内置的两个端点的实现细节，还提到了`HttpSecurity`过滤器，因为注销端点的实现就是通过过滤器的作用。核心的过滤器主要有：

- FilterSecurityInterceptor
- UsernamePasswordAuthenticationFilter
- SecurityContextPersistenceFilter
- ExceptionTranslationFilter

这一节将重点介绍其中的`UsernamePasswordAuthenticationFilter`和`FilterSecurityInterceptor`。   

### 3.1 `UsernamePasswordAuthenticationFilter`
笔者在刚开始看关于过滤器的文章，对于`UsernamePasswordAuthenticationFilter`有不少的文章介绍。如果只是引入Spring-Security，必然会与`/login`端点熟悉。SpringSecurity强制要求我们的表单登录页面必须是以POST方式向/login URL提交请求，而且要求用户名和密码的参数名必须是username和password。如果不符合，则不能正常工作。原因在于，当我们调用了HttpSecurity对象的formLogin方法时，其最终会给我们注册一个过滤器`UsernamePasswordAuthenticationFilter`。看一下该过滤器的源码。

```java
public class UsernamePasswordAuthenticationFilter extends
		AbstractAuthenticationProcessingFilter {
	//用户名、密码
	public static final String SPRING_SECURITY_FORM_USERNAME_KEY = "username";
	public static final String SPRING_SECURITY_FORM_PASSWORD_KEY = "password";

	private String usernameParameter = SPRING_SECURITY_FORM_USERNAME_KEY;
	private String passwordParameter = SPRING_SECURITY_FORM_PASSWORD_KEY;
	private boolean postOnly = true;

	//post请求/login
	public UsernamePasswordAuthenticationFilter() {
		super(new AntPathRequestMatcher("/login", "POST"));
	}
	//实现抽象类AbstractAuthenticationProcessingFilter的抽象方法，尝试验证
	public Authentication attemptAuthentication(HttpServletRequest request,
			HttpServletResponse response) throws AuthenticationException {
		if (postOnly && !request.getMethod().equals("POST")) {
			throw new AuthenticationServiceException(
					"Authentication method not supported: " + request.getMethod());
		}

		String username = obtainUsername(request);
		String password = obtainPassword(request);
		
		//···

		username = username.trim();

		UsernamePasswordAuthenticationToken authRequest = new UsernamePasswordAuthenticationToken(
				username, password);
		//···
		return this.getAuthenticationManager().authenticate(authRequest);
	}
}
```

```java
public abstract class AbstractAuthenticationProcessingFilter extends GenericFilterBean
		implements ApplicationEventPublisherAware, MessageSourceAware {
	//...
	
	//调用requiresAuthentication，判断请求是否需要authentication，如果需要则调用attemptAuthentication
	//有三种结果可能返回：
	//1.Authentication对象
	//2. AuthenticationException
	//3. Authentication对象为空
	public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
			throws IOException, ServletException {

		HttpServletRequest request = (HttpServletRequest) req;
		HttpServletResponse response = (HttpServletResponse) res;
		//不需要校验，继续传递
		if (!requiresAuthentication(request, response)) {
			chain.doFilter(request, response);
			return;
		}
		Authentication authResult;

		try {
			authResult = attemptAuthentication(request, response);
			if (authResult == null) {
				// return immediately as subclass has indicated that it hasn't completed authentication
				return;
			}
			sessionStrategy.onAuthentication(authResult, request, response);
		}
		//...
		catch (AuthenticationException failed) {
			// Authentication failed
			unsuccessfulAuthentication(request, response, failed);

			return;
		}

		// Authentication success
		if (continueChainBeforeSuccessfulAuthentication) {
			chain.doFilter(request, response);
		}

		successfulAuthentication(request, response, chain, authResult);
	}

	//实际执行的authentication，继承类必须实现该抽象方法
	public abstract Authentication attemptAuthentication(HttpServletRequest request,
			HttpServletResponse response) throws AuthenticationException, IOException,
			ServletException;
	//成功authentication的默认行为
	protected void successfulAuthentication(HttpServletRequest request,
			HttpServletResponse response, FilterChain chain, Authentication authResult)
			throws IOException, ServletException {

		//...
	}
	//失败authentication的默认行为
	protected void unsuccessfulAuthentication(HttpServletRequest request,
			HttpServletResponse response, AuthenticationException failed)
			throws IOException, ServletException {
	//...			
	}

	...
	//设置AuthenticationManager
	public void setAuthenticationManager(AuthenticationManager authenticationManager) {
		this.authenticationManager = authenticationManager;
	}
	...
}
```

`UsernamePasswordAuthenticationFilter`因为继承了`AbstractAuthenticationProcessingFilter`才拥有过滤器的功能。`AbstractAuthenticationProcessingFilter`要求设置一个authenticationManager，authenticationManager的实现类将实际处理请求的认证。`AbstractAuthenticationProcessingFilter`将拦截符合过滤规则的request，并试图执行认证。子类必须实现 attemptAuthentication 方法，这个方法执行具体的认证。   
认证之后的处理和上注销的差不多。如果认证成功，将会把返回的Authentication对象存放在SecurityContext，并调用SuccessHandler，也可以设置指定的URL和指定自定义的处SuccessHandler。如果认证失败，默认会返回401代码给客户端，也可以设置URL，指定自定义的处理FailureHandler。   

基于`UsernamePasswordAuthenticationFilter`自定义的`AuthenticationFilte`还是挺多案例的，这边推荐一篇博文[Spring Security(五)--动手实现一个IP_Login](https://www.cnkirito.moe/2017/10/01/spring-security-5/)，写得比较详细。

### 3.2 `FilterSecurityInterceptor`
`FilterSecurityInterceptor`是filterchain中比较复杂，也是比较核心的过滤器，主要负责web应用安全授权的工作。首先看下对于自定义的`FilterSecurityInterceptor`配置。


```java
	@Override
    public void configure(HttpSecurity http) throws Exception {
		
		...
		//添加CustomSecurityFilter，过滤器的顺序放在FilterSecurityInterceptor
        http.antMatcher("/oauth/check_token").addFilterAt(customSecurityFilter(), FilterSecurityInterceptor.class);

    }
    //提供实例化的自定义过滤器
    @Bean
    public CustomSecurityFilter customSecurityFilter() {
        return new CustomSecurityFilter();
    }
```

从上述配置可以看到，在`FilterSecurityInterceptor`的位置注册了`CustomSecurityFilter`，对于匹配到`/oauth/check_token`，则会调用该进入该过滤器。下图为`FilterSecurityInterceptor`的类图，在其中还添加了`CustomSecurityFilter`和相关实现的接口的类，方便读者对比着看。

![FilterSecurityInterceptor](http://ovcjgn2x0.bkt.clouddn.com/filterSecurity.png "FilterSecurityInterceptor类图")

`CustomSecurityFilter`是模仿`FilterSecurityInterceptor`实现，继承`AbstractSecurityInterceptor`和实现`Filter`接口。整个过程需要依赖`AuthenticationManager`、`AccessDecisionManager`和`FilterInvocationSecurityMetadataSource`。
`AuthenticationManager`是认证管理器，实现用户认证的入口；`AccessDecisionManager`是访问决策器，决定某个用户具有的角色，是否有足够的权限去访问某个资源；`FilterInvocationSecurityMetadataSource`是资源源数据定义，即定义某一资源可以被哪些角色访问。   
从上面的类图中可以看到自定义的`CustomSecurityFilter`同时又实现了
`AccessDecisionManager`和`FilterInvocationSecurityMetadataSource`。分别为`SecureResourceFilterInvocationDefinitionSource`和`SecurityAccessDecisionManager`。下面分析下主要的配置。

```java
//通过一个实现的filter，对HTTP资源进行安全处理
public class FilterSecurityInterceptor extends AbstractSecurityInterceptor implements Filter {
	//被filter chain真实调用的方法，通过invoke代理
	public void doFilter(ServletRequest request, ServletResponse response,
			FilterChain chain) throws IOException, ServletException {
		FilterInvocation fi = new FilterInvocation(request, response, chain);
		invoke(fi);
	}
	//代理的方法
	public void invoke(FilterInvocation fi) throws IOException, ServletException 	{
		//...省略
	}
}

```

上述代码是`FilterSecurityInterceptor`中的实现，具体实现细节就没列出了，我们这边重点在于对自定义的实现进行讲解。

```java
public class CustomSecurityFilter extends AbstractSecurityInterceptor implements Filter {
   
    @Autowired
    SecureResourceFilterInvocationDefinitionSource invocationSource;

    @Autowired
    private AuthenticationManager authenticationManager;

    @Autowired
    private SecurityAccessDecisionManager decisionManager;

	//设置父类中的属性
    @PostConstruct
    public void init() {
        super.setAccessDecisionManager(decisionManager);
        super.setAuthenticationManager(authenticationManager);
    }
	//主要的过滤方法，与原来的一致
    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        //logger.info("doFilter in Security ");
        //构造一个FilterInvocation，封装request, response, chain
        FilterInvocation fi = new FilterInvocation(servletRequest, servletResponse, filterChain);
        //beforeInvocation会调用SecureResourceDataSource中的逻辑，类似于aop中的before 
        InterceptorStatusToken token = super.beforeInvocation(fi);
        try {
        	//执行下一个拦截器
            fi.getChain().doFilter(fi.getRequest(), fi.getResponse());            
        } finally {
        	//完成后续工作，类似于aop中的after 
            super.afterInvocation(token, null);
        }
    }
    
    //...
    
    //资源源数据定义，设置为自定义的SecureResourceFilterInvocationDefinitionSource
    @Override
    public SecurityMetadataSource obtainSecurityMetadataSource() {
        return invocationSource;
    }
}
```

上面自定义的`CustomSecurityFilter`，与我们之前的讲解是一样的流程。主要依赖的三个接口都有在实现中实例化注入。看下父类的beforeInvocation方法，其中省略了一些不重要的代码片段。

```java
protected InterceptorStatusToken beforeInvocation(Object object) {  
    //根据SecurityMetadataSource获取配置的权限属性  
    Collection<ConfigAttribute> attributes = this.obtainSecurityMetadataSource().getAttributes(object);  
    //...  
    //判断是否需要对认证实体重新认证，默认为否  
    Authentication authenticated = authenticateIfRequired();  
  
    // Attempt authorization  
    try {  
        //决策管理器开始决定是否授权，如果授权失败，直接抛出AccessDeniedException  
        this.accessDecisionManager.decide(authenticated, object, attributes);  
    }  
    catch (AccessDeniedException accessDeniedException) {  
        publishEvent(new AuthorizationFailureEvent(object, attributes, authenticated,  
                accessDeniedException));  
  
        throw accessDeniedException;  
    }  
}  
```
上面代码可以看出，第一步是根据SecurityMetadataSource获取配置的权限属性，accessDecisionManager会用到权限列表信息。然后判断是否需要对认证实体重新认证，默认为否。第二步是接着决策管理器开始决定是否授权，如果授权失败，直接抛出AccessDeniedException。

(1). 获取配置的权限属性

```java
public class SecureResourceFilterInvocationDefinitionSource implements FilterInvocationSecurityMetadataSource, InitializingBean {
	private PathMatcher matcher;
	//map保存配置的URL对应的权限集
    private static Map<String, Collection<ConfigAttribute>> map = new HashMap<>();

	//根据传入的对象URL进行循环
    @Override
    public Collection<ConfigAttribute> getAttributes(Object o) throws IllegalArgumentException {
        logger.info("getAttributes");
        //应该做instanceof
        FilterInvocation filterInvocation = (FilterInvocation) o;
        //String method = filterInvocation.getHttpRequest().getMethod();
        String requestURI = filterInvocation.getRequestUrl();
        //循环资源路径，当访问的Url和资源路径url匹配时，返回该Url所需要的权限
        for (Iterator<Map.Entry<String, Collection<ConfigAttribute>>> iterator = map.entrySet().iterator(); iter.hasNext(); ) {
            Map.Entry<String, Collection<ConfigAttribute>> entry = iterator.next();
            String url = entry.getKey();

            if (matcher.match(url, requestURI)) {
                return map.get(requestURI);
            }
        }
        return null;
    }
    
	//... 
	
	//设置权限集，即上述的map
    @Override
    public void afterPropertiesSet() throws Exception {
        logger.info("afterPropertiesSet");
        //用来匹配访问资源路径
        this.matcher = new AntPathMatcher();
        //可以有多个权限
        Collection<ConfigAttribute> atts = new ArrayList<>();
        ConfigAttribute c1 = new SecurityConfig("ROLE_ADMIN");
        atts.add(c1);
        map.put("/oauth/check_token", atts);
    }
}
```

上面是getAttributes()实现的具体细节，将请求的URL取出进行匹配事先设定的受限资源，最后返回需要的权限、角色。系统在启动的时候就会读取到配置的map集合，对于拦截到请求进行匹配。代码中注释比较详细，这边不多说。

(2). 决策管理器

```java
public class SecurityAccessDecisionManager implements AccessDecisionManager {
	//...
	
    @Override
    public void decide(Authentication authentication, Object o, Collection<ConfigAttribute> collection) throws AccessDeniedException, InsufficientAuthenticationException {
        logger.info("decide url and permission");
        //集合为空
        if (collection == null) {
            return;
        }

        Iterator<ConfigAttribute> ite = collection.iterator();
        //判断用户所拥有的权限，是否符合对应的Url权限，如果实现了UserDetailsService，则用户权限是loadUserByUsername返回用户所对应的权限
        while (ite.hasNext()) {
            ConfigAttribute ca = ite.next();
            String needRole = ca.getAttribute();
            for (GrantedAuthority ga : authentication.getAuthorities()) {
                logger.info("GrantedAuthority: {}", ga);
                if (needRole.equals(ga.getAuthority())) {
                    return;
                }
            }
        }
        logger.error("AccessDecisionManager: no right!");
        throw new AccessDeniedException("no right!");
    }
    
	//...
}
```

上面的代码是决策管理器的实现，其逻辑也比较简单，将请求所具有的权限与设定的受限资源所需的进行匹配，如果具有则返回，否则抛出没有正确的权限异常。默认提供的决策管理器有三种，分别为AffirmativeBased、ConsensusBased、UnanimousBased，篇幅有限，我们这边不再扩展了。   

补充一下，所具有的权限是通过之前配置的认证方式，有password认证和client认证两种。我们之前在授权服务器中配置了`withClientDetails`，所以用frontend身份验证获得的权限是我们预先配置在数据库中的authorities。

## 4. 总结
Auth系统主要功能是授权认证和鉴权。项目微服务化后，原有的单体应用基于HttpSession认证鉴权不能满足微服务架构下的需求。每个微服务都需要对访问进行鉴权，每个微应用都需要明确当前访问用户以及其权限，尤其当有多个客户端，包括web端、移动端等等，单体应用架构下的鉴权方式就不是特别合适了。权限服务作为基础的公共服务，也需要微服务化。

笔者的设计中，Auth服务一方面进行授权认证，另一方面是基于token进行身份合法性和API级别的权限校验。对于某个服务的请求，经过网关会调用Auth服务，对token合法性进行验证。同时笔者根据当前项目的整体情况，存在部分遗留服务，这些遗留服务又没有足够的时间和人力立马进行微服务改造，而且还需要继续运行。为了适配当前新的架构，采取的方案就是对这些遗留服务的操作API，在Auth服务进行API级别的操作权限鉴定。API级别的操作权限校验需要的上下文信息需要结合业务，与客户端进行商定，应该在token能取到相应信息，传递给Auth服务，不过应尽量减少在header取上下文校验的信息。

笔者将本次开发Auth系统所涉及的大部分代码及源码进行了解析，至于没有讲到的一些内容和细节，读者可以自行扩展。

## 5. 不足与后续工作
### 5.1 存在的不足
- API级别操作权限校验的通用性   

	(1). 对于API级别操作权限校验，需要在网关处调用时构造相应的上下文信息。上下文信息基本依赖于	token中的payload，如果信息太多引起token太长，导致每次客户端的请求头部长度变长。  
	 
	(2). 并不是所有的操作接口都能覆盖到，这个问题是比较严重的，根据上下文集合很可能出现好多接口	的权限没法鉴定，最后的结果就是API级别操作权限校验失败的是绝对没有权限访问该接口，而通过不一定能访问，因为该接口涉及到的上下文根本没法完全得到。我们的项目在现阶段，定义的最小上下文集合能勉强覆盖到，但是对于后面扩增的服务接口真的是不乐观。   
	
	(3). 每个服务的每个接口都在Auth服务注册其所需要的权限，太过麻烦，Auth服务需要额外维护这样的信息。
- 网关处调用Auth服务带来的系统吞吐量瓶颈   

	(1). 这个其实很容易理解，Auth服务作为公共的基础服务，大多数服务接口都会需要鉴权，Auth服务需要经过复杂。   
	
	(2). 网关调用Auth服务，阻塞调用，只有等Auth服务返回校验结果，才会做进一步处理。虽说Auth服务可以多实例部署，但是并发量大了之后，其瓶颈明显可见，严重可能会造成整个系统的不可用。


### 5.2 后续工作

- 从整个系统设计角度来讲，API级别操作权限后期将会分散在各个服务的接口上，由各个接口负责其所需要的权限、身份等。Spring Security对于接口级别的权限校验也是支持的，之所以采用这样的做法，也是为了兼容新服务和遗留的服务，主要是针对遗留服务，新的服务采用的是分散在各个接口之上。
- 将API级别操作权限分散到各个服务接口之后，相应的能提升Auth服务的响应。网关能够及时的对请求进行转发或者拒绝。
- API级别操作权限所需要的上下文信息对各个接口真的设计的很复杂，这边我们确实花了时间，同时管理移动服务的好几百操作接口所对应的权限，非常烦。！

**本文的源码地址：   
GitHub：https://github.com/keets2012/Auth-service   
码云： https://gitee.com/keets/Auth-Service**


---
### 参考
1. [配置表单登录](http://www.tianshouzhi.com/api/tutorials/spring_security_4/265)
2. [Spring Security3源码分析-FilterSecurityInterceptor分析](http://dead-knight.iteye.com/blog/1513913)
3. [Core Security Filters](https://docs.spring.io/spring-security/site/docs/3.0.x/reference/core-web-filters.html)
4. [Spring Security(四)--核心过滤器源码分析](https://www.cnkirito.moe/2017/09/30/spring-security-4/)


### 相关阅读
[认证鉴权与API权限控制在微服务架构中的设计与实现（一）](http://blueskykong.com/2017/10/19/security1/)   
[认证鉴权与API权限控制在微服务架构中的设计与实现（二）](http://blueskykong.com/2017/10/22/security2/)
[认证鉴权与API权限控制在微服务架构中的设计与实现（三）](http://blueskykong.com/2017/10/24/security3/)


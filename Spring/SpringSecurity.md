## SpringSecurity

### 主要组件

`SecurityContextHolder`：保存SecurityContext到当前线程上下文

`SecurityContext`：保存已通过认证的Authentication

`SecurityContext`：安全上下文，从`SecurityContextHolder`中获取，存储Authentication的容器

`Authentication`：用于存储用户认证信息，最常见的就是用户名和密码

`UserDetailsService`和`UserDetails`：定义如何获取保存在服务端的用户信息

`AuthenticationManager`：校验用户提供的认证信息和保存在服务器中的用户信息

`AuthenticationProvider`：用于处理Authentication的接口，可实现不同的校验方式

`AbstractAuthenticationProcessingFilter`：认证处理过滤器，主要是定义如何处理用户认证数据

### 核心过滤器

**SecurityContextPersistenceFilter**： 整个Spring Security 过滤器链的开端，它有两个作用：一是当请求到来时，检查`Session`中是否存在`SecurityContext`,如果不存在，就创建一个新的`SecurityContext`。二是请求结束时将`SecurityContext`放入 `Session`中，并清空 `SecurityContextHolder`。

**UsernamePasswordAuthenticationFilter**： 继承自抽象类 `AbstractAuthenticationProcessingFilter`，当进行表单登录时，该Filter将用户名和密码封装成一个 `UsernamePasswordAuthentication`进行验证。

**AnonymousAuthenticationFilter**: 匿名身份过滤器，当前面的Filter认证后依然没有用户信息时，该Filter会生成一个匿名身份——`AnonymousAuthenticationToken`。一般的作用是用于匿名登录。

**ExceptionTranslationFilter**： 异常转换过滤器，用于处理 `FilterSecurityInterceptor`抛出的异常。

**FilterSecurityInterceptor**： 过滤器链最后的关卡，从 SecurityContextHolder中获取 Authentication，比对用户拥有的权限和所访问资源需要的权限。

### 表单登录认证流程

1. 进入 UsernamePasswordAuthenticationFilter 构建一个未认证的 UsernamePasswordAuthenticationToken
2. 随后交给 AuthenticationManager 进行验证
3. AuthenticationManager 找到对应的 AuthenticationProvider进行认证
4. AuthenticationProvider找到上下文中的UserDetailsService 中寻找用户然后对比
5. 验证成功返回 Authentication 放入 SecurityContextHolder中

#### UsernamePasswordAuthenticationFilter

进入过滤器链中的 `AbstractAuthenticationProcessingFilter`，此类是`UsernamePasswordAuthenticationFilter`的父类

```java
public abstract class AbstractAuthenticationProcessingFilter extends GenericFilterBean
    implements ApplicationEventPublisherAware, MessageSourceAware {
    ......
        @Override
        public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
        throws IOException, ServletException {
        doFilter((HttpServletRequest) request, (HttpServletResponse) response, chain);
    }

    private void doFilter(HttpServletRequest request, HttpServletResponse response, FilterChain chain)
        throws IOException, ServletException {
        if (!requiresAuthentication(request, response)) {
            chain.doFilter(request, response);
            return;
        }
        try {
            //调用子类 UsernamePasswordAuthenticationFilter 的 attemptAuthentication 方法
            Authentication authenticationResult = attemptAuthentication(request, response);
            if (authenticationResult == null) {
                // return immediately as subclass has indicated that it hasn't completed
                //子类未完成认证，立即返回
                return;
            }
            this.sessionStrategy.onAuthentication(authenticationResult, request, response);
            // Authentication success
            if (this.continueChainBeforeSuccessfulAuthentication) {
                chain.doFilter(request, response);
            }
            //认证成功,主要做以下几件事
            //1.将认证成功的Authentication对象放入SecurityContextHolder
            //2.调用RememberMeServices处理“记住我”功能
            //3.调用认证成功处理器
            //4.发布认证成功事件
            successfulAuthentication(request, response, chain, authenticationResult);
        }
        catch (InternalAuthenticationServiceException failed) {
            this.logger.error("An internal error occurred while trying to authenticate the user.", failed);
            //认证失败，主要做以下几件事
            //1.清空SecurityContextHolder
            //2.将异常信息存入session
            //3.调用RememberMeServices处理“记住我”功能
            //4.调用认证失败处理器
            unsuccessfulAuthentication(request, response, failed);
        }
        catch (AuthenticationException ex) {
            // Authentication failed
            unsuccessfulAuthentication(request, response, ex);
        }
    }
    ......
}
```

主要处理认证的逻辑在子类的 `attemptAuthentication`方法

```java
public class UsernamePasswordAuthenticationFilter extends AbstractAuthenticationProcessingFilter {
	......
    @Override
    public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response)
        throws AuthenticationException {
        if (this.postOnly && !request.getMethod().equals("POST")) {
            throw new AuthenticationServiceException("Authentication method not supported: " + request.getMethod());
        }
        //获取表单中的用户名和密码
        String username = obtainUsername(request);
        username = (username != null) ? username : "";
        username = username.trim();
        String password = obtainPassword(request);
        password = (password != null) ? password : "";
        //将用户名和密码封装成一个未认证的UsernamePasswordAuthenticationToken对象
        UsernamePasswordAuthenticationToken authRequest = new UsernamePasswordAuthenticationToken(username, password);
        // Allow subclasses to set the "details" property
        setDetails(request, authRequest);
        //交给AuthenticationManager处理认证逻辑，并返回Authentication
        return this.getAuthenticationManager().authenticate(authRequest);
    }
    ......
}
```

`UsernamePasswordAuthenticationFilter`会将认证逻辑交给 `AuthenticationManager`处理，`AuthenticationManager`是身份认证的核心接口，它的实现类是 `ProviderManager`，而 `ProviderManager`又将请求委托给一个 `AuthenticationProvider`列表，列表中的每一个 AuthenticationProvider将会被依次查询是否需要通过其进行验证,每个 provider的验证结果只有两个情况：抛出一个异常或者完全填充一个 Authentication对象的所有属性

选择其中一个 `AbstractUserDetailsAuthenticationProvider`来看

```java
public abstract class AbstractUserDetailsAuthenticationProvider
    implements AuthenticationProvider, InitializingBean, MessageSourceAware {
    ......
    @Override
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {
        Assert.isInstanceOf(UsernamePasswordAuthenticationToken.class, authentication,
                            () -> this.messages.getMessage("AbstractUserDetailsAuthenticationProvider.onlySupports",
                                                           "Only UsernamePasswordAuthenticationToken is supported"));
        String username = determineUsername(authentication);
        boolean cacheWasUsed = true;
        UserDetails user = this.userCache.getUserFromCache(username);
        if (user == null) {
            cacheWasUsed = false;
            try {
                //调用子类DaoAuthenticationProvider的retrieveUser()方法获取UserDetails
                user = retrieveUser(username, (UsernamePasswordAuthenticationToken) authentication);
            }
            //为获取到UserDetails抛出异常信息
            catch (UsernameNotFoundException ex) {
                this.logger.debug("Failed to find user '" + username + "'");
                if (!this.hideUserNotFoundExceptions) {
                    throw ex;
                }
                throw new BadCredentialsException(this.messages
                                                  .getMessage("AbstractUserDetailsAuthenticationProvider.badCredentials", "Bad credentials"));
            }
            Assert.notNull(user, "retrieveUser returned null - a violation of the interface contract");
        }
        try {
            //对UserDetails的一些属性进行预检查，即判断用户是否锁定，是否可用以及用户是否过期
            this.preAuthenticationChecks.check(user);
            //对UserDetails附加的检查，对传入的Authentication与从数据库中获取的UserDetails进行密码匹配
            additionalAuthenticationChecks(user, (UsernamePasswordAuthenticationToken) authentication);
        }
        catch (AuthenticationException ex) {
            if (!cacheWasUsed) {
                throw ex;
            }
            // There was a problem, so try again after checking
            // we're using latest data (i.e. not from the cache)
            cacheWasUsed = false;
            user = retrieveUser(username, (UsernamePasswordAuthenticationToken) authentication);
            this.preAuthenticationChecks.check(user);
            additionalAuthenticationChecks(user, (UsernamePasswordAuthenticationToken) authentication);
        }
        //对UserDetails进行后检查，检查UserDetails的密码是否过期
        this.postAuthenticationChecks.check(user);
        if (!cacheWasUsed) {
            this.userCache.putUserInCache(user);
        }
        Object principalToReturn = user;
        if (this.forcePrincipalAsString) {
            principalToReturn = user.getUsername();
        }
        //上面所有检查成功后，用传入的用户信息和获取的UserDetails生成一个成功验证的Authentication
        return createSuccessAuthentication(principalToReturn, authentication, user);
    }
    ......
}
```

下面重点看一下`DaoAuthenticationProvider`的`retrieveUser()`方法

```java
@Override
protected final UserDetails retrieveUser(String username, UsernamePasswordAuthenticationToken authentication)
    throws AuthenticationException {
    prepareTimingAttackProtection();
    try {
        //通过UserDetailsService获取UserDetails
        UserDetails loadedUser = this.getUserDetailsService().loadUserByUsername(username);
        if (loadedUser == null) {
            throw new InternalAuthenticationServiceException(
                "UserDetailsService returned null, which is an interface contract violation");
        }
        return loadedUser;
    }
    catch (UsernameNotFoundException ex) {
        mitigateAgainstTimingAttack(authentication);
        throw ex;
    }
    catch (InternalAuthenticationServiceException ex) {
        throw ex;
    }
    catch (Exception ex) {
        throw new InternalAuthenticationServiceException(ex.getMessage(), ex);
    }
}
```

此方法中最核心的就是 `getUserDetailsService().loadUserByUsername(username);`，本质就是调用UserDetailsService的loadUserByUsername()方法加载UserDetails，UserDetails包含了比Authentication更加详细的用户信息。UserDetailsService常见的实现类有JdbcDaoImpl，InMemoryUserDetailsManager，前者从数据库加载用户，后者从内存中加载用户。我们也可以自己实现UserDetailsServices接口，比如我们是如果是基于数据库进行身份认证，那么我们可以手动实现该接口，而不用JdbcDaoImpl。

最后将调用 `AbstractUserDetailsAuthenticationProvider.authenticate()`的 `createSuccessAuthentication()`生成一个认证成功的Authentication对象，就是组合UserDetails和传入的Authentication。

```java
protected Authentication createSuccessAuthentication(Object principal, Authentication authentication,
                                                     UserDetails user) {
    // Ensure we return the original credentials the user supplied,
    // so subsequent attempts are successful even with encoded passwords.
    // Also ensure we return the original getDetails(), so that future
    // authentication events after cache expiry contain the details
    UsernamePasswordAuthenticationToken result = new UsernamePasswordAuthenticationToken(principal,
                                                                                         authentication.getCredentials(), this.authoritiesMapper.mapAuthorities(user.getAuthorities()));
    result.setDetails(authentication.getDetails());
    this.logger.debug("Authenticated user");
    return result;
}
```

最终返回的Authentication对象将一步步向上返回，到 `AbstractAuthenticationProcessingFilter`中，放入 `SecurityContextHolder`。

### FilterSecurityInterceptor

`FilterSecurityInterceptor` 过滤器是最后的关卡，之前的请求最终会来到这里，它的大致工作流程是

- 封装请求信息
- 从系统中读取配信息，即资源所需的权限信息
- 从 `SecurityContextHolder`中获取之前认证过的 `Authentication`对象，即表示当前用户所拥有的权限
- 然后根据上面获取到的三种信息，传入一个权限校验器中，对于当前请求来说，比对用户拥有的权限和资源所需的权限。若比对成功，则进入真正系统的请求处理逻辑，反之，会抛出相应的异常

```java
public class FilterSecurityInterceptor extends AbstractSecurityInterceptor implements Filter {
    ......
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
        throws IOException, ServletException {
        //封装request、response请求，并调用核心方法
        invoke(new FilterInvocation(request, response, chain));
    }
	......
    public void invoke(FilterInvocation filterInvocation) throws IOException, ServletException {
        if (isApplied(filterInvocation) && this.observeOncePerRequest) {
            // filter already applied to this request and user wants us to observe
            // once-per-request handling, so don't re-do security checking
            filterInvocation.getChain().doFilter(filterInvocation.getRequest(), filterInvocation.getResponse());
            return;
        }
        // first time this request being called, so perform security checking
        //判断是否第一次经过此过滤器
        if (filterInvocation.getRequest() != null && this.observeOncePerRequest) {
            //若已经历过此拦截器，则不再执行后续逻辑，直接进入下一个拦截器
            filterInvocation.getRequest().setAttribute(FILTER_APPLIED, Boolean.TRUE);
        }
        //调用父类的方法，执行授权判断逻辑
        InterceptorStatusToken token = super.beforeInvocation(filterInvocation);
        try {
            filterInvocation.getChain().doFilter(filterInvocation.getRequest(), filterInvocation.getResponse());
        }
        finally {
            super.finallyInvocation(token);
        }
        super.afterInvocation(token, null);
    }
    ......
}
```

核心处理逻辑需要父类 `AbstractSecurityInterceptor`的 `beforeInvocation()`方法

```java
protected InterceptorStatusToken beforeInvocation(Object object) {
    Assert.notNull(object, "Object was null");
    if (!getSecureObjectClass().isAssignableFrom(object.getClass())) {
        throw new IllegalArgumentException("Security invocation attempted for object " + object.getClass().getName()
                                           + " but AbstractSecurityInterceptor only configured to support secure objects of type: "
                                           + getSecureObjectClass());
    }
    //读取Spring Security的配置信息，将其封装成 ConfigAttribute
    Collection<ConfigAttribute> attributes = this.obtainSecurityMetadataSource().getAttributes(object);
    if (CollectionUtils.isEmpty(attributes)) {
        Assert.isTrue(!this.rejectPublicInvocations,
                      () -> "Secure object invocation " + object
                      + " was denied as public invocations are not allowed via this interceptor. "
                      + "This indicates a configuration error because the "
                      + "rejectPublicInvocations property is set to 'true'");
        if (this.logger.isDebugEnabled()) {
            this.logger.debug(LogMessage.format("Authorized public object %s", object));
        }
        publishEvent(new PublicInvocationEvent(object));
        return null; // no further work post-invocation
    }
    if (SecurityContextHolder.getContext().getAuthentication() == null) {
        credentialsNotFound(this.messages.getMessage("AbstractSecurityInterceptor.authenticationNotFound",
                                                     "An Authentication object was not found in the SecurityContext"), object, attributes);
    }
    //从SpringContextHolder中获取Authentication
    Authentication authenticated = authenticateIfRequired();
    if (this.logger.isTraceEnabled()) {
        this.logger.trace(LogMessage.format("Authorizing %s with attributes %s", object, attributes));
    }
    // Attempt authorization
    //启动授权流程
    attemptAuthorization(object, attributes, authenticated);
    if (this.logger.isDebugEnabled()) {
        this.logger.debug(LogMessage.format("Authorized %s with attributes %s", object, attributes));
    }
    if (this.publishAuthorizationSuccess) {
        publishEvent(new AuthorizedEvent(object, attributes, authenticated));
    }

    // Attempt to run as a different user
    Authentication runAs = this.runAsManager.buildRunAs(authenticated, object, attributes);
    if (runAs != null) {
        SecurityContext origCtx = SecurityContextHolder.getContext();
        SecurityContextHolder.setContext(SecurityContextHolder.createEmptyContext());
        SecurityContextHolder.getContext().setAuthentication(runAs);

        if (this.logger.isDebugEnabled()) {
            this.logger.debug(LogMessage.format("Switched to RunAs authentication %s", runAs));
        }
        // need to revert to token.Authenticated post-invocation
        return new InterceptorStatusToken(origCtx, true, attributes, object);
    }
    this.logger.trace("Did not switch RunAs authentication since RunAsManager returned null");
    // no further work post-invocation
    return new InterceptorStatusToken(SecurityContextHolder.getContext(), false, attributes, object);

}
```

`attemptAuthorization(object, attributes, authenticated);`中将会把配置信息和用户信息，和请求信息一起传入`AccessDecisionManager`的`decide()`方法，执行授权校验逻辑。

`AccessDecisionManager` 本身是一个接口，它的 实现类是 `AbstractAccessDecisionManager`,而 `AbstractAccessDecisionManager`也是一个抽象类，它的实现类有三个，常用的是 `AffirmativeBased`，最终的授权校验逻辑是 AffirmativeBased 实现的

```java
@Override
@SuppressWarnings({ "rawtypes", "unchecked" })
public void decide(Authentication authentication, Object object, Collection<ConfigAttribute> configAttributes)
    throws AccessDeniedException {
    int deny = 0;
    for (AccessDecisionVoter voter : getDecisionVoters()) {
        //投票器执行投票
        int result = voter.vote(authentication, object, configAttributes);
        switch (result) {
            case AccessDecisionVoter.ACCESS_GRANTED:
                return;
            case AccessDecisionVoter.ACCESS_DENIED:
                deny++;
                break;
            default:
                break;
        }
    }
    if (deny > 0) {
        throw new AccessDeniedException(
            this.messages.getMessage("AbstractAccessDecisionManager.accessDenied", "Access is denied"));
    }
    // To get this far, every AccessDecisionVoter abstained
    checkAllowIfAllAbstainDecisions();
}
```

该方法中执行`AccessDecisionVoter`的校验逻辑，如果校验失败就抛出`AccessDeniedException`异常。

### 自定义登录

理解了SpringSecurity的认证流程，我们可以自己仿照UsernamePasswordAuthentication这种用户名密码认证方式来自定义一种登录认证方式。

1. 实现**UserDetailsService**接口，自定义loadUserByUsername()方法

```java
@Service
public class UserService implements UserDetailsService {

    @Autowired
    private  IUserService iUserService;

    @Override
    public UserDetails loadUserByUsername(String s) throws UsernameNotFoundException {
        SysUser user = iUserService.findByUsername(s);
        if (user == null) {
            throw new UsernameNotFoundException("用户不存在");
        }
        //把角色放入认证器里
        Collection<GrantedAuthority> authorities = new ArrayList<>();
        List<String> roles = user.getRoles();
        for (String role : roles) {
            authorities.add(new SimpleGrantedAuthority(role));
        }
        return new User(user.getUserName(), user.getPassword(), authorities);
    }

}
```

2. 自定义**AuthenticationFilter**实现对自定义登录的拦截，继承AbstractAuthenticationProcessingFilter，并构建未认证的SmsCodeAuthenticationToken

```java
/**
 * 短信登录的鉴权过滤器，模仿 UsernamePasswordAuthenticationFilter 实现
 */
public class SmsCodeAuthenticationFilter extends AbstractAuthenticationProcessingFilter {
    /**
     * form表单中手机号码的字段name
     */
    public static final String SPRING_SECURITY_FORM_MOBILE_KEY = "mobile";

    private String mobileParameter = SPRING_SECURITY_FORM_MOBILE_KEY;
    /**
     * 是否仅 POST 方式
     */
    private boolean postOnly = true;

    public SmsCodeAuthenticationFilter() {
        // 短信登录的请求 post 方式的 /sms/login
        super(new AntPathRequestMatcher("/sms/login", "POST"));
    }

    @Override
    public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response) throws AuthenticationException {
        if (postOnly && !request.getMethod().equals("POST")) {
            throw new AuthenticationServiceException(
                    "Authentication method not supported: " + request.getMethod());
        }

        String mobile = obtainMobile(request);

        if (mobile == null) {
            mobile = "";
        }

        mobile = mobile.trim();

        SmsCodeAuthenticationToken authRequest = new SmsCodeAuthenticationToken(mobile);

        // Allow subclasses to set the "details" property
        setDetails(request, authRequest);

        return this.getAuthenticationManager().authenticate(authRequest);
    }

    protected String obtainMobile(HttpServletRequest request) {
        return request.getParameter(mobileParameter);
    }

    protected void setDetails(HttpServletRequest request, SmsCodeAuthenticationToken authRequest) {
        authRequest.setDetails(authenticationDetailsSource.buildDetails(request));
    }

    public String getMobileParameter() {
        return mobileParameter;
    }

    public void setMobileParameter(String mobileParameter) {
        Assert.hasText(mobileParameter, "Mobile parameter must not be empty or null");
        this.mobileParameter = mobileParameter;
    }

    public void setPostOnly(boolean postOnly) {
        this.postOnly = postOnly;
    }
}
```
自定义认证对象**AuthenticationToken**

```java
//替换原有系统的 UsernamePasswordAuthenticationToken 用来做验证
public class SmsCodeAuthenticationToken extends AbstractAuthenticationToken {
    private static final long serialVersionUID = SpringSecurityCoreVersion.SERIAL_VERSION_UID;

    //在 UsernamePasswordAuthenticationToken 中该字段代表登录的用户名，在这里就代表登录的手机号码
    private final Object principal;
	
    //构建一个没有鉴权的 SmsCodeAuthenticationToken
    public SmsCodeAuthenticationToken(Object principal) {
        super(null);
        this.principal = principal;
        setAuthenticated(false);
    }

	//构建拥有鉴权的 SmsCodeAuthenticationToken
    public SmsCodeAuthenticationToken(Object principal, Collection<? extends GrantedAuthority> authorities) {
        super(authorities);
        this.principal = principal;
        super.setAuthenticated(true); // must use super, as we override
    }

    public Object getCredentials() {
        return null;
    }

    public Object getPrincipal() {
        return this.principal;
    }

    public void setAuthenticated(boolean isAuthenticated) throws IllegalArgumentException {
        if (isAuthenticated) {
            throw new IllegalArgumentException(
                    "Cannot set this token to trusted - use constructor which takes a GrantedAuthority list instead");
        }

        super.setAuthenticated(false);
    }

    @Override
    public void eraseCredentials() {
        super.eraseCredentials();
    }
}
```

3. 自定义**AuthenticationProvider**进行验证

```java
//短信登陆鉴权 Provider，要求实现 AuthenticationProvider 接口
public class SmsCodeAuthenticationProvider implements AuthenticationProvider {
	//上下文中的 userDetailsService
    private UserDetailsService userDetailsService;

    @Override
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {
        SmsCodeAuthenticationToken authenticationToken = (SmsCodeAuthenticationToken) authentication;

        String mobile = (String) authenticationToken.getPrincipal();

        checkSmsCode(mobile);

        UserDetails userDetails = userDetailsService.loadUserByUsername(mobile);

        // 此时鉴权成功后，应当重新 new 一个拥有鉴权的 authenticationResult 返回
        SmsCodeAuthenticationToken authenticationResult = new SmsCodeAuthenticationToken(userDetails, userDetails.getAuthorities());

        authenticationResult.setDetails(authenticationToken.getDetails());

        return authenticationResult;
    }

    private void checkSmsCode(String mobile) {
        HttpServletRequest request = ((ServletRequestAttributes) Objects.requireNonNull(RequestContextHolder.getRequestAttributes())).getRequest();
      
        String inputCode = request.getParameter("smsCode");
        
        //这里的验证码我们放session里，这里拿出来跟用户输入的做对比
        Map<String, Object> smsCode = (Map<String, Object>) request.getSession().getAttribute("smsCode");
        if (smsCode == null) {
            throw new BadCredentialsException("未检测到申请验证码");
        }

        String applyMobile = (String) smsCode.get("mobile");

        int code = (int) smsCode.get("code");

        if (!applyMobile.equals(mobile)) {
            throw new BadCredentialsException("申请的手机号码与登录手机号码不一致");
        }
        if (code != Integer.parseInt(inputCode)) {
            throw new BadCredentialsException("验证码错误");
        }
    }

    @Override
    public boolean supports(Class<?> authentication) {
        // 判断 authentication 是不是 SmsCodeAuthenticationToken 的子类或子接口
        return SmsCodeAuthenticationToken.class.isAssignableFrom(authentication);
    }

    public UserDetailsService getUserDetailsService() {
        return userDetailsService;
    }

    public void setUserDetailsService(UserDetailsService userDetailsService) {
        this.userDetailsService = userDetailsService;
    }
}
```

4. 通过SecurityConfigurerAdapter将自定义的登录和security进行绑定

自定义**成功处理器**和**失败处理器**

```java
@Component
@Slf4j
public class CustomAuthenticationSuccessHandler extends SavedRequestAwareAuthenticationSuccessHandler {

    @Override
    public void onAuthenticationSuccess(HttpServletRequest request, HttpServletResponse response, Authentication authentication) throws IOException {
        log.info("登录成功");
        response.setStatus(HttpStatus.OK.value());
        ModelMap modelMap = GenerateModelMap.generateMap(HttpStatus.OK.value(), "登录成功");
        response.setContentType("application/json;charset=UTF-8");
        response.getWriter().write(JSON.toJSONString(modelMap));
    }

}
```

```java
@Component
@Slf4j
public class CustomAuthenticationFailureHandler extends SimpleUrlAuthenticationFailureHandler {

    @Override
    public void onAuthenticationFailure(HttpServletRequest request, HttpServletResponse response, AuthenticationException exception) throws IOException, ServletException {
        log.info("登录失败！");
        response.setStatus(HttpStatus.INTERNAL_SERVER_ERROR.value());
        ModelMap modelMap = GenerateModelMap.generateMap(HttpStatus.INTERNAL_SERVER_ERROR.value(), "验证失败");
        response.setContentType("application/json;charset=UTF-8");
        response.getWriter().write(JSON.toJSONString(modelMap));
    }
}
```
**加入到过滤链里**

SecurityConfigurerAdapter 顾名思义就是 SecurityConfigurer的适配器，需要把自定义的AuthenticationFilter AuthenticationToken AuthenticationProvider 放进去。

```java
@Component
public class SmsCodeAuthenticationSecurityConfig extends SecurityConfigurerAdapter<DefaultSecurityFilterChain, HttpSecurity> {
    @Autowired //自定义的UserDetailsService
    private UserService userService;
    @Autowired
    private AuthenticationSuccessHandler customAuthenticationSuccessHandler;
    @Autowired
    private AuthenticationFailureHandler customAuthenticationFailureHandler;


    @Override
    public void configure(HttpSecurity http) throws Exception {
        SmsCodeAuthenticationFilter smsCodeAuthenticationFilter = new SmsCodeAuthenticationFilter();
        //设置AuthenticationManager
        smsCodeAuthenticationFilter.setAuthenticationManager(http.getSharedObject(AuthenticationManager.class));
        //设置成功、失败处理器
        smsCodeAuthenticationFilter.setAuthenticationSuccessHandler(customAuthenticationSuccessHandler);
        smsCodeAuthenticationFilter.setAuthenticationFailureHandler(customAuthenticationFailureHandler);
        //设置UserDetailsService
        SmsCodeAuthenticationProvider smsCodeAuthenticationProvider = new SmsCodeAuthenticationProvider();
        smsCodeAuthenticationProvider.setUserDetailsService(userService);
        //把自定义的Provider放在过滤链中，addFilterAfter表示将其放在过滤链的哪个位置
        http.authenticationProvider(smsCodeAuthenticationProvider)
            .addFilterAfter(smsCodeAuthenticationFilter, UsernamePasswordAuthenticationFilter.class);
    }
}
```
这样只是加入到了security的过滤链里，但是并没有生效，要想使这些配置生效，还要在 WebSecurityConfigurerAdapter 里配置一下。

**WebSecurityConfigurerAdapter**

在配置中加入 http.apply(config) ，使自定义配置生效。

```java
@Configuration
@EnableWebSecurity
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    @Autowired
    private CustomAuthenticationFailureHandler customAuthenticationFailureHandler;
    @Autowired
    private CustomAuthenticationSuccessHandler customAuthenticationSuccessHandler;
    @Autowired //注入自定义的登陆流程
    private SmsCodeAuthenticationSecurityConfig smsCodeAuthenticationSecurityConfig;
    @Autowired
    private UserService userService;


    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.userDetailsService(userService).passwordEncoder(
                new PasswordEncoder() {
                    @Override
                    public String encode(CharSequence charSequence) {
                        return charSequence.toString();
                    }

                    @Override
                    public boolean matches(CharSequence charSequence, String s) {
                        return s.equals(charSequence.toString());
                    }
                });
    }


    @Override
    protected void configure(HttpSecurity http) throws Exception {
        //表单登陆配置
        http.formLogin()
                .failureHandler(customAuthenticationFailureHandler)
                .successHandler(customAuthenticationSuccessHandler)
                .loginPage("/login")
                .loginProcessingUrl("/authentication/form")
                .and();

        http.apply(smsCodeAuthenticationSecurityConfig)
                .and()
                .logout()
                .logoutUrl("/logout")
                .and()
                .authorizeRequests()
                // 如果有允许匿名的url，填在下面
                .antMatchers("/login", "/sms/**", "/authentication/form").permitAll()
                .anyRequest().authenticated();

        // 关闭CSRF跨域
        http.csrf().disable();
    }
}
```
至此已经完成自定义的登陆流程。

### RememberMe
#### 两种方式
**cookie**

在WebSecurityConfigurerAdapter 中 的 configure() 方法添加一个 `rememberMe() `

```java
@Override
protected void configure(HttpSecurity http) throws Exception {
        //http.httpBasic()  //httpBasic 登录
        http.formLogin()
            .loginPage("/login")
            .loginProcessingUrl("/authentication/form") // 自定义登录路径
            .failureHandler(failureAuthenticationHandler) // 自定义登录失败处理
            .successHandler(successAuthenticationHandler) // 自定义登录成功处理
            .and()
            .logout()
            .logoutUrl("/logout")
            .and()
            .authorizeRequests()// 对请求授权
            .antMatchers("/login", "/authentication/require",
                         "/authentication/form").permitAll()
            .anyRequest() // 任何请求
            .authenticated()//; // 都需要身份认证
            .and()
            .rememberMe()
            .rememberMeCookieName("remember")
            .tokenValiditySeconds(3600)
            .and()
            .csrf().disable();// 禁用跨站攻击
}
```
在登陆的时候勾选一下，这时候登陆成功后，会在浏览器cookie里自动保存一条名为remember-me 的cookie如果你需要改变这个名字可以在 HttpSecurity 链上加入`rememberMeCookieName("remember")` 即可。

**数据库存储**

使用 Cookie 存储虽然很方便，但是 Cookie 毕竟是保存在客户端的，而且 Cookie 的值还与用户名、密码这些敏感数据相关，虽然加密了，但是将敏感信息存在客户端，毕竟不太安全。

Spring security 还提供了另一种相对更安全的实现机制：在客户端的 Cookie 中，仅保存一个无意义的加密串（与用户名、密码等敏感数据无关），然后在数据库中保存该加密串-用户信息的对应关系，自动登录时，用 Cookie 中的加密串，到数据库中验证，如果通过，自动登录才算通过。

需要创建一张表来存储 token 信息

```sql
CREATE TABLE `persistent_logins` (
  `username` varchar(64) NOT NULL,
  `series` varchar(64) NOT NULL,
  `token` varchar(64) NOT NULL,
  `last_used` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`series`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```
创建配置类

```java
@Configuration
public class RemberMeConfig {

    private final DataSource dataSource;

    public RemberMeConfig(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    @Bean
    public PersistentTokenRepository persistentTokenRepository(){
        JdbcTokenRepositoryImpl tokenRepository = new JdbcTokenRepositoryImpl();
        tokenRepository.setDataSource(dataSource);
        // 如果token表不存在，使用下面语句可以初始化该表；若存在，请注释掉这条语句，否则会报错。
        //tokenRepository.setCreateTableOnStartup(true);
        return tokenRepository;
    }
}
```
然后在 configure() 中进行修改

```java
@Autowired
private  PersistentTokenRepository persistentTokenRepository;

@Override
protected void configure(HttpSecurity http) throws Exception {
        //http.httpBasic()  //httpBasic 登录
        http.formLogin()
            .loginPage("/login")
            .loginProcessingUrl("/authentication/form") // 自定义登录路径
            .failureHandler(failureAuthenticationHandler) // 自定义登录失败处理
            .successHandler(successAuthenticationHandler) // 自定义登录成功处理
            .and()
            .logout()
            .logoutUrl("/logout")
            .and()
            .authorizeRequests()// 对请求授权
            .antMatchers("/login", "/authentication/require",
                         "/authentication/form").permitAll()
            .anyRequest() // 任何请求
            .authenticated()//; // 都需要身份认证
            .and()
            .rememberMe()
            .tokenRepository(persistentTokenRepository())
            .tokenValiditySeconds(3600)
            .userDetailsService(userDetailsService)
            .and()
            .csrf().disable();// 禁用跨站攻击
}
```
#### 原理
1. UsernamePasswordAuthenticationFilter 认证成功后会走successfulAuthentication 方法
2. 然后经过successfulAuthentication 的 RememberMeService 其中有个 TokenRepository
3. TokenRepository 生成token，首先将 token 写入到浏览器的 Cookie 中，然后将 token、认证成功的用户名写入到数据库中
4. 下次请求时，会经过RememberMeAuthenticationFilter 它会读取 Cookie 中的 token，交给 RememberMeService 从数据库中查询记录
5. 如果存在记录，会读取用户名并去调用 UserDetailsService，获取用户信息，并将用户信息放入Spring Security 中，实现自动登陆。

### 权限控制
#### 静态权限

##### 方式一：configure


```java
@Configuration
@EnableWebSecurity
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    private final FailureAuthenticationHandler failureAuthenticationHandler;
    private final SuccessAuthenticationHandler successAuthenticationHandler;
    private final UserService userService;
    private final AccessDeniedAuthenticationHandler accessDeniedAuthenticationHandler;
   

    public WebSecurityConfig(UserService userService, FailureAuthenticationHandler failureAuthenticationHandler, SuccessAuthenticationHandler successAuthenticationHandler,AccessDeniedAuthenticationHandler accessDeniedAuthenticationHandler)   {
        this.userService = userService;
        this.failureAuthenticationHandler = failureAuthenticationHandler;
        this.successAuthenticationHandler = successAuthenticationHandler;
        this.accessDeniedAuthenticationHandler = accessDeniedAuthenticationHandler;
    }
    /**
     * 注入身份管理器bean
     *
     * @return
     * @throws Exception
     */
    @Bean
    @Override
    public AuthenticationManager authenticationManagerBean() throws Exception {
        return super.authenticationManagerBean();
    }
    /**
     * 注入自定义权限管理
     *
     * @return
     * @throws Exception
     */
    @Bean
    public DefaultWebSecurityExpressionHandler webSecurityExpressionHandler() {
        DefaultWebSecurityExpressionHandler handler = new DefaultWebSecurityExpressionHandler();
        handler.setPermissionEvaluator(new CustomPermissionEvaluator());
        return handler;
    }

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.userDetailsService(userService).passwordEncoder(
                new PasswordEncoder() {
                    @Override
                    public String encode(CharSequence charSequence) {
                        return charSequence.toString();
                    }

                    @Override
                    public boolean matches(CharSequence charSequence, String s) {
                        return s.equals(charSequence.toString());
                    }
                });
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.formLogin()
                .failureHandler(failureAuthenticationHandler) // 自定义登录失败处理
                .successHandler(successAuthenticationHandler) // 自定义登录成功处理
                .and()
                .logout()
                .logoutUrl("/logout")
                .and()
                .formLogin()
                .loginPage("/login")
                .loginProcessingUrl("/authentication/form") // 自定义登录路径
                .and()
                .authorizeRequests()// 对请求授权
                .antMatchers("/login", "/authentication/require",
                        "/authentication/form").permitAll()// 这些页面不需要身份认证
                .antMatchers("/docker").hasRole("DOCKER")
                .antMatchers("/java").hasRole("JAVA")
                .antMatchers("/java").hasRole("JAVA")
                .antMatchers("/custom")
                .access("@testPermissionEvaluator.check(authentication)")
                .anyRequest()//其他请求需要认证
                .authenticated().and().exceptionHandling()
                .accessDeniedHandler(accessDeniedAuthenticationHandler)
                .and()
                .csrf().disable();// 禁用跨站攻击
    }

}
```
**自定义权限表达式**

access("@testPermissionEvaluator.check(authentication)") 的意思就是 去testPermissionEvaluator这个bean里来执行check方法，这里需要注意check 方法必须返回值是boolean的因为这个是要给投票器投票的


```java
interface TestPermissionEvaluator {
    boolean check(Authentication authentication);
}

@Service("testPermissionEvaluator")
public class TestPermissionEvaluatorImpl implements TestPermissionEvaluator {

    public boolean check(Authentication authentication) {
        //这里可以拿到登陆信息然后随便的去定制自己的权限 随便你怎么查询
        //true就是过，false就是不过
        System.out.println("进入了自定义的匹配器" + authentication);
        return false;
    }
}
```
**权限异常处理器**

实现`AccessDeniedHandler` 接口

```java
@Component
@Slf4j
public class AccessDeniedAuthenticationHandler implements AccessDeniedHandler {
    private final ObjectMapper objectMapper;

    public AccessDeniedAuthenticationHandler(ObjectMapper objectMapper) {
        this.objectMapper = objectMapper;
    }


    @Override
    public void handle(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, AccessDeniedException e) throws IOException, ServletException {
        log.info("没有权限");
        httpServletResponse.setStatus(HttpStatus.FORBIDDEN.value());
        httpServletResponse.setContentType("application/json;charset=UTF-8");
        httpServletResponse.getWriter().write(objectMapper.writeValueAsString(e.getMessage()));
    }
}
```
在`configure()`方法配置

```java
.authenticated().and().exceptionHandling().accessDeniedHandler(accessDeniedAuthenticationHandler)
```
##### 方式二：注解
**使用**

Spring Security默认是禁用注解的，要想开启注解，要在继承WebSecurityConfigurerAdapter的类加@EnableMethodSecurity注解，并在该类中将AuthenticationManager定义为Bean。

```java
@Configuration
@EnableWebSecurity
@EnableGlobalMethodSecurity(prePostEnabled = true,securedEnabled=true,jsr250Enabled=true)
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {
    @Bean
    @Override
    public AuthenticationManager authenticationManagerBean() throws Exception {
        return super.authenticationManagerBean();
    }
}
```
`@EnableGlobalMethodSecurity` 分别有`prePostEnabled` 、`securedEnabled`、`jsr250Enabled` 三个字段，其中每个字段代码一种注解支持，默认为false，true为开启。

**JSR-250注解**

* @DenyAll
* @RolesAllowed
* @PermitAll

**securedEnabled注解**

* @Secured

使用@Secured在方法上指定安全性要求 角色/权限等 只有对应 角色/权限 的用户才可以调用这些方法。 如果有人试图调用一个方法，但是不拥有所需的 角色/权限，那会将会拒绝访问将引发异常。

**prePostEnabled注解**

* @PreAuthorize --适合进入方法之前验证授权
* @PostAuthorize --检查授权方法之后才被执行
* @PostFilter --在方法执行之后执行，而且这里可以调用方法的返回值，然后对返回值进行过滤或处理或修改并返回
* @PreFilter --在方法执行之前执行，而且这里可以调用方法的参数，然后对参数值进行过滤或处理或修改

#### 权限原理
* 如果认证没有成功则会默认由AnonymousAuthenticationFilter 设置一个匿名用户
* 进入FilterSecurityInterceptor 去调用 accessDecisionManager 来进行权限认证
* accessDecisionManager 用相应的策略循环 不同的 AccessDecisionVoter 进行投票
* 投票完毕后进行判断来决定是否抛出 AccessDeniedException 异常
* 抛出的异常由ExceptionTranslationFilter 处理
    * 判断是否是匿名用户是的出重新去认证
    * 不是的话调用AccessDeniedException 去处理异常

#### 动态权限
**思路**
1. 定义一个自己的SecurityMetadataSource 使其从数据库中查询权限自己构建 ConfigAttribute
2. 定义一个具有自己需求的AccessDecisionVoter
3. 规定一个自己的AccessDecisionManager

**实践**

自定义`SecurityMetadataSource`
```java
public class DynamicFilterInvocationSecurityMetadataSource implements FilterInvocationSecurityMetadataSource {

    private FilterInvocationSecurityMetadataSource superMetadataSource;

    public DynamicFilterInvocationSecurityMetadataSource(FilterInvocationSecurityMetadataSource expressionBasedFilterInvocationSecurityMetadataSource) {
        this.superMetadataSource = expressionBasedFilterInvocationSecurityMetadataSource;
        //TODO 在这里去查询数据库
    }

    private final AntPathMatcher antPathMatcher = new AntPathMatcher();

    /**
     * 假设这就是从数据库中查询到的数据
     * 意思就是 ROLE_JAVA 的角色 才能访问 /tt
     */
    private final Map<String, String> urlRoleMap = new HashMap<String, String>() {{
        put("/tt", "ROLE_JAVA");
    }};

    @Override
    public Collection<ConfigAttribute> getAttributes(Object object) throws IllegalArgumentException {
        FilterInvocation fi = (FilterInvocation) object;
        String url = fi.getRequestUrl();
        for (Map.Entry<String, String> entry : urlRoleMap.entrySet()) {
            if (antPathMatcher.match(entry.getKey(), url)) {
                return SecurityConfig.createList(entry.getValue());
            }
        }
        //如果没有匹配到就拿 咱们自定义的配置
        return superMetadataSource.getAttributes(object);
    }

    @Override
    public Collection<ConfigAttribute> getAllConfigAttributes() {
        return null;
    }


    @Override
    public boolean supports(Class<?> clazz) {
        return FilterInvocation.class.isAssignableFrom(clazz);
    }
}
```
自定义`AccessDecisionVoter`

```java
public class MyDynamicVoter implements AccessDecisionVoter<Object> {
    /**
     * supports 方法说明这个投票器是否可以传递到下一个投票器
     * 可以支持传递，则返回true
     *
     * @param attribute
     * @return
     */
    @Override
    public boolean supports(ConfigAttribute attribute) {
        return true;
    }

    @Override
    public boolean supports(Class<?> clazz) {
        return true;
    }

    @Override
    public int vote(Authentication authentication, Object object, Collection<ConfigAttribute> attributes) {
        //如果没有进行认证则永远返回 -1
        if (authentication == null) {
            return ACCESS_DENIED;
        }
        int result = ACCESS_ABSTAIN;
        Collection<? extends GrantedAuthority> authorities = extractAuthorities(authentication);
        for (ConfigAttribute attribute : attributes) {
            if (attribute.getAttribute() == null) {
                continue;
            }
            if (this.supports(attribute)) {
                result = ACCESS_DENIED;
                for (GrantedAuthority authority : authorities) {
                    if (attribute.getAttribute().equals(authority.getAuthority())) {
                        return ACCESS_GRANTED;
                    }
                }
            }
        }
        return result;
    }

    private Collection<? extends GrantedAuthority> extractAuthorities(
            Authentication authentication) {
        return authentication.getAuthorities();
    }

}
```
**加入配置项**

```java
 protected void configure(HttpSecurity http) throws Exception {
        //http.httpBasic()  //httpBasic 登录
        http.formLogin()
                .failureHandler(failureAuthenticationHandler) // 自定义登录失败处理
                .successHandler(successAuthenticationHandler) // 自定义登录成功处理
                .and()
                .logout()
                .logoutUrl("/logout")
                .and()
                .formLogin()
                .loginPage("/login")
                .loginProcessingUrl("/authentication/form") // 自定义登录路径
                .and()
                .authorizeRequests()// 对请求授权
                // 自定义FilterInvocationSecurityMetadataSource
                .withObjectPostProcessor(new ObjectPostProcessor<FilterSecurityInterceptor>() {
                    @Override
                    public <O extends FilterSecurityInterceptor> O postProcess(O fsi) {
                        fsi.setSecurityMetadataSource(mySecurityMetadataSource(fsi.getSecurityMetadataSource()));
                        fsi.setAccessDecisionManager(accessDecisionManager());
                        return fsi;
                    }
                })
                .antMatchers("/login", "/authentication/require",
                        "/authentication/form").permitAll()
                .anyRequest()
                .authenticated().and().exceptionHandling().accessDeniedHandler(accessDeniedAuthenticationHandler)
                .and()
                .csrf().disable();// 禁用跨站攻击
    }


@Bean
public DynamicFilterInvocationSecurityMetadataSource mySecurityMetadataSource(FilterInvocationSecurityMetadataSource filterInvocationSecurityMetadataSource) {
   return new DynamicFilterInvocationSecurityMetadataSource(filterInvocationSecurityMetadataSource);
}


@Bean
public AccessDecisionManager accessDecisionManager() {
   List<AccessDecisionVoter<?>> decisionVoters = new ArrayList<>();
   decisionVoters.add(new AuthenticatedVoter());
   decisionVoters.add(new WebExpressionVoter());
   decisionVoters.add(new MyDynamicVoter());
   return new AffirmativeBased(decisionVoters);
}
```
### JWT
在认证过滤器前加入JWT过滤器

引入jwt

```xml
<dependency>
   <groupId>io.jsonwebtoken</groupId>
   <artifactId>jjwt</artifactId>
   <version>0.9.1</version>
</dependency>
```
yml中配置jwt参数

```xml
jwt:
  # jwt的密钥
  secret: jwtkey
  # jwt过期时间
  expiration: 360000
  tokenHeader: Authorization #JWT存储的请求头
  tokenHead: Bearer  #JWT负载中拿到开头
```

用户实体

```java
package com.lingluoyu.model;

import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.userdetails.UserDetails;

import java.util.Collection;

/**
 * @author lius
 * @date 2020/12/17
 */
public class SysUser implements UserDetails {
    private Integer id;
    private String username;
    private String nickname;
    private String password;

    public Integer getId() {
        return id;
    }

    public void setId(Integer id) {
        this.id = id;
    }

    @Override
    public String getUsername() {
        return username;
    }

    public void setUsername(String username) {
        this.username = username;
    }

    public String getNickname() {
        return nickname;
    }

    public void setNickname(String nickname) {
        this.nickname = nickname;
    }

    public void setPassword(String password) {
        this.password = password;
    }

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return null;
    }

    @Override
    public String getPassword() {
        return password;
    }

    @Override
    public boolean isAccountNonExpired() {
        return false;
    }

    @Override
    public boolean isAccountNonLocked() {
        return false;
    }

    @Override
    public boolean isCredentialsNonExpired() {
        return false;
    }

    @Override
    public boolean isEnabled() {
        return false;
    }
}
```
`SysUserService`

```java
@Service
public class SysUserService {
    private static final Logger LOGGER = LoggerFactory.getLogger(SysUserService.class);
    @Autowired
    private SysUserMapper sysUserMapper;
    @Autowired
    private UserDetailsService userDetailsService;
    @Autowired
    private JwtTokenUtil jwtTokenUtil;

    public SysUser selectById(Integer id) {
        return sysUserMapper.selectById(id);
    }

    public SysUser selectByName(String name) {
        return sysUserMapper.selectByName(name);
    }

    public String login(String username, String password) {
        String token = null;
        try {
            UserDetails userDetails = userDetailsService.loadUserByUsername(username);
            if (!password.equals(userDetails.getPassword())) {
                throw new BadCredentialsException("密码不正确");
            }
            UsernamePasswordAuthenticationToken authentication = new UsernamePasswordAuthenticationToken(userDetails, null, userDetails.getAuthorities());
            SecurityContextHolder.getContext().setAuthentication(authentication);
            token = jwtTokenUtil.generateToken(userDetails);
        } catch (AuthenticationException e) {
            LOGGER.warn("登录异常:{}", e.getMessage());
        }
        return token;

    }
}
```
`userDetailService`
```java
@Service
@Qualifier("userDetailsService")
public class UserDetailsServiceImpl implements UserDetailsService {
    @Autowired
    private SysUserService sysUserService;

    @Autowired
    private SysRoleService sysRoleService;

    @Autowired
    private SysUserRoleService sysUserRoleService;


    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        Collection<GrantedAuthority> authorities = new ArrayList<>();
        // 从数据库中取出用户信息
        SysUser sysUser = sysUserService.selectByName(username);

        // 判断用户是否存在
        if(sysUser == null) {
            throw new UsernameNotFoundException("用户名不存在");
        }

        // 添加权限
        List<SysUserRole> sysUserRoles = sysUserRoleService.listByUserId(sysUser.getId());
        for (SysUserRole sysUserRole : sysUserRoles) {
            SysRole sysRole = sysRoleService.selectById(sysUserRole.getRoleId());
            authorities.add(new SimpleGrantedAuthority(sysRole.getRoleName()));
        }

        // 返回UserDetails实现类
        return new User(sysUser.getUsername(), sysUser.getPassword(), authorities);
    }
}
```
`JwtTokenUtil`

```java
@Component
public class JwtTokenUtil {
    private static final Logger LOGGER = LoggerFactory.getLogger(JwtTokenUtil.class);
    private static final String CLAIM_KEY_USERNAME = "sub";
    private static final String CLAIM_KEY_CREATED = "created";
    @Value("${jwt.secret}")
    private String secret;
    @Value("${jwt.expiration}")
    private Long expiration;

    /**
     * 根据负责生成JWT的token
     */
    private String generateToken(Map<String, Object> claims) {
        return Jwts.builder()
                .setClaims(claims)
                .setExpiration(generateExpirationDate())
                .signWith(SignatureAlgorithm.HS512, secret)
                .compact();
    }

    /**
     * 从token中获取JWT中的负载
     */
    private Claims getClaimsFromToken(String token) {
        Claims claims = null;
        try {
            claims = Jwts.parser()
                    .setSigningKey(secret)
                    .parseClaimsJws(token)
                    .getBody();
        } catch (Exception e) {
            LOGGER.info("JWT格式验证失败:{}", token);
        }
        return claims;
    }

    /**
     * 生成token的过期时间
     */
    private Date generateExpirationDate() {
        return new Date(System.currentTimeMillis() + expiration * 1000);
    }

    /**
     * 从token中获取登录用户名
     */
    public String getUserNameFromToken(String token) {
        String username;
        try {
            Claims claims = getClaimsFromToken(token);
            username = claims.getSubject();
        } catch (Exception e) {
            username = null;
        }
        return username;
    }

    /**
     * 验证token是否还有效
     *
     * @param token       客户端传入的token
     * @param userDetails 从数据库中查询出来的用户信息
     */
    public boolean validateToken(String token, UserDetails userDetails) {
        String username = getUserNameFromToken(token);
        return username.equals(userDetails.getUsername()) && !isTokenExpired(token);
    }

    /**
     * 判断token是否已经失效
     */
    private boolean isTokenExpired(String token) {
        Date expiredDate = getExpiredDateFromToken(token);
        return expiredDate.before(new Date());
    }

    /**
     * 从token中获取过期时间
     */
    private Date getExpiredDateFromToken(String token) {
        Claims claims = getClaimsFromToken(token);
        return claims.getExpiration();
    }

    /**
     * 根据用户信息生成token
     */
    public String generateToken(UserDetails userDetails) {
        Map<String, Object> claims = new HashMap<>();
        claims.put(CLAIM_KEY_USERNAME, userDetails.getUsername());
        claims.put(CLAIM_KEY_CREATED, new Date());
        return generateToken(claims);
    }

    /**
     * 判断token是否可以被刷新
     */
    public boolean canRefresh(String token) {
        return !isTokenExpired(token);
    }

    /**
     * 刷新token
     */
    public String refreshToken(String token) {
        Claims claims = getClaimsFromToken(token);
        claims.put(CLAIM_KEY_CREATED, new Date());
        return generateToken(claims);
    }
}
```

开启配置类

```java
@SpringBootApplication
public class SecuritySimpleJwtApplication {
    public static void main(String[] args) {
        SpringApplication.run(SecuritySimpleJwtApplication.class, args);
    }
}
```
颁发Token，放在成功处理器去做，这里为了让jwt具有刷新功能，特意准备了一个具备刷新功能的 token `refreshToken`

```java
@Slf4j
@Component
public class AppAuthenticationSuccessHandler extends SavedRequestAwareAuthenticationSuccessHandler {


    @Autowired
    private JwtTokenUtil jwtTokenUtil;

    @Override
    public void onAuthenticationSuccess(HttpServletRequest request, HttpServletResponse response, Authentication authentication) throws IOException {
        final String randomKey = jwtTokenUtil.getRandomKey();
        String username = ((UserDetails) authentication.getPrincipal()).getUsername();
        log.info("username：{}", username);
        //生产JWT 令牌
        final String token = jwtTokenUtil.generateToken(username, randomKey);
        final String refreshToken = jwtTokenUtil.generateRefreshToken(username, randomKey);
        log.info("登录成功！");
        ModelMap modelMap = GenerateModelMap.generateMap(HttpStatus.OK.value(), "登陆成功");
        modelMap.put("token", JwtAuthenticationTokenFilter.TOKEN_PREFIX + token);
        modelMap.put("refreshToken", JwtAuthenticationTokenFilter.TOKEN_PREFIX + refreshToken);
        response.setContentType("application/json;charset=UTF-8");
        response.getWriter().write(JSON.toJSONString(modelMap));
    }
}
```
自定义jwt过滤器

```java
public class JwtAuthenticationTokenFilter extends OncePerRequestFilter {
    private static final Logger LOGGER = LoggerFactory.getLogger(JwtAuthenticationTokenFilter.class);
    @Autowired
    private UserDetailsService userDetailsService;
    @Autowired
    private JwtTokenUtil jwtTokenUtil;
    @Value("${jwt.tokenHeader}")
    private String tokenHeader;
    @Value("${jwt.tokenHead}")
    private String tokenHead;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain) throws ServletException, IOException {
        String authHeader = request.getHeader(this.tokenHeader);
        if (authHeader != null && authHeader.startsWith(this.tokenHead)) {
            String authToken = authHeader.substring(this.tokenHead.length());// The part after "Bearer "
            String username = jwtTokenUtil.getUserNameFromToken(authToken);
            LOGGER.info("checking username:{}", username);
            if (username != null && SecurityContextHolder.getContext().getAuthentication() == null) {
                UserDetails userDetails = this.userDetailsService.loadUserByUsername(username);
                if (jwtTokenUtil.validateToken(authToken, userDetails)) {
                    UsernamePasswordAuthenticationToken authentication = new UsernamePasswordAuthenticationToken(userDetails, null, userDetails.getAuthorities());
                    authentication.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
                    LOGGER.info("authenticated user:{}", username);
                    SecurityContextHolder.getContext().setAuthentication(authentication);
                }
            }
        }
        chain.doFilter(request, response);
    }
}
```
自定义认证异常处理器

```java
@Component
public class RestAuthenticationEntryPoint implements AuthenticationEntryPoint {
    @Override
    public void commence(HttpServletRequest request, HttpServletResponse response, AuthenticationException authException) throws IOException, ServletException {
        response.setCharacterEncoding("UTF-8");
        response.setContentType("application/json");
        response.getWriter().println(JSON.toJSONString(CommonResult.unauthorized(authException.getMessage())));
        response.getWriter().flush();
    }
}
```

自定义授权异常处理器

```java
@Component
public class RestfulAccessDeniedHandler implements AccessDeniedHandler {
    @Override
    public void handle(HttpServletRequest request,
                       HttpServletResponse response,
                       AccessDeniedException e) throws IOException, ServletException {
        response.setCharacterEncoding("UTF-8");
        response.setContentType("application/json");
        response.getWriter().println(JSON.toJSONString(CommonResult.forbidden(e.getMessage())));
        response.getWriter().flush();
    }
}
```

加入过滤链

```java
@EnableWebSecurity
@EnableGlobalMethodSecurity(prePostEnabled = true) // 开启方法级安全验证
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    @Autowired
    private UserDetailsServiceImpl userDetailsService;
    @Autowired
    private RestfulAccessDeniedHandler restfulAccessDeniedHandler;
    @Autowired
    private RestAuthenticationEntryPoint restAuthenticationEntryPoint;

    /**
     * anyRequest          |   匹配所有请求路径
     * access              |   SpringEl表达式结果为true时可以访问
     * anonymous           |   匿名可以访问
     * denyAll             |   用户不能访问
     * fullyAuthenticated  |   用户完全认证可以访问（非remember-me下自动登录）
     * hasAnyAuthority     |   如果有参数，参数表示权限，则其中任何一个权限可以访问
     * hasAnyRole          |   如果有参数，参数表示角色，则其中任何一个角色可以访问
     * hasAuthority        |   如果有参数，参数表示权限，则其权限可以访问
     * hasIpAddress        |   如果有参数，参数表示IP地址，如果用户IP和参数匹配，则可以访问
     * hasRole             |   如果有参数，参数表示角色，则其角色可以访问
     * permitAll           |   用户可以任意访问
     * rememberMe          |   允许通过remember-me登录的用户访问
     * authenticated       |   用户登录后可访问
     */
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.csrf()// 由于使用的是JWT，我们这里不需要csrf
                .disable()
                .sessionManagement()// 基于token，所以不需要session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
                .and()
                .authorizeRequests()
                .antMatchers(HttpMethod.GET, // 允许对于网站静态资源的无授权访问
                        "/",
//                        "/*.html",
//                        "/**/*.html",
                        "/toLogin",
                        "/**/*.css",
                        "/**/*.js"
                )
                .permitAll()
                .antMatchers("/login")// 对登录要允许匿名访问
                .permitAll()
                .antMatchers(HttpMethod.OPTIONS)//跨域请求会先进行一次options请求
                .permitAll()
//                .antMatchers("/**")//测试时全部运行访问
//                .permitAll()
                .anyRequest()// 除上面外的所有请求全部需要鉴权认证
                .authenticated();
        // 禁用缓存
        http.headers().cacheControl();
        // 添加JWT filter
        http.addFilterBefore(jwtAuthenticationTokenFilter(), UsernamePasswordAuthenticationFilter.class);
        //添加自定义未授权和未登录结果返回
        http.exceptionHandling()
                .accessDeniedHandler(restfulAccessDeniedHandler)
                .authenticationEntryPoint(restAuthenticationEntryPoint);
    }

    /**
     * 身份认证接口
     */
    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.userDetailsService(userDetailsService).passwordEncoder(new PasswordEncoder() {
            @Override
            public String encode(CharSequence charSequence) {
                return charSequence.toString();
            }

            @Override
            public boolean matches(CharSequence charSequence, String s) {
                return s.equals(charSequence.toString());
            }
        });
    }

    @Bean
    public JwtAuthenticationTokenFilter jwtAuthenticationTokenFilter(){
        return new JwtAuthenticationTokenFilter();
    }
}
```


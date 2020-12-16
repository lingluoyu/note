### SpringSecurity认证流程
1. 进入 UsernamePasswordAuthenticationFilter 然后构建一个没有认证的 UsernamePasswordAuthenticationToken
2. 随后交给 AuthenticationManager 进行验证
3. AuthenticationManager 找到对应的 AuthenticationProvider进行认证
4. AuthenticationProvider找到上下文中的UserDetailsService 中寻找用户然后对比
5. 验证成功返回 Authentication 放入 SecurityContextHolder中

### 自定义登录
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

**自定义成功处理器和失败处理器**

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

SecurityConfigurerAdapter 顾名思义就是 SecurityConfigurer的适配器，我们只需要吧我们刚才写的 AuthenticationFilter AuthenticationToken AuthenticationProvider 都放进来就可以与security挂上了。

```java
@Component
public class SmsCodeAuthenticationSecurityConfig extends SecurityConfigurerAdapter<DefaultSecurityFilterChain, HttpSecurity> {
    @Autowired //我们自己定义的UserDetailsService
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
        //设置失败成功处理器
        smsCodeAuthenticationFilter.setAuthenticationSuccessHandler(customAuthenticationSuccessHandler);
        smsCodeAuthenticationFilter.setAuthenticationFailureHandler(customAuthenticationFailureHandler);
        //设置UserDetailsService
        SmsCodeAuthenticationProvider smsCodeAuthenticationProvider = new SmsCodeAuthenticationProvider();
        smsCodeAuthenticationProvider.setUserDetailsService(userService);
        //这里说明要把我们自己写的Provider放在过滤链的哪里
        http.authenticationProvider(smsCodeAuthenticationProvider)
                .addFilterAfter(smsCodeAuthenticationFilter, UsernamePasswordAuthenticationFilter.class);
    }
}
```
这样只是加入到了security的过滤链里 但是并没有生效，那么怎么配置呢？对就是还要在 WebSecurityConfigurerAdapter 里配置一下。

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
    @Autowired //注入咱们自己定义的登陆流程
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

```
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
在登陆的时候勾选一下，这时候登陆成功后，会在浏览器cookie里自动保存一条名为remember-me 的cookie如果你需要改变这个名字可以在 HttpSecurity 链上加入.rememberMeCookieName("remember") 即可。

**数据库存储**

使用 Cookie 存储虽然很方便，但是大家都知道 Cookie 毕竟是保存在客户端的，而且 Cookie 的值还与用户名、密码这些敏感数据相关，虽然加密了，但是将敏感信息存在客户端，毕竟不太安全。

Spring security 还提供了另一种相对更安全的实现机制：在客户端的 Cookie 中，仅保存一个无意义的加密串（与用户名、密码等敏感数据无关），然后在数据库中保存该加密串-用户信息的对应关系，自动登录时，用 Cookie 中的加密串，到数据库中验证，如果通过，自动登录才算通过。

需要创建一张表来存储 token 信息

```
CREATE TABLE `persistent_logins` (
  `username` varchar(64) NOT NULL,
  `series` varchar(64) NOT NULL,
  `token` varchar(64) NOT NULL,
  `last_used` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`series`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```
创建配置类

```
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
然后在 configure() 中稍微做修改

```
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
**权限配置**

#### 方式一：configure


```
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


```
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

```
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

```
.authenticated().and().exceptionHandling().accessDeniedHandler(accessDeniedAuthenticationHandler)
```
#### 注解
**使用**

Spring Security默认是禁用注解的，要想开启注解，要在继承WebSecurityConfigurerAdapter的类加@EnableMethodSecurity注解，并在该类中将AuthenticationManager定义为Bean。

```
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
* 进入FilterSecurityInterceptor 去调用 accessDecisionManager 来进行一个权限认证
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
```
public class DynamicFilterInvocationSecurityMetadataSource implements FilterInvocationSecurityMetadataSource {


    private FilterInvocationSecurityMetadataSource superMetadataSource;


    public DynamicFilterInvocationSecurityMetadataSource(FilterInvocationSecurityMetadataSource expressionBasedFilterInvocationSecurityMetadataSource) {
        this.superMetadataSource = expressionBasedFilterInvocationSecurityMetadataSource;
        //TODO 在这里去查询你的数据库
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

```
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

```
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

```
<dependency>
   <groupId>io.jsonwebtoken</groupId>
   <artifactId>jjwt</artifactId>
   <version>0.9.0</version>
</dependency>
```
用户实体

```
@Data
public class User implements UserDetails {

    private Long id;
    private String userName;
    private String password;
    private List<String> roles;


    public User(Long id, String userName, String password, List<String> roles) {
        this.id = id;
        this.userName = userName;
        this.password = password;
        this.roles = roles;
    }

    @Override
    public boolean isAccountNonExpired() {
        return true;
    }

    @Override
    public boolean isAccountNonLocked() {
        return true;
    }

    @Override
    public boolean isCredentialsNonExpired() {
        return true;
    }

    @Override
    public boolean isEnabled() {
        return true;
    }

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        List<GrantedAuthority> authorities = new ArrayList<>();
        for (String role : roles) {
            authorities.add(new SimpleGrantedAuthority(role));
        }
        return authorities;
    }

    @Override
    public String getUsername() {
        return userName;
    }

    @Override
    public String getPassword() {
        return password;
    }

}
```
`IUserService`接口
```
public interface IUserService {
    User findByUsername(String userName);
}
```
`UserServiceImpl`实现类
```
@Service
public class UserServiceImpl implements IUserService {

    private static final Set<User> users = new HashSet<>();
	// 密码123    md5加密 
    static {
        users.add(new User(1L, "fulin", "1dc568b64c0f67e7a86c89a12fa5bd5f", Arrays.asList("admin", "docker")));
        users.add(new User(1L, "xiaohan", "1dc568b64c0f67e7a86c89a12fa5bd5f", Arrays.asList("admin", "docker")));
        users.add(new User(1L, "longlong", "1dc568b64c0f67e7a86c89a12fa5bd5f", Arrays.asList("admin", "docker")));
    }

    @Override
    public User findByUsername(String userName) {
        return users.stream().filter(o -> StringUtils.equals(o.getUsername(), userName)).findFirst().get();
    }
}

```
`userDetailService`
```
@Service
public class UserService implements UserDetailsService {

    final IUserService iUserService;

    public UserService(IUserService iUserService) {
        this.iUserService = iUserService;
    }

    @Override
    public UserDetails loadUserByUsername(String s) throws UsernameNotFoundException {
        User user = iUserService.findByUsername(s);
        if (user == null) {
            throw new UsernameNotFoundException("用户不存在");
        }
        return user;
    }

}
```
**jwt**

jwt配置
```
@Data
@ConfigurationProperties(prefix = "security.jwt")
public class SecurityProperties {
    private JwtProperties jwt = new JwtProperties();
}
```
```
/**
 * Jwt的基本配置
 */
@Data
public class JwtProperties {
    /**
     * 默认前面秘钥
     */
    private String secret = "defaultSecret";

    /**
     * token默认有效期时长，1小时
     */
    private Long expiration = 3600L;
    /**
     * token默认有效期时长，1个半小时
     */
    private Long refreshExpiration = 5400L;

    /**
     * token的唯一标记
     */
    private String md5Key = "randomKey";

    /**
     * GET请求是否需要进行Authentication请求头校验，true：默认校验；false：不拦截GET请求
     */
    private boolean preventsGetMethod = true;
}
```
开启配置类
```
@SpringBootApplication
@EnableConfigurationProperties(SecurityProperties.class)
public class SecuritySimpleJwtApplication {
    public static void main(String[] args) {
        SpringApplication.run(SecuritySimpleJwtApplication.class, args);
    }
}
```
`JwtTokenUtil`
```
@Component
public class JwtTokenUtil {

    private final SecurityProperties securityProperties;

    public JwtTokenUtil(SecurityProperties securityProperties) {
        this.securityProperties = securityProperties;
    }

    /**
     * 获取用户名从token中
     */
    public String getUsernameFromToken(String token) {
        return getClaimFromToken(token).getSubject();
    }

    /**
     * 获取jwt失效时间
     */
    public Date getExpirationDateFromToken(String token) {
        return getClaimFromToken(token).getExpiration();
    }

    /**
     * 获取私有的jwt claim
     */
    public String getPrivateClaimFromToken(String token, String key) {
        return getClaimFromToken(token).get(key).toString();
    }

    /**
     * 获取md5 key从token中
     */
    public String getMd5KeyFromToken(String token) {
        return getPrivateClaimFromToken(token, securityProperties.getJwt().getMd5Key());
    }

    /**
     * 获取jwt的payload部分
     */
    public Claims getClaimFromToken(String token) {
        return Jwts.parser()
                .setSigningKey(generalKey())
                .parseClaimsJws(token)
                .getBody();
    }

    /**
     * <pre>
     *  验证token是否失效
     *  true:过期   false:没过期
     * </pre>
     */
    public Boolean isTokenExpired(String token) {
        try {
            final Date expiration = getExpirationDateFromToken(token);
            return expiration.before(new Date());
        } catch (ExpiredJwtException expiredJwtException) {
            return true;
        }
    }

    /**
     * 生成token(通过用户名和签名时候用的随机数)
     */
    public String generateToken(String userName, String randomKey) {
        Map<String, Object> claims = new HashMap<>();
        claims.put(securityProperties.getJwt().getMd5Key(), randomKey);
        return doGenerateToken(claims, userName);
    }

    /**
     * 生成token(通过用户名和签名时候用的随机数)
     */
    public String generateRefreshToken(String userName, String randomKey) {
        Map<String, Object> claims = new HashMap<>();
        claims.put(securityProperties.getJwt().getMd5Key(), randomKey);
        return doGenerateRefreshToken(claims, userName);
    }

    /**
     * 由字符串生成加密key
     *
     * @return
     */
    public SecretKey generalKey() {
        byte[] encodedKey = Base64.decodeBase64(securityProperties.getJwt().getSecret());
        return new SecretKeySpec(encodedKey, 0, encodedKey.length, "AES");
    }

    /**
     * 生成token
     */
    private String doGenerateToken(Map<String, Object> claims, String subject) {
        final Date createdDate = new Date();
        final Date expirationDate = new Date(createdDate.getTime() + securityProperties.getJwt().getExpiration() * 1000);

        return Jwts.builder()
                .setClaims(claims)
                .setSubject(subject)
                .setIssuedAt(createdDate)
                .setExpiration(expirationDate)
                .signWith(SignatureAlgorithm.HS512, generalKey())
                .compact();
    }

    /**
     * 生成token
     */
    private String doGenerateRefreshToken(Map<String, Object> claims, String subject) {
        final Date createdDate = new Date();
        final Date expirationDate = new Date(createdDate.getTime() + securityProperties.getJwt().getRefreshExpiration() * 1000);
        return Jwts.builder()
                .setClaims(claims)
                .setSubject(subject)
                .setIssuedAt(createdDate)
                .setExpiration(expirationDate)
                .signWith(SignatureAlgorithm.HS512, generalKey())
                .compact();
    }

    /**
     * 获取混淆MD5签名用的随机字符串
     */
    public String getRandomKey() {
        return getRandomString(6);
    }

    /**
     * 获取随机位数的字符串
     */
    public String getRandomString(int length) {
        final String base = "abcdefghijklmnopqrstuvwxyz0123456789";
        Random random = new Random();
        StringBuffer sb = new StringBuffer();
        for (int i = 0; i < length; i++) {
            int number = random.nextInt(base.length());
            sb.append(base.charAt(number));
        }
        return sb.toString();
    }

    /**
     * 刷新token
     *
     * @param token：token
     * @return
     */
    public String refreshToken(String token, String randomKey) {
        String refreshedToken;
        try {
            final Claims claims = getClaimFromToken(token);
            refreshedToken = generateToken(claims.getSubject(), randomKey);
        } catch (Exception e1) {
            refreshedToken = null;
        }
        return refreshedToken;
    }

}
```
**颁发Token**

都准备齐全了，颁发token呢我们就放在成功处理器去做，这里为了让jwt具有刷新功能，特意准备了一个具备刷新功能的 token `refreshToken`

```
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
**jwt过滤器**

自己定义一个jwt过滤器，这是我自己实现的代码如下，这里访问/refreshToken 重新颁发一个token和 `refreshToken`给前台

```
/**
 * JWT过滤器
 * <p>
 * OncePerRequestFilter，顾名思义，
 * 它能够确保在一次请求中只通过一次filter，而需要重复的执行。
 */
@Component
@Slf4j
public class JwtAuthenticationTokenFilter extends OncePerRequestFilter {
    private static final String HEADER_NAME = "Authorization";
    public static final String TOKEN_PREFIX = "Bearer ";

    @Autowired
    private JwtTokenUtil jwtTokenUtil;

    @Autowired
    private SecurityProperties securityProperties;

    @Autowired
    private UserService userService;


    private AntPathMatcher antPathMatcher = new AntPathMatcher();


    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
        log.info("请求路径：{}，请求方式为：{}", request.getRequestURI(), request.getMethod());
        if (antPathMatcher.match("/favicon.ico", request.getRequestURI())) {
            log.info("jwt不拦截此路径：{}，请求方式为：{}", request.getRequestURI(), request.getMethod());
            filterChain.doFilter(request, response);
            return;
        }
        /*
         * get请求是否需要进行Authentication请求头校验，true：默认校验；false：不拦截GET请求
         * 因为get请求比较特殊
         */
        if (!securityProperties.getJwt().isPreventsGetMethod()) {
            if (Objects.equals(RequestMethod.GET.toString(), request.getMethod())) {
                log.info("jwt不拦截此路径因为开启了不拦截GET请求：{}，请求方式为：{}", request.getRequestURI(), request.getMethod());
                filterChain.doFilter(request, response);
                return;
            }
        }
        /*
         * 排除路径，并且如果是options请求是cors跨域预请求，设置allow对应头信息
         * permitUrls可以自定义不需要验证的url
         */
        String[] permitUrls = {"/authentication"};
        for (String permitUrl : permitUrls) {
            if (antPathMatcher.match(permitUrl, request.getRequestURI())
                    || Objects.equals(RequestMethod.OPTIONS.toString(), request.getMethod())) {
                log.info("jwt不拦截此路径：{}，请求方式为：{}", request.getRequestURI(), request.getMethod());
                filterChain.doFilter(request, response);
                return;
            }
        }
        // 获取请求头Authorization
        String authHeader = request.getHeader(HEADER_NAME);
        if (StringUtils.isBlank(authHeader) || !authHeader.startsWith(TOKEN_PREFIX)) {
            log.error("Authorization的开头不是Bearer，Authorization===>{}", authHeader);
            responseEntity(response, HttpStatus.UNAUTHORIZED.value(), "暂无权限！");
            return;
        }
        // 截取token
        String authToken = authHeader.substring(TOKEN_PREFIX.length());
        //判断token是否失效
        if (jwtTokenUtil.isTokenExpired(authToken)) {
            log.info("token已过期！");
            responseEntity(response, HttpStatus.UNAUTHORIZED.value(), "token已过期！");
            return;
        }
        String randomKey = jwtTokenUtil.getMd5KeyFromToken(authToken);
        String username = jwtTokenUtil.getUsernameFromToken(authToken);
        //如果访问的是刷新Token的请求
        if (antPathMatcher.match("/refreshToken", request.getRequestURI()) && Objects.equals(RequestMethod.POST.toString(), request.getMethod())) {
            final String getRandomKey = jwtTokenUtil.getRandomKey();
            refreshEntity(response, HttpStatus.OK.value(), jwtTokenUtil.generateToken(username, getRandomKey), jwtTokenUtil.refreshToken(authToken, jwtTokenUtil.getRandomKey()));
            return;
        }
        /*
         * 验证token是否合法
         */
        if (StringUtils.isBlank(username) || StringUtils.isBlank(randomKey)) {
            log.info("username{}或randomKey{} 可能为null！", username, randomKey);
            responseEntity(response, HttpStatus.UNAUTHORIZED.value(), "暂无权限！");
            return;
        }
        //获得用户名信息放入上下文中
        if (SecurityContextHolder.getContext().getAuthentication() == null) {
            UserDetails userDetails = this.userService.loadUserByUsername(username);
            UsernamePasswordAuthenticationToken authentication = new UsernamePasswordAuthenticationToken(
                    userDetails, null, userDetails.getAuthorities());
            authentication.setDetails(new WebAuthenticationDetailsSource().buildDetails(
                    request));
            SecurityContextHolder.getContext().setAuthentication(authentication);
        }
        // token过期时间
        long tokenExpireTime = jwtTokenUtil.getExpirationDateFromToken(authToken).getTime();

        // token还剩余多少时间过期
        long surplusExpireTime = (tokenExpireTime - System.currentTimeMillis()) / 1000;
        log.info("Token剩余时间:" + surplusExpireTime);

        filterChain.doFilter(request, response);

    }

    private void responseEntity(HttpServletResponse response, Integer status, String message) {
        response.setContentType("application/json;charset=UTF-8");
        response.setStatus(status);
        ModelMap modelMap = GenerateModelMap.generateMap(status, message);
        try {
            response.getWriter().write(JSON.toJSONString(modelMap));
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    private void refreshEntity(HttpServletResponse response, Integer status, String token, String refreshToken) {
        response.setContentType("application/json;charset=UTF-8");
        response.setStatus(status);
        ModelMap modelMap = new ModelMap();
        modelMap.put("token", JwtAuthenticationTokenFilter.TOKEN_PREFIX + token);
        modelMap.put("refreshToken", JwtAuthenticationTokenFilter.TOKEN_PREFIX + refreshToken);
        try {
            response.getWriter().write(JSON.toJSONString(modelMap));
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```
**加入过滤链**

```
@Configuration
public class ValidateSecurityCoreConfig extends WebSecurityConfigurerAdapter {

    @Autowired
    private AuthenticationFailureHandler authenticationFailureHandler;

    @Autowired
    private AuthenticationSuccessHandler authenticationSuccessHandler;

    @Autowired
    private UserService userService;

    @Autowired
    private JwtAuthenticationTokenFilter jwtAuthenticationTokenFilter;

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.userDetailsService(userService).passwordEncoder(
                new PasswordEncoder() {

                    @Override
                    public String encode(CharSequence rawPassword) {
                        return MD5Util.encode((String) rawPassword);
                    }

                    @Override
                    public boolean matches(CharSequence rawPassword, String encodedPassword) {
                        return encodedPassword.equals(MD5Util.encode((String) rawPassword));
                    }
                });
    }


    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.formLogin()
                .loginProcessingUrl("/authentication")
                .successHandler(authenticationSuccessHandler)
                .failureHandler(authenticationFailureHandler)
                .and()
                .csrf().disable();
        http
                .sessionManagement()
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS).and()
                .addFilterBefore(jwtAuthenticationTokenFilter, UsernamePasswordAuthenticationFilter.class);
        http.authorizeRequests().antMatchers("/authentication").permitAll();
    }
}
```
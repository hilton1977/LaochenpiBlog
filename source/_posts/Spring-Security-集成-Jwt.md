---
title: Spring Security 集成 Jwt
tags:
  - Spring
categories:
  - Java
toc: false
date: 2020-12-21 15:57:28
---

### maven集成
Spring Security 权限安全框架


``` xml
<!--安全-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>

<!--jwt-->
<dependency> <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt</artifactId>
    <version>0.9.1</version>
</dependency>
```

### 安全配置
继承`WebSecurityConfigurerAdapter`编写相关配置、`@EnableWebSecurity`开启web安全，`@EnableGlobalMethodSecurity`开启全局方法安全设置，`prePostEnabled `开启方法请求前后校验例如`@PreAuthorize("hasAuthority('KHZX.MXGL.CK')")` 请求用户是否拥有权限


``` java
/**
 * 权限配置中心
 *
 * @author Major.Fang
 */
@Slf4j
@Configuration
@EnableWebSecurity
@EnableGlobalMethodSecurity(prePostEnabled = true)
public class SecurityConfigurer extends WebSecurityConfigurerAdapter implements ApplicationContextAware {

    @Autowired
    private RedisUtil redisUtil;


    /**
     * 路径过滤白名单无需携带token
     *
     * @param web
     * @throws Exception
     */
    @Override
    public void configure(WebSecurity web)  {
        web.ignoring()
                .antMatchers("/swagger-ui.html")
                .antMatchers("/v2/**")
                .antMatchers("/swagger-resources/**")
                .antMatchers("/index/*")
                .antMatchers("/common/upload")
                .antMatchers(whiteList());
    }


    /**
     * 白名单处理
     * @return
     */
    private String[] whiteList(){
        RequestMappingHandlerMapping requestMappingHandlerMapping = super.getApplicationContext().getBean(RequestMappingHandlerMapping.class);
        Map<RequestMappingInfo, HandlerMethod> map = requestMappingHandlerMapping.getHandlerMethods();
        Set<String> whiteList = new HashSet<>();
        for(Map.Entry<RequestMappingInfo,HandlerMethod> entry:map.entrySet()){
            HandlerMethod handlerMethod = entry.getValue();
            RequestMappingInfo requestMappingInfo = entry.getKey();
            if(handlerMethod.hasMethodAnnotation(WhiteList.class)){
                whiteList.addAll(requestMappingInfo.getPatternsCondition().getPatterns());
            }
        }
        return whiteList.toArray(new String[whiteList.size()]);
    }

    /**
     * 请求认证配置
     *
     * @param httpSecurity
     * @throws Exception
     */
    @Override
    protected void configure(HttpSecurity httpSecurity) throws Exception {
        // 除去特殊路径意外所有请求需要认证
        httpSecurity.authorizeRequests()
                .antMatchers("/index/*").permitAll()
                .antMatchers("/common/upload").permitAll()
                .anyRequest().authenticated();
        // 无需开启csrf 非法跨域请求拦截
        httpSecurity.csrf().disable();
        // 禁用 Security 的 Session策略
        httpSecurity.sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS);
        // 访问异常处理 认证用户访问无权限资源时的异常
        httpSecurity.exceptionHandling().accessDeniedHandler(new MyAccessDeniedHandler());
        // 添加 JWT 过滤器
        httpSecurity.addFilterAfter(new JwtAuthenticationTokenFilter(redisUtil), UsernamePasswordAuthenticationFilter.class);
        // 禁用缓存
        httpSecurity.headers().cacheControl();

    }

    /**
     * 认证用户没有权限访问资源
     */
    private class MyAccessDeniedHandler implements AccessDeniedHandler {

        @Override
        public void handle(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, AccessDeniedException e) throws IOException {
            HttpResponseUtil.writeError(HttpStatus.HS_FORBIDDEN, httpServletResponse);
        }
    }
```

#### Jwt认证过滤器
``` java 
/**
 * JWT Token 认证过滤器
 * 1.请求头Headers 是否包含 token
 * 2.token进行校验是否合法、是否过期、签名是否校验成功
 * 3.通过token获取用户userId获取缓存中的用户信息
 * 4.上述都校验成功构建用户资源权限校验实体交予安全上下文进行权限处理
 *
 * @author Major.Fang
 */
public class JwtAuthenticationTokenFilter extends OncePerRequestFilter {

    private RedisUtil redisUtil;

    public JwtAuthenticationTokenFilter(RedisUtil redisUtil) {
        this.redisUtil = redisUtil;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, FilterChain filterChain) throws ServletException, IOException {
        String token = httpServletRequest.getHeader("Authorization");
        if (StrUtil.isEmpty(token)) {
            HttpResponseUtil.writeError(HttpStatus.HS_ILLEGAL_TOKEN, httpServletResponse);
            return;
        }
        HttpStatus errorStatus = JwtUtil.verifyToken(token);
        if (errorStatus != null) {
            HttpResponseUtil.writeError(errorStatus, httpServletResponse);
            return;
        }
        SecurityUserDetails securityUserDetails = redisUtil.getValue(RedisKeyConstant.SYS_USER_TOKEN + token, SecurityUserDetails.class);
        if (securityUserDetails == null) {
            HttpResponseUtil.writeError(HttpStatus.HS_ILLEGAL_TOKEN,httpServletResponse);
            return;
        }
        // 构建验证所需对象参数交予安全上下文进行权限校验
        UsernamePasswordAuthenticationToken authentication = new UsernamePasswordAuthenticationToken(
                securityUserDetails, null, securityUserDetails.getAuthorities());
        authentication.setDetails(new WebAuthenticationDetailsSource().buildDetails(
                httpServletRequest));
        SecurityContextHolder.getContext().setAuthentication(authentication);
        filterChain.doFilter(httpServletRequest, httpServletResponse);
    }
}
```
#### JwtUtil 工具类
``` java
/**
 * Jwt 工具
 *
 * @author Major.Fang
 */
public class JwtUtil {
    /**
     * token的密码
     **/
    private static String tokenPassword = "ZK-PRODUCT";

    /**
     * token的有效时长，小时为单位
     **/
    private static Long liveHours = 24L;

    /**
     * 根据用户id生成token
     *
     * @param userId
     * @return
     */
    public static String genTokenByUserId(String userId) {
        return Jwts.builder()
                .setClaims(null)
                .setId(userId)
                .setIssuedAt(new Date())
                .setExpiration(new Date(System.currentTimeMillis() + liveHours * 60 * 60 * 1000))
                .signWith(SignatureAlgorithm.HS256, tokenPassword)
                .compact();
    }

    /**
     * 校验Token
     *
     * @param token
     * @return  错误状态码
     */
    public static HttpStatus verifyToken(String token) {
        try {
            getClaims(token);
        } catch (ExpiredJwtException e) {
            return HttpStatus.HS_TOKEN_EXPIRED;
        } catch (SignatureException e) {
            return HttpStatus.HS_TOKEN_INSUFFICIENT;
        } catch (MalformedJwtException e){
            return HttpStatus.HS_ILLEGAL_TOKEN;
        }
        return null;
    }

    /**
     * 根据Token获取userId
     *
     * @param token
     * @return
     */
    public static String getUserIdByToken(String token) {
        Claims claims = getClaims(token);
        return Optional.of(claims).map(Claims::getId).orElse(null);
    }

    private static Claims getClaims(String token) {
        return Jwts.parser().setSigningKey(tokenPassword).parseClaimsJws(token).getBody();
    }
}
```
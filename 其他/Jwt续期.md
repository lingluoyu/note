**JwtFilter**

```java
import org.apache.commons.lang3.StringUtils;
import org.springframework.boot.web.servlet.FilterRegistrationBean;
import org.springframework.context.annotation.Bean;
import org.springframework.stereotype.Component;

import javax.servlet.Filter;
import javax.servlet.FilterChain;
import javax.servlet.FilterConfig;
import javax.servlet.ServletException;
import javax.servlet.ServletRequest;
import javax.servlet.ServletResponse;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;

/**
 * @author lius
 * @date 2021/3/17
 */
@Component
public class JwtFilter implements Filter {

    private String[] allowUrls;

    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
        String allowUrl = filterConfig.getInitParameter("allowUrls");
        if (StringUtils.isNotBlank(allowUrl)) {
            allowUrls = allowUrl.split(";");
        }
    }

    @Bean
    public FilterRegistrationBean testFilterRegistration() {

        FilterRegistrationBean registration = new FilterRegistrationBean();
        registration.setFilter(new JwtFilter());
        registration.addUrlPatterns("/*");
        registration.addInitParameter("allowUrls", "/login.htm;/tologin.htm");
        registration.setName("jwtFilter");
        registration.setOrder(1);
        return registration;
    }

    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) throws IOException, ServletException {
        HttpServletRequest request = (HttpServletRequest) req;
        HttpServletResponse response = (HttpServletResponse) res;

        String username = (String) request.getAttribute("username");

        Object cacheUser = LocalCache.getCache(username);

        if (!allowUrl(request.getRequestURI())) {
            if (null == cacheUser || JwtUtil.isExpiration((String) cacheUser) || JwtUtil.getUsername((String) cacheUser) != username) {
                response.sendRedirect(request.getContextPath() + "/auth.htm");
            } else {
                LocalCache.setCache(username, JwtUtil.getToken(username), 30 * 60);
            }
        }

        chain.doFilter(request, response);
    }

    private Boolean allowUrl(String allowUrl) {
        for (String url :
                allowUrls) {
            if (allowUrl.contains(url)) {
                return true;
            }
        }
        return false;
    }

    @Override
    public void destroy() {

    }
}
```

**JwtUtil**

```java
import io.jsonwebtoken.Claims;
import io.jsonwebtoken.ExpiredJwtException;
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.SignatureAlgorithm;

import java.util.Date;

/**
 * @author lius
 * @date 2021/3/17
 */
public class JwtUtil {

    public static final long EXPIRE = 30 * 60 * 1000;  //过期时间，毫秒

    //秘钥
    public static final String APPSECRET = "app";

    public static String getToken(String username) {
        if (username == null) {
            return null;
        }

        String token = Jwts.builder().setSubject(username)
                .setIssuedAt(new Date())
                .setExpiration(new Date(System.currentTimeMillis() + EXPIRE))
                .signWith(SignatureAlgorithm.HS256, APPSECRET).compact();
        return token;
    }

    // 从token中获取用户名
    public static String getUsername(String token) {
        try {
            return getTokenBody(token).getSubject();
        } catch (ExpiredJwtException e) {
            return e.getClaims().getSubject();
        }

    }

    // 是否已过期
    public static boolean isExpiration(String token) {
        try {
            return getTokenBody(token).getExpiration().before(new Date());
        } catch (ExpiredJwtException e) {
            return e.getClaims().getExpiration().before(new Date());
        }
    }

    private static Claims getTokenBody(String token) {
        return Jwts.parser()
                .setSigningKey(APPSECRET)
                .parseClaimsJws(token)
                .getBody();
    }
}
```

**超时cache**

```java
import java.util.ArrayList;
import java.util.Collection;
import java.util.Date;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.TimerTask;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;

/**
 * 带有超时时间的缓存
 * @author lius
 * @date 2021/3/17
 */
public class LocalCache {
    // private static final long SECOND_TIME = 1000;//默认过期时间 20秒
    // private static final int DEFUALT_VALIDITY_TIME = 20;//默认过期时间 20秒
//    private static final Timer timer;

    private static final ScheduledExecutorService pool;

    private static final ConcurrentHashMap<String, Object> map;
    static Object mutex = new Object();

    static {
//        timer = new Timer();
        pool = Executors.newScheduledThreadPool(1);
        map = new ConcurrentHashMap<String, Object>();
    }

    /**
     * 增加缓存对象
     *
     * @param key
     * @param ce
     * @param validityTime 有效时间
     */
    public static void setCache(String key, Object ce, int validityTime) {
        map.put(key, new CacheWrapper(validityTime, ce));
        pool.schedule(new TimeoutTimerTask(key), validityTime, TimeUnit.SECONDS);
//        timer.schedule(new TimeoutTimerTask(key), validityTime * 1000);
    }

    // 获取缓存KEY列表
    public static Set<String> getCacheKeys() {
        return map.keySet();
    }

    public static List<String> getKeysFuzz(String patton) {
        List<String> list = new ArrayList<String>();
        for (String tmpKey : map.keySet()) {
            if (tmpKey.contains(patton)) {
                list.add(tmpKey);
            }
        }
        if (isNullOrEmpty(list)) {
            return null;
        }
        return list;
    }

    public static Integer getKeySizeFuzz(String patton) {
        Integer num = 0;
        for (String tmpKey : map.keySet()) {
            if (tmpKey.startsWith(patton)) {
                num++;
            }
        }
        return num;
    }

    /**
     * 增加缓存对象
     *
     * @param key
     * @param ce
     */
    public static void setCache(String key, Object ce) {
        map.put(key, new CacheWrapper(ce));
    }

    /**
     * 获取缓存对象
     *
     * @param key
     * @return
     */
    public static <T> T getCache(String key) {
        CacheWrapper wrapper = (CacheWrapper) map.get(key);
        if (wrapper == null) {
            return null;
        }
        return (T) wrapper.getValue();
    }

    /**
     * 检查是否含有制定key的缓冲
     *
     * @param key
     * @return
     */
    public static boolean contains(String key) {
        return map.containsKey(key);
    }

    /**
     * 删除缓存
     *
     * @param key
     */
    public static void delCache(String key) {
        map.remove(key);
    }

    /**
     * 删除缓存
     *
     * @param key
     */
    public static void delCacheFuzz(String key) {
        for (String tmpKey : map.keySet()) {
            if (tmpKey.contains(key)) {
                map.remove(tmpKey);
            }
        }
    }

    /**
     * 获取缓存大小
     */
    public static int getCacheSize() {
        return map.size();
    }

    /**
     * 清除全部缓存
     */
    public static void clearCache() {
        map.clear();
    }

    /**
     * @projName：lottery
     * @className：TimeoutTimerTask
     * @description：清除超时缓存定时服务类
     */
    static class TimeoutTimerTask extends TimerTask {
        private String ceKey;

        public TimeoutTimerTask(String key) {
            this.ceKey = key;
        }

        @Override
        public void run() {
            CacheWrapper cacheWrapper = (CacheWrapper) map.get(ceKey);
            if (cacheWrapper == null || cacheWrapper.getDate() == null) {
                return;
            }
            if (System.currentTimeMillis() < cacheWrapper.getDate().getTime()) {
                return;
            }
            LocalCache.delCache(ceKey);
        }
    }

    public static boolean isNullOrEmpty(Object obj) {
        try {
            if (obj == null) {
                return true;
            }
            if (obj instanceof CharSequence) {
                return ((CharSequence) obj).length() == 0;
            }
            if (obj instanceof Collection) {
                return ((Collection<?>) obj).isEmpty();
            }
            if (obj instanceof Map) {
                return ((Map<?, ?>) obj).isEmpty();
            }
            if (obj instanceof Object[]) {
                Object[] object = (Object[]) obj;
                if (object.length == 0) {
                    return true;
                }
                boolean empty = true;
                for (int i = 0; i < object.length; i++) {
                    if (!isNullOrEmpty(object[i])) {
                        empty = false;
                        break;
                    }
                }
                return empty;
            }
            return false;
        } catch (Exception e) {
            return true;
        }

    }

    private static class CacheWrapper {
        private Date date;
        private Object value;

        public CacheWrapper(int time, Object value) {
            this.date = new Date(System.currentTimeMillis() + time * 1000);
            this.value = value;
        }

        public CacheWrapper(Object value) {
            this.value = value;
        }

        public Date getDate() {
            return date;
        }

        public Object getValue() {
            return value;
        }
    }
}
```

**使用**

```java
import org.springframework.web.bind.annotation.RequestMapping;

/**
 * @author lius
 * @date 2021/3/17
 */
@RequestMapping("/login")
public class LoginController {
    @RequestMapping("/login.htm")
    public String login(String username, String password) {
        //验证username和password

        //生成jwt
        String token = JwtUtil.getToken(username);
        LocalCache.setCache(username, token, 30 * 60);


        return "success";
    }


    @RequestMapping("/test.htm")
    public String test(String username, String code) {
        System.out.println("已登录！");
        return "success";
    }
}
```


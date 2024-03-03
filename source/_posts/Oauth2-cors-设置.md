---
title: Oauth2 cors 设置
catalog: true
subtitle: spring oauth2 cors 设置简单记录
header-img: 'http://pvpmnqrh8.bkt.clouddn.com/tag-bg.png'
tags:
  - spring security
  - spring oauth2
categories:
  - web
abbrlink: e8cdb6bd
date: 2019-08-04 16:32:04
---
遇到的问题:
>系统中同时使用了 spring security 与 spring oauth2 ，需要跨域支持。
但是Spring security 的跨域设置未能覆盖到 oauth2 的token端点。

## 分析过程与结果如下：
根据spring 官方文档 security 的跨域设置方式：
  ```
  http.cors()
```
```
    @Bean
    public CorsConfigurationSource corsConfigurationSource () {
        final UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        final CorsConfiguration config = new CorsConfiguration();
        config.setAllowCredentials(false); // 禁止cookies跨域
        config.addAllowedOrigin("*");// 允许向该服务器提交请求的URI，*表示全部允许。。这里尽量限制来源域，比如http://xxxx:8080 ,以降低安全风险。。
        config.addAllowedHeader("*");// 允许访问的头信息,*表示全部
        config.setMaxAge(18000L);// 预检请求的缓存时间（秒），即在这个时间段里，对于相同的跨域请求不会再预检了
        config.addAllowedMethod("*");// 允许提交请求的方法，*表示全部允许，也可以单独设置GET、PUT等
        source.registerCorsConfiguration("/**", config);
        return source;
    }
```
未生效,分析如下:
由于 AuthorizationServer 的token 端点受httpBasic 认证限制。
因此当跨域请求时。options 请求是不携带 认证信息的。
导致跨域请求失败。
如果需要设置 token端点 支持跨域。需要设置  处理跨域请求的filter order级别高于 0（小于0）。

>注解@EnableAuthorizationServer 中关于security 鉴权的配置 由AuthorizationServerSecurityConfiguration 负责,代码如下：
```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import({AuthorizationServerEndpointsConfiguration.class, AuthorizationServerSecurityConfiguration.class})
public @interface EnableAuthorizationServer
```
>AuthorizationServerSecurityConfiguration 配置类的代码如下：
```
@Configuration
@Order(0)
@Import({ClientDetailsServiceConfiguration.class, AuthorizationServerEndpointsConfiguration.class})
public class AuthorizationServerSecurityConfiguration extends WebSecurityConfigurerAdapter 
```
- 具体 httpSecurity 配置可查看当前类-方法：
protected void configure(HttpSecurity http) throws Exception
该类中定义了相关的 token端点安全配置。如果配置了spring security 文档中提供的跨域处理没有生效，就是因为该 filterChain中没有cors 的相关处理。

因此如果需要 为oauth token端点添加 cors 配置。则需要设置cors filter的优先级别 高于 order(0).

## 添加跨域方案: 
1.添加独立的cors filter，同时设置 优先级别高于 order(0)。
- 可参考其他方案的 cors 添加方式，主要处理优先级

2.定义自己的 @EnableAuthorizationServer 注解，重写其中 AuthorizationServerSecurityConfiguration 关于httpSecurity 相关配置，添加跨域处理。
例子：
```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import({ AuthorizationServerEndpointsConfiguration.class, PersonAuthorizationServerSecurityConfiguration.class })
public @interface EnablePersonAuthorizationServer {

}
```

```
@Configuration
@Order(0)
@Import({ ClientDetailsServiceConfiguration.class, AuthorizationServerEndpointsConfiguration.class })
public class PersonAuthorizationServerSecurityConfiguration extends AuthorizationServerSecurityConfiguration {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        super.configure(http);
        http.cors();
    }
}
```

```
    @Bean
    CorsConfigurationSource corsConfigurationSource(OAuth2PlatformCorsConfig oauth2PlatformCorsConfig) {
        CorsConfiguration configuration = new CorsConfiguration();
        configuration.setAllowedOrigins("*");
        configuration.setAllowedMethods("*");
        configuration.setAllowedHeaders("*");
        configuration.setMaxAge(36000);
        configuration.setAllowCredentials(true);
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", configuration);
        return source;
    }
    ```

总结：
核心思想解决方式：要求处理跨域的filter优先级别高于 spring oauth2自身的 httpBasic鉴权filter优先级。将跨域的options 请求，拦截处理。（因为spirng oauth2的token端点受httpBasic认证校验）。只有这样cors filter 才能正确生效。
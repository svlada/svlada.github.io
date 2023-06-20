---
title: Spring Session JDBC Configuration
description: This article will show you how to configure and use Spring Session to manage session data in your web application.
date: 2018-05-01
tags:
  - java
  - spring-boot
  - spring-security
layout: layouts/post.njk
permalink: "spring-session-tutorial/index.html"
---

This article shows how to configure [Spring Session](https://projects.spring.io/spring-session/) to use database for session storage.

Please checkout the code from the following GitHub [repository](https://github.com/svlada/springsession-jdbc).

## Table of contents
1. <a title="Introduction: Spring Session" href="#introduction">Introduction</a>
2. <a title="Project setup using Maven" href="#project-setup">Project setup</a>
2. <a title="Test" href="#test">Test</a>

## <a name="introduction" id="introduction">Introduction</a>
Before going into the more details of [Spring Session](https://projects.spring.io/spring-session/) configuration, I will share my view on the hype around Stateful vs Stateless session management.

Lately, a lot of people started using JSON Web Token (JWT) as a stateless mechanism for handling sessions. A couple of years ago I wrote an [article on that topic](http://www.svlada.com/jwt-token-authentication-with-spring-boot/) and honestly didn't know that it will be abused by so many people. The main idea was to show how to override and extend various parts of Spring Security. I would strongly recommend not using JWT for handling sessions. Let's see what are the pros and cons of stateless and stateful session management approaches.

### Stateless

Pros

1. No need to scale session data on the server-side as the session is maintained through cryptographically signed JSON Web Token (JWT). 

Cons

1. No <strong>log-out</strong> feature without introducing state on server side.
2. Potential token explosion as JSON Web Token becomes larger.
3. Sending JSON Web Token (JWT) payload on each request can be expensive.

### Stateful

Pros

1. Ability to log-out user
2. Out-of-box sliding session 

Cons

1. /

In short, don't use JSON Web Token to manage session data for your web applications. Most of the web applications will be fine with storing session-related data on Redis.

If you have microservices architecture, you can use API Gateway as a translation layer that would validate session-id and create a federated token to be used by the services. That's one use case where JSON Web Token fits nicely. 

## <a name="project-setup" id="project-setup">Project setup</a>

Include ``spring-session-core`` and ``spring-session-jdbc`` in your ``pom.xml`` file. 

**Maven dependencies**

```java
<dependency>
  <groupId>org.springframework.session</groupId>
  <artifactId>spring-session-jdbc</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.session</groupId>
  <artifactId>spring-session-core</artifactId>
</dependency>
```

**Spring security configuration**

The following class shows how to configure REST API security with the Spring Session:

```java
@Configuration
@EnableWebSecurity
@EnableJdbcHttpSession
public class WebSecurityConfig  extends WebSecurityConfigurerAdapter {
    private final RestAuthenticationEntryPoint restAuthenticationEntryPoint;
    private final AuthenticationProvider provider;

    @Autowired
    public WebSecurityConfig(final RestAuthenticationEntryPoint restAuthenticationEntryPoint,
        final AuthenticationProvider provider) {
        this.restAuthenticationEntryPoint = restAuthenticationEntryPoint;
        this.provider = provider;
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .csrf().disable()
            .exceptionHandling()
            .authenticationEntryPoint(restAuthenticationEntryPoint)
            .and()
                .formLogin()
                .successHandler(new SessionAuthenticationSuccessHandler())
                .failureHandler(new SimpleUrlAuthenticationFailureHandler())
            .and()
                .logout()
                    .defaultLogoutSuccessHandlerFor(new HttpStatusReturningLogoutSuccessHandler(),
                        new AntPathRequestMatcher("/logout"))
            .and()
                .authorizeRequests()
                    .antMatchers("/login").permitAll()
                    .antMatchers("/h2/**").permitAll()
            .and()
                .authorizeRequests().antMatchers("/api/**").hasAnyRole("ADMIN")
            .and()
                .requestCache()
                .requestCache(new NullRequestCache());
    }

    @Bean
    public HttpSessionIdResolver httpSessionIdResolver() {
        return new HeaderHttpSessionIdResolver("X-Auth-Token");
    }

    @Override
    protected void configure(AuthenticationManagerBuilder auth) {
        auth.authenticationProvider(provider);
    }
}
```

The following list describes the WebSecurityConfig elements:

1. **RestAuthenticationEntryPoint** - The entry point implementation which returns 401 status, indicating that the request requires authentication.
2. **SessionAuthenticationSuccessHandler** - Success authentication handler that returns 200 status on successful authentication.
3. **HttpSessionIdResolver** - Use ``HeaderHttpSessionIdResolver`` if you want to send authentication token through Http headers. Please check the following [git commit](https://github.com/spring-projects/spring-session/commit/6f05c84aa7c1f7c4efcf2c0d3c20709a79b0785f) regarding class name changes.
4. **@EnableJdbcHttpSession** - This annotation is needed as it exposes ``SessionRepositoryFilter`` that will use the database for storing session data.

## <a name="test" id="test">Test</a>

### User login

Check for the ```x-auth-token``` in response and include it with the subsequent requests.

```java
curl -X POST \
  http://localhost:1999/login \
  -H 'cache-control: no-cache' \
  -H 'content-type: multipart/form-data; boundary=----WebKitFormBoundary7MA4YWxkTrZu0gW' \
  -F username=test \
  -F password=test
```

### User logout

```java
curl -X GET \
  http://localhost:1999/logout \
  -H 'cache-control: no-cache' \
  -H 'x-auth-token: 2eabcc45-0bb5-40f7-8d48-8aec0fdf0bbc'
```

### Access to protected resource

This is an example of how to access the protected resource by including the access token in the headers:

```java
curl -X GET \
  http://localhost:1999/api/sample \
  -H 'cache-control: no-cache' \
  -H 'x-auth-token: 30ab6295-7b63-4172-9fb3-3514d5e46390'
```

## Source code

**Session Authentication Success Handler**

```java
public class SessionAuthenticationSuccessHandler implements AuthenticationSuccessHandler {
    @Override
    public void onAuthenticationSuccess(HttpServletRequest request, HttpServletResponse response,
        Authentication authentication) throws IOException, ServletException {
        response.setStatus(HttpServletResponse.SC_OK);
    }
}
```

**RestAuthenticationEntryPoint**

```java
@Component
public class RestAuthenticationEntryPoint implements AuthenticationEntryPoint {
    @Override
    public void commence(HttpServletRequest httpServletRequest,
        HttpServletResponse httpServletResponse, AuthenticationException e)
        throws IOException, ServletException {

        httpServletResponse.sendError(HttpServletResponse.SC_UNAUTHORIZED, "Unauthorized");
    }
}
```

**AuthenticationProviderConfig**

```java
@Configuration
public class AuthenticationProviderConfig {
    private final PasswordEncoder passwordEncoder;
    private final UserDetailsService userDetailsService;

    public AuthenticationProviderConfig(PasswordEncoder passwordEncoder,
        @Qualifier("databaseUserDetailsService") UserDetailsService userDetailsService) {
        this.passwordEncoder = passwordEncoder;
        this.userDetailsService = userDetailsService;
    }

    @Bean
    public AuthenticationProvider databaseAuthenticationProvider() {
        DaoAuthenticationProvider daoAuthenticationProvider = new DaoAuthenticationProvider();
        daoAuthenticationProvider.setUserDetailsService(userDetailsService);
        daoAuthenticationProvider.setPasswordEncoder(passwordEncoder);
        return daoAuthenticationProvider;
    }
}
```

**Password encoder configuration**

```java
@Configuration
public class PasswordEncoderConfig {
    @Bean
    protected PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(11);
    }
}
```
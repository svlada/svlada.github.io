---
title: JWT Authentication Tutorial
description: This article will guide you on how you can implement JWT authentication with Spring Boot.
date: 2016-09-05
tags:
  - java
  - spring-boot
  - jwt
  - spring-security
layout: layouts/post.njk
permalink: "jwt-token-authentication-with-spring-boot/index.html"
---

## Introduction

This article is a guide to implementing JWT authentication with Spring Boot.

At a minimum, the client must exchange a username and password to obtain a JWT. This token is then used to authenticate subsequent API calls.

The core workflows for securing APIs with JWT are:
- AJAX Login Authentication
- JWT Token Authentication

## Prerequisite

See the linked GitHub [repo](https://github.com/svlada/springboot-security-jwt) for code.

This project is using H2 in-memory database to store sample user data. To simplify setup, I created data fixtures and configured Spring Boot to automatically load them at startup from: `/jwt-demo/src/main/resources/data.sql`

The overall project structure is shown below:

```java
+---main
|   +---java
|   |   \---com
|   |       \---svlada
|   |           +---common
|   |           +---entity
|   |           +---profile
|   |           |   \---endpoint
|   |           +---security
|   |           |   +---auth
|   |           |   |   +---ajax
|   |           |   |   \---jwt
|   |           |   |       +---extractor
|   |           |   |       \---verifier
|   |           |   +---config
|   |           |   +---endpoint
|   |           |   +---exceptions
|   |           |   \---model
|   |           |       \---token
|   |           \---user
|   |               +---repository
|   |               \---service
|   \---resources
|       +---static
|       \---templates
```

## Ajax authentication

In the context of AJAX authentication, the user provides credentials in a JSON payload, sent as part of an XMLHttpRequest.

In this first part of the tutorial, AJAX authentication is implemented using the standard patterns provided by the Spring Security framework. The following components need to be implemented:

1. AjaxLoginProcessingFilter
2. AjaxAuthenticationProvider
3. AjaxAwareAuthenticationSuccessHandler
4. AjaxAwareAuthenticationFailureHandler
5. RestAuthenticationEntryPoint
6. WebSecurityConfig

Before going into the implementation details, let's look at the request/response authentication flow.

**Ajax authentication request example**

The Authentication API allows users to exchange their credentials for an authentication token.

In this example, the client initiates the process by calling the authentication endpoint: `/api/auth/login`

Raw HTTP request:

```java
POST /api/auth/login HTTP/1.1
Host: localhost:9966
X-Requested-With: XMLHttpRequest
Content-Type: application/json
Cache-Control: no-cache

{
    "username": "svlada@gmail.com",
    "password": "test1234"
}
```

CURL:

```java
curl -X POST -H "X-Requested-With: XMLHttpRequest" -H "Content-Type: application/json" -H "Cache-Control: no-cache" -d '{
    "username": "svlada@gmail.com",
    "password": "test1234"
}' "http://localhost:9966/api/auth/login"
```

**Ajax authentication response example**

If the client supplies valid credentials, the Authentication API responds with an HTTP response that includes the following details:

1. HTTP status _200 OK_
2. Signed JWT Access and Refresh tokens included in the response body

**JWT Access token** - used to authenticate against protected API resources. It must be set in ```X-Authorization``` header.

**JWT Refresh token** - used to acquire new Access Token. Token refresh is handled by the following API endpoint: ```/api/auth/token```.

Raw HTTP Response:

```json
{
  "token": "eyJhbGciOiJIUzUxMiJ9.eyJzdWIiOiJzdmxhZGFAZ21haWwuY29tIiwic2NvcGVzIjpbIlJPTEVfQURNSU4iLCJST0xFX1BSRU1JVU1fTUVNQkVSIl0sImlzcyI6Imh0dHA6Ly9zdmxhZGEuY29tIiwiaWF0IjoxNDcyMDMzMzA4LCJleHAiOjE0NzIwMzQyMDh9.41rxtplFRw55ffqcw1Fhy2pnxggssdWUU8CDOherC0Kw4sgt3-rw_mPSWSgQgsR0NLndFcMPh7LSQt5mkYqROQ",
  
  "refreshToken": "eyJhbGciOiJIUzUxMiJ9.eyJzdWIiOiJzdmxhZGFAZ21haWwuY29tIiwic2NvcGVzIjpbIlJPTEVfUkVGUkVTSF9UT0tFTiJdLCJpc3MiOiJodHRwOi8vc3ZsYWRhLmNvbSIsImp0aSI6IjkwYWZlNzhjLTFkMmUtNDg2OS1hNzdlLTFkNzU0YjYwZTBjZSIsImlhdCI6MTQ3MjAzMzMwOCwiZXhwIjoxNDcyMDM2OTA4fQ.SEEG60YRznBB2O7Gn_5X6YbRmyB3ml4hnpSOxqkwQUFtqA6MZo7_n2Am2QhTJBJA1Ygv74F2IxiLv0urxGLQjg"
}
```

**JWT Access Token**

The JWT access token is used for both authentication and authorization:
- Authentication is performed by verifying the token's signature. If the signature is valid, access to the requested API resource is granted.
- Authorization is determined by checking the privileges in the **scope** attribute of the token.

A decoded JWT access token consists of three parts: Header, Claims, and Signature, as shown below:

Header
```json

{
    "alg": "HS512"
}
```

Claims
```json
{
  "sub": "svlada@gmail.com",
  "scopes": [
    "ROLE_ADMIN",
    "ROLE_PREMIUM_MEMBER"
  ],
  "iss": "http://svlada.com",
  "iat": 1472033308,
  "exp": 1472034208
}
```

Signature (base64 encoded)

```java
41rxtplFRw55ffqcw1Fhy2pnxggssdWUU8CDOherC0Kw4sgt3-rw_mPSWSgQgsR0NLndFcMPh7LSQt5mkYqROQ
```

**JWT Refresh Token**

A refresh token is a long-lived token used to request new access tokens. Its expiration time is longer than that of an access token.

In this tutorial, we'll use the `jti` claim to maintain a list of blacklisted or revoked tokens. The JWT ID (`jti`) claim, defined by [RFC 7519](https://tools.ietf.org/html/rfc7519#section-4.1.7), uniquely identifies each refresh token.

A decoded refresh token consists of three parts: Header, Claims, and Signature, as shown below:

Header
```json
{
  "alg": "HS512"
}
```

Claims
```json
{
  "sub": "svlada@gmail.com",
  "scopes": [
    "ROLE_REFRESH_TOKEN"
  ],
  "iss": "http://svlada.com",
  "jti": "90afe78c-1d2e-4869-a77e-1d754b60e0ce",
  "iat": 1472033308,
  "exp": 1472036908
}
```

Signature (base64 encoded)
```
SEEG60YRznBB2O7Gn_5X6YbRmyB3ml4hnpSOxqkwQUFtqA6MZo7_n2Am2QhTJBJA1Ygv74F2IxiLv0urxGLQjg
```

#### AjaxLoginProcessingFilter

The first step is to extend `AbstractAuthenticationProcessingFilter` to provide custom processing for AJAX authentication requests.
- Deserialization and basic validation of the incoming JSON payload are handled in the `AjaxLoginProcessingFilter#attemptAuthentication` method. If validation succeeds, the authentication logic is delegated to the `AjaxAuthenticationProvider` class.
- In case of successful authentication, the `AjaxLoginProcessingFilter#successfulAuthentication` method is invoked.
- In case of failed authentication, the `AjaxLoginProcessingFilter#unsuccessfulAuthentication` method is invoked.

```java
public class AjaxLoginProcessingFilter extends AbstractAuthenticationProcessingFilter {
    private static Logger logger = LoggerFactory.getLogger(AjaxLoginProcessingFilter.class);

    private final AuthenticationSuccessHandler successHandler;
    private final AuthenticationFailureHandler failureHandler;

    private final ObjectMapper objectMapper;
    
    public AjaxLoginProcessingFilter(String defaultProcessUrl, AuthenticationSuccessHandler successHandler, 
            AuthenticationFailureHandler failureHandler, ObjectMapper mapper) {
        super(defaultProcessUrl);
        this.successHandler = successHandler;
        this.failureHandler = failureHandler;
        this.objectMapper = mapper;
    }

    @Override
    public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response)
            throws AuthenticationException, IOException, ServletException {
        if (!HttpMethod.POST.name().equals(request.getMethod()) || !WebUtil.isAjax(request)) {
            if(logger.isDebugEnabled()) {
                logger.debug("Authentication method not supported. Request method: " + request.getMethod());
            }
            throw new AuthMethodNotSupportedException("Authentication method not supported");
        }

        LoginRequest loginRequest = objectMapper.readValue(request.getReader(), LoginRequest.class);
        
        if (StringUtils.isBlank(loginRequest.getUsername()) || StringUtils.isBlank(loginRequest.getPassword())) {
            throw new AuthenticationServiceException("Username or Password not provided");
        }

        UsernamePasswordAuthenticationToken token = new UsernamePasswordAuthenticationToken(loginRequest.getUsername(), loginRequest.getPassword());

        return this.getAuthenticationManager().authenticate(token);
    }

    @Override
    protected void successfulAuthentication(HttpServletRequest request, HttpServletResponse response, FilterChain chain,
            Authentication authResult) throws IOException, ServletException {
        successHandler.onAuthenticationSuccess(request, response, authResult);
    }

    @Override
    protected void unsuccessfulAuthentication(HttpServletRequest request, HttpServletResponse response,
            AuthenticationException failed) throws IOException, ServletException {
        SecurityContextHolder.clearContext();
        failureHandler.onAuthenticationFailure(request, response, failed);
    }
}
```

#### AjaxAuthenticationProvider

The responsibility of the `AjaxAuthenticationProvider` class is to:
1. Verify user credentials against a database, LDAP, or another system that stores user data.
2. Throw an authentication exception if the username and password do not match any record.
3. Create a `UserContext` and populate it with the required user data (in this case, just the username and user privileges).
4. Delegate JWT token creation to the `AjaxAwareAuthenticationSuccessHandler` upon successful authentication.

```java
@Component
public class AjaxAuthenticationProvider implements AuthenticationProvider {
    private final BCryptPasswordEncoder encoder;
    private final DatabaseUserService userService;

    @Autowired
    public AjaxAuthenticationProvider(final DatabaseUserService userService, final BCryptPasswordEncoder encoder) {
        this.userService = userService;
        this.encoder = encoder;
    }

    @Override
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {
        Assert.notNull(authentication, "No authentication data provided");

        String username = (String) authentication.getPrincipal();
        String password = (String) authentication.getCredentials();

        User user = userService.getByUsername(username).orElseThrow(() -> new UsernameNotFoundException("User not found: " + username));
        
        if (!encoder.matches(password, user.getPassword())) {
            throw new BadCredentialsException("Authentication Failed. Username or Password not valid.");
        }

        if (user.getRoles() == null) throw new InsufficientAuthenticationException("User has no roles assigned");
        
        List<GrantedAuthority> authorities = user.getRoles().stream()
                .map(authority -> new SimpleGrantedAuthority(authority.getRole().authority()))
                .collect(Collectors.toList());
        
        UserContext userContext = UserContext.create(user.getUsername(), authorities);
        
        return new UsernamePasswordAuthenticationToken(userContext, null, userContext.getAuthorities());
    }

    @Override
    public boolean supports(Class<?> authentication) {
        return (UsernamePasswordAuthenticationToken.class.isAssignableFrom(authentication));
    }
}
```

#### AjaxAwareAuthenticationSuccessHandler

We'll implement the `AuthenticationSuccessHandler` interface, which is invoked when a client has been successfully authenticated.

Our custom implementation is the `AjaxAwareAuthenticationSuccessHandler` class. Its responsibility is to add a JSON payload containing the JWT access token and refresh token to the HTTP response body.

```java
@Component
public class AjaxAwareAuthenticationSuccessHandler implements AuthenticationSuccessHandler {
    private final ObjectMapper mapper;
    private final JwtTokenFactory tokenFactory;

    @Autowired
    public AjaxAwareAuthenticationSuccessHandler(final ObjectMapper mapper, final JwtTokenFactory tokenFactory) {
        this.mapper = mapper;
        this.tokenFactory = tokenFactory;
    }

    @Override
    public void onAuthenticationSuccess(HttpServletRequest request, HttpServletResponse response,
            Authentication authentication) throws IOException, ServletException {
        UserContext userContext = (UserContext) authentication.getPrincipal();
        
        JwtToken accessToken = tokenFactory.createAccessJwtToken(userContext);
        JwtToken refreshToken = tokenFactory.createRefreshToken(userContext);
        
        Map<String, String> tokenMap = new HashMap<String, String>();
        tokenMap.put("token", accessToken.getToken());
        tokenMap.put("refreshToken", refreshToken.getToken());

        response.setStatus(HttpStatus.OK.value());
        response.setContentType(MediaType.APPLICATION_JSON_VALUE);
        mapper.writeValue(response.getWriter(), tokenMap);

        clearAuthenticationAttributes(request);
    }

    /**
     * Removes temporary authentication-related data which may have been stored
     * in the session during the authentication process..
     * 
     */
    protected final void clearAuthenticationAttributes(HttpServletRequest request) {
        HttpSession session = request.getSession(false);

        if (session == null) {
            return;
        }

        session.removeAttribute(WebAttributes.AUTHENTICATION_EXCEPTION);
    }
}
```

Let's focus for a moment on how JWT Access token is created. In this tutorial we are using [Java JWT](https://github.com/jwtk/jjwt) library created by [Stormpath](https://stormpath.com/).

Make sure that ```JJWT``` dependency is included in your ```pom.xml```.

```java
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt</artifactId>
    <version>${jjwt.version}</version>
</dependency>
```

We have created factory class (```JwtTokenFactory```) to decouple token creation logic.

Method ```JwtTokenFactory#createAccessJwtToken```  creates signed JWT Access token.

Method ```JwtTokenFactory#createRefreshToken``` creates signed JWT Refresh token.


```java
@Component
public class JwtTokenFactory {
    private final JwtSettings settings;

    @Autowired
    public JwtTokenFactory(JwtSettings settings) {
        this.settings = settings;
    }

    /**
     * Factory method for issuing new JWT Tokens.
     * 
     * @param username
     * @param roles
     * @return
     */
    public AccessJwtToken createAccessJwtToken(UserContext userContext) {
        if (StringUtils.isBlank(userContext.getUsername())) 
            throw new IllegalArgumentException("Cannot create JWT Token without username");

        if (userContext.getAuthorities() == null || userContext.getAuthorities().isEmpty()) 
            throw new IllegalArgumentException("User doesn't have any privileges");

        Claims claims = Jwts.claims().setSubject(userContext.getUsername());
        claims.put("scopes", userContext.getAuthorities().stream().map(s -> s.toString()).collect(Collectors.toList()));

        DateTime currentTime = new DateTime();

        String token = Jwts.builder()
          .setClaims(claims)
          .setIssuer(settings.getTokenIssuer())
          .setIssuedAt(currentTime.toDate())
          .setExpiration(currentTime.plusMinutes(settings.getTokenExpirationTime()).toDate())
          .signWith(SignatureAlgorithm.HS512, settings.getTokenSigningKey())
        .compact();

        return new AccessJwtToken(token, claims);
    }

    public JwtToken createRefreshToken(UserContext userContext) {
        if (StringUtils.isBlank(userContext.getUsername())) {
            throw new IllegalArgumentException("Cannot create JWT Token without username");
        }

        DateTime currentTime = new DateTime();

        Claims claims = Jwts.claims().setSubject(userContext.getUsername());
        claims.put("scopes", Arrays.asList(Scopes.REFRESH_TOKEN.authority()));
        
        String token = Jwts.builder()
          .setClaims(claims)
          .setIssuer(settings.getTokenIssuer())
          .setId(UUID.randomUUID().toString())
          .setIssuedAt(currentTime.toDate())
          .setExpiration(currentTime.plusMinutes(settings.getRefreshTokenExpTime()).toDate())
          .signWith(SignatureAlgorithm.HS512, settings.getTokenSigningKey())
        .compact();

        return new AccessJwtToken(token, claims);
    }
}
```

Please note: if you instantiate a `Claims` object outside of `Jwts.builder()`, you must first call `Jwts.builder().setClaims(claims)`.

Why? By default, `Jwts.builder()` creates an empty `Claims` object. This means that if you call `.setClaims()` after setting the subject with `.setSubject()`, your subject will be overwritten. A new instance of `Claims` replaces the one created internally by `Jwts.builder()`.

#### AjaxAwareAuthenticationFailureHandler

The `AjaxAwareAuthenticationFailureHandler` is invoked by the `AjaxLoginProcessingFilter` in case of authentication failures. It can be customized to return specific error messages based on the type of exception that occurs during the authentication process.

```java
@Component
public class AjaxAwareAuthenticationFailureHandler implements AuthenticationFailureHandler {
    private final ObjectMapper mapper;
    
    @Autowired
    public AjaxAwareAuthenticationFailureHandler(ObjectMapper mapper) {
        this.mapper = mapper;
    }   
    
    @Override
    public void onAuthenticationFailure(HttpServletRequest request, HttpServletResponse response,
            AuthenticationException e) throws IOException, ServletException {
        
        response.setStatus(HttpStatus.UNAUTHORIZED.value());
        response.setContentType(MediaType.APPLICATION_JSON_VALUE);
        
        if (e instanceof BadCredentialsException) {
            mapper.writeValue(response.getWriter(), ErrorResponse.of("Invalid username or password", ErrorCode.AUTHENTICATION, HttpStatus.UNAUTHORIZED));
        } else if (e instanceof JwtExpiredTokenException) {
            mapper.writeValue(response.getWriter(), ErrorResponse.of("Token has expired", ErrorCode.JWT_TOKEN_EXPIRED, HttpStatus.UNAUTHORIZED));
        } else if (e instanceof AuthMethodNotSupportedException) {
            mapper.writeValue(response.getWriter(), ErrorResponse.of(e.getMessage(), ErrorCode.AUTHENTICATION, HttpStatus.UNAUTHORIZED));
        }

        mapper.writeValue(response.getWriter(), ErrorResponse.of("Authentication failed", ErrorCode.AUTHENTICATION, HttpStatus.UNAUTHORIZED));
    }
}

```

## JWT Authentication

Token-based authentication schemes have become immensely popular in recent years, as they offer important benefits compared to traditional session and cookie-based approaches:

1. CORS
2. No need for CSRF protection
3. Better integration with mobile
4. Reduced load on authorization server
5. No need for distributed session store

Some trade-offs have to be made with this approach:

1. More vulnerable to XSS attacks
2. Access token can contain outdated authorization claims (e.g when some of the user privileges are revoked)
3. Access tokens can grow in size in case of increased number of claims
4. File download API can be tricky to implement
5. True statelessness and revocation are mutually exclusive

In this article we'll investigate how JWT's can used for token based authentication.

JWT Authentication flow is very simple: 

1. User obtains Refresh and Access tokens by providing credentials to the Authorization server
2. User sends Access token with each request to access protected API resource
3. Access token is signed and contains user identity (e.g. user id) and authorization claims. 

It's important to note that authorization claims are embedded in the access token. Why does this matter? If authorization data (e.g., user privileges stored in the database) changes during the lifetime of an access token, those changes will not take effect until a new token is issued.

In most cases, this is not a major issue since access tokens are typically short-lived. However, if up-to-the-minute accuracy is required, consider using the opaque token pattern instead.

Before diving into the implementation details, let's look at a sample request to a protected API resource.

**Signed request to protected API resource**

The pattern for access tokens is: `<header-name>: Bearer <access_token>`

In our example, the header name is `X-Authorization`.

Raw HTTP request:
```java
GET /api/me HTTP/1.1
Host: localhost:9966
X-Authorization: Bearer eyJhbGciOiJIUzUxMiJ9.eyJzdWIiOiJzdmxhZGFAZ21haWwuY29tIiwic2NvcGVzIjpbIlJPTEVfQURNSU4iLCJST0xFX1BSRU1JVU1fTUVNQkVSIl0sImlzcyI6Imh0dHA6Ly9zdmxhZGEuY29tIiwiaWF0IjoxNDcyMzkwMDY1LCJleHAiOjE0NzIzOTA5NjV9.Y9BR7q3f1npsSEYubz-u8tQ8dDOdBcVPFN7AIfWwO37KyhRugVzEbWVPO1obQlHNJWA0Nx1KrEqHqMEjuNWo5w
Cache-Control: no-cache
```

CURL:
```java
curl -X GET -H "X-Authorization: Bearer eyJhbGciOiJIUzUxMiJ9.eyJzdWIiOiJzdmxhZGFAZ21haWwuY29tIiwic2NvcGVzIjpbIlJPTEVfQURNSU4iLCJST0xFX1BSRU1JVU1fTUVNQkVSIl0sImlzcyI6Imh0dHA6Ly9zdmxhZGEuY29tIiwiaWF0IjoxNDcyMzkwMDY1LCJleHAiOjE0NzIzOTA5NjV9.Y9BR7q3f1npsSEYubz-u8tQ8dDOdBcVPFN7AIfWwO37KyhRugVzEbWVPO1obQlHNJWA0Nx1KrEqHqMEjuNWo5w" -H "Cache-Control: no-cache" "http://localhost:9966/api/me"
```

Let's look at the implementation details. The following components are required to implement JWT authentication:

1. JwtTokenAuthenticationProcessingFilter
2. JwtAuthenticationProvider
3. SkipPathRequestMatcher
4. JwtHeaderTokenExtractor
5. BloomFilterTokenVerifier
6. WebSecurityConfig

#### JwtTokenAuthenticationProcessingFilter

`JwtTokenAuthenticationProcessingFilter` applies to all API endpoints under `/api/**`, except for the refresh-token endpoint `/api/auth/token` and the login endpoint `/api/auth/login`.

Responsibilities:
1. Extract the access token from the `X-Authorization` header.
   - If a token is present, delegate authentication to `JwtAuthenticationProvider`.
   - If no token is found (or the format is invalid), throw an authentication exception.
2. Invoke success or failure handlers based on the outcome of `JwtAuthenticationProvider`.

Important: On successful authentication, ensure you call `chain.doFilter(request, response)` so processing continues to the next filter. The final filter—`FilterSecurityInterceptor#doFilter`—is responsible for invoking your controller method that handles the requested API resource.

```java
public class JwtTokenAuthenticationProcessingFilter extends AbstractAuthenticationProcessingFilter {
    private final AuthenticationFailureHandler failureHandler;
    private final TokenExtractor tokenExtractor;
    
    @Autowired
    public JwtTokenAuthenticationProcessingFilter(AuthenticationFailureHandler failureHandler, 
            TokenExtractor tokenExtractor, RequestMatcher matcher) {
        super(matcher);
        this.failureHandler = failureHandler;
        this.tokenExtractor = tokenExtractor;
    }

    @Override
    public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response)
            throws AuthenticationException, IOException, ServletException {
        String tokenPayload = request.getHeader(WebSecurityConfig.JWT_TOKEN_HEADER_PARAM);
        RawAccessJwtToken token = new RawAccessJwtToken(tokenExtractor.extract(tokenPayload));
        return getAuthenticationManager().authenticate(new JwtAuthenticationToken(token));
    }

    @Override
    protected void successfulAuthentication(HttpServletRequest request, HttpServletResponse response, FilterChain chain,
            Authentication authResult) throws IOException, ServletException {
        SecurityContext context = SecurityContextHolder.createEmptyContext();
        context.setAuthentication(authResult);
        SecurityContextHolder.setContext(context);
        chain.doFilter(request, response);
    }

    @Override
    protected void unsuccessfulAuthentication(HttpServletRequest request, HttpServletResponse response,
            AuthenticationException failed) throws IOException, ServletException {
        SecurityContextHolder.clearContext();
        failureHandler.onAuthenticationFailure(request, response, failed);
    }
}
```

#### JwtHeaderTokenExtractor

The `JwtHeaderTokenExtractor` is a simple class used to extract the authorization token from the request header. You can extend the `TokenExtractor` interface and provide a custom implementation—for example, one that extracts a token from a URL instead of a header.

```java
@Component
public class JwtHeaderTokenExtractor implements TokenExtractor {
    public static String HEADER_PREFIX = "Bearer ";

    @Override
    public String extract(String header) {
        if (StringUtils.isBlank(header)) {
            throw new AuthenticationServiceException("Authorization header cannot be blank!");
        }

        if (header.length() < HEADER_PREFIX.length()) {
            throw new AuthenticationServiceException("Invalid authorization header size.");
        }

        return header.substring(HEADER_PREFIX.length(), header.length());
    }
}
```

#### JwtAuthenticationProvider

The `JwtAuthenticationProvider` has the following responsibilities:
1. Verify the access token's signature.
2. Extract identity and authorization claims from the access token and use them to create a `UserContext`.
3. Throw an authentication exception if the access token is malformed, expired, or not signed with the correct signing key.

```java
@Component
public class JwtAuthenticationProvider implements AuthenticationProvider {
    private final JwtSettings jwtSettings;
    
    @Autowired
    public JwtAuthenticationProvider(JwtSettings jwtSettings) {
        this.jwtSettings = jwtSettings;
    }

    @Override
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {
        RawAccessJwtToken rawAccessToken = (RawAccessJwtToken) authentication.getCredentials();

        Jws<Claims> jwsClaims = rawAccessToken.parseClaims(jwtSettings.getTokenSigningKey());
        String subject = jwsClaims.getBody().getSubject();
        List<String> scopes = jwsClaims.getBody().get("scopes", List.class);
        List<GrantedAuthority> authorities = scopes.stream()
                .map(authority -> new SimpleGrantedAuthority(authority))
                .collect(Collectors.toList());
        
        UserContext context = UserContext.create(subject, authorities);
        
        return new JwtAuthenticationToken(context, context.getAuthorities());
    }

    @Override
    public boolean supports(Class<?> authentication) {
        return (JwtAuthenticationToken.class.isAssignableFrom(authentication));
    }
}

```

#### SkipPathRequestMatcher

The `JwtTokenAuthenticationProcessingFilter` is configured to skip the following endpoints: `/api/auth/login` and `/api/auth/token`. This is achieved using the `SkipPathRequestMatcher` implementation of `RequestMatcher`.

```java
public class SkipPathRequestMatcher implements RequestMatcher {
    private OrRequestMatcher matchers;
    private RequestMatcher processingMatcher;
    
    public SkipPathRequestMatcher(List<String> pathsToSkip, String processingPath) {
        Assert.notNull(pathsToSkip);
        List<RequestMatcher> m = pathsToSkip.stream().map(path -> new AntPathRequestMatcher(path)).collect(Collectors.toList());
        matchers = new OrRequestMatcher(m);
        processingMatcher = new AntPathRequestMatcher(processingPath);
    }

    @Override
    public boolean matches(HttpServletRequest request) {
        if (matchers.matches(request)) {
            return false;
        }
        return processingMatcher.matches(request) ? true : false;
    }
}
```

#### WebSecurityConfig

The `WebSecurityConfig` class extends `WebSecurityConfigurerAdapter` to provide custom security configuration.

The following beans are configured and instantiated in this class:
- `AjaxLoginProcessingFilter`
- `JwtTokenAuthenticationProcessingFilter`
- `AuthenticationManager`
- `BCryptPasswordEncoder`

In addition, the `WebSecurityConfig#configure(HttpSecurity http)` method is used to define patterns for protected and unprotected API endpoints. Note that CSRF protection is disabled, since the application does not use cookies.

```java
@Configuration
@EnableWebSecurity
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {
    public static final String JWT_TOKEN_HEADER_PARAM = "X-Authorization";
    public static final String FORM_BASED_LOGIN_ENTRY_POINT = "/api/auth/login";
    public static final String TOKEN_BASED_AUTH_ENTRY_POINT = "/api/**";
    public static final String TOKEN_REFRESH_ENTRY_POINT = "/api/auth/token";
    
    @Autowired private RestAuthenticationEntryPoint authenticationEntryPoint;
    @Autowired private AuthenticationSuccessHandler successHandler;
    @Autowired private AuthenticationFailureHandler failureHandler;
    @Autowired private AjaxAuthenticationProvider ajaxAuthenticationProvider;
    @Autowired private JwtAuthenticationProvider jwtAuthenticationProvider;
    
    @Autowired private TokenExtractor tokenExtractor;
    
    @Autowired private AuthenticationManager authenticationManager;
    
    @Autowired private ObjectMapper objectMapper;
        
    protected AjaxLoginProcessingFilter buildAjaxLoginProcessingFilter() throws Exception {
        AjaxLoginProcessingFilter filter = new AjaxLoginProcessingFilter(FORM_BASED_LOGIN_ENTRY_POINT, successHandler, failureHandler, objectMapper);
        filter.setAuthenticationManager(this.authenticationManager);
        return filter;
    }
    
    protected JwtTokenAuthenticationProcessingFilter buildJwtTokenAuthenticationProcessingFilter() throws Exception {
        List<String> pathsToSkip = Arrays.asList(TOKEN_REFRESH_ENTRY_POINT, FORM_BASED_LOGIN_ENTRY_POINT);
        SkipPathRequestMatcher matcher = new SkipPathRequestMatcher(pathsToSkip, TOKEN_BASED_AUTH_ENTRY_POINT);
        JwtTokenAuthenticationProcessingFilter filter 
            = new JwtTokenAuthenticationProcessingFilter(failureHandler, tokenExtractor, matcher);
        filter.setAuthenticationManager(this.authenticationManager);
        return filter;
    }

    @Bean
    @Override
    public AuthenticationManager authenticationManagerBean() throws Exception {
        return super.authenticationManagerBean();
    }
    
    @Override
    protected void configure(AuthenticationManagerBuilder auth) {
        auth.authenticationProvider(ajaxAuthenticationProvider);
        auth.authenticationProvider(jwtAuthenticationProvider);
    }
    
    @Bean
    protected BCryptPasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
        .csrf().disable() // We don't need CSRF for JWT based authentication
        .exceptionHandling()
        .authenticationEntryPoint(this.authenticationEntryPoint)
        
        .and()
            .sessionManagement()
            .sessionCreationPolicy(SessionCreationPolicy.STATELESS)

        .and()
            .authorizeRequests()
                .antMatchers(FORM_BASED_LOGIN_ENTRY_POINT).permitAll() // Login end-point
                .antMatchers(TOKEN_REFRESH_ENTRY_POINT).permitAll() // Token refresh end-point
                .antMatchers("/console").permitAll() // H2 Console Dash-board - only for testing
        .and()
            .authorizeRequests()
                .antMatchers(TOKEN_BASED_AUTH_ENTRY_POINT).authenticated() // Protected API End-points
        .and()
            .addFilterBefore(buildAjaxLoginProcessingFilter(), UsernamePasswordAuthenticationFilter.class)
            .addFilterBefore(buildJwtTokenAuthenticationProcessingFilter(), UsernamePasswordAuthenticationFilter.class);
    }
}
```

#### PasswordEncoderConfig

BCrypt encoder that is in AjaxAuthenticationProvider.

```java
@Configuration
public class PasswordEncoderConfig {
    @Bean
    protected BCryptPasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

#### BloomFilterTokenVerifier

This is dummy class. You should ideally implement your own TokenVerifier to check for revoked tokens.

```java
@Component
public class BloomFilterTokenVerifier implements TokenVerifier {
    @Override
    public boolean verify(String jti) {
        return true;
    }
}
```

### Conclusion

I've often heard it said that losing a JWT token is like losing your house keys—anyone who finds it can walk right in. So handle your tokens with care.

## References

- [I don’t see the point in Revoking or Blacklisting JWT](https://www.dinochiesa.net/?p=1388)
- [Spring Security Architecture - Dave Syer](https://github.com/dsyer/spring-security-architecture)
- [Invalidating JWT](http://stackoverflow.com/questions/21978658/invalidating-json-web-tokens/36884683#36884683)
- [Secure and stateless JWT implementation](http://stackoverflow.com/questions/38557379/secure-and-stateless-jwt-implementation)
- [Learn JWT](https://github.com/dwyl/learn-json-web-tokens)
- [Opaque access tokens and cloud foundry](https://www.cloudfoundry.org/opaque-access-tokens-cloud-foundry/)
- [The unspoken vulnerability of JWTS](http://by.jtl.xyz/2016/06/the-unspoken-vulnerability-of-jwts.html)
- [How To Control User Identity Within Micro-services](http://nordicapis.com/how-to-control-user-identity-within-microservices/)
- [Why Does OAuth v2 Have Both Access and Refresh Tokens?](http://stackoverflow.com/questions/3487991/why-does-oauth-v2-have-both-access-and-refresh-tokens/12885823)
- [RFC-6749](https://tools.ietf.org/html/rfc6749)
- [Are breaches of JWT-based servers more damaging?](https://www.sslvpn.online/are-breaches-of-jwt-based-servers-more-damaging/)

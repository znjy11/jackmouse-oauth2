### oauth认证服务器搭建流程
- #### 1.导入依赖
``` xml
    <dependencies>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
        </dependency>
        <!--web模块-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <!--监测-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
<!--        auth2-->
        <dependency>
            <groupId>org.springframework.security.oauth.boot</groupId>
            <artifactId>spring-security-oauth2-autoconfigure</artifactId>
            <version>2.6.3</version>
        </dependency>
<!--        JWS JWE 的实现-->
        <dependency>
            <groupId>org.springframework.security</groupId>
            <artifactId>spring-security-oauth2-jose</artifactId>
        </dependency>
<!--        jdbc-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-jdbc</artifactId>
        </dependency>
<!--        mysql-->
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
        </dependency>
<!--        nacos服务注册-->
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
            <version>2021.1</version>
        </dependency>
<!--        服务追踪-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-sleuth</artifactId>
            <version>3.1.3</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-sleuth-zipkin</artifactId>
            <version>3.1.3</version>
        </dependency>


<!--        测试依赖-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.security</groupId>
            <artifactId>spring-security-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
``` 
- #### 2.配置文件
```yaml
# 应用名称
server:
  port: 10000
spring:
  application:
    name: oauth2-auth-server
  #配置数据源，方便单个服务测试使用，待启动多服务调用其他服务获得数据
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://127.0.0.1:3306/mook?useUnicode=true&&characterEncoding=utf-8&&serverTimezone=Asia/Shanghai
    username: root
    password: 123456
  zipkin:
    base-url: http://zipkin:9411/
```
- #### 3.启动类
```java
/**
 * 开启oauth认证服务器
 * @author zhoujiaangyao
 */
@SpringBootApplication
@EnableAuthorizationServer
public class Oauth2AuthServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(Oauth2AuthServerApplication.class, args);
    }

}
```
- #### 4.WebSecurityConfig类配置
```java
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    /**
     * 配置登录系统的用户信息（内存实现）
     * @return 用户信息
     */
    @Bean
    @Override
    public UserDetailsService userDetailsService() {
        return new InMemoryUserDetailsManager(
                User.withDefaultPasswordEncoder()
                        .username("jackmouse")
                        .password("123456")
                        .roles("USER")
                        .build());
    }

    /**
     * 配置http请求访问权限
     * @param http http
     * @throws Exception Exception
     */
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
                .authorizeRequests()
                //基于jws的认证，放行请求jws数据的请求
                .mvcMatchers("/.well-known/jwks.json").permitAll()
                //任何请求都需要认证
                .anyRequest().authenticated()
                .and()
                //开启httpBasic认证方式
                .httpBasic()
                .and()
                //关闭csrf
                .csrf()
//                .ignoringRequestMatchers((request) -> "/introspect".equals(request.getRequestURI()))
                .disable();
    }

}

```
- #### 5.认证服务器AuthorizationServerConfig类配置
```java
@Configuration
public class AuthorizationServerConfig extends AuthorizationServerConfigurerAdapter {
    /**
     * 密码加密
     * @return PasswordEncoder
     */
    @Bean
    PasswordEncoder passwordEncoder() {
        return PasswordEncoderFactories.createDelegatingPasswordEncoder();
    }

    /**
     * 配置客户端信息（内存实现）
     * @param clients 客户端
     * @throws Exception Exception
     */
    @Override
    public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
        clients.inMemory()
                //客户端名称
                .withClient("first-client")
                //客户端密码
                .secret(passwordEncoder().encode("noonewilleverguess"))
                //权限
                .scopes("resource:read")
                //支持认证的方式
                .authorizedGrantTypes("authorization_code", "password")
                //重定向url
                .redirectUris("http://localhost:19021/login/oauth2/code/goodskill",
                        "http://www.goodskill.com:8080/login/oauth2/code/goodskill")
                .and()
                .withClient("second-client")
                .secret(passwordEncoder().encode("noonewilleverguess"))
                .scopes("resource:read")
                .authorizedGrantTypes("authorization_code", "password")
                .redirectUris(
                        "http://www.goodskill.com:8080/login/oauth2/code/goodskill",
                        "http://localhost:19021/login/oauth2/code/goodskill"
                )
        ;
    }
    
}

```
- #### 5.JWT配置类JwkSetConfiguration配置
```java
@Configuration
public class JwkSetConfiguration extends AuthorizationServerConfigurerAdapter {

    AuthenticationManager authenticationManager;
    KeyPair keyPair;
    UserDetailsService userDetailsService;

    public JwkSetConfiguration(AuthenticationConfiguration authenticationConfiguration,
                               KeyPair keyPair, UserDetailsService userDetailsService) throws Exception {

        this.authenticationManager = authenticationConfiguration.getAuthenticationManager();
        this.keyPair = keyPair;
        this.userDetailsService = userDetailsService;
    }

    /**
     * jwt_token接入点
     * @param endpoints endpoints
     */
    @Override
    public void configure(AuthorizationServerEndpointsConfigurer endpoints) {
        // @formatter:off
        TokenEnhancerChain enhancerChain = new TokenEnhancerChain();
        List<TokenEnhancer> delegates = new ArrayList<>();
        delegates.add(tokenEnhancer());
        delegates.add(accessTokenConverter());
        //配置JWT的内容增强器
        enhancerChain.setTokenEnhancers(delegates); 
        endpoints
                //配置管理器
                .authenticationManager(authenticationManager)
                //设置用户
                //配置加载用户信息的服务
                .userDetailsService(userDetailsService) 
                //设置token
                .accessTokenConverter(accessTokenConverter())
                //设置增强token
                .tokenEnhancer(enhancerChain);
        // @formatter:on
    }

    /**
     * token增强
     * @return TokenEnhancer
     */
    @Bean
    public TokenEnhancer tokenEnhancer() {
        return (accessToken, authentication) -> {
            User securityUser = (User) authentication.getPrincipal();
            Map<String, Object> info = new HashMap<>(8);
            //把用户ID设置到JWT中
            info.put("name", securityUser.getUsername());
            ((DefaultOAuth2AccessToken) accessToken).setAdditionalInformation(info);
            return accessToken;
        };
    }

    /**
     * token存储方式 JWT
     * @return TokenStore
     */
    @Bean
    public TokenStore tokenStore() {
        return new JwtTokenStore(accessTokenConverter());
    }

    @Bean
    public JwtAccessTokenConverter accessTokenConverter() {
        JwtAccessTokenConverter converter = new JwtAccessTokenConverter();
        converter.setKeyPair(this.keyPair);
        return converter;
    }
    
}
```
- #### 6.JwkSetEndpoint配置
```java
@FrameworkEndpoint
public class JwkSetEndpoint {
    KeyPair keyPair;

    public JwkSetEndpoint(KeyPair keyPair) {
        this.keyPair = keyPair;
    }
    @GetMapping("/.well-known/jwks.json")
    @ResponseBody
    public Map<String, Object> getKey(Principal principal) {
        RSAPublicKey publicKey = (RSAPublicKey) this.keyPair.getPublic();
        RSAKey key = new RSAKey.Builder(publicKey).build();
        return new JWKSet(key).toJSONObject();
    }
}
```
- #### 7.KeyConfig配置
```java
public class KeyConfig {
    @Bean
    KeyPair keyPair() {
        try {
            String privateExponent = "3851612021791312596791631935569878540203393691253311342052463788814433805390794604753109719790052408607029530149004451377846406736413270923596916756321977922303381344613407820854322190592787335193581632323728135479679928871596911841005827348430783250026013354350760878678723915119966019947072651782000702927096735228356171563532131162414366310012554312756036441054404004920678199077822575051043273088621405687950081861819700809912238863867947415641838115425624808671834312114785499017269379478439158796130804789241476050832773822038351367878951389438751088021113551495469440016698505614123035099067172660197922333993";
            String modulus = "18044398961479537755088511127417480155072543594514852056908450877656126120801808993616738273349107491806340290040410660515399239279742407357192875363433659810851147557504389760192273458065587503508596714389889971758652047927503525007076910925306186421971180013159326306810174367375596043267660331677530921991343349336096643043840224352451615452251387611820750171352353189973315443889352557807329336576421211370350554195530374360110583327093711721857129170040527236951522127488980970085401773781530555922385755722534685479501240842392531455355164896023070459024737908929308707435474197069199421373363801477026083786683";
            String exponent = "65537";

            RSAPublicKeySpec publicSpec = new RSAPublicKeySpec(new BigInteger(modulus), new BigInteger(exponent));
            RSAPrivateKeySpec privateSpec = new RSAPrivateKeySpec(new BigInteger(modulus), new BigInteger(privateExponent));
            KeyFactory factory = KeyFactory.getInstance("RSA");
            return new KeyPair(factory.generatePublic(publicSpec), factory.generatePrivate(privateSpec));
        } catch ( Exception e ) {
            throw new IllegalArgumentException(e);
        }
    }
}
```
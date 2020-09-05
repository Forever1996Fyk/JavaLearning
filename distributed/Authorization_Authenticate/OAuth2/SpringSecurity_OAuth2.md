## <center>Spring Security OAuth2</center>

我们知道`OAuth2`是一个关于授权的开放标准协议, 通过各种认证手段认证用户身份, 并颁发token(令牌), 使得第三方应用可以使用该令牌在**限定时间, 限定范围**访问指定资源。

`OAuth2.0`中的重要概念:

- `resource owner`: 拥有被访问资源的用户
- `user-agent`: 一般是浏览器
- `client`: 第三方应用
- `Authorization Server`: 认证授权服务器, 用来进行用户认证并颁发token
- `Resource Server`: 资源服务器, 拥有被访问资源的服务器, 需要通过token来确定是否拥有权限访问

`Spring Security`是一套安全框架, 基于RBAC(基于角色的权限控制)对用户的访问权限进行控制, 核心思想是通过一系列的`filter chain`来进行拦截过滤。其中最核心的是`filter_security_interceptor`, 通过`FilterInvocationSecurityMethodadaSource`来进行资源权限的匹配, `AccessDecisionManager`来执行访问策略。


### 1. Spring Cloud Security OAuth2认证流程

根据上面的解释, 将`OAuth2`与`Spring Security`集成, 就可以得到一套完整的安全解决方案。

具体的流程如下:

1. 用户登录A网站, 准备访问B网站的资源
2. 但是B网站的资源是受保护的, 于是返回302将用户重定向到了B网站的登录页面
3. 用户完成认证后, B网站提示用户是否将资源授权给A网站使用
4. 用户确认后, A网站通过`授权码模式`获取了B网站颁发的`access_token`
5. A网站携带该token访问B网站的获取用户的接口(`https://api.b.com/user`), B网站网站验证token正确后返回了与该token绑定的用户信息
6. A网站的`Spring Security`框架根据返回的用户信息构造了`principal`对象并保存在服务器中
7. A网站再次携带该token访问B网站的资源, B网站根据**该token对应的用户**返回该用户的信息资源
8. 该用户后续只要在token有效期以及权限范围内均可正常获取

上面的流程中, A网站就是第三方应用(client), A网站及充当了认证服务器, 又充当了资源服务器。但是有几点需要注意的是:

**在标准的`OAuth2`授权过程中, 5，6，8这几部都不是必须的, 只要有1,2,3,4,7就完成了授权的整个过程。因为`OAuth2`只关心token的颁发, token的续签, token访问被保护的资源。那么为什么`Spring Security`还要做5,6两步呢?**

那是因为`Spring Security`是一套完整的安全框架, 它必须关心用户身份!在实际的使用场景中, `OAuth2`不仅仅用来进行被保护资源的访问, 还会被用来做单点登录(SSO)。在SSO的场景中, 用户身份就是核心, 而token本身是不携带用户信息的, 这样clinet就无法知道认证服务器发的token到底对应哪个用户。

> `OAuth2`, `Spring Security`, `JWT`到底是扮演什么角色? 它们之间的关系到底是什么? 为什么要集成这三者?

首先我个人的理解就是, `OAuth2`只是作为认证服务, `Spring Security`作为资源服务。

`OAuth2`并不在意用户到底能不能访问某个资源(或者是某个api), 它只是用来认证和生成token的。也就是说, 用户在访问A网站时, 只要token不存在或者token解析有问题, 就直接跳转到登录页, 进行`OAuth2`的认证, 然后生成token返回。这样的过程仅仅只是认证, 也就是说用户可以进入这个A系统而已, 至于A系统中的资源或者功能是否能访问, `OAuth2`是无法控制的。这就需要`Spring Security`来进行资源访问的控制。

`Spring Security`是真正的对用户的资源进行控制, 也就是这个用户能否访问这个api是`Spring Security`来控制的。所以`Spring Security`必须要知道用户信息, 而本身`OAuth2`的令牌token是不具有携带用户信息的功能, 唯一的方案就是将令牌跟用户信息存入服务器中, 但是这样每次资源服务都需要从服务器中获取用户信息, 对服务器压力会增大, 于是出现了`JWT`。

`JWT`所生产的token, 是可以携带用户信息的, 而且是通过签名加密的, 所以正好可以用作携带用户信息的令牌, 进行传递。

所以用户访问系统A, 整体的框架用途为:

1. 用户访问系统A, 经过请求过滤器发现没有携带token, 或者token失效, 解析失败时, 返回302状态, 前端判断该状态跳转到登录认证页面。
2. 用户输入账号密码进行登录, 这时通过`OAuth2`认证协议(clientId, secret等参数)并且查询用户账号和密码是否正确, 返回用户信息(基本用户信息, 用户的权限列表), 并使用`JWT`将用户信息加密保存在token中, 并返回前端200状态码和token。
3. 前端根据状态码200, 判断认证成功, 跳转到系统A首页, 并将token存入cookie或者localStorage中。
4. 访问首页时, 根据api网关转发到对应的资源服务并携带token, 资源服务解析token, 拿到用户信息构造`Spring Security`的`principal`对象中(目的是为了方便获取当前登录人的信息)。获取用户信息中的权限列表, 判断用户哪些权限是可用的。
5. 根据权限的判断, 最终决定用户是否能够访问系统A中的资源

### 2. OAuth2 与 SSO

首先要注意的是的`OAuth2`并不是一个SSO框架, 但是可以实现SSO功能。

**通过将用户信息这个资源设置为被保护资源, 可以使用`OAuth2`技术实现单点登录(SSO), 而`Spring Security OAuth2`就是这种OAuth2 SSO方案的一个实现。**

根据上面的解释, 其实`OAuth2`认证完成之后, 就完成了SSO的登录认证过程, 后续是用`Spring Security`去进行资源的保护。如果不用`JWT`的话, `Spring Security`需要从`principal`中取出这个用户的token, 再去访问资源服务器, 不需要每次去进行用户的认证授权。而且**此时浏览器与client之间仍然是通过传统的cookie-session机制来保持会话, 而非通过token。实际上在SSO的过程中, 使用到token的访问只要client与resources server之间获取user信息的那一次, token的信息是保存在client的session中的, 而不是用户本地。**

#### 2.1 OAuth2 SSO 与CAS, SAML2的比较

在CAS和SAML2中, 没有资源服务器的概念, 只有认证客户端和认证服务器。因为CAS与SAML2是专门为SSO设计的, 在令牌的结构上都考虑到了用户信息的问题(SAML2规范还携带了权限信息), 而`OAuth2`本身不是专门为SSO设计的, 主要是为了解决资源第三方授权访问的问题, 所以在用户信息方面, 还需要提供额外的接口。

#### 2.2 OAuth2 SSO的优势

从上面的流程可以看出, 通过`OAuth2`进行SSO认证, 有一个好处就是做到了**认证与授权的解耦**。在平常的开发中, 认证时必要容易做到统一个抽象的, 但是不同系统中的角色, 确实千差万别。同时角色的设计又是与资源服务器的设计强相关的。所以资源服务单独的配置, 是非常方便的。为权限的控制带来了更大的灵活性。


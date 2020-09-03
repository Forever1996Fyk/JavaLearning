## <center>认证授权的基本认识</center>

### 1. 认证(Authenticate)和授权(Authorization)的区别是什么?

简单通俗的来说就是:

- **Authenticate(认证)**: 验证用户身份的凭证(例如用户名/用户ID和密码), 通过这个凭证, 系统得以知道这是用户, 也就是说系统存在这个用户。所以, Authenticate又被称为身份/用户验证。
- **Authorization(授权)**: 发生在认证通过之后, 对系统的访问权限的控制。

### 2. 常见的认证授权方式

#### 2.1 Cookie & Session

Cookie与Session都是跟踪用户会话的方式, 只不过**Cookie是在客户端保存用户信息, Session在服务端保存用户信息。**

Cookie的经典案例:

1. 我们在Cookie中保存我们已经登录过的信息, 下次访问网站就可以自动填充登录的信息
2. 使用Cookie保存Session或Token, 向后端发送请求的时候带上 Cookie，这样后端就能取到session或者token了。
3. Cookie通常还可以用来分析和记录用户行为。举个简单的例子你在网上购物的时候，因为HTTP协议是没有状态的，如果服务器想要获取你在某个页面的停留状态或者看了哪些商品，一种常用的实现方式就是将这些信息存放在Cookie

##### 2.1.1 如何用Session进行身份认证?

大部分情况下我们通过SessionID来实现特定的用户, SessionID一般存放在Redis中。例如: 用户成功登录系统, 然后返回给客户端具有SessionID的Cookie, 当用户向后端发起请求的时候会把SessionID带上, 在根据SessionID去Redis中查询身份状态。

##### 2.1.2 没有Cookie的话, Session还能用吗?

一般通过Cookie来保存SessionID, 假如使用Cookie保存SessionID的话, 如果客户端禁用了Cookie, 那么Session就无法正常工作。

但是, 你也可以把SessionID放在请求url或者请求头中, `https://api.com?session_id=xxx`。这种方案可行, 但是安全性和用户体验感降低。

##### 2.1.3 为什么Cookie无法防止CSRF攻击, 而Token可以?

**CSRF(Cross Site Request Forgery) 跨站请求伪造**。简单来说就是通过获取你的身份信息去发送一些对你不好的请求。

之前说过, 进行Session认证时, 我们用Cookie来存储SessionID。但是如果别人通过Cookie拿到了SessionID后就可以代替你的身份访问系统了。

使用Token时, 我们一般讲TOken存放在local storage中。然后通过请求头给每个请求都加上这个token, 这样就不会出现CSRF问题。因为即使你点击了非法链接发送了请求到服务端, 这个请求也不会携带Token。

要注意的时无论Cookie还是Token都无法避免跨站脚本攻击(Cross Site Scrptiing) XSS。

#### 2.2 JWT 

上面已经讨论了Cookie和Session，但是都需要依赖Cookie，不适合移动端, 而且需要保证保存Session信息服务器的可用性。

**JWT(JSON Web Token)**是一种无状态的认证方式, 不需要在服务端保存Session数据, 只需要在客户端保存服务端返回的Token即可。

**JWT本质上就一段带有签名的JSON格式的数据, 由于带有签名所以接受者可以验证它的真实性。**

JWT由三部分构成:

1. Header: 描述JWT的元数据, 定义了生成签名的算法以及Token类型。
2. Payload(负载): 用来存放实际需要传递的数据
3. Signature(签名): 服务器通过`Payload`, `Header`和一个密钥(`secret`)使用Header中指定的签名算法(默认是HMAC SHA256)生成。

所以JWT的过程为, 服务器通过`Payload`, `Header`和密钥(`secret`)创建令牌(`Tokne`)并将`Token`返回客户端, 客户端将`Token`保存在Cookie或者localStorage中, 每次发送请求将`Token`放入`HTTP Header`的`Authorization`字段中: `Authorization: Bearer Token`。不要使用Cookie自动发送, 因为这样就不能跨域了。

#### 2.3 OAuth2.0

OAuth是行业的标准授权协议, 主要用来授权第三方应用获取有限的权限。OAuth1.0已经被废弃了。

实际上它就是一种授权机制, 最终目的是为第三方应用产生一个有时效性的令牌Token, 使第三方应用能够通过该令牌获取相关的资源。

 > SSO与OAuth2.0的区别?

 OAuth是一个标准的授权协议, 而SSO解决多个系统之间的登录认证问题。

 ### 3. 重点的认证授权框架

在之后的章节中, 将深入研究以下几个框架:

- `Spring Security` & `Spring Security OAuth2`
- `Shiro`
- `JWT`
- 分布式认证授权解决方案`Spring Cloud Security OAuth2` & `JWT`整合


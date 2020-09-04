## <center>深入理解OAuth 2.0</center>

`OAuth 2.0`是目前最流行的授权机制, 用来授权第三方应用, 获取用户数据。

> 简单来说: OAuth就是一种授权机制。数据的所有者告诉系统, 同意授权第三方应用进入系统, 获取这些数据。系统会产生一个短期的进入令牌(token), 用来代替密码, 供第三方应用使用。

### 1. 令牌与密码

令牌(token)与密码(password)的作用是一样的, 都可以进入系统, 但是这两种是有差异的:

- 令牌是短期的, 到期会自动失效, 用户自己无法修改。密码一般长期有效, 用户不修改, 就不会发生变化。
- 令牌可以被数据所有者撤销, 会立即失效。而密码一般不允许被他人撤销
- 令牌有权限范围(scope), 对于互联网服务来说, 只读令牌比读写令牌更安全。密码一般是完整权限。

注意: 只要知道了令牌就能进入系统, 系统一般不会再次确认身份, 所以**令牌必须保密, 泄露令牌与泄露密码的后果一样, 这也就是为什么令牌的有效期, 一般都设置很短的原因。**

### 2. 授权方式

我们知道`OAuth`的核心就是向第三方办法令牌, 这个令牌相当于授权给第三方应用。

- **授权码(`authorization code`)**
- **隐藏式(`implicit`)**
- **密码式(`password`)**
- **客户端凭证(`client credentials`)**

不管哪一种授权方式, 第三方应用申请令牌之前都必须要有系统的标识, 说明自己的身份, 然后第三方应用会拿到两个身份识别码: **客户端ID(`clientId`)**和**客户端密钥(`client secret`)**。这是为了防止令牌被滥用, 没有这两个标识的第三方应用, 无法拿到令牌。

#### 2.1 授权码(authorization code)

**授权码(`authorization code`)方式, 是第三方应用先申请一个授权码, 然后再用该码获取令牌。**

这种方式是最常用的流程, 安全性最高, 适用于有后端的Web的应用。授权码通过前端传送, 令牌则是存储在后端, 而且所有与资源服务器通信都在后端完成。装前后端分离, 可以避免令牌泄露。

1. A网站提供一个链接, 用户点击就会跳转到B网站, 授权用户数据给A网站使用。例如: `https://b.com/oauth/authorize?response_type=codeclient_id=CLIENT_IDredirect_uri=CALLBACK_URL&scope=read`

    `response_type`参数标识要求返回授权码类型, `client_id`参数是让B知道是谁在请求, `redirect_uri`参数是B接受或拒绝请求后的跳转网址, `scope`参数表示授权的范围(这里是只读)。

2. 用户跳转后, B网站会要求用户登录, 然后询问是否同意给予A网站授权。用户同意, B网站就会跳回`redirect_uri`参数指定的网址。跳转时, 会传回一个授权码。`https://a.com/callback?code=AUTHORIZATION_CODE`

    `code`参数就是授权码

3. A网站拿到授权码, 就可以在后端向B网站请求令牌。`https://b.com/oauth/token?client_id=CLIENT_ID&client_secret=CLIENT_SECRET&grant_type=authorization_code&code=AUTHORIZATION_CODE&redirect_uri=CALLBACK_URL`

    上面的URL中, `client_id`参数和`client_secret`参数用来让B确认A的身份(`client_secret`参数是保密的,所以只能在后端发送), `grant_type`参数的值是`authorization_code`, 表示采用的授权方式是授权码,`code`参数就是第2步获取到的授权码, `redirect_uri`参数是令牌颁发后的回调网址。

4. B网站收到请求后, 就会发送令牌。具体是向`redirect_uri`指定的网址, 发送一段JSON数据。

    ```json
    {
        "access_token":"token",
        "token_type":"bearer",
        "expires_in":"259200",
        "refresh_token":"refresh_token",
        "scope":"read",
        "uid":10001
        "info":{...}
    }
    ```

#### 2.2 隐藏式(implicit)

这种方式是针对Web应用是纯前端应用, 没有后端。**允许直接向前端颁发令牌。这种方式没有授权码这个步骤, 所以称为"隐藏式"**。

这里就不详细描述这种方式了。

#### 2.3 密码式(password)

**如果你高度信任某个应用, 用户可以把用户名和密码, 直接告诉该应用。该应用就是使用你的密码, 申请令牌, 这种方式称为"密码式"(password)**。

1. A网站要求用户提供B网站的用户名和密码。A就直接向B请求令牌。`https://oauth.b.com/token?grant_type=password&username=USERNAME&password=PASSWORD&client_id=CLIENT_ID`。

    上面的URL, `grant_type`参数是授权方式密码式(`password`), `username`和`password`是B的用户名和密码。

2.  B网站验证身份通过后, 直接给出令牌。无须跳转, 直接把令牌放在JSON数据中, 作为HTTP响应。

这种方式需要用户给出自己的用户名/密码, 显然风险很大，因此只适用于其他授权方式都无法采用的情况，而且必须是用户高度信任的应用。

#### 2.4 客户端凭证(client credentials)

这种方式适用于没有前端的命令行应用, 即在命令行下请求令牌。

1. A应用在命令行向B发出请求`https://oauth.b.com/token?grant_type=client_credentials&client_id=CLIENT_ID&client_secret=CLIENT_SECRET`

    上面的URL中, `grant_type`参数等于`client_credentials`表示客户端凭证式, `client_id`和`client_secret`用来让B确认A的身份。

2. B网站验证通过后, 直接返回令牌

这种方式给出的令牌, 是针对第三方应用的, 而不是针对用户, 即有可能多个用户共享同一个令牌。

### 3. 令牌(token)的使用

A网站拿到令牌后, 就可以向B网站的API请求数据了。此时每发送API请求, 都必须带有令牌。在请求头信息上, 加上一个 **Authorization**字段。

### 4. 更新令牌

令牌的有效期到了, 如果让用户重新走一遍上面的流程, 在申请一个新的令牌, 用户体验非常不好, 而且没必要。`OAuth 2.0`允许用户自动更新令牌。

B网站颁发令牌时, 一次性颁发两个令牌, 一个用于获取数据, 另一个用户获取新的令牌(refresh_token字段)。令牌到期前, 用户使用refresh token发一个请求去更新令牌。
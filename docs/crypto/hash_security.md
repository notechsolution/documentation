# 一文详解密码学Hash算法的安全问题(加盐+HMAC)

上一篇文章- [一文详解密码学中的Hash 算法](./hash.md), 我们介绍了密码学中Hash算法的性质、分类以及使用场景。在使用场景中，我们介绍了这些场景的具体使用方式方法。为了保持文章的连贯性和可读性， 对于其中的潜在安全问题，我们简单一笔带过。今天我们另开一篇文章，着重介绍密码学Hash算法的主要安全问题以及对应的解决办法。希望能大家使用密码学Hash算法带来更多维度的考量。

## 密码学Hash算法作为身份验证的安全问题

上一篇文章我们介绍了怎样使用密码学Hash算法的单向特性来作为用户身份验证的方案：

> 密码学Hash算法必须具备有单向性，也就是不可逆，不能从Hash值逆向推算出原始值，基于这种特性，我们可以用来进行用户身份验证。用户登录都需要进行用户名密码的校验，如果我们将用户密码存在数据库里面，一旦数据库泄露，那就所有人的密码都泄露了，这样的事情前几年发生了不少！那么如果我们将用户的原始密码进行Hash运算，并只是把Hash值保存在数据库里面，当用户登录时，我们计算用户输入密码的Hash值，并与数据库的Hash值进行比较，如果相同则验证通过，不同则失败。而用户的明文密码不进行保存，这样一来，万一数据库泄露了，也不会一下子泄露了全部的用户密码

简单地说 用户的密码在系统中变成了 f(p) = h(p), 而验证的时候， 计算h(p') = h(p), 其中p为用户设置的密码，p'为登录时用户输入的密码，这样系统就可以不用明文保存用户的密码。比如常见密码`password1` 用SHA256保存在数据库就变成了`0b14d501a594442a01c6859541bcb3e8164d183d32937b851835442f69d5c94e` , 登录时系统校验 SHA(input)是否等于这个Hash值，相同则通过验证。

感觉这样很可行，就连系统管理员直接查DB也不知道你的密码是什么，因为目前常用的Hash算法不可逆的特性都还没被打破，也就是基本没办法从Hash值反推出原始消息。

## 重要声明

我们这里将会根据计算机网络安全所遇到的问题进行介绍，目的是提高我们开发人员的密码安全意识，提供系统的攻防能力！一定要在合法的前提下进行网络安全的攻防演练！

### 彩虹表 

#### 什么是彩虹表

道高一尺魔高一丈！反推不行，那我就干脆“正推”。上一篇文章我们介绍Hash算法的时候有一个特性`相同的输入消息总是能得到相同的Hash值`. 那如果有一个Hash值的表，里面包含了原文以及Hash值，如果我的Hash值跟你系统的Hash值一样，那我就可以知道你的密码是什么了！！！比如你的密码是`password1`, 然后数据库存的是SHA的值`0b14d501a594442a01c6859541bcb3e8164d183d32937b851835442f69d5c94e`,  这时如果攻击者的密码表中 有如下数据：

| sequence | hash                                                         | password        |
| -------- | ------------------------------------------------------------ | --------------- |
| 1        | e5e8b2d214db8f3689be77f6fde9b64164b3e792efb329e9a9b53993055d6c8e | strongPassword1 |
| 2        | 0b14d501a594442a01c6859541bcb3e8164d183d32937b851835442f69d5c94e | password1       |



经过匹配，很快就可以知道第2条数据的hash值跟你密码的hash值是一样的，那么他就可以推断出你的密码是`password1`, 因为Hash值相同。 这个表就叫做`彩虹表(rainbow table)` , 彩虹表都是很大的，才能更快更准地 *破解* 搞到的密码库。一般的彩虹表都要100GB以上的, 而像 [CMD5](https://www.cmd5.com) 这种提供公开Hash算法的反向查询（就是你给个Hash值，它能查出原文是什么），他需要的彩虹表则是更大了，声称有90万亿条，数据库的大小都有500TB了: 

>  本站针对md5、sha1等全球通用公开的加密算法进行反向查询，通过穷举字符组合的方式，创建了明文密文对应查询数据库，创建的记录约90万亿条，占用硬盘超过500TB，查询成功率95%以上，很多复杂密文只有本站才可查询。自2006年已稳定运行十余年，国内外享有盛誉。

假设我们的密码是 "strongpassword" 的SHA256为`05926fd3e6ec8c13c5da5205b546037bdcf861528e0bdb22e9cece29e567a1bc`, 在CMD5反查的结果为:

![](D:\workspace\github\documentation\docs\crypto\image\hashSecurity01.png)

当然，为了生存，免费的账户只能查询简单的原文组成规则，复杂的，比如大小写+字母+数字等，则要收费



#### 彩虹表从哪来的

比较合理合法的做法是自己穷举字符组合，然后算出各种组合在各种Hash算法下的值。也有人提供了彩虹表生成工具，比如`RainbowCrack` , 它可以根据你要的密码复杂度，生成对应的彩虹表。

还有一种情况，黑客通过SQL注入等手段，侵入到目标网站去，然后把用户密码表给导出来，这种行为叫做 “脱库”。万一这个网站心比较大，密码都是明文保存的，那就公共事件了，直接密码泄露了。这时就会有人根据这些`真实密码` 去积累彩虹表的数据了。 所以大家一定要给用户密码做好Hash等密码保护手段，同时要做好SQL注入等安全问题



#### 彩虹表怎么用

"脱库" 这个黑客用到的术语，而这个术语一般会伴随着 "撞库" 而来。脱库积累了彩虹表的数据，那么下一步就要根据已推测出来的密码，逐一到目标网站去尝试登陆，登陆成功就得到验证过的密码了，这个过程就叫 "撞库".  "撞库" 行为是用户验证模块的开发人员要特别注意的，我们需要对这种行为作出各种防护处理。比如常见的如果密码错误5次就要等几分钟再可以重试，或者像银行这种，试错3次就只能第二天才能尝试，再错几次可能就要去柜台了。这种就极大地阻止了撞库的成功率，降低了他们的投入产出比，从而放弃啃这块硬骨头。 



## Hash加盐

看到上面可能的安全问题，难道我们就不能用Hash进行密码保存来作为身份验证码？也是可以的! 只不过再多做一点事情让它更安全一点就行。

### 静态盐

既然常见的密码hash很容易被推算出来，那么我可以在原来密码的基础上，加上一些其它字符串再计算Hash，当验证的时候我也在用户输入的密码加上对应的字符串再进行验证，这样别人再反推就很难了。比如前面的`strongpassword` 一下就个CMD5查出来了，如果我混进一个随机字符串`ghuarxlheq`后面, 变成`strongpasswordghuarxlheq` , 这会计算出来的hash为`664d9d6484cd8988d7a3c313c03b3df148ad01dd7b6d434deb231e0b34a2b7bb`, 那么CMD5就查不出来了：

![](D:\workspace\github\documentation\docs\crypto\image\hashSecurity02.png)

使用静态盐有一个潜在问题，如果某一个密码碰巧被猜出来了，或者盐被泄露出来，而由于其它密码也使用了相同的盐，那就它们被攻破的可能性就很大。

### 随机盐

加盐确实能很大程度地避免彩虹表攻击，但是上面说到的静态盐确实存在着风险，有点像密码学中讲到的 `前向安全性` 。 如果每个密码都用随机生成的盐，那么即使某一个密码泄露，或者某一个盐泄露了，受影响也只是一个密码而已，其它密码基本都还安全。这样随机生成的盐就叫做 随机盐，其实就是一串随机字符。

### 盐的存储

既然每一密码都有一个独立的盐，那么这个盐该怎么存呢？ 如果条件允许，把盐也独立存放在一张表，这样即使密码表泄露了，盐表没泄露，那么基本还是安全的。但这样就随着带来了性能问题，毕竟每次身份验证都需要查询两张表。如果担心性能，可以把盐跟密码存在同一张表，这样身份验证的时候就只需要查询一张表，当然随着而来就是表泄露的时候同时泄露了密码hash跟盐值。不过话说回来，即使拿到hash以及盐，也彩虹表攻击的难度也增大了很多，你需要知道加盐的算法，然后把整个彩虹表重新算一遍hash值。当然，架构设计不就是各种平衡取舍嘛!  需要根据实际业务系统需求选择不同的密码保存方式。 

还有一种是不额外产生盐，而是用用户表中的已有字段，通过自己的算法产生一个盐，然后进行Hash计算。比如用 ID+Email的hash值作为盐值 等等。这些都可以自己很方便地实现!

### Java加盐Hash的实现

实现加盐Hash不是特别复杂，一般根据自己的加盐算法，把原文跟盐连在一起，再进行hash计算，比如下面的实现：

```java
 public static String sha256WithSalt(String content, String salt) throws NoSuchAlgorithmException {
    MessageDigest digest = MessageDigest.getInstance("SHA-256");
    String forHashContent = content + salt;
    digest.update(forHashContent.getBytes());
    return HexUtils.toHexString(digest.digest());
  }
```



## HMAC

### Hash一致性校验的问题

上一篇文章我们介绍了用Hash进行文件或者消息内容一致性的校验，在大多数情况都是没问题的，比如对于docker镜像文件的下载。但在一些特殊情况下，比如JWT 口令，这种需要设计用户的身份信息的场景下，如果有人恶意串改内容，比如把里面的用户权限从只读 改成读写来骗过验证系统，那么就会有风险。尤其是在现在的很多微服务架构中，很多设计都是以JWT进行统一身份认证。而为了提供性能，很多微服务在收到JWT口令时，不会去调用统一认证平台进行校验，而是选择自行校验这个口令的真实性，因为里面有Hash值。

![](D:\workspace\github\documentation\docs\crypto\image\hashSecurity03.png)

这种情况我们，我们就需要对Hash进行更为安全的处理，比如简单的HMAC。

### HMAC

HMAC - Hash-base Message Authentication Code, 基于Hash的消息验证码。从名字就可以看出它主要是用来验证消息的一致性以及真实性。HMAC算法的基本思想也跟加盐类似，只不过实现方式不同。加盐算是简单粗暴地把原文跟盐加在一起进行hash运算，而HMAC则是将密码key补位，然后与明文分组进行异或运算，并且将该输出与下一个分组进行异或运算，直到算出最后的Hash值。具体算法可以观摩一下Coursera课程用到的这张图：

![](D:\workspace\github\documentation\docs\crypto\image\hashSecurity04.png)



JWT的一般验证方式是 token统一认证平台将约定的key通知给各个微服务，大家在收到JWT之后，用这个key去计算JWT的Hash，再与JWT的hash值进行比较来实现消息验证。比如下图，我们用了mySSOCenter作为HMAC的key进行消息的Hash运算得到一个签名，同时也可以进行消息的验证：

![](D:\workspace\github\documentation\docs\crypto\image\hashSecurity05.png)



### Java HMAC的实现

JDK中自带了HMAC的实现，下面的代码就可以计算以及验证Hash了：

```java
  public static void main(String[] args) throws InvalidKeyException, NoSuchAlgorithmException {
    String content = "this is origin content";
    String secret = "rootPassword";
    String hmac = generateHmac(ALGORITHM_HmacSHA1, content, secret);
    System.out.println("hmac -> "+ hmac);
  }

  public static String generateHmac(String algorithm, String content, String key)
      throws NoSuchAlgorithmException, InvalidKeyException {
    SecretKeySpec secretKey = new SecretKeySpec(key.getBytes(), algorithm);
    Mac mac = Mac.getInstance(algorithm);
    mac.init(secretKey);
    byte[] hashByte = mac.doFinal(content.getBytes());
    return HexUtils.toHexString(hashByte);
  }
```



好了，关于密码学Hash算法的安全问题到此就告了一段落了，希望能有助于大家理解系统中密码保存的机制。





> 今天我们另开一篇文章，着重介绍密码学Hash算法的主要安全问题以及对应的解决办法。希望能大家使用密码学Hash算法带来更多维度的考量。内容将会涉及到彩虹表，撞库等黑客部分网络安全攻击，对于用加盐的方式来避免这种攻击，以及如何用HMAC来确保消息的一致性
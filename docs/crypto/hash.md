# 一文详解密码学中的Hash 算法 #

上一篇文章里面，我们介绍了随机数以及随机数中的应用，可以看到密码学中到处都有随机数的身影，这种作为大部分密码学算法的基本组成被称之为 “加密基元“。今天我们一起来看一下另外一个加密基元 - 密码学Hash算法



## 什么是密码学Hash算法

密码学Hash算法是一个非常重要，而且常见的算法，是计算机密码学中的核心组成部分。密码学Hash算法是指将任何长度的二进制值映射成较短的固定长度二进制值的算法，这个较短的固定长度二进制值就是Hash值。先说一下：这个表述其实不是特别严谨，“任意长度”其实应该是 “算法允许长度范围内的任意长度”，因为有些密码学Hash算法是有输入长度限制的。既然很长的输入可以变成很短的输出，这就像我们写文章之后，需要写一个摘要一样，所以Hash值很多时候也叫做 “消息摘要”， Java中计算密码学Hash值的基类更是直接叫 MessageDigest.

下面代码能够更直观一点，我们来看一些密码学Hash的例子：

```java
  public static void main(String[] args) throws NoSuchAlgorithmException {
    System.out.println(md5("abcd") + " <- abcd");
    System.out.println(md5("abcd") + " <- abcd");
    System.out.println(md5("明月几时有，把酒问青天，不知天上宫阙，今夕是何年") + " <- 明月几时有，把酒问青天，不知天上宫阙，今夕是何年");
    System.out.println(md5("1") + " <- 1 ");
  }

  public static String md5(String content) throws NoSuchAlgorithmException {
    MessageDigest instance = MessageDigest.getInstance("MD5");
    instance.update(content.getBytes());
    return HexUtils.toHexString(instance.digest());
  }
```

输出结果为:

```shell
e2fc714c4727ee9395f324cd2e7f331f <- abcd
e2fc714c4727ee9395f324cd2e7f331f <- abcd
06d9a9f655c1ac1f0391e7dacfac6cbb <- 明月几时有，把酒问青天，不知天上宫阙，今夕是何年
c4ca4238a0b923820dcc509a6f75849b <- 1 
```



## 密码学Hash算法的特性

密码学Hash算法有好几个特性：

- 相同的输入消息总是能得到相同的Hash值。给定的Hash算法，不管消息长度多少，最终的Hash值长度是相同的。上面的例子可以看到 输入"abcd" 的MD5 Hash值一直都是`e2fc714c4727ee9395f324cd2e7f331f`
- 不可逆. 不可逆也叫单向性 (pre-image resistance)。 很难通过Hash值反推出原始消息是什么！设想一下，要是可逆的话，那么我们将10GB的文件变成一个128位的hash值，然后进行传输，对方接到后进行逆运算就可以得到原来的文件。那么我们的网络2G就够了，不用去争取5G,6G的了，大华为最近也就不用成天被灭总欺负了!
- 很难冲突。很难找到两个不同的消息能够产生相同的Hash值。这里用"很难" 而不是直接写“无冲突”是为了稍微严谨一点，因为Hash算法MD5已经在实践中产生碰撞了, 也就是攻击者不断地运算，能够找到两个不同的消息，使用MD5算出来的Hash值是一样的。

## 密码学Hash算法的分类

密码学Hash算法大致可以分为两个类别：普通的密码学Hash算法以及安全的密码学Hash算法。 前面列的几个密码学Hash算法的特性是所有Hash算法都具备的，不管是普通的还是安全的。而安全的密码学Hash算法则多了如下几个特性：

-  强抗碰撞 Collision Resistance

  如果两个不同的原文能产生相同的Hash，这个就是产生了Hash碰撞。而如果能随机地找到这两个原文M1, M2，使得h(M1) = h(M2), 那么这个就是强碰撞。而能抵抗这种碰撞的特性就叫 强抗碰撞。

- 弱抗碰撞 Second pre-image Resistance

  如果给定消息M1，能够找到M2，使得h(M1) = h(M2)，这个就是弱碰撞。而能抵抗这种碰撞的特性叫弱抗碰撞。

对于攻击者来说，Hash算法的破解难度为  强抗碰撞 < 弱抗碰撞 < 单向性。也就是说 首先破解的是 强抗碰撞，随便找，只要能找到两个不同的输入有相同的输出就算 破解了。这里的`破解` 也分为理论破解和实际破解， 一个Hash算法理论上产生了破解，不代表在实际使用场景中不能使用该算法，因为现实世界发生碰撞的可能性还是很大的。再退一步说，即使发生了实际破解，也不代表不能用，比如MD5 早就理论跟实际都破解了，但你如果只是用于内部系统进行消息完整性地校验，那应该可以大胆地用。

接下来我们来看几种常见的Hash算法：

### MD5

MD5 - Message Digest #5 是MIT教授1992年公开一种Hash算法，它接收任意长度的输入，输出一个128位的Hash值。这个算法其实挺佛性的：不管你来多大的输入，1位也行，100GB也行，我最终的输出有且仅有128位。比如前面数字 `1` 的MD5 hash值是`c4ca4238a0b923820dcc509a6f75849b <- 1 ` , 由32个十六进制的字符，所以总输出位数为 32 x 4 =128位。2004年，这个算法被中科院王小云院士证明了可以产生碰撞。所以在一些安全要求比较高的场合下，慢慢地不再用MD5算法了。但是MD5还是出现在很多实现当中，因为它虽然打破了强碰撞性，但是单一性还是有的，而且它的输出是常见的Hash算法里面最短的，只有128位。

### SHA

安全的密码学算法SHA - Secured Hash Algorithm， 是一组密码学Hash算法的总称。由美国国家标准与技术研究院(NIST)指定的算法。主要有3类算法： SHA-1, SHA2, SHA-3.

- SHA-1

  SHA-1类似于MD5的算法， 输出的长度固定位160位。还是前面的数字`1` ， 其SHA-1的输出为40个十六进制的字符  `356a192b7913b04c54574d18c28d46e6395428ab ` . 跟MD5一样，SHA-1也是被王小云院士在2005年证明为不安全，在实践中产生了碰撞。但还是那句话，实践中被证实发生碰撞，不代表不能用。比如很多文件下载的都提供一个SHA-1的Hash值供用户进行校验。

- SHA-2

  SHA-2可以说是SHA-1的升级版，Hash的构造跟实现也不同。SHA-1是固定160位，而SHA-2 可以是224， 384， 256， 512位，分别对应的算法名字是 SHA-224, SHA-256, SHA-384, SHA-512. 简单的不同就是位数越多，其产生碰撞的概率就越低，当然也运算需要的时间也相应的就长一些。SHA-2截止目前安全的，其中SHA-256是数字证书SSL的标准hash算法，打开带有www.bing.com, 我们打开证书就能看到具体的签名hash算法是SHA256。 所以如果没有特殊的要求，建议采用SHA256作为密码学Hash算法的首选。

  ![](D:\workspace\github\documentation\docs\crypto\image\hash01.png)

  

- SHA-3

  SHA-2 和SHA-1使用相同的处理引擎， 所以理论上都存在碰撞的风险.于是NIST提出了设计新的安全Hash算法的竞赛，号召大家去设计新的安全Hash算法， 命名为SHA-3。这些算法需要满足如下4个要求:

  - Hash函数需要容易实现
  - 必须保守安全，能抵抗已知的碰撞攻击。要和SHA-2一样，有4个相同的Hash大小(224位，256位，384位，512位)
  - Hash算法必须接收密码分析，源代码和分析结果要公开给第三方审查
  - 算法实现多样性

  最终在2008年，Keccak算法（读作为“ket-chak”）从51个候选者中脱颖而出，成为SHA-3的标准算法。现在很多产品应用已经开始使用SHA-3作为Hash算法，比如区块链实现 以太坊Ethereum的基础Hash算法就是SHA-3.

  SHA3对应的4种位数的算法名字为： SHA3-224, SHA3-256, SHA3-384, SHA3-512.

## Java 中密码学Hash算法的使用

Java的SDK中已经内置了MD5， SHA1, SHA2的Hash算法，我们可以很方便地使用这些算法. 而SHA-3不包含在JDK里面，需要在maven里面引入Bouncy Castle的包

```xml
<dependency>
    <groupId>org.bouncycastle</groupId>
    <artifactId>bcprov-jdk15on</artifactId>
    <version>1.56</version>
</dependency>
```

```java
public class HashUtil {

  public static void main(String[] args) throws NoSuchAlgorithmException {
    System.out.println("abcd -> "+md5("abcd"));
    System.out.println("abcd -> "+sha1("abcd"));
    System.out.println("abcd -> "+sha2("abcd"));
    System.out.println("abcd -> "+sha3("abcd"));
  }

  public static String md5(String content) throws NoSuchAlgorithmException {
    MessageDigest instance = MessageDigest.getInstance("MD5");
    instance.update(content.getBytes());
    return HexUtils.toHexString(instance.digest());
  }

  public static String sha1(String content) throws NoSuchAlgorithmException {
    MessageDigest instance = MessageDigest.getInstance("SHA-1");
    instance.update(content.getBytes());
    return HexUtils.toHexString(instance.digest());
  }
  public static String sha2(String content) throws NoSuchAlgorithmException {
    MessageDigest instance = MessageDigest.getInstance("SHA-256");
    instance.update(content.getBytes());
    return HexUtils.toHexString(instance.digest());
  }

  public static String sha3(String content) throws NoSuchAlgorithmException {
    Digest digest = new SHA3Digest(256);
    digest.update(content.getBytes(),0, content.length());
    byte[] hashArray = new byte[digest.getDigestSize()];
    digest.doFinal(hashArray, 0);
    return HexUtils.toHexString(hashArray);
  }
}
```



可以看到输出为:

```shell
abcd -> e2fc714c4727ee9395f324cd2e7f331f
abcd -> 81fe8bfe87576c3ecb22426f8e57847382917acf
abcd -> 88d4266fd4e6338d13b845fcf289579d209c897823b9217da3e161936f031589
abcd -> 6f6f129471590d2c91804c812b5750cd44cbdfb7238541c451e1ea2bc0193177
```



## 密码学Hash算法的使用场景

### 数据一致性

密码学Hash算法有 根据相同的输入不管运行多少次都会得到相同的输出这个特性，那么我们就可以用来作为数据一致性的校验：

- 文件比较

  第一种就是文件的比较，经常可以在各种软件下载，或者Docker镜像下载的地方，都会提供该文件的Hash值以及对应的Hash算法，这样就可以在本地验证文件是否完整下载下来。

- 数字签名

  很多地方都会用到数字签名，比如前面说到的SSL证书，或者是JWT，这里面都需要对原始数据计算Hash，并进行加密处理

### 身份验证

前面提到密码学Hash算法必须具备有单向性，也就是不可逆，不能从Hash值逆向推算出原始值，基于这种特性，我们可以用来进行用户身份验证。用户登录都需要进行用户名密码的校验，如果我们将用户密码存在数据库里面，一旦数据库泄露，那就所有人的密码都泄露了，这样的事情前几年发生了不少！那么如果我们将用户的原始密码进行Hash运算，并只是把Hash值保存在数据库里面，当用户登录时，我们计算用户输入密码的Hash值，并与数据库的Hash值进行比较，如果相同则验证通过，不同则失败。而用户的明文密码不进行保存，这样一来，万一数据库泄露了，也不会一下子泄露了全部的用户密码。当然，这样也避免不了彩虹表以及字典攻击，不过我们可以通过对Hash算法加盐进行处理。具体的操作后面再详细介绍。

### 文件秒传

用没有试过向云盘上传一部1-2G的电影，结果几秒就上传成功？ 以我们目前的网络基础设施，应该到不了这样的速度的。那这是怎么做的呢? 其中的一种实现方式就是Hash值的计算，在上传之前，客户端先计算出要上传文件的Hash值，然后将这个值发送回后台，后台检查是否已经存在该Hash值，如果已经存在，则告诉客户端说存在相同的文件，无需再上传，并且把对应的文件ID发给客户端就行，这样客户端就实现了秒传，同时也可以在服务器看到这个文件!










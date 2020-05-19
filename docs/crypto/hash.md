# 密码学中的Hash 算法 #

上一篇文章里面，我们介绍了随机数以及随机数中的应用，可以看到密码学中到处都有随机数的身影，这种作为大部分密码学算法的基本组成被称之为 “加密基元“。今天我们一起来看一下另外一个加密基元 - 密码学Hash算法



## 什么是密码学Hash算法

密码学Hash算法是一个非常重要，而且常见的算法，是计算机密码学中的核心组成部分。密码学Hash算法是指将任何长度的二进制值映射成较短的固定长度二进制值的算法，这个较短的固定长度二进制值就是Hash值。先说一下：这个表述其实不是特别严谨，“任意长度”其实应该是 “算法允许长度范围内的任意长度”，因为有些密码学Hash算法是有输入长度限制的。既然很长的输入可以变成很短的输出，这就像我们写文章之后，需要写一个摘要一样，使用Hash值很多时候也叫做 “消息摘要”， Java中计算密码学Hash值的基类更是直接叫 MessageDigest.

下面直观点，我们来看一些密码学Hash的例子：

```java
  public static void main(String[] args) throws NoSuchAlgorithmException {
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
06d9a9f655c1ac1f0391e7dacfac6cbb <- 明月几时有，把酒问青天，不知天上宫阙，今夕是何年
c4ca4238a0b923820dcc509a6f75849b <- 1 
```



## 密码学Hash算法的特性

密码学Hash算法有好几个特性：

- 相同的输入消息总是能得到相同的Hash值。给定的Hash算法，不管消息长度多少，最终的Hash值长度是相同的
- 不可逆. 不可逆也叫单向性 (pre-image resistance)。 很难通过Hash值反推出原始消息是什么！设想一下，要是可逆的话，那么我们将10GB的文件变成一个128位的hash值，然后进行传输，对方接到后进行逆运算就可以得到原来的文件。那么我们的网络2G就够了，不用去争取5G,6G的了，大华为最近也就不用成天被灭总欺负了
- 很难冲突。很难找到两个不同的消息能够产生相同的Hash值。这里用"很难" 而不是直接写“无冲突”是为了稍微严谨一点，因为Hash算法MD5已经在实践中产生碰撞了.

## 密码学Hash算法的分类

### 普通的Hash算法  -  MD5

### 安全的Hash 算法 - SHA

## Java 中密码学Hash算法的使用



## 密码学Hash算法的使用场景

### 数据一致性： 文件比较，前后端分离架构的验证

### 身份验证： 密码保存与验证

文件秒传：



## Extentions:

### Hash算法 v.s 密码学的Hash算法

### 用Hash保存密码 带来的可能攻击








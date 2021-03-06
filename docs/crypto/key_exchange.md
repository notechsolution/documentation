# 一文详解密码协商算法

>  TL;DR: 前面的文章我们介绍了对称密码算法，非对称密码算法，以及这两种对应的使用场景。对称密码加密算法与非对称密码加密算法都有很好的安全性，但是对称密码有一个麻烦点就是需要加密方把对称密钥给到解密方，这里就会涉及到密钥的安全问题。这篇文章将介绍对称密钥加密算法的密钥如何生成/传递的问题。

## 密钥协商算法 - 对称与非对称的配合使用

前面几篇文章我们了解到了对称加密算法有很好的安全性与较好的性能，但是需要将密钥传递给解密方，这个过程是存在安全隐患的，尤其是在不安全的环境下，比如开放网络环境-互联网等。非对称加密的安全性也很高，而由于需要进行大素数的指数运算, 性能不是很好，但非对称加密算法有另外一个优点，那即使公开密钥不怕在不完全的环境下传递，即使是被人偷听到了也不怕，很多情况下还希望更多的人知道呢！ 所以我们可以将对称密钥加密与非对称密钥加密进行配合使用： 用非对称进行对称密钥的传递-- 得到不安全环境下的安全传递，拿到对称密钥之后，以对称密钥加密进行明文的加密 -- 得到高性能。 这个过程也叫 密钥协商算法。 下面我们介绍几种常见的重要的密钥协商算法。

## RSA密钥协商算法

现在我们来用之前以前介绍过的RSA来实现密钥协商。主要的思路如上述说明：用RSA来实现对称密钥传递，用对称加密算法AES/DES等来实现高性能加密。下面我们以HTTPS协议的密钥协商场景为例。首先我们有浏览器跟服务器，双方的通讯需要进行加密。如果双方直接将对称密码在公网上进行传输，而TCP上是明文的，那么就有可能被人嗅探到密钥，从而破解了传输的内容。下面介绍HTTPS中基于RSA的密钥协商算法：

- 首先服务器有一对公私钥
- 浏览器端向服务器端请求公钥信息，服务器端将自己的公钥pubKey发送给客户端
- 客户端收到公钥后，用随机算法生成一个随机数，作为本次会话的对称密钥k。并且用服务器端的公钥pubKey对这个密钥k进行加密，得到DK，然后将DK发送给服务端
- 服务器端收到DK之后，用自己的私钥priKey进行解密，得到对称密钥k，然后回复确认消息给客户端。 实际算法中还需要校验这个对称密钥k的安全强度够不够等
- 至此，浏览器端与服务器端就可以通过密钥k进行加密信息传递和接收了

可以看到，以上整个过程中，对称密钥k均没明文出现在传递过程当中，这样就可以在不安全的网络环境中，进行安全的加解密操作了。

这里有一个问题：密钥的生成只在客户端进行，服务器端只是接受，顶多就做一些验证而已，服务器端决定不了密钥的生成。如果客户端生成的对称密钥太短，比如只有3位，而服务器端没有进行校验的话，那么破解者就可以直接绕过RSA的加密问题，转而直接破解对称密钥。这种密钥协商算法，只能算法密钥生成及通知算法，没有看到`协商`的过程。

下面会介绍另一种密钥协商算法，一种真正具有`协商` 过程的密钥协商算法

## DHKE

DHKE全称是 Differ-Hellman Key Exchange, 是一种1976年就设计出来的公开密钥算法，只不过这种算法不提供直接的加密或者解密，而是用来进行密钥的传输和分配。

### 混色实验[^1]

DHKE算法的一个核心思想可以用`混色`实验来做一个比较直观的展示。现在我们来介绍这个`混色`实验。

首先这里要假设有两种液体颜料，我们可以很容易地由这两种颜色混出另外一种新的颜色，但是你很难由混出来的新颜色推断出是由哪两种颜色混合出来的，即使告诉你其中的一种颜色，也很难推出原来的颜色以及比例。

下面我们一步步来看这个混色的过程：

- 首先有俩人叫Alice跟Bob，闲着无聊说咱来个混色实验呗。并且俩人说好了用黄色作为底色，这个底色的选择不用保密，即使被人知道也无妨

- Alice和Bob各自再选了一种秘密颜色，自己记得这个颜色。 Alice选了红色，Bob选了大海绿

- 俩人分别将黄色以及秘密颜色 混成一种新的颜色。 Alice得到橘色，Bob得到浅天蓝色

- 到这里，双方都可进行交换了

  ![生成公钥](.\image\key-exchange-by-color-mixing-part-1.png)





接下来开始颜色的交换，从而产生最终的颜色：

- 接着Alice跟Bob交换他们的新混出来的颜色，Alice把橘色给了Bob， Bob把浅天蓝色给了Alice

  最后Alice跟Bob分别将收到的颜色与自己前面挑选的秘密颜色混起来

  - Alice把收到的天蓝色与自己的秘密颜色-红色 混起来得到黄棕色
  - Bob把收到的橘色与自己的秘密颜色-大海绿 混起来，也得到了黄棕色

  ![](.\image\key-exchange-by-color-mixing-part-2.png)

我们可以再看一下，为什么会一样呢？ 我们可以来看看他们俩各自都加了什么颜色

- Alice： Bob分享的浅天蓝色 + 自己的秘密颜色红色 = 黄色 + 大海绿 + 红色
- Bob： Alice分享的橘色 + 自己的秘密颜色大海绿  = 黄色 + 红色 + 大海绿

可以看到他们最终加的颜色组成都一样的，所以可以得到一样的颜色。而这个颜色自始至终都没有出现在传输过程中，而且也实现了双方协商的过程。

### DHKE

DHKE的核心思想跟上述的颜色混合实验的思想是高度一致的。首先我们来看一下DHKE背后的数学内容。DHKE的基本数学是模幂运算：

$$ (g^a)^b \ mod \ p = (g^b)^a \ mod \ p$$

 其中g,p,a,b 都是正整数。假设$A = (g^a)\ mod\ p$, $B = (g^b)\ mod \ p$, 我们我们就可以在不知道对方的a或者b的情况下，算出 $(g^a)^b\ mod\ p = A^b\ mod\ p = B^a\ mod\ p$.

这里我们再来说清楚这些字母都代表什么？ 

- 首先 p 跟 g可以公开的两个数，p是质数，一般要512位以上，而g是一个小整数，一般是2. 

- a是发起方的密钥，这里是Alice的密钥，而$A = (g^a)\ mod\ p$ 则是Alice的公钥

- b是对方的密钥，这里是Bob的密钥，而$B = (g^b)\ mod \ p$ 这是Bob的公钥

- Alice推算出来的shareKey 为 $(g^a)^b \ mod \ p$

- Bob推算出来的shareKey为 $(g^b)^a \ mod \ p$

  

有了上面这个数学基础之后，我们可以来看DHKE协议是怎么工作的：

- 首先Alice选择一个g和p，以及自己的私钥a，然后算出自己的公钥 $A = g^a \ mod\ p$
- Alice把A,p,g 通过公开网络发送给Bob
- Bob也产生一个自己的私钥b，然后根据拿到的p, g，算出自己的公钥 $B = g^b \ mod\ p$
- Bob把自己的公钥B通过公开网络发送给Alice
- 这时Alice和Bob都拿到对方的公钥，于是可以计算出shareKey
  - Alice: $sharedKey = B^a \ mod\ p$
  - Bob: $sharedKey = A^b\ mod\ p$  

下图描述了这么一个过程：

![key-exchange-Diffie-Hellman-Protocol](.\image\key-exchange-Diffie-Hellman-Protocol.png)

至此我们就可以在不安全的网络环境下，通信双方实现密钥协商的过程。

### DHKE在Java中的实践

 下面我们用Java代码来实现DHKE的过程：

```java
public class DHUtil {

  static Logger logger = LoggerFactory.getLogger("DHUtil");

  public static void main(String[] args)
      throws Exception {
    BigInteger p = BigInteger.probablePrime(512, new Random());
    BigInteger g = new BigInteger("2");
    KeyPair keyPairA = generateDHKeyPair(p, g);
    KeyPair keyPairB = generateDHKeyPair(p, g);

    String publicKeyA = new String(Base64.getEncoder().encode(keyPairA.getPublic().getEncoded()));
    String publicKeyB = new String(Base64.getEncoder().encode(keyPairB.getPublic().getEncoded()));
    BigInteger publicKeyAY = ((DHPublicKey) keyPairA.getPublic()).getY();
    BigInteger publicKeyBY = ((DHPublicKey) keyPairB.getPublic()).getY();

    logger.info("publicKeyA, format {},  {}", keyPairA.getPublic().getFormat(), publicKeyA);
    logger.info("publicKeyB, format {},  {}", keyPairB.getPublic().getFormat(), publicKeyB);
    logger.info("publicKeyA Y,  {}", publicKeyAY);
    logger.info("publicKeyB Y,  {}", publicKeyBY);
//
    byte[] secretA = computeSharedSecret(keyPairA.getPrivate(), publicKeyBY.toByteArray(), p, g);
    String secretAString = new String(Base64.getEncoder().encode(secretA));

    byte[] secretB = computeSharedSecret(keyPairB.getPrivate(), publicKeyAY.toByteArray(), p, g);
    String secretBString = new String(Base64.getEncoder().encode(secretB));

    logger.info("secret A equals to secret B: {}", secretAString.equals(secretBString));
  }

  public static KeyPair generateDHKeyPair(BigInteger p, BigInteger g)
      throws NoSuchAlgorithmException, InvalidAlgorithmParameterException {
    DHParameterSpec dhParameterSpec = new DHParameterSpec(p, g);
    KeyPairGenerator keyGenerator = KeyPairGenerator.getInstance("DiffieHellman");
    keyGenerator.initialize(dhParameterSpec);
    return keyGenerator.generateKeyPair();
  }

  public static byte[] computeSharedSecret(PrivateKey myPrivateKey, byte[] hisPublicKeyByte,
      BigInteger p, BigInteger g)
      throws NoSuchAlgorithmException, InvalidKeyException, InvalidKeySpecException {
    KeyFactory keyFactory = KeyFactory.getInstance("DiffieHellman");
    BigInteger hisPublicKeyY = new BigInteger(1, hisPublicKeyByte);
    PublicKey hisPublicKey = keyFactory.generatePublic(new DHPublicKeySpec(hisPublicKeyY, p, g));

    KeyAgreement keyAgreement = KeyAgreement.getInstance("DiffieHellman");
    keyAgreement.init(myPrivateKey);
    keyAgreement.doPhase(hisPublicKey, true);
    byte[] secret = keyAgreement.generateSecret();
    logger.info("secret {} generated by privateKey {} with hisPublicKey {}", new String(Base64.getEncoder().encode(secret)),
        new String(Base64.getEncoder().encode(myPrivateKey.getEncoded())),
        hisPublicKeyY);
    return secret;
  }
```

执行这个程序可以看到如下的输出：

```verilog
09:01:59.588 [main] INFO DHUtil - publicKeyA, format X.509,  MIGeMFcGCSqGSIb3DQEDATBKAkEAsMVOj0kSL1sahj1drMXxkLMXuWFPk6cVpzgl6y9YYC78lahsiuBopR3gi+5a//+4toJi67cotEgZ1qiBLa7cWQIBAgICAYADQwACQEXlVq1nqdYc6DsNhVM2Q4kTvyURfTQ7hoKThmxM/RFkh250VT3KZu0oToU/cJ2BoF98478TYex/COk+71OjXws=
09:01:59.591 [main] INFO DHUtil - publicKeyB, format X.509,  MIGeMFcGCSqGSIb3DQEDATBKAkEAsMVOj0kSL1sahj1drMXxkLMXuWFPk6cVpzgl6y9YYC78lahsiuBopR3gi+5a//+4toJi67cotEgZ1qiBLa7cWQIBAgICAYADQwACQF2ZXEW5XBVn4axuPSnHrmOW1Sp4CSjkt+saP0QDy1Hmn0HtX0AMuewPJG4p9kbMZS0+DDDCPh+O7bsAYfakZZU=
09:01:59.591 [main] INFO DHUtil - publicKeyA Y,  3660742903935543138340844783893407768074467034614604813879913560329513058452449951529057767433562802574483137410428594544885303740866499524026390160826123
09:01:59.591 [main] INFO DHUtil - publicKeyB Y,  4902180763320310394510982200977225381779825174323107369251201843502834164278055264473286139839124487961551802995085460566799994538103539244890003081553301
09:01:59.704 [main] INFO DHUtil - secret ofRXrtOfHWbiyKH53LwEvg/wpTzoHBOaDgbTaYg0m5kckWtx+ekBtpim+5/DpOduno6SosrZJiDUgXcgMtrdEg== generated by privateKey MIGRAgEAMFcGCSqGSIb3DQEDATBKAkEAsMVOj0kSL1sahj1drMXxkLMXuWFPk6cVpzgl6y9YYC78lahsiuBopR3gi+5a//+4toJi67cotEgZ1qiBLa7cWQIBAgICAYAEMwIxANGxLNL/rXPxcCErbRLYzNEcGNq0SbWW+Sly88bCh3PeRH+PdUmM7WMTuzGDUM4tzg== with hisPublicKey 4902180763320310394510982200977225381779825174323107369251201843502834164278055264473286139839124487961551802995085460566799994538103539244890003081553301
09:01:59.704 [main] INFO DHUtil - secret ofRXrtOfHWbiyKH53LwEvg/wpTzoHBOaDgbTaYg0m5kckWtx+ekBtpim+5/DpOduno6SosrZJiDUgXcgMtrdEg== generated by privateKey MIGRAgEAMFcGCSqGSIb3DQEDATBKAkEAsMVOj0kSL1sahj1drMXxkLMXuWFPk6cVpzgl6y9YYC78lahsiuBopR3gi+5a//+4toJi67cotEgZ1qiBLa7cWQIBAgICAYAEMwIxAP5sl3GDhNt5mqHjknrdUOboy+YSzT9wAZPCvwEWGoVA0hw4X3CYn9JTXiG4QCswag== with hisPublicKey 3660742903935543138340844783893407768074467034614604813879913560329513058452449951529057767433562802574483137410428594544885303740866499524026390160826123
09:01:59.704 [main] INFO DHUtil - secret A equals to secret B: true
```



## ECDH

ECDH 全名叫 Elliptic-Curve Diffie-Hellman, 也是一种密钥协商算法。主要的做法也是双方都可以有自己的一对椭圆曲线算法的公私钥对，来进行密钥协商。ECDH也是按照DHKE协议进行密钥协商，只不过DHKE是基于模幂相等数学基础，而ECDH是基于曲线打点的数学基础。具体的ECDH的介绍我们可以在椭圆曲线算法的时候在一起介绍！



[^1]: https://cryptobook.nakov.com/key-exchange/diffie-hellman-key-exchange


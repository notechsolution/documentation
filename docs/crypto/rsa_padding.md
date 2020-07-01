# 一文详解非对称加密算法之RSA Padding

>  TL;DR: 上一篇文章我们介绍了非对称算法RSA，介绍了RSA的基本原理，公私钥的产生以及加解密的过程，并且用Java 以及OpenSSL做一些实践。这篇文章我们来介绍RSA加密算法的安全问题以及处理方法，主要是补位 - Padding。 将会介绍为什么要补位，如果补位，有什么补位的算法。



## RSA 可能存在的安全问题

RSA提供了非对称加密算法的一种实现，也是重要的广泛流行的非对称加密算法。现在我们来看它的一个潜在的安全问题：前文我们介绍到公钥的组成是{n, e}, 其中n的二进制位数就是我们所说的公钥的长度。 假如我们的公钥采用目前最长的key长度4096位，但是指数e=3的情况下，前方战线请示总部是否开战，总部收到消息后电文速回`No`. 会出现什么问题呢？[^1]

- 首先总部查看ASCII表，得出`"No"`的 `ASCII` 为 `0x4E 0x6F`， 所以得到明文 `M = 0x4E6F`.
- 接着总部进行机密, 密文 C = Pow (0x4E6F, 3) mod n , 计算 Pow (0x4E6F, 3) = 0x75CCE07084F，  也就是 C  = 0x75CCE07084F mod n 
- 我们知道n是一个4096位的大数，肯定大于`0x75CCE07084F ` 的，所以我们可以直接得到密文 C = ``0x75CCE07084F ` `

这时敌军监听拦截到这个传输的密文`0x75CCE07084F ` , 觉得很数字很短，有机会进行破解，于是：

- 敌军将`0x75CCE07084F ` 转成10进制，得到8095174953039
- 再尝试着进行开根操作，当小哥进行到开3次根的时候 $\sqrt[3]{8095174953039} = 20079$
- 接着小哥将它转成十六进制 `hex(20079)= 0x4E4F`. 
- 小哥赶紧查一下ASCII表，得到了`No`的明文，遂奔向作战指挥部

从上面的操作我们可以看到，由于n特别大，而e特别小，导致整个加解密的过程都可以不用到n.  从而使得即使公钥的key很大，也有可能容易被破解。



## 如何化解这种安全问题

上面这种情况的重点是密文进行指数e运算的时候，得到的值小于n的值，从而变得加密过程用不到n，所以解密过程也可以用不到n。既然这样，那么我们可以把n变大，比如现在大多数RSA算法的默认e大小65537，这样指数运算后的值肯定大于4096位的大小， n在加解密过程中就必须参与到。

这个问题是解决，但是另外一个问题来了: `相同的明文在不管加密多少次，它的密文都是一样的`。这在密码学中是一个大忌，很容易被破解者猜到内容。我们在对称加密算法中也说到这个问题，需要用添加随机初始化向量IV，以实现相同的加密请求，每一次出来的结果都要不同。 (题外话：这与分布式系统幂等性刚好相反)

那这问题怎么办呢？跟对称加密算法类似，我们可以采用补位的算法来实现。对称加密常用的是PKCS#5 以及 PKCS#7 Padding。而非对称的补位算法常用的是 PKCS#1_v1.5 以及 OAEP (PKCS#1_v2)。



## RSA Padding - PKCS#1_v1.5

为了同时解决上述的两个问题，于是RSA公司设计了一种补位算法，归入到PKCS#1 - RSA Cryptography Standard 里面，称之为PKCS#1_v1.5。其组成是由几个固定位 + 随机数 + 明文消息，其格式如下：

`00 02 [a bunch of non-zero random bytes] 00 [the message]` 

- 第一位默认为00
- 第二位是块类型，`01`表示签名，`02`表示加解密
- 第三部分是一串非0的随机数
- 第四部分是一个0
- 最后一部分就是明文

补位编码后的明文大小应该接近或者等于公钥的位数，所以 `随机数的大小= 公钥n的位数 - 3*8  - 明文的位数`。 而PKCS#1建议最小的随机数为8位才足够安全，所以明文的大小最大是 (公钥n的位数 - 11 * 8), 从而推出下面不同公钥大小能加密的明文最大：

- 1024:  (1024 - 11* 8) / 8=  117
- 2048: (2048 - 11*8) /8 = 245
- 4096: (4096 - 11*8)/8 = 501

这也是RSA加密算法的一个短板：一次性加密的明文大小有限，目前最大的key也只能加密501个字节的明文。PKCS#1_v5 是目前RSA算法的默认补位算法,所以选择RSA加密时，除非特别指定，要不默认的就是PKCS#1_v1.5. 大家打开RSACipher的源码就可以看到paddingType的默认值：

```java
public final class RSACipher extends CipherSpi {
  private int mode;
  private String paddingType = "PKCS1Padding";
```



## RSA Padding - OAEP

这下是不是就万事大吉呢？其实还有一丢丢问题，那就是解密者没法知道解出来的明文是不是`正确`的，只能说按照算法是可以解开的。记得我们在密码学Hash算法安全问题一文中介绍到为了确保消息和原文没有被篡改，我们可以将hash值进行加密运算，也就是消息验证码 - HMAC算法。这里我们反过来，在加密的时候把原文hash也带进来， 这也就是RSA补位算法OAEP - Optimal Asymmetric Encyprtion Padding的核心。听着名字就霸气：最优的非对称加密补位。所以 OAEP算法理所当然地也归入PKCS#1，作为PKCS#1_v1.5的升级版，排名为 PKCS#1_v2.

OAEP的组成由原文的Hash + 随机数+分隔符+原文, 具体的操作比较复杂，大致如下：

- 生成一个k0位的随机数r
- 原文m补上 k1位0，得到消息M， k1 = n - size(m) - k0
- 补位后的消息M与hash函数 G(r)进行异或操作，得到X
- 然后随机数r与X的hash值进行异或操作，得到Y
- 将X与Y合并成补位后的消息

下面的图展示了补位的过程：

![](.\image\rsa_padding01.png)

这样一来就解决了密文重复性问题以及消息一致性问题。目前大厂的RSA加密算法都支持OAEP补位，阿里云的加密算法服务更是只支持OAEP补位算法。下面的Java代码实现了RSA PKCS#1_v15补位以及OAEP补位算法：[^2]

```java
public class RSAUtil {

  static Logger logger = LoggerFactory.getLogger("RSAUtil");

  public static void main(String[] args) throws Exception {
    KeyPair keyPair = generateKeyPair(1024);
    RSAPublicKey publicKey = (RSAPublicKey) keyPair.getPublic();
    RSAPrivateKey privateKey = (RSAPrivateKey) keyPair.getPrivate();
    String publicKeyString = new String(Base64.getEncoder().encode(publicKey.getEncoded()));
    String privateKeyString = new String(Base64.getEncoder().encode(privateKey.getEncoded()));
    logger.info("generate {} bits public key, format {},  {}", publicKey.getModulus().bitLength(),
        publicKey.getFormat(), publicKeyString);
    logger.info("private key format {}, {}", privateKey.getFormat(), privateKeyString);

    String message = "I am Coco Cola!";
    String cipherText = encrypt(publicKeyString, message, false);
    logger.info("[PKCS#1_v15]plainText '{}' encrypted as: {}", message, cipherText);
    String plainText = decrypt(privateKeyString, cipherText, false);
    logger.info("[PKCS#1_v15]cipherText '{}' decrypted as: {}", cipherText, plainText);

    String cipherText1 = encrypt(publicKeyString, message, true);
    logger.info("[OAEP]plainText '{}' encrypted as: {}", message, cipherText1);
    String plainText1 = decrypt(privateKeyString, cipherText1, true);
    logger.info("[OAEP]cipherText '{}' decrypted as: {}", cipherText1, plainText1);

  }


  public static String encrypt(String publicKeyString, String message, boolean usingOAEP)
      throws NoSuchAlgorithmException, InvalidKeySpecException, NoSuchPaddingException, InvalidKeyException, BadPaddingException, IllegalBlockSizeException, InvalidAlgorithmParameterException {
    X509EncodedKeySpec publicKeySpec = new X509EncodedKeySpec(
        Base64.getDecoder().decode(publicKeyString));
    RSAPublicKey pubKey = (RSAPublicKey) KeyFactory.getInstance("RSA")
        .generatePublic(publicKeySpec);
    Cipher cipher = Cipher.getInstance("RSA/ECB/OAEPWITHSHA-256ANDMGF1PADDING");
    if (usingOAEP) {
      OAEPParameterSpec oaepParameterSpec = new OAEPParameterSpec("SHA-256", "MGF1",
          MGF1ParameterSpec.SHA256, PSpecified.DEFAULT);
      cipher.init(Cipher.ENCRYPT_MODE, pubKey, oaepParameterSpec);
    } else {
      cipher.init(Cipher.ENCRYPT_MODE, pubKey);
    }
    byte[] encrypted = cipher.doFinal(message.getBytes());
    return new String(Base64.getEncoder().encode(encrypted));
  }


  public static String decrypt(String privateKeyString, String cipherText, boolean usingOAEP)
      throws NoSuchAlgorithmException, InvalidKeySpecException, NoSuchPaddingException, InvalidKeyException, BadPaddingException, IllegalBlockSizeException, InvalidAlgorithmParameterException {
    PKCS8EncodedKeySpec privateKeySpec = new PKCS8EncodedKeySpec(
        Base64.getDecoder().decode(privateKeyString));
    PrivateKey priKey = KeyFactory.getInstance("RSA").generatePrivate(privateKeySpec);
    Cipher cipher = Cipher.getInstance("RSA/ECB/OAEPWITHSHA-256ANDMGF1PADDING");
    if (usingOAEP) {
      OAEPParameterSpec oaepParameterSpec = new OAEPParameterSpec("SHA-256", "MGF1",
          MGF1ParameterSpec.SHA256, PSpecified.DEFAULT);
      cipher.init(Cipher.DECRYPT_MODE, priKey, oaepParameterSpec);
    } else {
      cipher.init(Cipher.DECRYPT_MODE, priKey);
    }

    byte[] decrypted = cipher.doFinal(Base64.getDecoder().decode(cipherText.getBytes()));
    return new String(decrypted);
  }

  public static KeyPair generateKeyPair(int keySize) throws NoSuchAlgorithmException {
    KeyPairGenerator keyPairGenerator = KeyPairGenerator.getInstance("RSA");
    keyPairGenerator.initialize(keySize);
    KeyPair keyPair = keyPairGenerator.generateKeyPair();
    return keyPair;
  }
}

```



到这里我们基本就介绍RSA有关的补位算法，希望可以让大家在使用过程中清楚地知道有哪些补位算法，他们有什么区别，该怎么用！



[^1]: https://security.stackexchange.com/questions/183179/what-is-rsa-oaep-rsa-pss-in-simple-terms/183330#183330

[^2]: https://stackoverflow.com/questions/50298687/bouncy-castle-vs-java-default-rsa-with-oaep


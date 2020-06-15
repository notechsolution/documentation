# 一文详解对称密钥加密

TL;DR: 前面几篇文章我们介绍了密码学的两大基本单元：随机数和Hash算法。有了这两大基元的加持，我们就可以开始密码学中的加解密算法的介绍。本文我们将重点介绍密码学中的对称密钥算法，包括流密码算法，块密码算法；各种算法的基本原理，重点介绍了主流的块密码算法的补位，迭代模式，加密器的实现等，最后动手实践了在Java中如何使用这些算法为我们业务所用！

## 数据加密及类型

数据加密，指的是根据一定规则，将数据处理成不规则的数据，使得人们除非有了关键的钥匙以及得知这个规则，难于得知无规则数据的真实含义。这个`一定规则` 就是加密算法，这个`钥匙`就是密钥。

数据加密分为对称密钥加密以及非对称密钥加密：

- 对称密钥加密： 双方共同持有这个密钥，发送方用这个密钥按照指定的算法将数据加密，再发出去；接收方用这个密钥将接收到的数据解密，以得到真实的数据含义。由于双方都持有这个密钥，而且内容相同，所以叫对称密钥
- 非对称密钥加密：这种加密方式的密钥是一对，发送方用其中的一把钥匙将数据加密，再发出去；接收方用这对密钥的另一把钥匙将数据解密，以得到真实的数据含义。发送方持有密钥中的一把钥匙，接收方持有另外一把。接收方持有的钥匙叫 `私钥`, 而接收方持有的这把钥匙叫`公钥` 。两把钥匙不一样，所以叫做非对称密钥加密，也叫做公开密钥算法。

这篇文章我们主要介绍对称密码加密算法。对称加密算法可以简单地概括为通过一个算法和一个密钥，对明文进行处理，变成一个无规则无意义数据的算法。明文在算法里面叫plaintext，密钥叫做key，而最终生成的密文叫ciphertext. 下图形象地描述了对称机密算法的操作：

![](.\image\symmetric.png)

我们也可以用如下的公式来简单表述对称加解密算法：

​					密文 = E（明文，算法，密钥）

​					明文 = D（密文，算法，密钥）



## 流密码算法

流密码加密是指将明文信息按字符（具体是按二进制位）逐位地加密的一类密码学算法。在计算机存储中，所有的信息最终都是用0和1进行表示。所有可以采用一定地算法，把某些0变成1，把某些1变成0，这样别人就很难知道真实的内容是什么。怎样把某些0变成1，把1变成0好呢？数学上的异或 XOR ⊕ 操作提供了极好的方案：

- 0 ⊕ 0 = 0
- 0 ⊕ 1 = 1
- 1 ⊕ 0 = 1
- 0 ⊕ 0 = 0

那么加密的时候，明文为M，密钥为K，这样加密的过程就是:  密文C = M ⊕ K, 而对应的解密过程则为 M = C ⊕ K . 破解者仅凭密文是很难破解出明文，即使采用一定的暴力手段，也无法确认破解的明文就是原始明文。

流密码算法的过程大概是这样的：

- 用户提供一个原始密钥key
- 算法将原始密钥key作为种子seed，生成一个伪随机数，这个伪随机数将作为加密的密码流。由于加解密双方知道算法以及种子seed，所以能生成相同的伪随机数
- 算法将原文M与密码流进行异或操作，从而得到密文C
- 信息的接受者同样根据密钥key以及约定的伪随机数生成算法, 生成一个密码流
- 接着将密文C与密码流进行异或操作，从而得出原文

流密码算法可以并行进行处理，而且主要是简单的异或操作，所以性能非常好。

#### 流密码算法的安全问题

但是流密码算法有个问题：如果原始key不变的话，之前的密文很容易被破解。比如有人用明文A，加密后得到的密文为EA。这时如果密钥不变，而我们拦截到密文EA， 那么我们怎样能破解密文呢？ 我们可以用明文B去请求一次加密，得到密文EB。这时我们手头上有对方的密文EA，我们自己明文B以及对应的密文EB，所以我们就可以推算出明文 A = EA  ⊕ EB  ⊕ B。 这个过程怎么推导出来的呢?

1. EA = A  ⊕ Key
2. EB = B  ⊕ Key

根据相同的值异或为0 这个规则，那么我们可以推出：

3. EA  ⊕ EB = A  ⊕ Key  ⊕  B  ⊕ Key  = A  ⊕ B  ⊕ (Key  ⊕ Key) = A  ⊕ B

这时两边同时异或B：

4. EA  ⊕ EB  ⊕ B = A  ⊕ B  ⊕ B = A  ⊕ (B  ⊕ B  ) = A 

所以明文A = EA  ⊕ EB  ⊕ B 

流密码算法的关键是一次性的密码本(One-time Pad), 这样才能确保消息不被破解。 由于速度快以及实现简单，在确保一次性密码的前提下，流密码算法一直活跃于各种业务场景下。RC4 (Rivest Cipher 4)就是一种常见的流密码加密算法的实现。之前一直用于HTTPS的TLS，对用户请求的内容进行对称加密，而其密钥则由RSA或者DH等密码协商算法一次性生成使用。 但由于后来有攻击者可以在4个小时破解一个HTTPS保护的cookie的内容。所以RC4现在基本被移除出 HTTPS的加密套件。在实际项目当中，大家也尽量能不用就不用RC4。可以采用我们下面要介绍的块密码加密算法，目前更为安全一点。



## 块密码算法

块密码(Block Cipher)算法加解密的时候，不是一次性完成，而是把原文分成固定长度的块，每次对这些数据块进行处理。所以块密码算法也叫分组密码算法。

块密码算法的流程大致如下：

- 将全量明文M按照算法要求拆解成固定大小的明文块m-0 ~ m-n, 也就是一小块一小块的分组
- 对于每一个明文块通过加密器进行加密
- 加个明文块加密的结果合并起来，成为最终的明文C

![](.\image\symmetric00.png)

#### Padding

分组很有可能会发生全量明文不是块大小的倍数，比如全量明文是64bits : {1,2,3,4,5,a,b,c,d,e} , 块大小是128bits，那么明文块0就只有64bits。这时怎么办呢？其中一种解决方式就是进行补位了。然后当解密成功后，将补位的内容去掉就得到完整的明文。补位的类型有下面几种：

##### 1) NoPadding

第一种最霸气，加密算法说“我不补位!” 。 你们的明文都必须是我指定的块大小的倍数，否则我就不干了！比如上面的情况，你提供一个128bits 倍数的明文，那我乖乖地加密，如果不是128bits的倍数，我就报错!  这样我解密也不用费事去除补位的内容了！

我相信大多数的码农打心里都想这样干，同时我也相信这样写大概率会被项目经理追杀！

##### 2) ZeroPadding

第二种比较乖，明文不是块大小的倍数是没问题的，我来帮你补齐，你差几位，我补几个0. 比如上面明文块差64bits，那我就补8个0字符。变成 {1,2,3,4,5,a,b,c,d,e,0,0,0,0,0,0,0,0}. 这样补齐128bits。 

这种补位方式加密的时候没问题，但是在解密的时候有可能会遇到问题：万一明文最后几个字符也是0，比如你的存款3000000000。这样如果银行的程序员小哥的加密算法补位是ZeroPadding，当你要去取款时系统解密余额，发现后面全是0，好家伙全干掉了. 小哥 + 项目经理卒...

##### 3) PCKS#5 / PKCS#7

既然有问题，那我们就改进。标准化组织IETF(Internet Engineering Task Force)的PKIX工作组维护了一系列的标准，从PKCS#1到PKCS#14，其中的PKCS#5 和PKCS#7里面有定义了关于补位的标准。PKCS#5 / #7的补位标准是：补位的字符等于缺少的字符数。比如上面明文 {1,2,3,4,5,a,b,c,d,e}，块大小是128bits，那么明文块缺少64bits，也就是8个字符，那么补位的字符就是8。补位后的明文变成 {1,2,3,4,5,a,b,c,d,e,8,8,8,8,8,8,8,8}。 

如果明文大小刚好了块大小的倍数呢？那就补齐一整个块。最后一块变成了 {16,16,16,16,16,16,16,16,16,16,16,16,16,16,16,16}. 

PKCS#5 /PCKS#7在补位方面是一样的，他们的区别在于明文块的大小：PKCS#5 限定了明文块的大小是128bits，而PKCS#7的分组大小可以1-255任意大小。



#### 迭代模式

补位完成后，我们可以开始加密了。全量明文分成这么N跟明文块，那么需要迭代对每一个明文块进行加密。块密码算法有很多种不同的迭代方式，下面我们介绍常用的以及推荐的迭代方式：

##### 1) ECB - Electronic CodeBook

ECB模式时最简单的一种迭代模式。它的加密过程如下：

- 将全量明文分成多个明文块，最后一块不够块大小的需要补位
- 一次对每个数据块进行迭代，得到每一个数据块对应的明文，最后将所有的密文块组合成完整的密文。

其解密的过程刚好是相反的。下图展示了ECB的加密过程图示：



![](.\image\symmetric02.png)

ECB的优点是： 简单，由于每一块都是独立加密运算的，可以并行运算，并且误差不会被传递

ECB的缺点是：不能隐藏明文的模式，比如原文重复出现有“珍珠港”字样，那么密文也相同的密文字符重复出现。而且有可能被进行明文攻击，具体可以搜索一下`ECB模式攻击` 

**不推荐**使用ECB作为对称密码加密的迭代模式。

##### 2) CBC - Cipher Block Chaining

CBC是一种比较常见的迭代模式，基本解决了ECB模式的安全问题。它的迭代步骤如下:

- 同样地将明文分组，最后一块进行补位
- 处理第一个数据块前，生成一个初始化向量 IV (Initialization Vector). IV一般是随机数
- IV与明文块0进行异或，运算的结果经传给密码器进行加密，得出密文C0
- 接着处理后续明文块，将上一个明文块的输出C0 作为下个明文块加密的IV，进行与第一个块相同的操作，以此迭代下去。

下图展示CBC的迭代模式：

![](.\image\symmetric03.png)

CBC模式的优点是：引入了IV，如果采用随机IV，可以做到同样的明文和密钥，最终得到的明文不一样，更为安全一些。初始化向量IV是对着密文一起发送给解密者的，IV的长度是跟密码块的长度一致，一般作为第一个密码块传递给解密者

CBC的缺点是：每一块的解密都需要上一块解密的结果作为输入IV，所以只能串行处理。

推荐使用CBC作为对称密码加密的迭代模式。

##### 3) CTR - Counter

CTR模式在迭代的时候，类似于流密码的运行模式。每次迭代的时候要生成一个密码流。生成密码流的方法可以是任意的，但是各个密码之间是有关系的，最简单的方式就是密码流不断递增，所以这个就叫 计数器模式 Counter。它的处理过程大致如下：

- 将明文拆成N个明文块，但是不用补位
- 生成N个密码流，简单的方式比如递增
- 接下来进行迭代加密，密码流与密码进行加密操作，然后与明文进行异或XOR运算得到密文块
- 迭代运行每一个明文块，得到最终的密文

下图展示了CTR模式的大致流程：

![](.\image\symmetric04.png)

CTR的优点是:  引入Nonce，随机的Nonce可以确保同样的明文和密钥，可以产生不同的密文。由于密钥流按照一定的规则生成，所以个加密单元可以预先知道所有的输入 密钥， Nonce，明文块，那么就可以并行进行处理。 CTR每一块里面采用流加密算法，将明文与对应的流密钥位进行异或XOR操作就行，所以不需要补位填充

推荐使用CTR作为对称密码加密的迭代模式。



#### 加密器

上面这几种不同的迭代方式都说到了加密器，但到目前为止，这一部分仍然像个黑盒子一样，只知道给他密钥跟明文，他就能加密。不同的算法有不同的加密器的实现，即使相同算法，密钥长度不同加密器的实现也有不同，比如AES加密器里面，AES128的密钥迭代次数是10，AES192是12轮，而AES256是14轮。但加密器的基本操作的类型大致有，有字节替换，行位移，列混淆，加轮密钥。不一定每一个算法都实现了这4个步骤。

##### 字节替换

字节替换是将明文的字符根据一定的规则替换成另外的字符，根据什么规则呢？一般会有与明文块大小一致的二维常量数组，这个二维常量数组也叫替换表 Substation Box, 简称S盒。 算法实现根据明文的位置从S盒中找到对应的字符替换掉明文字符。比如AES128的块长度是16字节，那么它的S盒就是一个4x4的二维数组。如下图：

![](.\image\symmetric05.png)

##### 行位移

替换还不够，这里还要来个行位移，怎么弄呢？就是每一行的元素 “预备起”，按照要求向左或者向右移动几位。比如第0行不动，第1行左移1位，第2行左移2位，第3行左移3位。下图描述了这个过程：

![](.\image\symmetric06.png)

##### 列混淆

行已经移动了，列也进行移动就显得很弱，那我就进行混淆吧。每一列都要与一个叫修补矩阵(fixed matrix)的二维数组常量进行相乘，得到对应的输出列：

![](.\image\symmetric07.png)



##### 加轮密钥

行也移动了，列也混淆了，我们还有一个真正的密钥没动呢！密钥也排成对应的矩阵，比如AES128的就排成一个4x4的矩阵。然后将输入数组的每一个字节与密钥矩阵的对应位置的值进行异或XOR操作，生成最终的输出值。下图描述了这个过程：

![](.\image\symmetric08.png)

 这里要注意一下，这里的密钥不是原始128位的密钥，而是由原始128位的密钥扩展出来的子密钥。上面说了AES128的加密迭代是10轮，那么就需要从原始128位的密钥迭代生成10个128位的子密钥。下面描述了一个64bit密钥的迭代过程：

- 将64 位的key按照第一轮置换表PC-1进行置换,得出一个置换结果1。由于置换表只有56位，所以置换结果只有56位（每7位需要留1位作为校验码）
- 将56位的置换结果分成左右两部分，左边是C0， 右边是D0，每一边都是28位
- C0跟D0分别按照循环左移表进行进行左移操作，得到左移后的C1和D1
- C1和D1进行合并，再经过48位的PC-2表进行置换操作，得到一个48位的密钥K1
- 将上一轮产生的C1和D1作为下一轮的输入，迭代直到所有的轮数都完成

下图大致描述了这个过程：

![](.\image\symmetric09.png)

### 块密码算法的种类

上面我们介绍了块密码大部分的原理内容了，接着我们来看一下常见的块密码算法 DES, 3DES 以及AES。

#### DES

DES - Data Encryption Standard. 听着名字就霸气 - 数据加密标准，一般这么霸气都是来自米国。1977年米国采用了IBM公司设计的加密方案作为正式的数据加密标准，起名为DES。 DES 是分组密码算法的一种，它的key是64bits，它支持的迭代模式为ECB和CBC。它的主要流程是：

- 将明文分组
- 对每个分组进行初始字节置换
- 生成对应的子密钥
- 对每一个迭代进行置换，异或运算等运算，得到分组密文
- 迭代直到完成全部迭代过程

#### 3DES

随着时间的推移，摩尔定律持续生效，64位的密钥被破解的风险很高。而设计一种新密码算法又需要很长的时间，进行实际应用论证就更长了。所以不能轻易放弃DES。既然它长度不够，那么我们就扩展它密钥长度。一个64位的密钥不够长，那我就弄出3个64bits: K1,K2, K3. 把密钥变成 `Key = EK3(DK2(EK1))`.  

- 首先用K1进行DES加密
- 然后用K2进行DES解密 （由于`K1 != K2`, 所以`解密`也就变成`加密`）
- 最后用K3进行DES再加密

所以这个算法叫 3DES。这个算法基本解决了DES算法密钥长度不够的问题，但也引来了新的问题，同一个明文需要加密3次才能得出结果，性能自然而然就降下来了

#### AES

3DES虽然勉强能满足要求，但是作为宇宙科技最先进的NIST，不可能就此碌碌而为啊! 2000年召集群英会，要求设计出一款新的安全的对称加密算法。 Rijndael 被选中成为后宫佳丽，在2002年宣布正式成为新的加密算法标准 AES - Advanced Encryption Standard。 AES支持128,192,256位数的密钥， 支持 ECB，CBC， CTR等迭代模式。AES加密的步骤有：

- 将明文进行分组，最后一个块需要进行补位
- 对于每一个分组进行字节置换，行位移，列混淆以及加轮密钥的操作
- 不同的密钥长度进行不同的轮数操作，有10轮，12轮，14轮，得到分组密文
- 迭代直到完成全部迭代过程

目前大部分的加密场景下都使用了AES，也推荐大家使用AES进行加密。

## 对称密码算法在Java中的实践

### 流密码算法RC4

下面我们实现一个简单的流密码算法RC4：

```java
public class RC4Util {
  static Logger logger = LoggerFactory.getLogger("RC4Util");

  public static void main(String[] args) throws UnsupportedEncodingException {
    String message = "I am Vita";
    String password = "lemon";
    String cipherText = encrytRC4String(message, password);
    logger.info("message '{}' with password '{}', cipher text: '{}'", message, password, cipherText);
    String plainText = decrytRC4String(cipherText, password);
    logger.info("cipher '{}' with password '{}', plain text: '{}'", cipherText, password, plainText);
  }


  public static String encrytRC4String(String data, String key) throws UnsupportedEncodingException {
    if (data == null || key == null) {
      return null;
    }
    return HexUtils.toHexString(RC4Encryption(data.getBytes(), key));
  }


  public static String decrytRC4String(String cipherText, String key) {
    if (cipherText == null || key == null) {
      return null;
    }
    return new String(RC4Encryption(HexUtils.fromHexString(cipherText), key));
  }
  private static byte[] RC4Encryption(byte[] data, String mKkey) {
    int x = 0;
    int y = 0;
    byte key[] = keyDerivation(mKkey);
    int xorIndex;
    byte[] result = new byte[data.length];
    for (int i = 0; i < data.length; i++) {
      x = (x + 1) & 0xff;
      y = ((key[x] & 0xff) + y) & 0xff;
      byte tmp = key[x];
      key[x] = key[y];
      key[y] = tmp;
      xorIndex = ((key[x] & 0xff) + (key[y] & 0xff)) & 0xff;
      result[i] = (byte) (data[i] ^ key[xorIndex]);
    }
    return result;

  }

  private static byte[] keyDerivation(String aKey) {
    byte[] bkey = aKey.getBytes();
    byte state[] = new byte[256];

    for (int i = 0; i < 256; i++) {
      state[i] = (byte) i;
    }
    int index1 = 0;
    int index2 = 0;
    if (bkey.length == 0) {
      return null;
    }
    for (int i = 0; i < 256; i++) {
      index2 = ((bkey[index1] & 0xff) + (state[i] & 0xff) + index2) & 0xff;
      byte tmp = state[i];
      state[i] = state[index2];
      state[index2] = tmp;
      index1 = (index1 + 1) % bkey.length;
    }
    return state;
  }

}
```



执行结果为：

```verilog
11:35:16.118 [main] INFO RC4Util - message 'I am Vita' with password 'lemon', cipher text: '8db4d29023fb305dc8'
11:35:16.122 [main] INFO RC4Util - cipher '8db4d29023fb305dc8' with password 'lemon', plain text: 'I am Vita'
```



### 块密码算法

在Java中，块密码算法都抽象在Cipher这个class里面，我们可以参入不同的加密算法作为输入参数，Cipher会去寻找不同的算法实现在进行加解密。所以DES ， AES的调用都可以用相同的Util类实现。 这个Util类采用了默认初始化向量IV的实现 - SecureRandom，并且将这个动态的IV 放到密文的第一个分组，这样IV就不用在额外传递。

```java
public class SymmetricUtil {

  static Logger logger = LoggerFactory.getLogger("SymmetricUtil");

  public static void main(String[] args) throws Exception {
    String message = "I am Vita";

    String desPassword = "lemonIsG"; // 8bit
    testCryptgo("DES","DES/CBC/PKCS5Padding", message, desPassword );

    String aesPassword = "lemonIsGoodForYo"; // 128bit
    testCryptgo("AES","AES/CBC/PKCS5Padding", message, aesPassword );

  }

  public static void testCryptgo(String family, String algorithm, String message, String password) throws Exception {
    String cipherText = encrypt(family, algorithm, message, password);
    logger.info("[{}] message '{}' with password '{}', cipher text: '{}'", family, message, password, cipherText);
    String plainText = decrypt(family, algorithm, cipherText, password);
    logger.info("[{}] cipher '{}' with password '{}', plain text: '{}'", family, cipherText, password, plainText);
  }

  public static String encrypt(String family, String algorithm, String message, String password)
      throws Exception{
    SecretKeySpec key = new SecretKeySpec(password.getBytes(), family);
    Cipher cipher = Cipher.getInstance(algorithm);
//    REMARKS: if you want to use self defined IV, use below logic to generate IV. here use the system default IV by SecureRandom
//    String ivString = RandomStringUtils.randomAlphanumeric(cipher.getBlockSize());
//    IvParameterSpec iv = new IvParameterSpec(ivString.getBytes());
//    cipher.init(Cipher.ENCRYPT_MODE, key,iv);
    cipher.init(Cipher.ENCRYPT_MODE, key);
    byte[] cipherByte = cipher.doFinal(message.getBytes());
    return HexUtils.toHexString(ArrayUtils.addAll(cipher.getIV(), cipherByte));
  }

  public static String decrypt(String family,String algorithm, String message, String password)
      throws Exception{
    byte[] messageByte = HexUtils.fromHexString(message);
    SecretKeySpec key = new SecretKeySpec(password.getBytes(), family);
    Cipher cipher = Cipher.getInstance(algorithm);
    // retrieve IV from cipherText
    byte[] iv = ArrayUtils.subarray(messageByte,0, cipher.getBlockSize());
    // retrieve the real cipher content from input
    byte[] cipherContent = ArrayUtils.subarray(messageByte,cipher.getBlockSize(), messageByte.length);
    IvParameterSpec ivParams = new IvParameterSpec(iv);
    cipher.init(Cipher.DECRYPT_MODE, key, ivParams);
    byte[] cipherByte = cipher.doFinal(cipherContent);
    return new String(cipherByte);
  }

}

```

程序执行的结果为：

```verilog
16:32:02.182 [main] INFO SymmetricUtil - [DES] message 'I am Vita' with password 'lemonIsG', cipher text: '45e705887cf9c1646c8f8a75b1b1561e1a14304e2a44abcc'
16:32:02.191 [main] INFO SymmetricUtil - [DES] cipher '45e705887cf9c1646c8f8a75b1b1561e1a14304e2a44abcc' with password 'lemonIsG', plain text: 'I am Vita'
16:32:02.207 [main] INFO SymmetricUtil - [AES] message 'I am Vita' with password 'lemonIsGoodForYo', cipher text: '5f04ca7403eda616725b11b603df896d8c3bf91497f56e4a14f7e417ec77d43a'
16:32:02.207 [main] INFO SymmetricUtil - [AES] cipher '5f04ca7403eda616725b11b603df896d8c3bf91497f56e4a14f7e417ec77d43a' with password 'lemonIsGoodForYo', plain text: 'I am Vita'

```



## 小结

写了好几天，终于把对称密钥加密的内容大致写出来了！希望能通过这篇文章为大家理清对称密钥加密算法的轮廓，算法的实现原理，各种算法的区别以及如何使用！


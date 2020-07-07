# 一文详解非对称加密算法之RSA算法

TL;DR: 上一篇文章我们介绍了对称加密算法，其最主要的特点就是加密者和解密者持有相同的密钥，所以称之为`对称`。照理推想，有对称就有非对称。这篇文章我们来介绍另外一个重要的加密算法：非对称加密算法 (Asymmetric Cryptography), 也称为公开密钥加密算法 (Public Key Cryptography). 

## 公开密钥算法概要

首先，跟对称密钥算法一样，非对称密钥算法不是指一个算法，而是一种算法。这类算法与对称加密算法相比较，有如下的特点：

- 密钥是一对

  在上文的对称加密算法我们看到，密钥是一串数字或者字符串，加密者和解密者使用相同的密钥进行加解密。公开密钥算法则不同，它的密钥是一对的，分成公钥 - public key 和私钥  - private key。一般私钥是由密钥对的生成者持有，比如服务器端，不能泄露。而公钥是任何人都可以持有，是公开发布的，不怕泄露。由此这个算法而得到 `公开密钥`算法的名号。

- 功能不一样

  对称加密算法的主要功能是加密和解密，而公开密钥算法的功能除了加密和解密，还可用于密钥协商，数字签名，是数字证书， HTTPS等的最核心基础。

- 运行速度慢

  相对于对称加密算法，公开密钥算法由于其基础运算是指数运算再求余，而为了安全，指数一般是一个比较大的数值，所以其运算非常缓慢，而且由于算法的局限，一次加密的明文块很小，所以如果要加密一个很大的明文，比如一个文件的话，那性能是惨不忍睹的。所以一般情况下，会根据使用场景，只用公开密钥算法来加密对密钥保存要求更高的数据，而不是全部都用公开密钥算法来加密。

## RSA 算法

现在我们来介绍第一个公开密钥算法，也是比较常用而且重要的一个算法，叫RSA。 该算法是由Ron Rivest， Adi Shamir， Leenard Aldeman三个人创建的，以他们三个人的首字母来命名RSA。

 ### RSA算法原理

RSA算法的设计用到的数学知识很多，RSA会用到质数，互质数，公约数定理，欧几里得算法，同余和模求解，唯一质数分解定理，欧拉函数，欧拉定理和费马定理等。受限于时间和数学能力，这里就不一一展开了，有兴趣的可以参考这篇文章 https://www.jianshu.com/p/6aa7b59be872

#### 公私钥的生成

- 生成两个不相等的大质数p和q，它们的积 $n = pq$ ，这个n的二进制位数就是密钥的位数，通常是1024， 2048， 4096. 

- 计算p和q的乘积$n=pq$ , 以及欧拉公式  $\varphi(n)= (p-1)(q-1)$

- 选择一个整数e, 使得 $1 < e< \varphi(n)$, 且e和 $\varphi(n)$ 是互质的, 即 $gcd(e, \varphi(n))=1$. 在大多数RSA算法实现里面，e固定位65537
- 计算 e 对于 $\varphi(n)$ 的模反元素 d。即找到整数d，1 < d < φ，且满足 $ed≡1(mod  \varphi (n))$ 

- 把n 和e 封装成公钥，把n和d封装成私钥

n、e、d分别称之为:

- n : modulus 模数
- e: public exponent 公开指数
- d: private exponent 私有指数

所以我们得到RSA的公钥 {n, e}, 私钥 {n, d}



#### 加解密的过程

有了公私钥，我们就可以开始来看加解密的原理了。我们定义明文消息为M，而加密的内容为C，那么加密的过程为：计算明文消息的e次幂，然后与n求模： 


$$
C= M^e\quad mod\quad n
$$
而解密的过程为: 计算密文的d次幂，然后与n求模:
$$
M= C^d\quad mod\quad n
$$


具体的数学上的论证大家可以参考上面那篇文章，里面比较详细地介绍了论证过程。我们这里只介绍具体的使用。



#### RSA算法的安全性

幂运算的逆过程是对数问题，而模运算可以认为是离散问题，组合起来，RSA算法就是离散对数模型，只要密钥足够长，离散对数很难破解。密钥的长度也就是n的二进制位数。大家都可以获得公钥{n,e}, 而要计算出私钥d，那么需要知道p和q。而想通过一个巨大的n（一般为1024,2048 甚至4096位）获得p和q是一个因式分解问题，也叫大素数分解问题，暴力破解很难。 因此只要密钥足够长，目前推荐2048位，RSA算法是很安全的。

### RSA的实践

现在我们来介绍如何具体地实践RSA

- 公私钥对的生成

  首先是一个密码学神奇OpenSSL，这是一个开源的软件包，密码学的算法基本都包了。我们现在用OpenSSL来生成一个公私钥对:

  ```shell
  λ openssl genrsa -out myrsa.pem 2048
  Generating RSA private key, 2048 bit long modulus (2 primes)
  ............................+++++
  ..+++++
  e is 65537 (0x010001)
  ```

  这时OpenSSL会生成一个pem格式的文件，里面用ASCII的形式保存公私钥的关键信息n,e,d,p,q.  PEM( Privacy Enhanced Mail) 是RFCs 1421 https://tools.ietf.org/html/rfc1421 定义一个文件格式. 我们打myrsa.pem 文件，里面显示如下：

  ```
  -----BEGIN RSA PRIVATE KEY-----
  MIIEpQIBAAKCAQEAt8rHReYR+jIr2tetc1AhrrZkfj7ewbu4K7XscVPGhlyYR8Uk
  s2vn6MXJwglGjN5ETJmMBJ4MMhLHaATtW2Zj9iwuyZNHJtBndFjrILNpmoF+nxVl
  ......
  uBxc36lllGao2bN/EXcq+4yp4swQWfNVomK2kK7GQGONI9zokhJEfbRAb3Zxp0DM
  Fh85P7Hi50AVNWpZ2X+mCCaZt2gn8EB11G8r2BV/SPQTG6QqX+taXLQ=
  -----END RSA PRIVATE KEY-----
  
  ```

  我们可以看看这里面究竟都包含什么信息：`openssl rsa -in myrsa.pem -noout -text`

  ```
  λ openssl rsa -in myrsa.pem -noout -text                      
  RSA Private-Key: (2048 bit, 2 primes)                         
  modulus:                                                      
      00:b7:ca:c7:45:e6:11:fa:32:2b:da:d7:ad:73:50:             
      21:ae:b6:64:7e:3e:de:c1:bb:b8:2b:b5:ec:71:53:             
      c6:86:5c:98:47:c5:24:b3:6b:e7:e8:c5:c9:c2:09:             
      46:8c:de:44:4c:99:8c:04:9e:0c:32:12:c7:68:04:             
      ed:5b:66:63:f6:2c:2e:c9:93:47:26:d0:67:74:58:             
      eb:20:b3:69:9a:81:7e:9f:15:65:5b:ae:23:91:b3:             
      10:0f:8a:0c:33:5b:cb:09:a1:4c:82:35:18:ea:0c:             
      6d:09:27:04:19:08:62:1d:5b:39:70:8f:5e:3c:55:             
      de:01:ab:2c:33:2d:cd:ea:56:c5:81:66:7f:75:5a:             
      76:95:2e:29:61:28:e5:02:5d:f5:5c:37:1c:77:c5:             
      1b:f9:6f:36:b9:93:16:8b:de:bd:39:f1:93:58:0d:             
      10:34:22:5e:1f:ab:5b:fe:3f:84:e4:7d:9f:0c:06:             
      05:34:82:e8:fe:b8:e1:f0:6f:27:8a:fb:fe:b7:a7:             
      e9:e7:04:e0:38:5c:41:c2:12:f9:e4:e8:3b:5e:2b:             
      d5:30:1d:d6:7a:79:17:c0:93:f1:41:0a:9f:32:2a:             
      4d:37:2e:c6:5c:e0:a0:33:70:6e:41:d0:68:c3:4e:             
      b6:c5:b1:46:fc:36:c9:3c:70:e0:95:4a:f0:83:c3:             
      09:81                                                     
  publicExponent: 65537 (0x10001)                               
  privateExponent:                                              
      5e:00:31:81:57:95:94:40:7a:db:97:f9:d7:83:81:             
      ...          
      72:c5:8d:90:bb:62:51:e1:b4:da:3d:4d:34:a6:c4:             
      d1                                                        
  prime1:                                                       
      00:f0:b3:a7:ec:fe:11:ce:25:a7:b4:01:46:57:39:             
      09:39:a0:62:5b:06:f4:70:3b:9a:02:0c:6a:01:5e:             
      1f:69:16:51:7a:b4:03:34:09:99:13:5d:5e:c1:b8:             
      2d:92:73:6a:37:c6:66:5c:d8:02:6b:b7:41:58:3d:             
      a3:f1:70:a3:1a:42:11:fb:dc:e4:79:61:c5:39:16:             
      ea:d4:88:2a:f6:4c:c8:77:56:70:6e:e2:7f:14:f0:             
      dd:46:e4:ce:5a:2b:37:fe:04:98:09:0b:be:d5:8b:             
      7b:18:5a:ad:2d:cc:9e:d2:0a:5f:2f:83:2e:48:6a:             
      c8:99:3d:14:a7:46:4e:b3:6d                                
  prime2:                                                       
      00:c3:79:2c:14:57:e0:db:6a:54:cd:96:ca:33:74:             
      47:e1:62:85:e3:15:82:3e:00:41:03:c6:a9:75:f2:             
      79:df:2c:86:41:1c:5a:09:ea:8b:51:45:f8:d7:0d:             
      bf:45:c8:4d:4b:57:73:61:b1:76:cb:98:7a:f2:c9:             
      61:0f:c3:e9:9e:47:b1:09:76:ce:a0:4e:2f:17:e9:             
      81:f6:a9:d8:a5:cf:96:54:62:70:a2:17:fd:7d:ed:             
      09:59:09:f0:18:27:62:5a:97:59:29:b2:1f:57:fe:             
      b3:86:55:a9:1c:a3:23:8d:33:13:76:42:55:88:c2:             
      88:f3:ad:37:3e:b9:40:0d:e5                                
  exponent1:                                                    
      00:db:e3:63:be:ef:03:99:0d:71:3c:d2:05:4e:5d:             
     ...             
      d2:08:9b:72:28:b5:e3:e3:a9                                
  exponent2:                                                    
      00:a8:11:9e:89:db:49:55:be:e6:2d:62:b2:76:6d:             
  	....            
      c4:d2:27:a3:f1:85:5c:82:d5                                
  coefficient:                                                  
      00:bf:a4:d2:39:09:7a:66:1a:30:8e:9b:7e:10:ea:             
  	...             
      13:1b:a4:2a:5f:eb:5a:5c:b4                                
                                                                
  ```

  可以看到n, e,d 都在里面，甚至原始数据p 和q也存在里面。所以这个文件也是我们的私钥，只不过里面包含了公钥的信息

  接着我们从myrsa.pem里面剥离出公钥

  ```shell
  λ openssl rsa -in myrsa.pem -pubout -out mypubkey.pem
  -----BEGIN PUBLIC KEY-----
  MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAt8rHReYR+jIr2tetc1Ah
  rrZkfj7ewbu4K7XscVPGhlyYR8Uks2vn6MXJwglGjN5ETJmMBJ4MMhLHaATtW2Zj
  9iwuyZNHJtBndFjrILNpmoF+nxVlW64jkbMQD4oMM1vLCaFMgjUY6gxtCScEGQhi
  HVs5cI9ePFXeAassMy3N6lbFgWZ/dVp2lS4pYSjlAl31XDccd8Ub+W82uZMWi969
  OfGTWA0QNCJeH6tb/j+E5H2fDAYFNILo/rjh8G8nivv+t6fp5wTgOFxBwhL55Og7
  XivVMB3WenkXwJPxQQqfMipNNy7GXOCgM3BuQdBow062xbFG/DbJPHDglUrwg8MJ
  gQIDAQAB
  -----END PUBLIC KEY-----
  
  ```

  

  接着我们来看一下在Java中如何生成公私钥对：

  ```java
  public class RSAUtil {
  
    static Logger logger = LoggerFactory.getLogger("RSAUtil");
    
  public static void main(String[] args) throws Exception {
      KeyPair keyPair = generateKeyPair(1024);
      RSAPublicKey publicKey = (RSAPublicKey) keyPair.getPublic();
      RSAPrivateKey privateKey = (RSAPrivateKey) keyPair.getPrivate();
      String publicKeyString = new String(Base64.getEncoder().encode(publicKey.getEncoded()));
      String privateKeyString = new String(Base64.getEncoder().encode(privateKey.getEncoded()));
      logger.info("generate {} bits public key, format {},  {}", publicKey.getModulus().bitLength(), publicKey.getFormat(), publicKeyString);
      logger.info("private key format {}, {}", privateKey.getFormat(),  privateKeyString);
  
    public static KeyPair generateKeyPair(int keySize) throws NoSuchAlgorithmException {
      KeyPairGenerator keyPairGenerator = KeyPairGenerator.getInstance("RSA");
      keyPairGenerator.initialize(keySize);
      KeyPair keyPair = keyPairGenerator.generateKeyPair();
      return keyPair;
    }
  ```

  

  输出为：

  ```verilog
  18:11:53.604 [main] INFO RSAUtil - generate 1024 bits public key, format X.509,  MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQCP8+ujsUeg2hhgNSz1t8hoBHixoEy2tWVLo8Z22WC3LxqUfaGduvafIBlU9EYBU24ximn66N/AY4F9VzTKxVy3JmplIIiTptr+it5BMkJCO3YrsqPo6qKXHhpclvoc+YPfHB/8v13fmWlwI9aMCkI+mYF7m4V/gNouikBx8ZZcXQIDAQAB
  18:11:53.604 [main] INFO RSAUtil - private key format PKCS#8, MIICdwIBADANBgkqhkiG9w0BAQEFAASCAmEwggJdAgEAAoGBAI/z66OxR6DaGGA1LPW3yGgEeLGgTLa1ZUujxnbZYLcvGpR9oZ269p8gGVT0RgFTbjGKafro38BjgX1XNMrFXLcmamUgiJOm2v6K3kEyQkI7diuyo+jqopceGlyW+hz5g98cH/y/Xd+ZaXAj1owKQj6ZgXubhX+A2i6KQHHxllxdAgMBAAECgYAeIJqsg6nODFcVq4thUblrq6Pm6PmlM4mjrv8WWKBZNk6FzVVJwZtj6j/i+8y68k8ZpzJPBPXvOeQb62htF6kziUniqfEa78eoIwUbyPeMW6iOnPz8cSvMbDaKfR4GO6IufNajDQBG+8093+ILyU4eZiH8+UVDfGT1pdlpllkxsQJBAO+/sZXzZmjdBiP0xRQJJE9Jp6SonFxwqHXmRqK6Q84piOdAe9cnLxb+4v01UpndL8qHhtiEIOJV6+LPrb/GsW8CQQCZteacNMcYKsQqo+0vaCBTjqXYdbXbT8Pz7OMLTxctu/LuOepnl3IYLLQs+iW92n/Vw1LYBFd+7aNupNAk6xDzAkEA4DzqK3dBlNkNcjnwzsGSLXqViyONQ8S3O7bK4E7ZNo2Ql8KvUdg7agWyZuQlwvWnSoWiMQa7/xYgD77xIssDjwJAEdgEBW47DpsoWqrdBfvYhNqydgZ0Lhl8bfy5/r4Xur9u3CjtBUmXfSbzY6VGbFvJK0+ZdmpKnfmIV3fake6X8QJBANxWIz8EHPm0TWJAS6DSFLbo8XYbz5hcheUvRfEseE0TJoQDxW9RN5ikkpJusue3NaeeCuQe7RMZw4F440TuVy4=
  
  ```

  - 公私钥保存的标准

  公开密钥算法有一套标准叫Public Key Cryptgraphy Standards, 简称PKCS。 这套标准最早由RSA公司制定维护，目前交由标准化组织IETF(Internet Engineering Task Force)的 PKIX工作组来维护。这套标准从PKCS#1 到PKCS#14. 我们在对称加密算法的补位中就用到PKCS#5 和PKCS#7的部分标准。

  RSA的公钥是一般是以X.509标准的格式进行保存的，如上面的Java例子，` publicKey.getFormat()`的结果是`X.509`. 

  而私钥一般是以PKCS#8 (Private Key Information Syntax Standard)的格式保存. 如上面的例子中， `privateKey.getFormat()`的值为 PKCS#8.

  

- 加解密

  Java中security包已经带了RSA的实现，所以我们直接用Cipher类进行加载就行:

  ```java
   public static void main(String[] args) throws Exception {
      String publicKeyString = " MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQCP8+ujsUeg2hhgNSz1t8hoBHixoEy2tWVLo8Z22WC3LxqUfaGduvafIBlU9EYBU24ximn66N/AY4F9VzTKxVy3JmplIIiTptr+it5BMkJCO3YrsqPo6qKXHhpclvoc+YPfHB/8v13fmWlwI9aMCkI+mYF7m4V/gNouikBx8ZZcXQIDAQAB"
      String privateKeyString = "MIICdwIBADANBgkqhkiG9w0BAQEFAASCAmEwggJdAgEAAoGBAI/z66OxR6DaGGA1LPW3yGgEeLGgTLa1ZUujxnbZYLcvGpR9oZ269p8gGVT0RgFTbjGKafro38BjgX1XNMrFXLcmamUgiJOm2v6K3kEyQkI7diuyo+jqopceGlyW+hz5g98cH/y/Xd+ZaXAj1owKQj6ZgXubhX+A2i6KQHHxllxdAgMBAAECgYAeIJqsg6nODFcVq4thUblrq6Pm6PmlM4mjrv8WWKBZNk6FzVVJwZtj6j/i+8y68k8ZpzJPBPXvOeQb62htF6kziUniqfEa78eoIwUbyPeMW6iOnPz8cSvMbDaKfR4GO6IufNajDQBG+8093+ILyU4eZiH8+UVDfGT1pdlpllkxsQJBAO+/sZXzZmjdBiP0xRQJJE9Jp6SonFxwqHXmRqK6Q84piOdAe9cnLxb+4v01UpndL8qHhtiEIOJV6+LPrb/GsW8CQQCZteacNMcYKsQqo+0vaCBTjqXYdbXbT8Pz7OMLTxctu/LuOepnl3IYLLQs+iW92n/Vw1LYBFd+7aNupNAk6xDzAkEA4DzqK3dBlNkNcjnwzsGSLXqViyONQ8S3O7bK4E7ZNo2Ql8KvUdg7agWyZuQlwvWnSoWiMQa7/xYgD77xIssDjwJAEdgEBW47DpsoWqrdBfvYhNqydgZ0Lhl8bfy5/r4Xur9u3CjtBUmXfSbzY6VGbFvJK0+ZdmpKnfmIV3fake6X8QJBANxWIz8EHPm0TWJAS6DSFLbo8XYbz5hcheUvRfEseE0TJoQDxW9RN5ikkpJusue3NaeeCuQe7RMZw4F440TuVy4=";
      logger.info("generate {} bits public key, format {},  {}", publicKey.getModulus().bitLength(), publicKey.getFormat(), publicKeyString);
      logger.info("private key format {}, {}", privateKey.getFormat(),  privateKeyString);
  
      String message = "I am Coco Cola!";
      String cipherText = encrypt(publicKeyString, message);
      logger.info("plainText '{}' encrypted as: {}", message, cipherText);
      String plainText = decrypt(privateKeyString, cipherText);
      logger.info("cipherText '{}' decrypted as: {}", cipherText, plainText);
  
    }
  
  
    public static String encrypt(String publicKeyString, String message)
        throws  NoSuchAlgorithmException, InvalidKeySpecException, NoSuchPaddingException, InvalidKeyException, BadPaddingException, IllegalBlockSizeException {
      X509EncodedKeySpec publicKeySpec = new X509EncodedKeySpec(Base64.getDecoder().decode(publicKeyString));
      RSAPublicKey pubKey = (RSAPublicKey) KeyFactory.getInstance("RSA").generatePublic(publicKeySpec);
      Cipher cipher = Cipher.getInstance("RSA");
      cipher.init(Cipher.ENCRYPT_MODE, pubKey);
      byte[] encrypted = cipher.doFinal(message.getBytes());
      return new String(Base64.getEncoder().encode(encrypted));
    }
  
  
    public static String decrypt(String privateKeyString, String cipherText)
        throws  NoSuchAlgorithmException, InvalidKeySpecException, NoSuchPaddingException, InvalidKeyException, BadPaddingException, IllegalBlockSizeException {
      PKCS8EncodedKeySpec privateKeySpec = new PKCS8EncodedKeySpec(Base64.getDecoder().decode(privateKeyString));
      PrivateKey priKey = KeyFactory.getInstance("RSA").generatePrivate(privateKeySpec);
      Cipher cipher = Cipher.getInstance("RSA");
      cipher.init(Cipher.DECRYPT_MODE, priKey);
      byte[] decrypted = cipher.doFinal(Base64.getDecoder().decode(cipherText.getBytes()));
      return new String(decrypted);
    }
  ```



运行可得到如下结果：

```verilog
18:11:55.448 [main] INFO RSAUtil - plainText 'I am Coco Cola!' encrypted as: cnql6UxIv+4TFY8pjnfc+XhiOwzKZFeiTCvugDuKE23S22FFlil7WMlyBkwzw5lYHFjRAAEpwmlWUF5zoGoohEiUc9obcBPwWrwTc1YQ71Fe1vO9Xt5GzO0Dz+EcjocKMxCQ5JiRcbG/X3AGDgIWb/8J8tqn+NGh154y4tBeMk4=
18:11:55.464 [main] INFO RSAUtil - cipherText 'cnql6UxIv+4TFY8pjnfc+XhiOwzKZFeiTCvugDuKE23S22FFlil7WMlyBkwzw5lYHFjRAAEpwmlWUF5zoGoohEiUc9obcBPwWrwTc1YQ71Fe1vO9Xt5GzO0Dz+EcjocKMxCQ5JiRcbG/X3AGDgIWb/8J8tqn+NGh154y4tBeMk4=' decrypted as: I am Coco Cola!

```



#### RSA的使用及常见问题

- RSA的使用

由于RSA的原理是对明文计算e次幂(一般是65537次)，或者对密文进行d次幂的运算，再求模。所以性能回是一个很大的问题，特别当大数据量的运算的时候，性能确实不敢恭维。所以一般是用于比较重要的信息才采用RSA算法，比如对于对称密钥的加密，HTTPS RSA密码套件在进行3次握手的时候，客户端会生成一个临时密码，并用服务器端的公钥进行加密传给服务端，服务端收到加密的密钥之后，用它的私钥进行解密，从而得到对称加密的密钥。

还有一种就是对要发送内容计算hash1，并对hash的内容进行用自己的RSA私钥进行加密得到encryptedHash，然后把内容跟encryptedHash发给对方，接收方收到信息之后，统一对内容计算hash2值，同时用发送方的公钥解密接收到的encryptedHash值，再将解密得到的decryptedHash值跟计算得到的hash2值进行比较，如果相同就认为内容没有篡改过，而且是认定的发送方发的。这也就是一种签名算法的本质。

- 明文的大小限制

从RSA的加解密公式我们可以看出$C= M^e\quad mod\quad n$， $M= C^d\quad mod\quad n$ .所以解密的时候，算出的值是要去mod n，既然是n的余数，那就不能大于n。 如果明文的大于n,进行硬算，那么解密就会算错，算成求余后的值，比如n是91，而明文是95的话，那么解密后的值是4. 也就是所谓的回绕问题。所以明文必须小于n。 非对称密钥算法跟对称密钥算法一样，当明文内容小于n的时候，比如进行补位，而PKCS#1 (RSA Cryptograhy Standard)建议的补位是11个字节，所以 明文< n - 11*8。 比如公钥1024位，那么它能加密的明文大小为 (1024 - 88 )/ 8 = 117字节。



>  这篇文章我们来介绍另外一个重要的加密算法：非对称加密算法 (Asymmetric Cryptography), 也称为公开密钥加密算法 (Public Key Cryptography). 具体包括公开密钥算法概要，RSA算法原理，公私钥的生成，加解密的过程以及一些实践的问题，比如RSA加密的明文的位数为什么不能大于公钥的位数等



## RSA Padding

RSA的安全问题： 如果公钥的key长度为目前最长的4096位，但是指数e=3的情况下，就会出现下面的问题：

- `"No"` => `ASCII("No")` => `0x4E 0x6F` => `m=0x4E6F`.
- `c = POW(0x4E6F, 3) MOD n ` => `c = 0x75CCE07084F MOD n`
  - We know that `n` is a 4096-bit number (given) therefore it's larger than `0x75CCE07084F`, so `c=75CCE07084F`.
- 这时有人收到消息，直接开3次的平方根，就可以得到20079，转成十六进制，就可以得到`4E4F`, 从而得到原始消息 `No`

https://security.stackexchange.com/questions/183179/what-is-rsa-oaep-rsa-pss-in-simple-terms/183330#183330 



`00 02 [a bunch of non-zero random bytes] 00 [the message]`



还有问题就是 相同的消息加密出来的内容一样的



所以我们需要补位

- PKCS V1_5
- OEAP

## ECC 算法

### ECC 算法原理

### ECC公私钥对的生成

```Exception in thread "main" java.security.InvalidKeyException: Illegal key size or default parameters```

https://stackoverflow.com/questions/24907530/java-security-invalidkeyexception-illegal-key-size-or-default-parameters-in-and/24907555



## 密钥协商算法 DH

https://gist.github.com/ymnk/fec39e033394ee2ec47c

https://gist.github.com/wuyongzheng/0e2ed6d8a075153efcd3



## 公开密钥算法应用场景 

- 单步加密
- 双向加密
- 对称与非对称的配合
- 数字签名及衍生




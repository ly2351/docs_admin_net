---
sidebar_position: 13
---

# 13. 国密信创


:::tip 提示

完美适配国产化软硬件环境，支持国产中间件、国产数据库、麒麟操作系统、Windows、Linux部署使用；集成国密加解密插件，使用SM2、SM3、SM4等国密算法进行签名、数据完整性保护；软件层面全面遵循等级保护测评要求，完全符合等保、密评要求。
:::

:::tip 提示

Admin.NET 密码相关默认开启SM2国密加密传输和存储。MD5、SM2、SM4 任意切换。

有计划换掉 MD5 密码加密，采用 PBKDF2 加密。

var pbkdf2Hash = PBKDF2Encryption.Encrypt("Admin.NET"); // 加密

PBKDF2Encryption.Compare("Admin.NET", pbkdf2Hash); // 比较
:::

```json
  "Cryptogram": {
    "StrongPassword": false, // 是否开启密码强度验证
    "PasswordStrengthValidation": "(?=^.{6,16}$)(?=.*\\d)(?=.*\\W+)(?=.*[A-Z])(?=.*[a-z])(?!.*\\n).*$", // 密码强度验证正则表达式，必须须包含大小写字母、数字和特殊字符的组合，长度在6-16之间
    "PasswordStrengthValidationMsg": "密码必须包含大小写字母、数字和特殊字符的组合，长度在6-16之间", // 密码强度验证消息提示
    "CryptoType": "SM2", // 密码加密算法：MD5、SM2、SM4
    "PublicKey": "0484C7466D950E120E5ECE5DD85D0C90EAA85081A3A2BD7C57AE6DC822EFCCBD66620C67B0103FC8DD280E36C3B282977B722AAEC3C56518EDCEBAFB72C5A05312", // 公钥
    "PrivateKey": "8EDB615B1D48B8BE188FC0F18EC08A41DF50EA731FA28BF409E6552809E3A111" // 私钥
  }
```

系统已经封装 SM2、SM3、SM4 国密算法，具体调用方法如下：

1、GMUtil.SM2Encrypt() 进行SM2加密

2、GMUtil.SM2Decrypt() 进行SM2解密

3、GMUtil.SM4EncryptECB() 进行SM4加密（ECB）

4、GMUtil.SM4DecryptECB() 进行SM4解密（ECB）

5、GMUtil.SM4EncryptCBC() 进行SM4加密（CBC）

6、GMUtil.SM4DecryptCBC() 进行SM4解密（CBC）

具体实现类 Admin.NET.Core/Util/GM/GMUtil.cs

```csharp
/// <summary>
/// GM工具类
/// </summary>
public class GMUtil
{
    /// <summary>
    /// SM2加密
    /// </summary>
    /// <param name="publicKeyHex"></param>
    /// <param name="data_string"></param>
    /// <returns></returns>
    public static string SM2Encrypt(string publicKeyHex, string data_string)
    {
        // 如果是130位公钥，.NET使用的话，把开头的04截取掉
        if (publicKeyHex.Length == 130)
        {
            publicKeyHex = publicKeyHex.Substring(2, 128);
        }
        // 公钥X，前64位
        string x = publicKeyHex.Substring(0, 64);
        // 公钥Y，后64位
        string y = publicKeyHex.Substring(64);
        // 获取公钥对象
        AsymmetricKeyParameter publicKey1 = GM.GetPublickeyFromXY(new BigInteger(x, 16), new BigInteger(y, 16));
        // Sm2Encrypt: C1C3C2
        // Sm2EncryptOld: C1C2C3
        byte[] digestByte = GM.Sm2Encrypt(Encoding.UTF8.GetBytes(data_string), publicKey1);
        string strSM2 = Hex.ToHexString(digestByte);
        return strSM2;
    }

    /// <summary>
    /// SM2解密
    /// </summary>
    /// <param name="privateKey_string"></param>
    /// <param name="encryptedData_string"></param>
    /// <returns></returns>
    public static string SM2Decrypt(string privateKey_string, string encryptedData_string)
    {
        if (!encryptedData_string.StartsWith("04"))
            encryptedData_string = "04" + encryptedData_string;
        BigInteger d = new(privateKey_string, 16);
        // 先拿到私钥对象，用ECPrivateKeyParameters 或 AsymmetricKeyParameter 都可以
        // ECPrivateKeyParameters bcecPrivateKey = GmUtil.GetPrivatekeyFromD(d);
        AsymmetricKeyParameter bcecPrivateKey = GM.GetPrivatekeyFromD(d);
        byte[] byToDecrypt = Hex.Decode(encryptedData_string);
        byte[] byDecrypted = GM.Sm2Decrypt(byToDecrypt, bcecPrivateKey);
        string strDecrypted = Encoding.UTF8.GetString(byDecrypted);
        return strDecrypted;
    }

    /// <summary>
    /// SM4加密（ECB）
    /// </summary>
    /// <param name="key_string"></param>
    /// <param name="plainText"></param>
    /// <returns></returns>
    public static string SM4EncryptECB(string key_string, string plainText)
    {
        byte[] key = Hex.Decode(key_string);
        byte[] bs = GM.Sm4EncryptECB(key, Encoding.UTF8.GetBytes(plainText), GM.SM4_ECB_PKCS7PADDING);//NoPadding 的情况下需要校验数据长度是16的倍数. 使用 HandleSm4Padding 处理
        return Hex.ToHexString(bs);
    }

    /// <summary>
    /// SM4解密（ECB）
    /// </summary>
    /// <param name="key_string"></param>
    /// <param name="cipherText"></param>
    /// <returns></returns>
    public static string SM4DecryptECB(string key_string, string cipherText)
    {
        byte[] key = Hex.Decode(key_string);
        byte[] bs = GM.Sm4DecryptECB(key, Hex.Decode(cipherText), GM.SM4_ECB_PKCS7PADDING);
        return Encoding.UTF8.GetString(bs);
    }

    /// <summary>
    /// SM4加密（CBC）
    /// </summary>
    /// <param name="key_string"></param>
    /// <param name="iv_string"></param>
    /// <param name="plainText"></param>
    /// <returns></returns>
    public static string SM4EncryptCBC(string key_string, string iv_string, string plainText)
    {
        byte[] key = Hex.Decode(key_string);
        byte[] iv = Hex.Decode(iv_string);
        byte[] bs = GM.Sm4EncryptCBC(key, Encoding.UTF8.GetBytes(plainText), iv, GM.SM4_CBC_PKCS7PADDING);
        return Hex.ToHexString(bs);
    }

    /// <summary>
    /// SM4解密（CBC）
    /// </summary>
    /// <param name="key_string"></param>
    /// <param name="iv_string"></param>
    /// <param name="cipherText"></param>
    /// <returns></returns>
    public static string SM4DecryptCBC(string key_string, string iv_string, string cipherText)
    {
        byte[] key = Hex.Decode(key_string);
        byte[] iv = Hex.Decode(iv_string);
        byte[] bs = GM.Sm4DecryptCBC(key, Hex.Decode(cipherText), iv, GM.SM4_CBC_PKCS7PADDING);
        return Encoding.UTF8.GetString(bs);
    }

    /// <summary>
    /// 补足 16 进制字符串的 0 字符，返回不带 0x 的16进制字符串
    /// </summary>
    /// <param name="input"></param>
    /// <param name="mode">1表示加密，0表示解密</param>
    /// <returns></returns>
    private static byte[] HandleSm4Padding(byte[] input, int mode)
    {
        if (input == null)
        {
            return null;
        }
        byte[] ret = (byte[])null;
        if (mode == 1)
        {
            int p = 16 - input.Length % 16;
            ret = new byte[input.Length + p];
            Array.Copy(input, 0, ret, 0, input.Length);
            for (int i = 0; i < p; i++)
            {
                ret[input.Length + i] = (byte)p;
            }
        }
        else
        {
            int p = input[input.Length - 1];
            ret = new byte[input.Length - p];
            Array.Copy(input, 0, ret, 0, input.Length - p);
        }
        return ret;
    }
}
```

系统内置了获取国密公钥私钥对实现和接口 Admin.NET.Core/Service/Common/SysCommonService.cs，保证公匙前后端一致才行。接口地址 /api/sysCommon/smKeyPair

```csharp
    /// <summary>
    /// 获取国密公钥私钥对 🏆
    /// </summary>
    /// <returns></returns>
    [DisplayName("获取国密公钥私钥对")]
    public SmKeyPairOutput GetSmKeyPair()
    {
        var kp = GM.GenerateKeyPair();
        var privateKey = Hex.ToHexString(((ECPrivateKeyParameters)kp.Private).D.ToByteArray()).ToUpper();
        var publicKey = Hex.ToHexString(((ECPublicKeyParameters)kp.Public).Q.GetEncoded()).ToUpper();

        return new SmKeyPairOutput
        {
            PrivateKey = privateKey,
            PublicKey = publicKey,
        };
    }
```
    
前端公匙路径在 Web/.env。

```env
# port 端口号
VITE_PORT = 8888

# open 运行 npm run dev 时自动打开浏览器
VITE_OPEN = false

# 打包是否开启 cdn（源文件 utils/build.ts），可自行修改
VITE_OPEN_CDN = false

# public path 配置线上环境路径（打包）、本地通过 http-server 访问时，请置空即可
VITE_PUBLIC_PATH =

# SM公钥
VITE_SM_PUBLIC_KEY = "0484C7466D950E120E5ECE5DD85D0C90EAA85081A3A2BD7C57AE6DC822EFCCBD66620C67B0103FC8DD280E36C3B282977B722AAEC3C56518EDCEBAFB72C5A05312"
```

前端密码加密传输应用：Web/src/views/login/component/account.vue

```vue
   // SM2加密密码
   // const keys = SM2.generateKeyPair(); // 生成密匙对
   const publicKey = window.__env__.VITE_SM_PUBLIC_KEY;
   const password = sm2.doEncrypt(state.ruleForm.password, publicKey, 1);
```   
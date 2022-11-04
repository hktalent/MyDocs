# NTLM协议

NTLM，"NT  LAN  Manager".是一种网络认证协议，它是基于挑战（Chalenge）/响应（Response）认证机制的一种认证模式。

NTLM协议的认证， 包括 NTLMv1 和 NTLMv2 两个版本

这个协议只支持Windows

**NTLM协议的认证过程分为三步：**

- 协商
- 质询
- 验证

协商：主要用于确认双方协议版本，NTLM协议版本

质询：就是挑战（Chalenge）/响应（Response）认证机制起作用的范畴。

验证：验证主要是在质询完成后，验证结果。



**质询的完整过程**:

```
1.客户端向服务端

发送用户身份信息请求，将明文密码转化为NTLM HASH值发送给服务端

2.服务端接受请求后，在SAM数据库中验证判断是否存在对应的NTML HASH值。如果存在的话，生成一个16位的随机数"challenge",使用登录用户对应的NTLM HASH加密challenge，生成"challenge1"作为一个Net-NTLM HASH保存在内存中.并且将challenge发送给客户端

3.客户端在收到challenge后，使用登录用户的NTLM HSAH加密challenge，生成Response并将其发送给服务端进行验证
```



**NTLM认证大致流程:**

```
winlogon.exe    -> 接受用户输入   -> lsass.exe -> 认证
```

```
Windows Logon Process(即 winlogon.exe)，是Windows NT 用户登 陆程序，用于管理用户登录和退出。

LSASS用于微软Windows系统的安全机 制。它用于本地安全和登陆策略
```



用户登录时，操作系统通过winlogon.exe进程显示登录页面输入框，用户输入账号密码后，将密码传输给lsass进程将明文密码加密成NTLM HASH，然后与SAM中的HASH进行对比



**NTLM(V1/V2)的hash是存放在安全账户管理(SAM)数据库以及域控的NTDS.dit数据库中，获取该Hash值可以直接进行PtH攻击**



**NTML HASH的产生:**

假设我的密码是admin，那么操作系统会将admin转换为十六进制，经过Unicode转换后，再调用MD4加密算法加密，这个加密结果的十六进制就是NTLM Hash

```
admin -> hex(16进制编码) = 61646d696e
61646d696e -> Unicode = 610064006d0069006e00
610064006d0069006e00 -> MD4 = 209c6174da490caeb422f3fa5a7ae634
```



**windows系统下的hash密码格式**

```
用户名称:SID:LM-HASH值:NT-HASH值
```



**1)windows下Hash值生成原理**

**PS:可以看到LM Hash加密是对固定字符串“KGS!@#$%”进行DES加密(对称加密)，这也是可以破解的本质原因**

先将明文密码全部转换为大写，在将大写后的字符串转换为二进制字符串。(如果明文口令转换后的二进制字符串不足14字节的话，在其后添加0x00补成14个字节)

```
假设明文口令是“Welcome”，首先全部转换成大写“WELCOME”

再做将口令字符串大写转后后的字符串变换成二进制串： 
“WELCOME” -> 57454C434F4D4500000000000000
```



然后将其分割两组7字节的数据，分别经过str_to_key()函数进行处理得到两组8字节数据:

```
57454C434F4D45 -str_to_key()-> 56A25288347A348A
00000000000000 -str_to_key()-> 0000000000000000
```



然后使用这两组数据对魔术字符串"KGS!@#$%"进行DES加密

```
 "KGS!@#$%" -> 4B47532140232425
```

  56A25288347A348A -对4B47532140232425进行标准**DES加密**-> C23413A8A1E7665F

  0000000000000000 -对4B47532140232425进行标准**DES加密**-> AAD3B435B51404EE



最后拼接得到的LM HASH为：

```
C23413A8A1E7665FAAD3B435B51404EE
```




由于我们的密码不超过7字节，所以后面的一半是固定的:
- AA-D3-B4-35-B5-14-04-EE

并且根据LM Hash特征，也能够判断用户的密码是否是大于等于7位



**2）windows下NTLM Hash生成原理**

假设明文口令是“123456”，首先转换成Unicode字符串，与LM Hash算法不同，这次不需要添加0x00补足14字节

```
"123456" -> 310032003300340035003600
```

从ASCII串转换成Unicode串时，使用little-endian序，微软在设计整个SMB协议时就没考虑过big-endian序，ntoh*()、hton*()函数不宜用在SMB报文解码中。0x80之前的标准ASCII码转换成Unicode码，就是简单地从0x??变成0x00??。此类标准ASCII串按little-endian序转换成Unicode串，就是简单地在原有每个字节之后添加0x00。对所获取的Unicode串进行标准MD4单向哈希，无论数据源有多少字节，MD4固定产生128-bit的哈希值

然后对转换得到的Unicode字符串进行MD4单向哈希

```
310032003300340035003600 -> 32ED87BDB5FDC5E9CBA88547376818D4
```



就得到了最后的NTLM HASH：

NTLM Hash: 32ED87BDB5FDC5E9CBA88547376818D4



**与LM Hash算法相比，明文口令大小写敏感，无法根据NTLM Hash判断原始明文口令是否小于8字节，摆脱了魔术字符串"KGS!@#$%"。MD4是真正的单向哈希函数，穷举作为数据源出现的明文，难度较大**



**NTLM V2协议**

NTLM v1与NTLM v2最显著的区别就是Challenge与加密算法不同，共同点就是加密的原料都是NTLM Hash。

下面细说一下有什么不同:

- Challage:NTLM v1的Challenge有8位，NTLM v2的Challenge为16位。
- Net-NTLM Hash:NTLM v1的主要加密算法是DES，NTLM v2的主要加密算法是HMAC-MD5


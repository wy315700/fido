
### 通用的协议

#### Version

```java
dictionary Version {
 int mj; // 主要的
 int mn; // 次要的
}
```

#### Operation Header

```java
dictionary OperationHeader {
 Version upv; // UAF协议版本，必须是1.0
 DOMString op; //操作码
 DOMString appID;
 DOMString serverData;
 Extension[] exts;
}
```

其中：

  * **upv** UAF协议版本号，必须是1.0
  * **op** FIDO操作码
  * **appID** 必须是一个HTTPS的URL，FIDO客户端通过appID获取可使用的FacetID
  * **serverData** 是一个RP创建的回话ID，以及其他客户端难懂的数据。服务器使用MAC校验来保证serverData的完整性
  * **exts** UAF消息扩展的列表

#### Type of Authenticator Attestation ID (AAID)

唯一确定一个认证器型号，例如1234#4567

```java
typedef DOMString AAID; // string[9]
```

  * *AAID* ：每一个认证器必须有一个全局确认的AAID来唯一标识其型号。只有相同厂商，相同型号的认证器才可以使用相同的AAID。
    * *AAID* 是一个类似"V#M"的字符串，其中
      * "#" 是分隔符
      * "V" 代表了认证器的提供商，是一个4位的16进制数字
      * "M" 代表了认证器的型号，是一个4位的16进制数字
    * 其中的16进制数字是大小写不敏感的，如"03EF"和"03ef"是一样的
    * 当设备商对认证器进行改变以后，如修复BUG，更新固件，必须使用一个新的AAID

#### Type of KeyID

```java
typedef DOMString KeyID; // base64url(byte[32...2048])
```

  * *KeyID* 唯一代表了用户的私钥，由认证器生成并且在FIDO服务器端注册
  * 在一个认证器向一个FIDO服务器注册的时候，( *AAID* , *KeyID* ) 二元组必须是唯一的
  * *KeyID* 使用base64url编码并且被包装在UAF消息体里
  * *KeyID* 可以被没有存储器的认证器使用，这些认证器生成私钥以后会存储在FIDO服务器上
  * 在认证的时候FIDO服务器提供 *KeyID* 来告诉认证器使用对应的私钥并且进行签名

#### Type of ServerChallenge

由服务器产生的一个随机挑战值

```java
typedef DOMString ServerChallenge; // base64url(byte[8...64])
```
  * *ServerChallenge* 是服务器生成的一个随机挑战值，用于防止重放攻击
  * *ServerChallenge* 在生成的时候必须使用通过认证的伪随机生成数算法
  * *ServerChallenge* 最小需要8字节，推荐使用20字节
  * *ServerChallenge* 最长是64字节的，为了方便使用SHA-512的运算结果

#### Type of FinalChallengeParams

```java
dictionary FinalChallengeParams {
    DOMString appID;
    ServerChallenge challenge;
    DOMString facetID;
    TLSData tlsData;
}
```

  * *appID* 是从Operation Header里获取的
  * *challenge* 即服务器挑战值
  * *facetID* 是由客户端决定的用于选择用户代理的一个数据
  * *tlsData* 是客户端发送到服务器端的TLS信息

#### Type of TLSData

```java
dictionary TLSData {
    DOMString serverEndPoint; // 必须, not empty. base64url
    DOMString tlsServerCertificate; // Optional, not empty if present
    DOMString tlsUnique; // 必须, not empty. base64url
    DOMString cid_pubkey; // Optional, base64url encoded JwsKey
}
```

  * *TLSData* 包含了频道绑定信息，用于防止中间人攻击
  * *serverEndPoint* 必须支持：
    * 服务器证书的base64url编码字符串的Hash值，其中Hash函数按照以下规则使用
      * 如果证书的签名算法里提供的是MD5或者SHA-1，则使用SHA-256
      * 如果证书签名算法里使用的是除MD5或者SHA-1的其他算法，则使用证书指定的算法
      * 如果证书没有指定算法，则行为是未知的
    * 如果获取不到服务器证书或者无法指定算法，则设置为"None"
  * *tlsServerCertificate* 是可选的
    * 如果服务器证书不可用，则必须设置为"None"
    * 当且仅当客户端决定忽略服务器证书的时候忽略这个值
    * 如果该值有效，则必须是base64url编码的DER格式的TLS证书
  * *tlsUnique* 是base64url编码的TLS channel Finished结构
  * *cid_pubkey* 
    * 如果TLS协议栈没有提供ChannelID,则无需设置
    * 如果对应的ChannelID没有被服务器认可，则必须设置为None

更多要求：
  1. 如果TLS Channel ID数据被浏览器或者应用程序使用，则应该被FIDO客户端使用
  2. TLS Channel ID应该被FIDO服务器端支持
  3. 如果TLS 绑定数据对FIDO客户端可见，则应该被FIDO客户端使用

#### Type of JwkKey

```java
dictionary JwkKey {
  DOMString kty;
  DOMString crv;
  DOMString x;
  DOMString y;
}
```

#### Type of Extension

```java
dictionary Extension {
    DOMString id; // 必须. string[1..32].
    DOMString data; // 必须. base64url(byte[1..8192])
    boolean fail_if_unknown;// 必须.
}
```

#### Type of TrustedApps

```java
dictionary TrustedApps {
Version version; // 必须.
DOMString[] ids; // 必须. Each list element is string[1..512].
}
```

比如

```json
{
    "alg": "B64S256",
    "ids": [ "https://login.acme.com/",
    "android:apk-key-hash:2jmj7l5rSw0yVb/vlWAYkK/YBwk",
    "ios:bundle-id:com.acme.app" ]
}
```
需要注意的是：

  * 如果使用一个浏览器用户代理，则对应的facetID为一个HTTPS的URL，并且以/结尾
  * 如果使用一个IOS的用户代理，则对应的facetID为改程序的BundleID
  * 如果使用一个Android的用户代理，则对应的facetID为APK的公钥，使用以下方法提取

    `keytool -exportcert -alias androiddebugkey -keystore <path-to-apk-sign- ing-keystore> &>2 /dev/null | openssl sha1 -binary | openssl base64 | sed 's/=//g'`

#### Type of Policy

```java
dictionary MatchCriteria {
  AAID aaid; // Optional
  KeyID[] keyIDList; // Optional
  unsigned long long authenticationFactor; // Optional, set of bit flags
  unsigned long long keyProtection; // Optional, set of bit flags
  unsigned long long keyProtection; // Optional, set of bit flags
  unsigned long long secureDisplay; // Optional, set of bit flags
  unsigned long[] supportedAuthAlgs; // Optional
  DOMString[] supportedSchemes; // Optional
  Extension[] exts; // Optional
}

dictionary Policy {
  MatchCriteria[][] accepted;
  MatchCriteria[] disallowed;
}
```

  * *MatchCriteria* 结构包含以下信息
    * 如果只允许指定的AAID，则设置 *aaid* 字段
    * 如果只允许指定的KeyID列表，则设置 *keyIDList* 字段
    * authenticationFactor, keyProtection, keyProtection，secureDisplay是位数据
    * *supportedSchemes* 里包含了使用KeyRegistrationData和SignedData中支持的编码模式
    * *exts* 里包含了扩展信息
  * *Policy* 结构里包含了若干种允许的认证器和不允许的认证器
  ## TODO

例如

```json
{
    "accepted": [ [ {"authenticationFactor": 0x02, "attachment": 0x01},
                    {"authenticationFactor": 0x10, "attachment": 0x01}]
        ]
]

```

FIDO 客户端在执行Policy的步骤的时候的规则
  


### 注册操作

从server到client的是request,反之为response

#### Type of RegisterRequest

```java
dictionary RegisterRequest {
  OperationHeader header;// 必须,
  ServerChallenge challenge;// 必须.
  DOMString username;// 必须, string[1..128].
  Policy policy;// 必须
}

object[] uafRegisterRequest; // 必须, not more than one element per version

```

  * RegisterRequest 包含了一个注册请求
    * *header* 包含了请求头信息，其中的op字段被设置成"Reg"
    * *challenge* 即服务器挑战值
    * *username* 是用户在RP上注册的用户名
    * *policy* 定义了支持的认证器信息

例如：

```json
[{
    "header": { "op": "Reg", "upv": { "mj": 1, "mn": 0 }, "appID": "https://mycorp.-
com/fido"},
    "challenge": "qwudh827hddbawd8qbdqj3bduq3duq56t324zwasdq4wrt",
    "username":"banking_personal",
    "policy": {
         "accepted": [[{
            "authenticationFactor": 00000000000001ff,
            "keyProtection": 000000000000000e,
            "attachment": 00000000000000ff,
            "secureDisplay": 000000000000001e,
            "supportedSchemes": "UAFV1TLV"}]],
         "disallowed": {"aaid": "1234#5678"}
    }
}]
```

#### Type of RegisterResponse

```java
dictionary AuthenticatorRegistrationAssertion {
    AAID aaid; // 必须.
    DOMString attestationCertificateChain; // Optional. base64url(byte[1..])
    DOMString scheme;
    DOMString krd;
    Extension [] exts;
}
dictionary RegisterResponse {
    OperationHeader header; // 必须, OperationHeader.op must be “Reg” 
    DOMString fcParams; // 必须, base64url encoded FinalChallengeParams 
    AuthenticatorRegistrationAssertion[] assertions; // 必须.
}

RegisterResponse uafRegisterResponse;
```

  * *AuthenticatorRegistrationAssertion* 包含了注册响应的断言信息
    * *aaid* 指注册的时候的认证器的AAID
    * *attestationCertificateChain* 包含了认证器的证书链，但是不包含根证书
    * *scheme* 包含了编码KRD的模式
    * *krd* 是KeyRegistrationData，包含了使用认证器的私钥签名的用户公钥
    * *exts* 包含了扩展信息
  * *RegisterResponse* 里包含了注册响应信息
    * *header* 即服务器发送的请求头
    * *fcParams* 是使用base64url编码的FinalChallengeParams数据
    * *assertions* 包含了多个断言信息

#### Processing Rules

FIDO服务器生成注册请求的时候的步骤：

  * 生成适当的policy p
    * 对于每个认证器 a：
      * 生成MatchCriteria结构变量m
      * 如果 *m.aaid* 被设置了，那么 *keyID* , *attachment* 和 *exts* 必须被设置
      * 如果 *m.aaid* 没有设置，那么 *m.supportedAuthAlgs* 和 *m.supportedSchemes* 必须设置
      * 被禁用的认证器必须要在m.disallowed里设置
      * 允许的认证器必须在m.accepted里设置
  * 生成RegisterRequest变量r
  * 生成一个随机挑战值，写入到r.challenge里
  * 将p写入到r.policy里
  * 将r写入到uafRegisterRequest数组里
  * 发送给FIDO客户端

FIDO客户端解析注册请求的时候的步骤为：

  * 选择主版本号是1，次版本号是0的消息m
  * 解析消息m
    * 如果必要的UAF消息字段没有被设置或者不正确的被设置，则拒绝此次请求
  * 显示可用的认证器给用户选择，其中必须去除被禁用的认证器
  * 根据AppID，获取可信的应用的FacetID
    * 如果应用的FacetID没有在信任列表里，则拒绝此次操作
  * 如果可用，请求TLS数据
  * 计算FinalChallengeParams变量fcp，并且设置 *fcp.appID* , *fcp.challenge* , *fcp.facetID* 和 *fcp.tlsData* 
    * 计算方法： FinalChallenge = base64url(serialize(utf8encode(fcp)))
  * 对于所有符合UAF协议版本和用户同意的认证器：
    * 生成对应的ASMRequest数据
    * 将ASMRequest数据发送给ASM

FIDO客户端生成注册响应的步骤

  * 生成一个 *uafRegisterResponse* 数据
  * 将 *uafRegisterRequest.header* 复制到 *uafRegisterResponse.header*
  * 将 *FinalChallenge* 赋值给 *uafRegisterResponse.fcParams* 
  * 将认证器的响应添加到 uafRegisterResponse.assertions
  * 将uafRegisterResponse发送给FIDO服务器

FIDO服务器解析注册响应的步骤

  * 解析响应消息
    * 如果协议版本不支持，拒绝操作
    * 如果必要的字段没有被设置或者设置错误，拒绝操作
  * 验证 *uafRegisterResponse.header.serverData* 的完整性
  * 使用base64url解码 *uafRegisterResponse.fcParams* 并且转换成一个对象 fcp
  * 验证 *fcp* 里的各种字段
    * 确认 *fcp.appID* 符合服务器存储的值
    * 确认 *fcp.challenge* 是服务器生成的而且没过期
    * 确认 *fcp.facetID* 在信任列表里
    * 确认 *fcp.tlsData* 是符合要求的
    * 如果有任何一项不满足要求，则拒绝操作
  * 对于 *uafRegisterResponse.assertions* 里的断言a
    * 从认证器元数据里查找对应的认证算法
    * 从 *a.krd* 提取TLV数据并且保证包含了所有的必要数据
    * 计算 *uafRegisterResponse.fcParams* 数据的hash值FCHash，在认证器的元数据里寻找hash算法。
    * 保证 *a.krd.FinalChallenge* == FCHash
      * 如果失败，则处理下一个断言
    * 如果该AAID的元数据里的AttestationRootCertificate包含至少一个元素
      * 获取 *a.krd.Certificate* 并且和 *a.attestationCertificateChain* 联系起来
      * 从 *a.aaid* 的元数据里获取所有的 *AttestationRootCertificate*
      * 从根证书开始校验 *krd.Certificate*
        * 如果失败，则处理下一个断言
      * 使用 *krd.Certificate* 校验 *krd.Signature*
        * 如果失败，则处理下一个断言
    * 如果 *AttestationRootCertificate* 是空的
      * 使用 *krd.PublicKey* 检验 *krd.Signature*
        * 如果校验失败，则处理下一个断言
    * 检验 *a.aaid* == *krd.AAID*
      * 如果失败，则处理下一个断言
    * 保证成功通过以上选项的断言满足提供的 Policy
      * 如果失败，则处理下一个断言
  * 对于每一个通过认证的断言a
    * 将 *a.krd.PublicKey* ，*a.krd.KeyID* , *a.krd.SignCounter* , *a.krd.authenticatorVersion* , *a.krd.AAID* 存储并且和用户身份联系在一起，但是如果二元组 （AAID, KeyID）已经存在，则操作失败。



### 认证操作

#### Type of AuthenticationRequest

```java
dictionary Transaction {
  DOMString contentType;// 必须
  DOMString content;// 必须. base64url(byte[1..8192])
}
dictionary AuthenticationRequest {
    OperationHeader header;// 必须, header.op must be “Auth”
    ServerChallenge challenge;// 必须
    Transaction transaction;// Optional
    Policy policy;// 必须
}

object[] uafAuthRequest; // 必须, not more than one element per version
```

  * *Transaction* 包含了FIDO服务器提供的信息
    * *contentType* 应该是"text/plain"或者"image/png"
      * 如果是"text/plain"，则 *content* 最多包含200个ASCLL字符
    * *content* 包含了对应的数据
  * *AuthenticationRequest* 包含了UAF认证请求消息
    * *header* 包含了请求头，header.op 必须是 “Auth”
    * *challenge* 包含了服务器生成的挑战值
    * *transaction* 包含了让用户确认的消息
    * *policy* 包含了对应的规则

例如

```json
[{
    "header": {"op": "Auth", "upv": { "mj": 1, "mn": 0 }, "appID":
"https://mycorp.com/fido"},
    "challenge": "triz786ighwer8764g6574234515reg45z",
    "policy": {
         "accepted": [[{
             "authenticationFactor": 00000000000001ff,
             "keyProtection": 000000000000000e,
             "attachment": 00000000000000ff,
             "secureDisplay": 000000000000001e,
             "supportedSchemes": "UAFV1TLV"}]],
         "disallowed": {"aaid": "1234#5678"}
    }
}]
```

#### Type of AuthenticationResponse

```java
dictionary AuthenticatorSignAssertion {
 AAID aaid;// 必须
 KeyID keyID;// 必须
 DOMString scheme;// 必须, e.g. “UAFV1TLV”
 DOMString signedData;// 必须. base64url(byte[1..4096])
 Extension[] exts;// Optional
}
dictionary AuthenticationResponse {
    OperationHeader header; // 必须, header.op must be “Auth”
    DOMString fcParams; // 必须, base64url encoded FinalChallengeParams
    AuthenticatorSignAssertion[] assertions; // 必须
}
```

  * *AuthenticatorSignAssertion* 包含了认证器生成的认证响应断言
    * *aaid* 表示认证器的AAID
    * *keyID* 包含了代表Uauth.priv的KeyID
    * *scheme* 包含了编码 *signedData* 的格式
    * *signedData* 包含了使用Uauth.priv签名的数据
    * *exts* 包含了认证器的扩展
  * *AuthenticationResponse* 包含了认证响应
    * *header* 必须和认证请求一致
    * *fcParams* 是使用base64url编码的FinalChallengeParams数据
    * *assertions* 包含了多个断言信息

例如

```json
{
    "header": {"op": "Auth", "upv": { "mj": 1, "mn": 0 }},
    "fcParams": "eyJhcHBJRCI6Imh0dHBzOi8vbXljb3JwLmNvbS9maWRvIiwgImNoYWxsZW5nZSI6I-jU0Njk4emhmZGtzamdoODc2dWpoZ2hqNyIsICJmYWNldElEIjoiYW5kcm9pZDphcGsta2V5LWhhc2g6Mmpta- jdsNXJTdzB5VmIvdmxXQVlrSy9ZQndrIiwgInRsc0RhdGEiOiIifQ",
    "assertions": [
        {"AAID":"1234#abcd", "keyID": "1234def...", "scheme": "UAFV1TLV",
            "signedData": "..."},
        {"AAID":"1234#abce", "keyID": "fa73fg...", "scheme": "UAFV1TLV",
            "signedData":"..."}]
}
```

FIDO服务器生成认证请求的步骤：

* 生成一个随机的挑战值
* 生成适当的policy
      * 如果 *MatchCritera.aaid.aaid* 被设置了，那么 *keyID* , *attachment* 和 *exts* 必须被设置
      * 如果 *MatchCritera.aaid* 没有设置，那么 *supportedAuthenticationAlgs* 和 *supportedSchemes* 必须设置
      * 如果是两步验证，则 *Policy.accepted* 里的值必须包含认证器的 AAID 和 KeyID
* 生成UAF认证请求数据
* 发送给FIDO客户端

FIDO客户端执行认证请求的步骤：

  * 选择主版本号是1，次版本号是0的消息m
  * 解析消息m
    * 如果必要的UAF消息字段没有被设置或者不正确的被设置，则拒绝此次请求
  * 显示可用和允许的认证器给用户选择
  * 如果 *AuthRequest.policy.accepted* 是空的，则用户可以使用任意注册的认证器进行操作
  * 根据AppID，获取可信的应用的FacetID
    * 如果应用的FacetID没有在信任列表里，则拒绝此次操作
  * 如果可用，请求TLS数据
  * 计算FinalChallengeParams变量fcp，并且设置 *fcp.appID* , *fcp.challenge* , *fcp.facetID* 和 *fcp.tlsData* 
    * 计算方法： FinalChallenge = base64url(serialize(utf8encode(fcp)))
  * 对于所有符合UAF协议版本和用户同意的认证器：
    * 生成对应的ASMRequest数据，其中包含AppID, FinalChallenge, KeyID, Transaction Text等数据
    * 将ASMRequest数据发送给ASM

FIDO服务器解析认证响应的步骤

  * 解析响应消息
    * 如果协议版本不支持，拒绝操作
    * 如果必要的字段没有被设置或者设置错误，拒绝操作
  * 验证 *uafRegisterResponse.header.serverData* 的完整性
  * 使用base64url解码 *uafRegisterResponse.fcParams* 并且转换成一个对象 fcp
  * 验证 *fcp* 里的各种字段
    * 确认 *fcp.appID* 符合服务器存储的值
    * 确认 *fcp.challenge* 是服务器生成的而且没过期
    * 确认 *fcp.facetID* 在信任列表里
    * 确认 *fcp.tlsData* 是符合要求的
    * 如果有任何一项不满足要求，则拒绝操作
  * 对于 *uafRegisterResponse.assertions* 里的断言
    * 根据 *AuthResponse.assertions.keyID* 取出 Uauth.pub
      * 如果没有对应的数据，则处理下一个断言
    * 使用注册的时候的数据库，检验 *AuthResponse.assertions.aaid*
      * 如果失败，则处理下一个断言
    * 从认证器元数据里取出对应的算法
    * 解析 *AuthResponse.assertions.signedData* 并且保证包含所有的必要字段
    * 检验 Sign Counter，保证认证器支持并且增长
      * 如果没有增长，则处理下一个断言
    * 计算 *uafRegisterResponse.fcParams* 数据的hash值FCHash，在认证器的元数据里寻找hash算法。
    * 保证 *a.krd.FinalChallenge* == FCHash
      * 如果失败，则处理下一个断言
    * 如果 *authenticationMode* == 2
      * 保证RP端有缓存的transaction
        * 如果没有，则处理下一个断言
      * 使用和处理FinalChallenge相同的算法对 *cachedTransaction* 数据计算hash值
        * *cachedTransHash* = hash( *cachedTransaction* )
      * 确保 *cachedTransHash* == *signedData*
        * 如果失败，则处理下一个断言
    * 使用Uauth.pub和对应的算法检验SignedData里的签名
      * 如果签名失败，则处理下一个断言
  * 保证通过检验的断言满足最初的规则
    * 如果不满足，拒绝此次响应


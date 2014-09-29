
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

#### Type of Authenticator Attestation ID (AAID)

唯一确定一个认证器，例如1234#4567

```java
typedef DOMString AAID; // string[9]
```

#### Type of KeyID

```java
typedef DOMString KeyID; // base64url(byte[32...2048])
```

#### Type of ServerChallenge

由服务器产生的一个随机挑战值

```java
typedef DOMString ServerChallenge; // base64url(byte[8...64])
```

#### Type of FinalChallengeParams

```java
dictionary FinalChallengeParams {
    DOMString appID;
    ServerChallenge challenge;
    DOMString facetID;
    TLSData tlsData;
}
```

#### Type of TLSData

```java
dictionary TLSData {
    DOMString serverEndPoint; // Mandatory, not empty. base64url
    DOMString tlsServerCertificate; // Optional, not empty if present
    DOMString tlsUnique; // Mandatory, not empty. base64url
    DOMString cid_pubkey; // Optional, base64url encoded JwsKey
}
```

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
    DOMString id; // Mandatory. string[1..32].
    DOMString data; // Mandatory. base64url(byte[1..8192])
    boolean fail_if_unknown;// Mandatory.
}
```

#### Type of TrustedApps

```java
dictionary TrustedApps {
Version version; // Mandatory.
DOMString[] ids; // Mandatory. Each list element is string[1..512].
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

#### Type of Policy

```java
dictionary MatchCriteria {
  AAID aaid; // Optional
  KeyID[] keyIDList; // Optional
  unsigned long long authenticationFactor; // Optional, set of bit flags
  unsigned long long keyProtection; // Optional, set of bit flags
  unsigned long long attachment; // Optional, set of bit flags
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

例如

```json
{
    "accepted": [ [ {"authenticationFactor": 0x02, "attachment": 0x01},
                    {"authenticationFactor": 0x10, "attachment": 0x01}]
        ]
]

```

### 注册操作

从server到client的是request,反之为response

#### Type of RegisterRequest

```java
dictionary RegisterRequest {
  OperationHeader header;// Mandatory, header.op must be “Reg”
  ServerChallenge challenge;// Mandatory.
  DOMString username;// Mandatory, string[1..128].
  Policy policy;// Mandatory
}

object[] uafRegisterRequest; // Mandatory, not more than one element per version

```

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
    AAID aaid; // Mandatory.
    DOMString attestationCertificateChain; // Optional. base64url(byte[1..])
    DOMString scheme;
    DOMString krd;
    Extension [] exts;
}
dictionary RegisterResponse {
    OperationHeader header; // Mandatory, OperationHeader.op must be “Reg” 
    DOMString fcParams; // Mandatory, base64url encoded FinalChallengeParams 
    AuthenticatorRegistrationAssertion[] assertions; // Mandatory.
}

RegisterResponse uafRegisterResponse;
```

### 认证操作

#### Type of AuthenticationRequest

```java
dictionary Transaction {
  DOMString contentType;// Mandatory
  DOMString content;// Mandatory. base64url(byte[1..8192])
}
dictionary AuthenticationRequest {
    OperationHeader header;// Mandatory, header.op must be “Auth”
    ServerChallenge challenge;// Mandatory
    Transaction transaction;// Optional
    Policy policy;// Mandatory
}

object[] uafAuthRequest; // Mandatory, not more than one element per version
```

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
 AAID aaid;// Mandatory
 KeyID keyID;// Mandatory
 DOMString scheme;// Mandatory, e.g. “UAFV1TLV”
 DOMString signedData;// Mandatory. base64url(byte[1..4096])
 Extension[] exts;// Optional
}
dictionary AuthenticationResponse {
    OperationHeader header; // Mandatory, header.op must be “Auth”
    DOMString fcParams; // Mandatory, base64url encoded FinalChallengeParams
    AuthenticatorSignAssertion[] assertions; // Mandatory
}
```

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



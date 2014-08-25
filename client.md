## UAF Application API and Transport Binding Specification
==============

### UAF Status Codes

  * 1200 操作成功完成
  * 1202 等待RP完成操作
  * 1400 未知消息
  * 1401 未认证的用户
  * 1403 未允许的用户
  * 1404 未发现
  * 1408 请求超时
  * 1480 未知的AAID
  * 1481 未知的KeyID
  * 1490 频道绑定被拒绝
  * 1491 不合法的请求
  * 1492 无法连接认证器
  * 1493 过期的认证器
  * 1494 无法获取的Key
  * 1495 无法获取的算法
  * 1496 无法获取的尝试
  * 1497 无法获取客户端兼容性
  * 1498 无法获取内容
  * 1500 服务器内部错误

### DOM API

#### UAFMessage Dictionary

```java
Dictionary UAFMessage {
    DOMString uafProtocolMessage;
    Object additionalData;
}
```

#### UAFResponseCallback

```java
callback UAFResponseCallback = void (UAFMessage uafResponse);
```

#### ErrorCallback

```java
callback ErrorCallback = void (ErrorCode code);
interface ErrorCode {
    const short NO_ERROR = 0x0;
    FIDO UAF Application API and Transport Binding Specification
    const short WAIT_USER_ACTION
    const short INSECURE_TRANSPORT
    const short USER_CANCELLED
    const short UNSUPPORTED_VERSION
    const short NO_SUITABLE_AUTHENTICATOR = 0x5;
    const short PROTOCOL_ERROR const short UNTRUSTED_FACET_ID const short UNKNOWN
}
```

#### notifyUAFResult Operation

```java
void notifyUAFResult(int responseCode, DOMString uafResponse);
```

#### Version Interface

```java
Interface Version {
    readonly attribute int majorVersion; 
    readonly attribute int minorVersion;
}
```

#### Authenticator Interface

```java
interface Authenticator {
    readonly attribute DOMString AAID;
    readonly attribute DOMString description;
    readonly attribute DOMString logo;

    readonly attribute Version[] supportedUAFVersions;
    readonly attribute long userVerification;
    readonly attribute long keyProtection;
    readonly attribute long attachmentHint;
    readonly attribute long secureDisplay;
    
    readonly attribute int authenticationAlgorithm;
    readonly attribute DOMString assertionScheme;

    // for future use
    readonly attribute long additionalInfo;
    // See FIDO UAF Registry of Predefined Values for constant definitions 
}
```

#### Discovery Interface

```java
interface Discovery {
    readonly attribute Version[] supportedUAFVersions; readonly attribute DOMString clientVendor;
    readonly attribute Version clientVersion;
    readonly attribute Authenticator[] availableAuthenticators; void checkPolicy(DOMString message, ErrorCallback cb);
}
```

#### FIDOClient Interface

```java
interface FIDOClient {
    void processUAFOperation(
        UAFMessage          message,
        UAFResponseCallback completionCallback,
        ErrorCallback       errorCallback
    );
}
```

### Android API

#### IUAFClient.aidl

```java
package org.fidoalliance.uaf.client;
import org.fidoalliance.uaf.client.IUAFErrorCallback;
import org.fidoalliance.uaf.client.IUAFResponseCallback;
import org.fidoalliance.uaf.client.Discovery;
import org.fidoalliance.uaf.client.UAFMessage;
    
import java.util.Map;

interface IUAFClient
{
    Discovery getDiscovery();

    void notifyUAFResult(in int    responseCode,
                         in String uafResponse);
    void processUAFMessage(
        in UAFMessage msg,
        in String     origin,
        in Map        channelBindings,
        in boolean    checkPolicy,
        in IUAFResponseCallback cb,
        in IUAFErrorCallback errorCb
    );
```

#### IUAFErrorCallback.aidl

```java
package org.fidoalliance.uaf.client;

interface IUAFErrorCallback
{
    void response(long code);
}
```

#### IUAFResponseCallback.aidl

```java
package org.fidoalliance.uaf.client;

import org.fidoalliance.uaf.client.UAFMessage;

interface IUAFResponseCallback
{
    void response(in UAFMessage uafResponse);
}
```

#### UAFMesage.aidl

```java
package org.fidoalliance.uaf.client;

parcelable UAFMessage;
```

#### UAFMessage.java

```java
package org.fidoalliance.uaf.client;

import android.os.Parcel;
import android.os.Parcelable;

public class UAFMessage implements Parcelable {

    public String uafProtocolMessage;
    public String additionalData;
    public static final Parcelable.Creator<UAFMessage> CREATOR
        = new Parcelable.Creator<UAFMessage>() {
            public UAFMessage createFromParcel(Parcel in) {
                return new UAFMessage(in);
            }
            public UAFMessage[] newArray(int size) {
                return new UAFMessage[size];
            }
    };
    public UAFMessage() {
    }

    private UAFMessage(Parcel in) { 
        uafProtocolMessage = in.readString();
        additionalData = in.readString(); 
    }
        
    @Override
    public int describeContents() {
        return 0; 
    }
        
    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeString(uafProtocolMessage);
        dest.writeString(additionalData); }
    }
```

#### Version.aidl

```java
package org.fidoalliance.uaf.client;

parcelable Version;
```

#### Version.java

```java
package org.fidoalliance.uaf.client;
    
import java.util.ArrayList;
import java.util.List;

import android.os.Parcel; 
import android.os.Parcelable;

public class Version implements Parcelable {

    public int majorVersion;
    public int minorVersion;

    public static final Parcelable.Creator<Version> CREATOR 
        = new Parcelable.Creator<Version>() {
            public Discovery createFromParcel(Parcel in) { 
                return new Version(in);
            }
            public Version[] newArray(int size) { 
                return new Version[size];
            }
        };
    public Version() {
    }
    
    private Version(Parcel in) {
        majorVersion = in.readInt();
        minorVersion = in.readInt();
    }
    
    @Override
    public int describeContents() {
        return 0;
    }
    
    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeInt(majorVersion);
        dest.writeInt(minorVersion);
    }
}
```

#### Discovery.aidl

```java
package org.fidoalliance.uaf.client;
    
parcelable Discovery;
```

#### Discovery.java

```java
package org.fidoalliance.uaf.client; 718
    
import java.util.ArrayList;
import java.util.List;
   
import android.os.Parcel;
import android.os.Parcelable;
  
public class Discovery implements Parcelable {

    public List<Version> supportedUAFVersions =
        new ArrayList<Version>();
    public String  clientVendor;
    public Version clientVersion    

    public List<Authenticator> availableAuthenticators = 
                                   new ArrayList<Authenticator>(); 

    public static final Parcelable.Creator<Discovery> CREATOR = new Parcelable.Creator<Discovery>() {
        public Discovery createFromParcel(Parcel in) { 
            return new Discovery(in);
        }
        public Discovery[] newArray(int size) { 
            return new Discovery[size];
        } 
    };  

    public Discovery() { 
    }   

    private Discovery(Parcel in) { 
        in.readTypedList(supportedUAFVersions,Version.CREATOR); 
        clientVendor = in.readString();
        clientVersion = in.readParcellable(null); 
        in.readTypedList(availableAuthenticators,Authenticator.CREATOR);
    }   

    @Override
    public int describeContents() {
        return 0; 
    }   

    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeTypedList(version); 
        dest.writeString(clientVendor); 
        dest.writeParcelable(clientVersion, 0); 
        dest.writeTypedList(availableAuthenticators);
    } 
}
```

#### Authenticator.aidl

```java
package org.fidoalliance.uaf.client;
parcelable Authenticator;
```

#### Authenticator.java

```java
package org.fidoalliance.uaf.client;

import java.util.List;
import java.util.ArrayList;

import android.os.Parcel;
import android.os.Parcelable;

public class Authenticator implements Parcelable {
    public String AAID;
    public String description;
    public String logo;
    public long   userVerification;
    public long   keyProtection;
    public long   attachmentHint;
    public long   secureDisplay;
    public String assertionScheme;
    public long additionalInfo;
    public int authenticationAlgorithm;
    public List<Version> supportedUAFVersions;
        
    public static final Parcelable.Creator<Authenticator> CREATOR
        = new Parcelable.Creator<Authenticator>() {
            
        public Authenticator createFromParcel(Parcel in) {
            return new Authenticator(in);
        }

        public Authenticator[] newArray(int size) {
            return new Authenticator[size];
        }
    };
    private Authenticator(Parcel in) {
        AAID             = in.readString();
        description      = in.readString();
        logo             = in.readString();
        userVerification = in.readString();
        keyProtection    = in.readString();
        attachmentHint   = in.readString();
        secureDisplay    = in.readString();
        assertionScheme  = in.readString();
        additionalInfo   = in.readLong();   

        authenticationAlgorithm = in.readInt();
        in.readTypedList(supportedUAFVersions,
            Version.CREATOR);
    }
                
    @Override
    public int describeContents() {
        return 0;
    }
        
    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeString(AAID);
        dest.writeString(description);
        dest.writeString(logo);
        dest.writeLong(userVerification);
        dest.writeLong(keyProtection);
        dest.writeLong(attachmentHint);
        dest.writeLong(secureDisplay);
        dest.writeString(assertionScheme);
        dest.writeLong(additionalInfo);
        dest.writeInt(authenticationAlgorithm);
        dest.writeTypedArray(supportedUAFVersions);
    }
}
```

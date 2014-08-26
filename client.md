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

是一个原生的UAF协议的一个包装

```java
Dictionary UAFMessage {
    //最原始的UAF协议字符串
    DOMString uafProtocolMessage;
    
    //额外的信息，是一个JSON结构
    Object additionalData;
}
```

#### UAFResponseCallback

FIDO客户端提供的，供application异步回调的接口

```java
callback UAFResponseCallback = void (UAFMessage uafResponse);
```

#### ErrorCallback

当发生错误的时候的回调方法

```java
callback ErrorCallback = void (ErrorCode code);
interface ErrorCode {
    //没有错误被收集，当收到这个消息以后，程序不应该继续等待UAF响应
    const short NO_ERROR                  = 0x0;

    //未使用https或者是页面里有不安全的元素
    const short INSECURE_TRANSPORT        = 0x1;

    //等待用户操作
    const short WAIT_USER_ACTION          = 0x2;
    
    //用户取消
    const short USER_CANCELLED            = 0x3;
    
    //不支持的版本
    const short UNSUPPORTED_VERSION       = 0x4;

    //未发现受支持的认证器
    const short NO_SUITABLE_AUTHENTICATOR = 0x5;
    
    //协议消息错误
    const short PROTOCOL_ERROR            = 0x6;
    
    //不信任的FacetID
    const short UNTRUSTED_FACET_ID        = 0x7;

    //未知错误
    const short UNKNOWN                   = 0xFF;
}
```

#### notifyUAFResult Operation

UAF应用告诉FIDO客户端它收到的响应码和响应内容

```java
void notifyUAFResult(int responseCode, DOMString uafResponse);
```

#### Version Interface

描述协议版本信息

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
    //认证器的人性化描述
    readonly attribute DOMString description;
    //认证器的Logo，使用data: url [RFC2397]存储
    readonly attribute DOMString logo;

    //表示认证器支持的UAF协议
    readonly attribute Version[] supportedUAFVersions;

    //一组比特标志，表示认证器支持的用户认证方式，使用常数USER_VERIFY_表示
    readonly attribute long userVerification;
    //一组比特标志，表示认证器使用的密钥保护办法，使用常数KEY_PROTECTION表示
    readonly attribute long keyProtection;
    //一组比特标志，表示认证器连接FIDO客户端软件的方法，使用常数ATTACHMENT_HINT表示
    readonly attribute long attachmentHint;
    //一组比特标志，表示安全显示的种类，使用常数SECURE_DISPLAY_表示
    readonly attribute long secureDisplay;
    
    //表示认证器支持的算法
    readonly attribute int authenticationAlgorithm;
    //认证器使用的编码方式
    readonly attribute DOMString assertionScheme;

    // for future use
    readonly attribute long additionalInfo;
    // See FIDO UAF Registry of Predefined Values for constant definitions 
}
```

#### Discovery Interface

用于发现用户的客户端软件和设备是否支持UAF协议以及认证器的兼容性问题

```java
interface Discovery {
    //客户端支持的UAF协议，按照优先级排序
    readonly attribute Version[] supportedUAFVersions; 
    //客户端供应商
    readonly attribute DOMString clientVendor;
    //客户端的版本
    readonly attribute Version clientVersion;
    //可用的认证器
    readonly attribute Authenticator[] availableAuthenticators; 
    //在不需要用户允许的情况下，询问FIDO插件是否支持对应的消息，如果支持，必须返回NO_ERROR
    void checkPolicy(DOMString message, ErrorCallback cb);
}
```

#### FIDOClient Interface

应用可以异步的发送UAF请求消息给客户端，并且等待返回

```java
interface FIDOClient {
    void processUAFOperation(
        //客户端使用的UAF消息
        UAFMessage          message,
        //接收返回的调用
        UAFResponseCallback completionCallback,
        //发生错误的时候的调用
        ErrorCallback       errorCallback
    );
}
```

### Android API

#### IUAFClient.aidl

安卓设备和FIDO客户端的主要接口,

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

需要添加以下权限

```xml
<permission
    android:name="org.fidoalliance.uaf.permissions.ACT_AS_WEB_BROWSER"
    android:label="Act as a browser for FIDO registrations."
    android:description="This application may act as a web browser,
        creating new and accessing existing FIDO registrations for anydomain."
    android:protectionLevel="dangerous" />
```

#### IUAFErrorCallback.aidl

FIDO客户端返回错误信息使用的接口

```java
package org.fidoalliance.uaf.client;

interface IUAFErrorCallback
{
    void response(long code);
}
```

#### IUAFResponseCallback.aidl

*processUAFMessage* 调用成功返回以后使用的回调函数

如果 *checkPolicy* 被设置了，那么该方法不会被调用，只有 *IUAFErrorCallback* 会被调用

```java
package org.fidoalliance.uaf.client;

import org.fidoalliance.uaf.client.UAFMessage;

interface IUAFResponseCallback
{
    void response(in UAFMessage uafResponse);
}
```

#### UAFMesage.aidl

*UAFMessage.java* 的包装

```java
package org.fidoalliance.uaf.client;

parcelable UAFMessage;
```

#### UAFMessage.java

应用程序交互的主要消息，被用于双边交互，应用与FIDO客户端，应用与FIDO服务器端。
使用 *additionalData* 来包含应用程序相关的信息和协议扩展
应用程序必须测试消息内容是否合法

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

*Version.java* 的包装

```java
package org.fidoalliance.uaf.client;

parcelable Version;
```

#### Version.java

描述主版本号和次版本号

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

*Discovery.java* 的包装

```java
package org.fidoalliance.uaf.client;
    
parcelable Discovery;
```

#### Discovery.java

应用使用该类来查询用户设备上存在的FIDO客户端和FIDO认证器，详情见 *Discovery Interface*

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

*Authenticator.java* 的包装

```java
package org.fidoalliance.uaf.client;
parcelable Authenticator;
```

#### Authenticator.java

表示在 *Discovery* 里使用的认证器实例的元数据

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

# Flutter 中的 TLS/TLS 证书

Flutter 中进行 https 通信时 tls/ssl 证书的校验跟 Android/iOS 原生平台有很大的不同，有必要了解它以避免一些认知的错误。

这里会通过跟踪代码的方式查看 Flutter/Dart 中 TLS/TLS 证书管理、校验的实现原理，包括下面的几个方面：

1. 介绍证书相关的一些基本概念；

2. Flutter/Dart 中尽心 TLS/TLS 证书校验时使用的 Root CAs 来自哪里？来自系统、还是自己内置？

3. Flutter/Dart 如何进行 ssl pinning；

4. Flutter/Dart 如何进行 ssl 的双向认证

   

## 几个基本概念

要了解 Flutter 中进行 ```tls/ssl``` 校验时使用的是哪些 ***Root CAs***，需要先明确一些概念。

### Certicate 证书

证书是个比较宽泛的概念，在 TLS/SSL 下的证书一般指的就是我们从 CA 那里申请的证书，这个语境下的证书的格式是 X509 格式的。

#### X509

[X.509](https://zh.wikipedia.org/wiki/X.509) 是密码学里公钥证书的格式标准，主要定义了证书中应该包含哪些内容。SSL 使用的就是这种证书标准。

每个X.509证书都包含一个 公钥, 电子签名，以及有关与证书及其颁发者相关的身份的信息认证中心（CA）。

#### 证书编码格式

同样的 X.509 证书，可能有不同的编码格式。目前有以下两种编码格式：

- **PEM** - Privacy Enhanced Mail

打开看文本格式，以 ```-----BEGIN...``` 开头、```-----END``` 结尾，内容是BASE64编码。

查看PEM格式证书的信息：

````shell
openssl x509 -in certificate.pem -text -noout
````

Apache 和 *NIX 服务器偏向于使用这种编码格式。

- **DER** - Distinguished Encoding Rules

打开看的话是二进制格式。

查看DER格式证书的信息：

```````shell
openssl x509 -in certificate.der **-inform der** -text -noout
```````

Java和Windows服务器偏向于使用这种编码格式。

> 两种证书可以相互转换：

- **PEM转为DER** 

  ````shell
  openssl x509 -in cert.crt -outform der -out cert.der
  ````

- **DER转为PEM** 

  ```` shell
  openssl x509 -in cert.crt -inform der -outform pem -out cert.pem
  ````

#### 证书扩展名

证书有两种编码格式 **PEM** 和 **DER**，但编码格式不是文件扩展名，常见的证书扩展名有：***crt*** 和 ***cer***。

***crt*** 常见于 *NIX 系统，大多是 PEM 编码；

***cer*** 常见于 Windows 系统，大多是 DER 编码。

#### 常见的相关文件

- key 文件

  单纯的用来存放公钥或私钥的，例如 *NIX 中的 ```~/.ssh/id_rsa.pub``` 保存的就是 rsa 的公钥。

- csr 文件

  Certificate Signing Request,即证书签名请求,这个并不是证书，而是向权威证书颁发机构获得签名证书的申请，其核心内容是一个公钥(当然还附带了一些别的信息)，在生成这个申请的时候，同时也会生成一个私钥，私钥要自己保管好。iOS App 开发中使用的证书就是通过这种格式向苹果申请的。

  可以用 ```openssl req -noout -text -in my.csr``` 查看文件的信息。如果是DER格式的话加上参数 ```-inform der```。

#### 归档文件 Archive

前面的编码格式或者文件大多都是跟证书相关的，有时候我们需要把证书和私钥妥善的放在一个文件中，常见的方式就是 PKCS #12 和 JKS。

- [PKCS #12](https://zh.wikipedia.org/wiki/PKCS_12)

  PKCS 表示 Public Key Cryptography Standards

  在密码学中，PKCS #12 定义了一种归档文件格式，用于实现存储许多加密对象在一个单独的文件中。通常用它来打包一个私钥及有关的 X.509 证书，或者打包信任链的全部项目。

  PKCS #12 文件扩展名一般为 ".p12 "或者 ".pfx"，在 Apple 的生态中大量使用。

- [JKS](https://en.wikipedia.org/wiki/Java_KeyStore)

  Java KeyStore 的概念和作用同 PKCS #12 几乎是等同的，只不过是更多的用在 Java 相关的场景中。例如给 apk 签名的证书就是保存在 Java KeyStore 中。

  > A Java KeyStore (JKS) is a repository of security certificates – either authorization certificates or public key certificates – plus corresponding private keys, used for instance in TLS encryption.

  JKS 文件的扩展名一般为 jks 或者 keystore。

很明显，不管是 JKS 文件还是 PKCS #12 文件，由于包含了私钥信息，都应该是有密码保护的。

### Root CAs，issuer，leaf

证书 PKI 中起作用的一个关键环节就是得有个地方来验证一个证书的有效性，这个地方就是 CA: Certificate Authority。

CA 本质上也是一个证书，普通证书有效性的校验标准就是看它是否是 CA 这个特殊证书签发的（签发即用 CA 特殊证书的私钥进行签名）。

在实际运行中，

1. CA 不止一家，有多家 CA
2. 为了安全性和灵活性，CA 不会直接签发证书，而是 CA 签发一些中间证书，用这些中间证书去签发各个网站的证书

这样就有了一个证书链：CA 证书 -> 中间证书 -> 网站证书，相应的也产生了一些通用的叫法：

- 证书链 certificate chain

- 叶子证书

  网站证书处于最末端，往往也被称为***叶子证书(leaf)***

- 中间证书

  证书链中间的证书就叫做中间证书，中间证书可能不止一个。中间证书也被称为 issuer certificate

- 根证书

  最根部的证书就是根证书 Root Certificate，因为往往它是一个 CA 证书，所以也被称为 Root CA，又因为 CA 不止一家，所以 Root CAs。

当然也可以不牵涉 CA 进来，证书链上的根证书是自签证书。

操作系统一般会内置一批 Root CAs，需要校验证书的一方可以通过比对所要校验的证书是否是由这些 Root CAs 中的一个签发的来校验证书的有效性；有的 Application 甚至会自己内置一批 Root CAs 来完成证书的校验，例如 Firefox。

## HttpClient

在 Flutter 中进行 http 操作，不管是使用 ```dio``` 还是使用官方的 ```package http```，最终干活的 http client 都还是 flutter engine 的```dart:io``` package (准确说是 ```dart:io``` 中的 ```dart:_http```) 中的 ```HttpClient```

下面是一个使用  ```HttpClient``` 的示例代码：

````dart
		HttpClient client = new HttpClient();
    client.getUrl(Uri.parse("http://www.example.com/"))
        .then((HttpClientRequest request) {
          // Optionally set up headers...
          // Optionally write to the request object...
          // Then call close.
          ...
          return request.close();
        })
        .then((HttpClientResponse response) {
          // Process the response.
          ...
        });
````

关于  ```HttpClient``` 还有几个需要注意的地方：

- ```HttpClient``` 的代码位于 flutter sdk 的  ```bin/cache/pkg/sky_engine/lib/_http/http.dart``` 或 dart sdk 的 ```sdk/lib/_http/http.dart``` ，```http.dart``` 不单包含  ```HttpClient```  的实现，还包含了  ```HttpServer``` 的实现。

  >为什么叫 sky_engine？

  Flutter 最初的开发代号就叫 Sky，因此 Flutter engine 的核心 package 就叫 ```sky_engine```。

  根据 ```sky_engine``` 的 README：

  This package describes the ```dart:ui``` library, which is the interface between Dart and the Flutter Engine.

  可见，```sky_engine``` 就是 Flutter 最核心的  ```dart:ui```  *[library](https://api.flutter.dev/flutter/dart-ui/dart-ui-library.html)* 。

  >```dart:ui``` 的内容

  除了这里的 ```dart:io``` 以外， ```dart:ui``` *[library](https://api.flutter.dev/flutter/dart-ui/dart-ui-library.html)* 还包含下面的内容：

  Flutter 的：[animation](https://api.flutter.dev/flutter/animation/animation-library.html)、[material](https://api.flutter.dev/flutter/material/material-library.html)、[cupertino](https://api.flutter.dev/flutter/cupertino/cupertino-library.html)、[gestures](https://api.flutter.dev/flutter/gestures/gestures-library.html)、[widgets](https://api.flutter.dev/flutter/widgets/widgets-library.html) 等；

  Dart 的：[dart:async](https://api.flutter.dev/flutter/dart-async/dart-async-library.html)、[dart:collection](https://api.flutter.dev/flutter/dart-collection/dart-collection-library.html)、[dart:math](https://api.flutter.dev/flutter/dart-math/dart-math-library.html)、[dart:ffi](https://api.flutter.dev/flutter/dart-ffi/dart-ffi-library.html)、[dart:isolate](https://api.flutter.dev/flutter/dart-isolate/dart-isolate-library.html)。

  实际上，Flutter SDK 的第一个 commit 的 message 就是 Open the Sky ：

  ````shell
  commit 00882d626a478a3ce391b736234a768b762c853a
  Author: Adam Barth <abarth@chromium.org>
  Date:   Thu Oct 23 11:15:41 2014 -0700
  
      Open the Sky
  ````

  

- ```abstract HttpClient```

  ```HttpClient``` 是 abstract 的，实际上``` http.dart```中的 ```HttpClient```只是定义了一个接口/协议，具体的实现是在 ```http_impl.dart``` 中的 ```_HttpClient```。

  ```dart
  factory HttpClient({SecurityContext? context}) {
    HttpOverrides? overrides = HttpOverrides.current;
    if (overrides == null) {
      return new _HttpClient(context);
    }
    return overrides.createHttpClient(context);
  }
  ```

- Closing ```HttpClient```

  The ```HttpClient``` supports persistent connections and caches network connections to reuse them for multiple requests whenever possible. This means that network connections can be kept open for some time after a request has completed. Use ```HttpClient.close``` to force the HttpClient object to shut down and to close the idle network connections.

- Proxy

  By default the HttpClient uses the proxy configuration available from the environment, see findProxyFromEnvironment. To turn off the use of proxies set the findProxy property to null.

  ```dart
  HttpClient client = new HttpClient();
  client.findProxy = null;
  ```

  Tbd

## SecurityContext

 ```HttpClient``` 的构造函数有个重要的 named parameter：```SecurityContext```：

```dart
factory HttpClient({SecurityContext? context}) {}
```

```SecurityContext ``` 的作用就是用来管理  ```HttpClient``` 进行 Secure Connection 时使用的证书的。

> SecurityContext  API Doc:
>
> The object containing the certificates to trust when making a secure client connection, and the certificate chain and private key to serve from a secure server.

如上描述，```SecurityContext``` 就是用来管理证书的，```SecurityContext``` 的构造函数以及它提供的 API 也说明了这一点：

```dart
abstract class SecurityContext {
  factory SecurityContext({bool withTrustedRoots = false});
  external static SecurityContext get defaultContext;
  
  void usePrivateKey(String file, {String? password});
  void usePrivateKeyBytes(List<int> keyBytes, {String? password});
  
  void setTrustedCertificates(String file, {String? password});
  void setTrustedCertificatesBytes(List<int> certBytes, {String? password});
  
  void useCertificateChain(String file, {String? password});
  void useCertificateChainBytes(List<int> chainBytes, {String? password});
  
  void setClientAuthorities(String file, {String? password});
  void setClientAuthoritiesBytes(List<int> authCertBytes, {String? password});
}
```

可以看出， ```SecurityContext``` 通过构造函数确定是否使用 ***内置证书***，通过 API 提供添加各种格式的证书的功能。

> 这里的 ***内置证书*** 表示 Flutter 在各个平台上默认使用的 Root CAs，也是本文讨论的重点。下同



**到这里，不难猜到 Flutter 在不同平台上使用的 Root CAs 的差异肯定就在 ```SecurityContext``` 的具体实现里面有所体现了。**

### SecurityContext 的实现

```SecurityContext``` 也是 abstract 的：

```dart
abstract class SecurityContext {
  external factory SecurityContext({bool withTrustedRoots = false});
  external static SecurityContext get defaultContext;
  //...
}
```

它的具体实现是 ```dart sdk``` 的 ``` sdk/lib/_internal/vm/bin/secure_socket_patch.dart``` 中的 ```_SecurityContext```：

````dart
@patch
class SecurityContext {
  @patch
  factory SecurityContext({bool withTrustedRoots: false}) {
    return new _SecurityContext(withTrustedRoots);
  }

  @patch
  static SecurityContext get defaultContext {
    return _SecurityContext.defaultContext;
  }

  @patch
  static bool get alpnSupported => true;
}
````

> 这里的 ```external``` 关键字和 dart sdk 中的 ```@patch``` 应该是通过某种方式关联起来了，但具体是如何工作的呢？

继续跟踪  ```_SecurityContext``` 的实现：

````dart
class _SecurityContext extends NativeFieldWrapperClass1
    implements SecurityContext {
  // skip irrelevant codes
  void usePrivateKey(String file, {String? password}) {
    List<int> bytes = (new File(file)).readAsBytesSync();
    usePrivateKeyBytes(bytes, password: password);
  }
  void usePrivateKeyBytes(List<int> keyBytes, {String? password})
      native "SecurityContext_UsePrivateKeyBytes";

  void setTrustedCertificatesBytes(List<int> certBytes, {String? password})
      native "SecurityContext_SetTrustedCertificatesBytes";

  void useCertificateChainBytes(List<int> chainBytes, {String? password})
      native "SecurityContext_UseCertificateChainBytes";

  void setClientAuthoritiesBytes(List<int> authCertBytes, {String? password})
      native "SecurityContext_SetClientAuthoritiesBytes";

  void _trustBuiltinRoots() native "SecurityContext_TrustBuiltinRoots";
}
````

可以看出  ```_SecurityContext``` 只是一个简单的 wrapper，它的功能实现都是通过 native extension 在 native 层实现的。

###  native 层的实现

Native 层的实现在 Dart SDK 的  ``` .//runtime/bin/security_context.cc``` 中，并且依据平台不同，有不同的实现：

```shell
.//runtime/bin/security_context.cc
.//runtime/bin/security_context_macos.cc
.//runtime/bin/security_context_android.cc
.//runtime/bin/security_context_win.cc
.//runtime/bin/security_context_fuchsia.cc
.//runtime/bin/security_context_linux.cc
```

到这里， ```HttpClient``` 及 ```SecurityContext``` 实现链条已经清晰了。

### Recap

在 Flutter 中进行 http 操作，使用的是 ```dart:io```  (准确说是 ```dart:io``` 中的 ```dart:_http```) 中的 ```HttpClient```， ```HttpClient``` 的具体的实现是在 ```http_impl.dart``` 中的 ```_HttpClient```。

 ```HttpClient``` 的构造函数有唯一一个 named parameter：```SecurityContext```，```SecurityContext ``` 就是用来管理  ```HttpClient``` 进行 Secure Connection 时使用的证书的。

因此只要在搞清楚 ```SecurityContext``` 每个功能的具体实现，就能弄清晰 Flutter 中 Root CAs 的问题了。

并且从命名中不难看出， ```SecurityContext``` 的构造函数确定的是默认使用的  Root CAs ；API 用来手动添加  Root CAs 。下面就逐次分析 ```SecurityContext``` 的构造函数（默认使用的   Root CAs ）和 API（手动添加   Root CAs ）：



## 默认 Root CAs

``` dart
factory SecurityContext({bool withTrustedRoots = false});
```

如前所诉， ```SecurityContext``` 的具体实现是 ```dart sdk``` 的 ``` sdk/lib/_internal/vm/bin/secure_socket_patch.dart``` 中的 ```_SecurityContext```：

```dart
class _SecurityContext extends NativeFieldWrapperClass1 implements SecurityContext {
  _SecurityContext(bool withTrustedRoots) {
    _createNativeContext();
    if (withTrustedRoots) {
      _trustBuiltinRoots();
    }
  }
  
  void _createNativeContext() native "SecurityContext_Allocate";
  
  void _trustBuiltinRoots() native "SecurityContext_TrustBuiltinRoots";
}
```

可以看出，```_SecurityContext ```的构造函数就是在 native 层 allocate 一个对象，然后在```withTrustedRoots``` 为 ```true``` 的时候 ```_trustBuiltinRoots``` ，可以想到 native 层的 ```_trustBuiltinRoots```  的实现就是默认使用的 Root CAs 确定的地方。来看看。

如前所述， ```SecurityContext``` 在 native 层的实现是因平台而异的：

````shell
.//runtime/bin/security_context.cc
.//runtime/bin/security_context_macos.cc
.//runtime/bin/security_context_android.cc
.//runtime/bin/security_context_win.cc
.//runtime/bin/security_context_fuchsia.cc
.//runtime/bin/security_context_linux.cc
````

```SecurityContext_TrustBuiltinRoots``` 的实现是通过 ```SSLCertContext``` 的 ```TrustBuiltinRoots``` 来实现的：

````c++
void FUNCTION_NAME(SecurityContext_TrustBuiltinRoots)(
    Dart_NativeArguments args) {
  SSLCertContext* context = SSLCertContext::GetSecurityContext(args);

  ASSERT(context != NULL);

  context->TrustBuiltinRoots();
}
````

下面分平台看看 ```TrustBuiltinRoots```  的实现：

### Android

````c++
void SSLCertContext::TrustBuiltinRoots() {
  // First, try to use locations specified on the command line.
  if (root_certs_file() != NULL) {
    LoadRootCertFile(root_certs_file());
    return;
  }
  if (root_certs_cache() != NULL) {
    LoadRootCertCache(root_certs_cache());
    return;
  }

  // On Android, we don't compile in the trusted root certificates. Insead,
  // we use the directory of trusted certificates already present on the device.
  // This saves ~240KB from the size of the binary. This has the drawback that
  // SSL_do_handshake will synchronously hit the filesystem looking for root
  // certs during its trust evaluation. We call SSL_do_handshake directly from
  // the Dart thread so that Dart code can be invoked from the "bad certificate"
  // callback called by SSL_do_handshake.
  const char* android_cacerts = "/system/etc/security/cacerts";
  LoadRootCertCache(android_cacerts);
  return;
}
````

先来看两个参数 ```root_certs_file``` 和 ```root_certs_cache```，这两个是 Dart VM command line 启动时传入的两个参数，在 Dart SDK 的 ```runtime/bin/main_options.cc``` 中有详细说明：

````markdown
"--root-certs-file=<path>\n"
"  The path to a file containing the trusted root certificates to use for\n"
"  secure socket connections.\n"
"--root-certs-cache=<path>\n"
"  The path to a cache directory containing the trusted root certificates to\n"
"  use for secure socket connections.\n"
````

`--root-certs-file` 表示存放证书文件的路径；

`--root-certs-cache`  表示存放一堆证书的文件夹的路径。

这两个参数在所有平台上都有同样的含义，后面就不再赘述。

如果这两个参数不为空，则按照优先顺序使用相应的 Root CAs，其它的 Root CAs 都会被忽略。

在移动端平台上，这两个参数都为空。

这样，在 Android 平台上，默认的 Root CAs 就是 ```/system/etc/security/cacerts``` 下的内置证书。

这个目录存放的就是 Android 系统内置的根证书，可读不可写。

也就是说，在 Android 平台上，Flutter App 使用的 默认 Root CAs 就是系统内置的根证书，***但是不包括用户安装的证书***。

#### network_config.xml

Flutter 不支持 Android 7.0 后引入的 network_config.xml 配置文件对网络特性的配置。

### macOS/iOS

````c++
void SSLCertContext::TrustBuiltinRoots() {
  // First, try to use locations specified on the command line.
  if (root_certs_file() != NULL) {
    LoadRootCertFile(root_certs_file());
    return;
  }
  if (root_certs_cache() != NULL) {
    LoadRootCertCache(root_certs_cache());
    return;
  }
}
````

可以看出，在 macOS/iOS 平台上，没有使用任何的默认证书。那么在 macOS/iOS 平台上，是如何进行 ssl/tls 的证书校验的呢？

Https 的通信依赖 SecureSocket，在 Flutter 上 SecureSocket 连接的建立是在 native 层完成的：

````c++
// dart-lang-sdk/runtime/bin/secure_socket_filter.cc
void FUNCTION_NAME(SecureSocket_Connect)(Dart_NativeArguments args) {
  // Skip irrelevant codes
  SSLCertContext* context = NULL;
  if (!Dart_IsNull(context_object)) {
    ThrowIfError(Dart_GetNativeInstanceField(
        context_object, SSLCertContext::kSecurityContextNativeFieldIndex,
        reinterpret_cast<intptr_t*>(&context)));
  }
  GetFilter(args)->Connect(host_name, context, is_server,
                           request_client_certificate,
                           require_client_certificate, protocols_handle);
}
````

``` SSLFilter``` 的 ```connect``` 方法：

````c++
void SSLFilter::Connect(const char* hostname,
                        SSLCertContext* context,
                        bool is_server,
                        bool request_client_certificate,
                        bool require_client_certificate,
                        Dart_Handle protocols_handle) {
  // Skip irrelevant codes
  ssl_ = SSL_new(context->context());
  SSL_set_bio(ssl_, ssl_side, ssl_side);
  SSL_set_mode(ssl_, SSL_MODE_AUTO_RETRY);  // TODO(whesse): Is this right?
  SSL_set_ex_data(ssl_, filter_ssl_index, this);
  // 这里调用了 SSLCertContext 的 RegisterCallbacks 方法
  context->RegisterCallbacks(ssl_);
  // Skip irrelevant codes
}
````

重点就是 ```SSLCertContext``` 的 ```RegisterCallbacks``` 方法。

在除了 macOS/iOS 的其他所有平台上，这个方法的实现都是空实现：

````c++
void SSLCertContext::RegisterCallbacks(SSL* ssl) {
  // No callbacks to register for implementations using BoringSSL's built-in
  // verification mechanism.
}
````

但在 macOS/iOS 上，证书的校验就是在这里完成的：

````c++
void SSLCertContext::RegisterCallbacks(SSL* ssl) {
  SSL_set_custom_verify(ssl, SSL_VERIFY_PEER, CertificateVerificationCallback);
}
````

```SSL_set_custom_verify``` 是 `boringssl/openssl` 提供的 [API](https://commondatastorage.googleapis.com/chromium-boringssl-docs/ssl.h.html#SSL_set_custom_verify)：

```c++
OPENSSL_EXPORT void SSL_set_custom_verify(
    SSL *ssl, int mode,
    enum ssl_verify_result_t (*callback)(SSL *ssl, uint8_t *out_alert));
```

> **BoringSSL**
>
> SSL 只是一种协议，而 OpenSSL 是它的一种实现，并且 OpenSSL 还提供很多好用的相关工具。
>
> BoringSSL 是 Google 基于 OpenSSL 开发的项目，虽是开源项目，但是由于 API 变动频繁、主要是目的支持内部项目特性，因此 Google 并不建议其他厂家使用。
>
> Google 家的产品的 SSL/TLS 的实现基本都是使用的 BoringSSL ，包括不限于 Dart，Chrome 等。
>
> macOS/iOS 中也内置了 BoringSSL 。

#### 证书校验方式

```SSL_set_custom_verify``` 的第二个参数 mode 表示校验的方式，在 boringssl 中定义有四种校验方式：

- **SSL_VERIFY_NONE**

On a client, verifies the server certificate but does not make errors fatal. 

On a server it does not request a client certificate. 

This is the default.

```
#define SSL_VERIFY_NONE 0x00
```

- **SSL_VERIFY_PEER**

On a client, makes server certificate errors fatal.

On a server it requests a client certificate and makes errors fatal. However, anonymous clients are still allowed. 

```
#define SSL_VERIFY_PEER 0x01
```

- **SSL_VERIFY_FAIL_IF_NO_PEER_CERT** 

Configures a server to reject connections if the client declines to send a certificate. This flag must be used together with `SSL_VERIFY_PEER`, otherwise it won't work.

```
#define SSL_VERIFY_FAIL_IF_NO_PEER_CERT 0x02
```

- **SSL_VERIFY_PEER_IF_NO_OBC** 

Configures a server to request a client certificate if and only if Channel ID is not negotiated.

```
#define SSL_VERIFY_PEER_IF_NO_OBC 0x04
```

在 Flutter/Dart 中，选用的校验方式是 **SSL_VERIFY_PEER**。

#### 证书校验回调

```SSL_set_custom_verify``` 的第三个参数是证书校验的回调，boringssl 会调用这个 callback 来让 callback 完成证书的校验，callback 的返回值是  `ssl_verify_ok` 或 `ssl_verify_invalid` 。

> The callback should return `ssl_verify_ok` if the certificate is valid. If the certificate is invalid, the callback should return `ssl_verify_invalid` and optionally set `*out_alert` to an alert to send to the peer. Some useful alerts include `SSL_AD_CERTIFICATE_EXPIRED`, `SSL_AD_CERTIFICATE_REVOKED`, `SSL_AD_UNKNOWN_CA`, `SSL_AD_BAD_CERTIFICATE`, `SSL_AD_CERTIFICATE_UNKNOWN`, and `SSL_AD_INTERNAL_ERROR`. See RFC 5246 section 7.2.2 for their precise meanings. If unspecified, `SSL_AD_CERTIFICATE_UNKNOWN` will be sent by default.

在 Flutter 的 macOS/iOS 平台上，callback 的实现为 `CertificateVerificationCallback`，这又是一个比较复杂的实现：

````c++
static ssl_verify_result_t CertificateVerificationCallback(SSL* ssl,
                                                           uint8_t* out_alert) {

  /// 从 ssl 中取出 Flutter 的证书相关的上下文
  SSLFilter* filter = static_cast<SSLFilter*>(
      SSL_get_ex_data(ssl, SSLFilter::filter_ssl_index));
  SSLCertContext* context = static_cast<SSLCertContext*>(
      SSL_get_ex_data(ssl, SSLFilter::ssl_cert_context_index));

  /// 在 worker thread 中进行证书校验，防止阻塞 main isolate
  const X509TrustState* certificate_trust_state =
      filter->certificate_trust_state();
  
  /// Skip irrelevante codes

  /// 取出待校验的证书
  STACK_OF(X509)* unverified = sk_X509_dup(SSL_get_peer_full_cert_chain(ssl));

  // 把 BoringSSL formatted certificates 转换成 macOS/iOS 格式的 SecCertificate certificates.
  ScopedCFMutableArrayRef cert_chain(NULL);
  X509* root_cert = NULL;
  int num_certs = sk_X509_num(unverified);
  int current_cert = 0;
  cert_chain.set(CFArrayCreateMutable(NULL, num_certs, NULL));
  X509* ca;
  // 从待校验的证书中取出最后一个证书 - 根证书
  while ((ca = sk_X509_shift(unverified)) != NULL) {
    ScopedSecCertificateRef cert(CreateSecCertificateFromX509(ca));
    if (cert == NULL) {
      return ssl_verify_invalid;
    }
    CFArrayAppendValue(cert_chain.get(), cert.release());
    ++current_cert;

    if (current_cert == num_certs) {
      break;
    }
  }
  ASSERT(current_cert == num_certs);
  root_cert = ca;
  X509_up_ref(ca);

  /// 取出配置的 Root CAs，包括：
  /// 1. 默认的配置（在 macOS/iOS 平台上为空）、
  /// 2. 命令行参数配置的（在 iOS 平台上为空）、
  /// 3. 通过 SecurityContext 的 API 添加的证书（可能不为空）
  SSL_CTX* ssl_ctx = SSL_get_SSL_CTX(ssl);
  X509_STORE* store = SSL_CTX_get_cert_store(ssl_ctx);
  // 还是得转换成 macOS/iOS 支持的格式 SecCertificate
  ScopedCFMutableArrayRef trusted_certs(CFArrayCreateMutable(NULL, 0, NULL));
  ASSERT(store != NULL);

  if (store->objs != NULL) {
    for (uintptr_t i = 0; i < sk_X509_OBJECT_num(store->objs); ++i) {
      X509* ca = sk_X509_OBJECT_value(store->objs, i)->data.x509;
      ScopedSecCertificateRef cert(CreateSecCertificateFromX509(ca));
      if (cert == NULL) {
        return ssl_verify_invalid;
      }
      CFArrayAppendValue(trusted_certs.get(), cert.release());
    }
  }

  /// Apple 在证书校验时要求的 Policy
  /// Generate a policy for validating chains for SSL.
  CFStringRef cfhostname = NULL;
  if (filter->hostname() != NULL) {
    cfhostname = CFStringCreateWithCString(NULL, filter->hostname(),
                                           kCFStringEncodingUTF8);
  }
  ScopedCFStringRef hostname(cfhostname);
  ScopedSecPolicyRef policy(
      SecPolicyCreateSSL(filter->is_client(), hostname.get()));

  /// 把用户配置的 Root CAs 再转成 Apple Framwork 下的 trust object
  /// Create the trust object with the certificates provided by the user.
  ScopedSecTrustRef trust(NULL);
  /// 这是一个重点的 Apple Api 的调用
  OSStatus status = SecTrustCreateWithCertificates(cert_chain.get(),
                                                   policy.get(), trust.ptr());
  if (status != noErr) {
    return ssl_verify_invalid;
  }


  /// 把用户配置的 RootCAs 添加进 trust object 
  // If the user provided any additional CA certificates, add them to the trust
  // object.
  if (CFArrayGetCount(trusted_certs.get()) > 0) {
    /// 这是一个重点的 Apple Api 的调用
    status = SecTrustSetAnchorCertificates(trust.get(), trusted_certs.get());
    if (status != noErr) {
      return ssl_verify_invalid;
    }
  }

  // Specify whether or not to use the built-in CA certificates for
  // verification.
  status =
      /// 这是一个重点的 Apple Api 的调用
      /// 最后一个参数表示是否信任系统内置的证书，取值来自于 SSLCertContext 的 trust_builtin
      SecTrustSetAnchorCertificatesOnly(trust.get(), !context->trust_builtin());
  if (status != noErr) {
    return ssl_verify_invalid;
  }

  // TrustEvaluateHandler should release all handles.
  /// Skip some codes
  return ssl_verify_retry; // ?
}
````

可以看出，在 macOS/iOS 平台上，最终是通过调用 Apple Security Framework 的 API 来完成了证书的校验，主要使用的 API 有三个（在上面的注释中已有标出）：

```c++
/// 用 trust 中的证书对 cert_chain 用 policy 进行校验
OSStatus status = SecTrustCreateWithCertificates(cert_chain.get(),
                                                   policy.get(), trust.ptr());
/// 把用户配置的 Root CAs 添加进 trust
status = SecTrustSetAnchorCertificates(trust.get(), trusted_certs.get());

/// 是否仅用提供的 Root CAs 进行校验，还是也使用系统内置的 CAs
status =      
      SecTrustSetAnchorCertificatesOnly(trust.get(), !context->trust_builtin());
```

关键的是否是使用系统内置 CAs 的 `trust_builtin`  ，是在 macOS/iOS 平台上的 `TrustBuiltinRoots` 中确定的：

```c++
void SSLCertContext::TrustBuiltinRoots() {
  // Skip irrelevant codes
  set_trust_builtin(true);
}
```

而  `TrustBuiltinRoots`  只有在 Dart 端的 `SecurityContext` 的 `withTrustedRoots` 为 true 时才会被调用。

#### Recap

也就是说只要 Dart 端的 `SecurityContext` 的 `withTrustedRoots` 为 true，那么在 iOS 平台上就会使用系统内置的 Root CAs 来进行证书的校验（当然也会使用用户提供的 Root CAs ），否则就会只是用用户提供的 Root CAs 进行校验。

### **Linux**

再来看 Linux 平台的  `TrustBuiltinRoots` 的实现

```c++
void SSLCertContext::TrustBuiltinRoots() {
  // First, try to use locations specified on the command line.
  if (root_certs_file() != NULL) {
    LoadRootCertFile(root_certs_file());
    return;
  }
  if (root_certs_cache() != NULL) {
    LoadRootCertCache(root_certs_cache());
    return;
  }

  // On Linux, we use the compiled-in trusted certs as a last resort. First,
  // we try to find the trusted certs in various standard locations. A good
  // discussion of the complexities of this endeavor can be found here:
  //
  // https://www.happyassassin.net/2015/01/12/a-note-about-ssltls-trusted-certificate-stores-and-platforms/
  const char* bundle = "/etc/pki/tls/certs/ca-bundle.crt";
  const char* cachedir = "/etc/ssl/certs";
  if (File::Exists(NULL, bundle)) {
    LoadRootCertFile(bundle);
    return;
  }

  if (Directory::Exists(NULL, cachedir) == Directory::EXISTS) {
    LoadRootCertCache(cachedir);
    return;
  }

  // Fall back on the compiled-in certs if the standard locations don't exist,
  // or we aren't on Linux.
  if (SSL_LOG_STATUS) {
    Syslog::Print("Trusting compiled-in roots\n");
  }
  AddCompiledInCerts();
}
```

在 Linux 平台上，首先也是先要 Honor `root_certs_file` 和 `root_certs_cache`  这两个命令行参数。

然后，跟在 Android 平台上一样，加载指定目录下的 Root CAs。

> 在 Linux 平台上，由于 Linux 发行版众多，因此 Root CAs 的存放位置也是千差万别，Flutter/Dart 在这里选用了两个最常见的目录。

然后，跟 Android 平台不一样的是，如果上面指定的两个目录中都没有发现证书的话，Flutter/Dart 会调用 `AddCompiledInCerts()`：

```c++
void SSLCertContext::AddCompiledInCerts() {
  if (root_certificates_pem == NULL) {
    if (SSL_LOG_STATUS) {
      Syslog::Print("Missing compiled-in roots\n");
    }
    return;
  }
  /// 获取证书 store，后面会往这个 store 里面添加证书
  X509_STORE* store = SSL_CTX_get_cert_store(context());
  
  /// 下面就是从 root_certificates_pem 中把证书读出来，往 store 里面添加
  BIO* roots_bio =
      BIO_new_mem_buf(const_cast<unsigned char*>(root_certificates_pem),
                      root_certificates_pem_length);
  X509* root_cert;
  // PEM_read_bio_X509 reads PEM-encoded certificates from a bio (in our case,
  // backed by a memory buffer), and returns X509 objects, one by one.
  // When the end of the bio is reached, it returns null.
  while ((root_cert = PEM_read_bio_X509(roots_bio, NULL, NULL, NULL)) != NULL) {
    int status = X509_STORE_add_cert(store, root_cert);
    // X509_STORE_add_cert increments the reference count of cert on success.
    X509_free(root_cert);
    if (status == 0) {
      break;
    }
  }
  /// Skip some codes about resource release
}
```

可见， `AddCompiledInCerts()` 就是从 `root_certificates_pem` 中把 ***PEM*** 编码的证书读出来，添加都证书 store 中。

在 Dart SDK 中，所有 native 层的 `security_context`  事项中的  `root_certificates_pem`  都是空。那么  `root_certificates_pem` 这个包含  ***PEM*** 编码证书的内容是从哪里来的？

这里需要介绍一个 Mozilla 在安全方面一个知名的开源项目 NSS。

#### **NSS**

> **Network Security Services** (**NSS**) is a set of libraries designed to support cross-platform development of security-enabled client and server applications. Applications built with NSS can support SSL v3, TLS, PKCS #5, PKCS #7, PKCS #11, PKCS #12, S/MIME, X.509 v3 certificates, and other security standards.

NSS 是 Mozilla 主导的一个开源项目，和 OpenSSL 一样，是一个底层密码学库，包括了 TLS 实现。

NSS 并不是完全由 Mozilla 开发出来的，很多公司（包括     Google）和个人都贡献了代码，只是 Mozilla 提供了一些基础设施（比如代码仓库、bug 跟踪系统、邮件组、讨论组）。

NSS 是跨平台的，很多产品都使用了NSS 密码库，包括：

- [Mozilla](https://zh.wikipedia.org/wiki/Mozilla)客户端产品，包括[Firefox](https://zh.wikipedia.org/wiki/Firefox)、[Thunderbird](https://zh.wikipedia.org/wiki/Mozilla_Thunderbird)、[SeaMonkey](https://zh.wikipedia.org/wiki/SeaMonkey)和[Firefox for mobile](https://zh.wikipedia.org/wiki/Firefox_for_mobile)（Fennec）。

- [Opera](https://zh.wikipedia.org/wiki/Opera)；

- [Red Hat ](https://zh.wikipedia.org/wiki/紅帽公司)的服务器产品：Red Hat Directory     Server, Red Hat Certificate System以及Apache网页服务器的mod nss SSL模块；

- Sun Java Enterprise System的Sun服务器产品；

- [Google Chrome](https://zh.wikipedia.org/wiki/Google_Chrome)和[Chromium](https://zh.wikipedia.org/wiki/Chromium)曾使用NSS，但现在它们使用[BoringSSL](https://zh.wikipedia.org/wiki/BoringSSL)。

> refs

1. - [初识NSS，一文了解全貌](https://cloud.tencent.com/developer/news/238252)

2. - [Wiki](https://zh.wikipedia.org/wiki/网络安全服务)

3. - [Mozilla](https://developer.mozilla.org/en-US/docs/Mozilla/Projects/NSS)，[FAQ](https://developer.mozilla.org/en-US/docs/Mozilla/Projects/NSS/FAQ)

#### 回到 root_certificates_pem

Chromium 及 Chromium OS 之前就是使用的 NSS 来做 SSL/TLS 认证的（这是之前的 Chromium 关于 NSS 的[项目地址](https://chromium.googlesource.com/chromium/deps/nss/)）。

但是现在，Google 已经把 SSL/TLS 相关的工作都转到自家的 [BoringSSL](https://zh.wikipedia.org/wiki/BoringSSL) 上了。

Dart/Flutter 作为 Google 的项目也不例外， Dart 中 SSL/TLS 相关的工作使用也是 [BoringSSL ](https://zh.wikipedia.org/wiki/BoringSSL)。

但是，NSS 是平台无关的，它在进行 SSL/TLS 证书校验时使用的 Root CAs 也是平台无关的，而是在项目自己维护了一个 Root CAs 的列表。

Dart 虽然没有使用 NSS ，但是 fork 了这些 Root CAs ，用做在某些平台上的校验用。[这里是](https://github.com/dart-lang/root_certificates) Dart fork 的这些 Root CAs 的项目地址。

从这个项目的 README.google 不难看出，它会定期从 NSS 抓取证书，然后转换成自己的格式保存在自己的 [root_certifactes.cc](https://github.com/dart-lang/root_certificates/blob/master/root_certificates.cc) 中。

而前面 Linux 平台上使用的 `root_certificates_pem` 就是来自这个文件：

```c++
const unsigned char* root_certificates_pem = root_certificates_pem_;
unsigned int root_certificates_pem_length = 214078;
```

#### Recap

也就是说在 Linux 平台上，默认使用 Root CAs 是系统 `/etc/pki/tls/certs/ca-bundle.crt` 目录或 `/etc/ssl/certs` 目录里面的证书。如果这两个目录都没有证书，就会使用 NSS 项目中维护的 Root CAs。

### **Windows**

```c++
void SSLCertContext::TrustBuiltinRoots() {
  /// Skip
  X509_STORE* store = SSL_CTX_get_cert_store(context());
  if (AddCertificatesFromRootStore(store)) {
    return;
  }
  /// Skip some codes
  AddCompiledInCerts();
}
```

在 Windows 平台上，首先也是先要 Honor `root_certs_file` 和 `root_certs_cache`  这两个命令行参数。

然后，会尝试使用系统的 Root CAs， 如果失败，使用 NSS 的 Root CAs

### **Fuchsia**

```c++
void SSLCertContext::TrustBuiltinRoots() {
  // skip
  int status = SSL_CTX_set_default_verify_paths(context());
  SecureSocketUtils::CheckStatus(status, "TlsException",
                                 "Failure trusting builtin roots");
}
```

在 Fuchsia 平台上，首先也是先要 Honor `root_certs_file` 和 `root_certs_cache`  这两个命令行参数。

然后，使用 BoringSSL 默认的证书进行校验：[SSL_CTX_set_default_verify_paths](https://www.openssl.org/docs/man1.1.0/man3/SSL_CTX_set_default_verify_paths.html)

### withTrustedRoots 默认值

从前面的分析可以看出，Dart 端  ```SecurityContext```  的 ```withTrustedRoots``` 参数的取值决定了是否使用默认 Root CAs，上面所有的代码追踪都是基于这个取值为 true 的情况。但是这个参数的默认值为 false：

```dart
 factory SecurityContext({bool withTrustedRoots = false});
```

```withTrustedRoots``` 默认为 false，也就是说  ```SecurityContext``` 默认不包含任何的默认证书。

那么这个参数取值什么时候为 true，或者说  ```HttpClient``` 是怎么使用了一个 ```withTrustedRoots``` 为 true  的  ```SecurityContext```  的？

从一个简单的 http 请求开始看，即： 用  ```HttpClient```  open 一个 url：

```dart
  // http_impl.dart
  Future<HttpClientRequest> openUrl(String method, Uri url) =>
    _openUrl(method, url);

  Future<_HttpClientRequest> _openUrl(String method, Uri uri) {
    // ...
    return _getConnection(uri.host, port, proxyConf, isSecure, timeline).then(
        (_ConnectionInfo info) {
      _HttpClientRequest send(_ConnectionInfo info) {
        return info.connection
            .send(uri, port, method.toUpperCase(), info.proxy, timeline);
      }
      return send(info);
    });
  }

  Future<_ConnectionInfo> _getConnection(String uriHost, int uriPort,
      _ProxyConfiguration proxyConf, bool isSecure, TimelineTask? timeline) {
    Iterator<_Proxy> proxies = proxyConf.proxies.iterator;

    Future<_ConnectionInfo> connect(error) {
      if (!proxies.moveNext()) return new Future.error(error);
      _Proxy proxy = proxies.current;
      String host = proxy.isDirect ? uriHost : proxy.host!;
      int port = proxy.isDirect ? uriPort : proxy.port!;
      return _getConnectionTarget(host, port, isSecure)
          .connect(uriHost, uriPort, proxy, this, timeline)
          // On error, continue with next proxy.
          .catchError(connect);
    }

    return connect(new HttpException("No proxies given"));
  }

  _ConnectionTarget _getConnectionTarget(String host, int port, bool isSecure) {
    String key = _HttpClientConnection.makeKey(isSecure, host, port);
    return _connectionTargets.putIfAbsent(key, () {
      return new _ConnectionTarget(key, host, port, isSecure, _context);
    });
  }

  /// SecurityContext _context 被传入 _ConnectionTarget 并调用 _ConnectionTarget 的 connect
Future<_ConnectionInfo> connect(String uriHost, int uriPort, _Proxy proxy,
      _HttpClient client, TimelineTask? timeline) {
    if (hasIdle) {
      // ...
      return new Future.value(new _ConnectionInfo(connection, proxy));
    }
    var maxConnectionsPerHost = client.maxConnectionsPerHost;
    if (maxConnectionsPerHost != null &&
        _active.length + _connecting >= maxConnectionsPerHost) {
      // pending
      return completer.future;
    }


    Future<ConnectionTask> connectionTask = (isSecure && proxy.isDirect
        ? SecureSocket.startConnect(host, port, // SecureSocket 在这里连接
            context: context, onBadCertificate: callback)
        : Socket.startConnect(host, port));
    _connecting++;
    return connectionTask.then((ConnectionTask task) {
      _socketTasks.add(task);
      Future socketFuture = task.socket;
      final Duration? connectionTimeout = client.connectionTimeout;
      if (connectionTimeout != null) {
        socketFuture = socketFuture.timeout(connectionTimeout);
      }
      return socketFuture.then((socket) {
        // ... 把建立的 socket 转换成 _ConnectionInfo
      }
  }

  // SecureSocket 连接的建立
  static Future<ConnectionTask<SecureSocket>> startConnect(host, int port,
      {SecurityContext? context,
      bool onBadCertificate(X509Certificate certificate)?,
      List<String>? supportedProtocols}) {
    /// 先建立 RawSecureSocket
    return RawSecureSocket.startConnect(host, port,
            context: context,
            onBadCertificate: onBadCertificate,
            supportedProtocols: supportedProtocols)
        .then((rawState) {
      Future<SecureSocket> socket =
          rawState.socket.then((rawSocket) => new SecureSocket._(rawSocket));
      return new ConnectionTask<SecureSocket>._(socket, rawState._onCancel);
    });
  }

  /// RawSecureSocket 的建立
  static Future<ConnectionTask<RawSecureSocket>> startConnect(host, int port,
      {SecurityContext? context,
      bool onBadCertificate(X509Certificate certificate)?,
      List<String>? supportedProtocols}) {
    /// 要想建立 RawSecureSocket，先建立一个 RawSocket
    return RawSocket.startConnect(host, port)
        .then((ConnectionTask<RawSocket> rawState) {
      Future<RawSecureSocket> socket = rawState.socket.then((rawSocket) {
        /// 把建立的 RawSocket 转换成 RawSecureSocket
        return secure(rawSocket,
            context: context,
            onBadCertificate: onBadCertificate,
            supportedProtocols: supportedProtocols);
      });
      return new ConnectionTask<RawSecureSocket>._(socket, rawState._onCancel);
    });
  }

  /// RawSocket 转换成 RawSecureSocket
  static Future<RawSecureSocket> secure(RawSocket socket,
      {StreamSubscription<RawSocketEvent>? subscription,
      host,
      SecurityContext? context,
      bool onBadCertificate(X509Certificate certificate)?,
      List<String>? supportedProtocols}) {
    socket.readEventsEnabled = false;
    socket.writeEventsEnabled = false;
    return _RawSecureSocket.connect(
        host != null ? host : socket.address.host, socket.port, false, socket,
        subscription: subscription,
        context: context,
        onBadCertificate: onBadCertificate,
        supportedProtocols: supportedProtocols);
  }

  /// 基于 RawSocket 建立 _RawSecureSocket
  static Future<_RawSecureSocket> connect(
      dynamic /*String|InternetAddress*/ host,
      int requestedPort,
      bool isServer,
      RawSocket socket,
      {SecurityContext? context,
      StreamSubscription<RawSocketEvent>? subscription,
      List<int>? bufferedData,
      bool requestClientCertificate = false,
      bool requireClientCertificate = false,
      bool onBadCertificate(X509Certificate certificate)?,
      List<String>? supportedProtocols}) {
    return new _RawSecureSocket(
            address,
            requestedPort,
            isServer,
            /// 终于，如果传入的 SecureContext 为 null（默认的 HttpClient 就是这样的），
            /// 则使用 SecurityContext.defaultContext
            context ?? SecurityContext.defaultContext,
            socket,
            subscription,
            bufferedData,
            requestClientCertificate,
            requireClientCertificate,
            onBadCertificate,
            supportedProtocols)
        ._handshakeComplete
        .future;
  }
                             
```

从上面的代码可以看出， ```HttpClient``` 默认使用的 ```SecurityContext``` 是 ```SecurityContext.defaultContext```，即  ```SecurityContext``` 的静态 field ```defaultContext```， 下面看看 ``` SecurityContext.defaultContext``` 的实现。

如前所述，```SecurityContext``` 也是 abstract 的，它的具体实现是 ```dart sdk``` 的 ``` sdk/lib/_internal/vm/bin/secure_socket_patch.dart``` 中的 ```_SecurityContext```：

```dart
class _SecurityContext extends NativeFieldWrapperClass1 implements SecurityContext {
  static final SecurityContext defaultContext = new _SecurityContext(true);
}
```

可见， ```HttpClient``` 使用的默认的  ```SecurityContext```  为 `SecurityContext.defaultContext`。 它的 ```withTrustedRoots``` 为 true。

### 结论

1. 启动 Dart VM 的命令行参数有最高的优先级

   `--root-certs-file` 和  `--root-certs-cache`  有最高的优先级，如果指定了任意一个，则仅使用指定的 Root CAs。

2. 默认 Root CAs

   默认情况下， ```HttpClient``` 使用的  ```SecurityContext```  为 `SecurityContext.defaultContext`。 它的 ```withTrustedRoots``` 为 true，也就是使用默认 Root CAs。

   - 在 Android 平台上，使用  ```/system/etc/security/cacerts``` 目录下的 Root CAs 进行 ssl/tls 证书校验；
   - 在 macOS/iOS 平台上，默认使用系统的证书进行校验；
   - 在 Linux 平台上，默认使用的 Root CAs 是系统 `/etc/pki/tls/certs/ca-bundle.crt` 目录或 `/etc/ssl/certs` 目录里面的证书。如果这两个目录都没有证书，就会使用 NSS 项目中维护的 Root CAs。
   - 在 Windows 平台上，尝试使用系统的 Root CAs， 如果失败，使用 NSS 的 Root CAs；
   - 在 Fuchsia 平台上，使用 BoringSSL 默认的证书进行校验。

   如果 ```HttpClient``` 使用的  ```SecurityContext``` 的  ```withTrustedRoots``` 为 false，则不会使用上面的这些默认 Root CAs，而是仅仅使用通过  ```SecurityContext``` 的 API 添加的 Root CAs。

## 手动配置证书

通过分析  ```SecurityContext```  的构造函数在不同平台上的实现，可以知道 Flutter/Dart 在不同平台上使用的默认 Root CAs 的情况。

 ```SecurityContext``` 还提供了一组 API 用来手动的配置证书：

```dart
abstract class SecurityContext {
  void usePrivateKey(String file, {String? password});
  void usePrivateKeyBytes(List<int> keyBytes, {String? password});
  
  void setTrustedCertificates(String file, {String? password});
  void setTrustedCertificatesBytes(List<int> certBytes, {String? password});
  
  void useCertificateChain(String file, {String? password});
  void useCertificateChainBytes(List<int> chainBytes, {String? password});
  
  void setClientAuthorities(String file, {String? password});
  void setClientAuthoritiesBytes(List<int> authCertBytes, {String? password});
}
```

### SecurityContext 的 API

可以从两个维度上来分组  ```SecurityContext```  提供的 API：

- 从 input 的证书的形式上，可以分成两组；
- 从功能上， ```SecurityContext``` 的 API 可以分成四组。

#### 证书（input）的形式

从 input 的证书的形式上，有两组 API：使用 file 管理证书的 API，和使用 byte 数组管理证书的 API。

两者在功能上没有区别，区别仅仅在于是调用者是直接把文件交给  ```SecurityContext``` 去读取，还是自己把文件读出到 byte 数组里面后再交给  ```SecurityContext``` 去使用。

需要注意的是，直接把文件交给  ```SecurityContext``` 去读取是同步读取的，会阻塞当前线程。

以`_SecurityContext` 中的  `usePrivateKey` 为例：

```dart
  void usePrivateKey(String file, {String? password}) {
    /// 这里有一个同步的文件 i/o 操作
    List<int> bytes = (new File(file)).readAsBytesSync();
    usePrivateKeyBytes(bytes, password: password);
  }
```



#### 提供的功能

> Conventions
>
> 在阅读 SSL/TLS 相关的文档、代码时，有些常见的、但是非正式定义的术语：

- TrustStore 一般表示用来存放***证书*** 的地方
- Key 一般用来表示私钥
- KeyStore 用来存放***私钥*** 的地方

先来想想作为 Dart 的 TLS/SSL 的证书管理器的  ```SecurityContext```  应该提供哪些功能。

- 首先， ```SecurityContext```  应该能配置额外的证书，用以完成自定义的证书信任，例如希望信任指定的证书或指定的证书签发的证书的场景(ssl pinning)；

- 其次，还应该提供 two-way ssl 的能力，也就是能配置 client 端使用的证书链和私钥；

- 最后 ， ```SecurityContext```  还会放在 Http Server 中使用，因此作为 Server 端需要支持配置 Root CAs 用来验证 client 传递过来的证书。

### Ssl Pinning

 ```SecurityContext```  提供了配置额外的信任证书的能力：`setTrustedCertificates` ：

> API Doc
>

Sets the set of trusted X509 certificates used by SecureSocket client connections, when connecting to a secure server.
file is the path to a PEM or PKCS12 file containing X509 certificates, usually root certificates from certificate authorities. For PKCS12 files, password is the password for the file. For PEM files, password is ignored. Assuming it is well-formatted, all other contents of file are ignored.

> iOS note

On iOS, this call takes only the bytes for a single DER encoded X509 certificate. It may be called multiple times to add multiple trusted certificates to the context. A DER encoded certificate can be obtained from a PEM encoded certificate by using the openssl tool:
```$ openssl x509 -outform der -in cert.pem -out cert.der```

`setTrustedCertificates` 用来配置 client 端使用的、用以验证 server 端证书的证书，一般是一个 CA 的根证书。

证书一般通过 PEM 或 PKCS12 格式的文件来提供。

看看他的具体实现：

```dart
/// dart-lang-sdk/sdk/lib/_internal/vm/bin/secure_socket_patch.dart
class _SecurityContext extends NativeFieldWrapperClass1
    implements SecurityContext {
  void setTrustedCertificatesBytes(List<int> certBytes, {String? password})
    native "SecurityContext_SetTrustedCertificatesBytes";
}
```

具体实现还是在 natvie 层：

```c++
// dart-lang-sdk/runtime/bin/security_context.cc
void SSLCertContext::SetTrustedCertificatesBytes(Dart_Handle cert_bytes,
                                                 const char* password) {
  int status = 0;
  {
    ScopedMemBIO bio(cert_bytes);
    // 用 PEM 编码格式尝试读取
    status = SetTrustedCertificatesBytesPEM(context(), bio.bio());
    if (status == 0) {// 读取失败
      if (SecureSocketUtils::NoPEMStartLine()) {// 不是 PEM 格式
        ERR_clear_error();
        BIO_reset(bio.bio());
        // 用 PKCS12 格式读取
        status = SetTrustedCertificatesBytesPKCS12(context(), &bio, password);
      }
    } else {
      // The PEM file was successfully parsed.
      ERR_clear_error();
    }
  }
  SecureSocketUtils::CheckStatus(status, "TlsException",
                                 "Failure trusting builtin roots");
}

// 用 PEM 格式读取证书
static int SetTrustedCertificatesBytesPEM(SSL_CTX* context, BIO* bio) {
  // 获取 openssl 中 cert store 的引用
  X509_STORE* store = SSL_CTX_get_cert_store(context);

  int status = 0;
  X509* cert = NULL;
  while ((cert = PEM_read_bio_X509(bio, NULL, NULL, NULL)) != NULL) {
    // 把读取到的证书添加到 X509_STORE 中
    status = X509_STORE_add_cert(store, cert);
    // X509_STORE_add_cert increments the reference count of cert on success.
    X509_free(cert);
    if (status == 0) {
      return status;
    }
  }
  return SecureSocketUtils::NoPEMStartLine() ? status : 0;
}
// 用 PKCS12 格式读取证书
static int SetTrustedCertificatesBytesPKCS12(SSL_CTX* context,
                                             ScopedMemBIO* bio,
                                             const char* password) {
  CBS cbs;
  CBS_init(&cbs, bio->data(), bio->length());

  EVP_PKEY* key = NULL;
  ScopedX509Stack cert_stack(sk_X509_new_null());
  int status = PKCS12_get_key_and_certs(&key, cert_stack.get(), &cbs, password);
  if (status == 0) {
    return status;
  }
  // 获取 openssl 中 cert store 的引用
  X509_STORE* store = SSL_CTX_get_cert_store(context);
  X509* ca;
  while ((ca = sk_X509_shift(cert_stack.get())) != NULL) {
    // 把读取到的证书添加到 X509_STORE 中
    status = X509_STORE_add_cert(store, ca);
    // X509_STORE_add_cert increments the reference count of cert on success.
    X509_free(ca);
    if (status == 0) {
      return status;
    }
  }
  return status;
}
```

其中的 `SSL_CTX_get_cert_store` 、 `X509_STORE_add_cer` 、`PEM_read_bio_X509` 都是 openssl 的功能，`PKCS12_get_key_and_certs` 是 boringssl 的功能：

[SSL_CTX_get_cert_store](https://www.openssl.org/docs/man1.1.0/man3/SSL_CTX_get_cert_store.html)() returns a pointer to the current certificate verification storage.

[X509_STORE_add_cert](https://www.openssl.org/docs/man1.1.1/man3/X509_STORE_add_cert.html)() and X509_STORE_add_crl() add the respective object to the **X509_STORE**'s local storage.

可见， ```SecurityContext``` 的  `setTrustedCertificates` 就是用来往 openssl 的 cert store 添加证书的，添加的证书可以用 PEM 格式提供，也可以用 PKCS12 的格式提供。

有三个值得注意的细节：

- PKCS12 一般是用来保存私钥和证书的，这里添加到 openssl  的 cert store 中的仅仅是证书，因此，一般使用 PEM  编码的证书就可以了；
- 虽然 API  的名字是 setXXXX，实际上可以多次调用添加多个证书。也可以用 PEM  编码格式一次提供多个证书；
- API doc 的 iOS  note 中提到如果使用 PKCS12 格式提供证书，每次只会读取一个 DER 证书。

### 提供证书和私钥

简单来讲，TLS/SSL 的握手过程中，server 端要提供给 client 端一个**证书**，client 端校验证书通过后，用证书的公钥加密一个**协商密钥**传递给 server 端，server 端用***私钥*** 解密**协商密钥**，这个协**商密钥**就是后续 server/client 两端对称加密通信使用的秘钥。

如前所述，`SercurityContext` 不止提供给 `HttpClient` 使用，还会提供给 `HttpServer` 使用，因此 `SercurityContext` 需要提供 API 用以配置前面提到的 ***证书 *** 和 ***私钥*** 的功能，这就是 `useCertificateChain` 和  `usePrivateKey` 两个 API 的功能。

#### useCertificateChain

> API  Doc

Sets the chain of X509 certificates served by SecureServerSocket when making secure connections, including the server certificate.

`useCertificateChain` 配置的是证书提供方所提供的证书链。

在 ssl 通信中，证书提供方要提供自己的证书，另一侧会对这个证书进行验证。

##### 完整证书链

一般情况下，提供证书时，不仅仅要提供从 CA 申请到的证书，而是要提供完整的证书链。例如 google.com 提供的证书链可以用 openssl 来查看：

```shell
$ openssl s_client -connect  google.com:443
CONNECTED(00000005)
depth=2 OU = GlobalSign Root CA - R2, O = GlobalSign, CN = GlobalSign
verify return:1
depth=1 C = US, O = Google Trust Services, CN = GTS CA 1O1
verify return:1
depth=0 C = US, ST = California, L = Mountain View, O = Google LLC, CN = *.google.com
verify return:1
---
Certificate chain
 0 s:/C=US/ST=California/L=Mountain View/O=Google LLC/CN=*.google.com
   i:/C=US/O=Google Trust Services/CN=GTS CA 1O1
 1 s:/C=US/O=Google Trust Services/CN=GTS CA 1O1
   i:/OU=GlobalSign Root CA - R2/O=GlobalSign/CN=GlobalSign
```

至于为什么要提供完整的证书链和如果提供的证书链不完整会有什么问题，在 Android 的 Security with HTTPS and SSL 的 [Missing intermediate certificate authority](https://developer.android.com/training/articles/security-ssl#MissingCa) 有详细的说明。

##### 实现

```c++
void FUNCTION_NAME(SecurityContext_UseCertificateChainBytes)(Dart_NativeArguments args) {
  // skip
  int status = context->UseCertificateChainBytes(cert_chain_bytes, password);
}

int SSLCertContext::UseCertificateChainBytes(Dart_Handle cert_chain_bytes, const char* password) {
  ScopedMemBIO bio(cert_chain_bytes);
  return UseChainBytes(context(), &bio, password);
}

static int UseChainBytes(SSL_CTX* context, ScopedMemBIO* bio, const char* password) {
  // 先尝试 PEM 格式
  int status = UseChainBytesPEM(context, bio->bio());
  if (status == 0) {
    if (SecureSocketUtils::NoPEMStartLine()) {
      ERR_clear_error();
      BIO_reset(bio->bio());
      // 再尝试 PKCS12 格式
      status = UseChainBytesPKCS12(context, bio, password);
    }
  } else {// skip}
  return status;
}

static int UseChainBytesPEM(SSL_CTX* context, BIO* bio) {
  int status = 0;
  ScopedX509 x509(PEM_read_bio_X509_AUX(bio, NULL, NULL, NULL));
  // 调用 openssl 的 SSL_CTX_use_certificate 使用证书
  status = SSL_CTX_use_certificate(context, x509.get());

  // 清空原来的 chain certs
  SSL_CTX_clear_chain_certs(context);

  X509* ca;
  while ((ca = PEM_read_bio_X509(bio, NULL, NULL, NULL)) != NULL) {
    // 使用新的证书链
    status = SSL_CTX_add0_chain_cert(context, ca);
  }
  return SecureSocketUtils::NoPEMStartLine() ? status : 0;
}

static int UseChainBytesPKCS12(SSL_CTX* context, ScopedMemBIO* bio, const char* password) {
  CBS cbs;
  CBS_init(&cbs, bio->data(), bio->length());

  EVP_PKEY* key = NULL;
  ScopedX509Stack certs(sk_X509_new_null());
  int status = PKCS12_get_key_and_certs(&key, certs.get(), &cbs, password);

  X509* ca = sk_X509_shift(certs.get());
  // 调用 openssl 的 SSL_CTX_use_certificate 使用证书
  status = SSL_CTX_use_certificate(context, ca);
  X509_free(ca);

  // 清空原来的 chain certs
  SSL_CTX_clear_chain_certs(context);

  while ((ca = sk_X509_shift(certs.get())) != NULL) {
    // 使用新的证书链
    status = SSL_CTX_add0_chain_cert(context, ca);
  }
  return status;
}
```

从具体的实现可以看出，不管是传入的证书是 PEM 格式，还是 PKCS12 格式， `useCertificateChain` 的工作流程都是一样的，可以分为三步：

1. 把传入的证书链的第一个取出，调用 openssl 的 `SSL_CTX_use_certificate` 使用证书。

   这里**使用证书**的证书是证书链的 leaf 证书，可能是从 CA 申请的证书，或者是自签证书链的最叶子证书。这个证书是跟 ***私钥*** 对应的，两端在 TLS/SSL 握手阶段协商秘钥的加密就是用的这个证书的公钥；

   > [SSL_CTX_use_certificate](https://www.openssl.org/docs/man1.0.2/man3/SSL_CTX_use_PrivateKey.html)

2. 清空证书链；

   使用的是 openssl  的 `SSL_CTX_clear_chain_certs`

3. 挨个读入传入的证书链，构建新的证书链。

   使用的是 openssl 的 `SSL_CTX_add0_chain_cert`

可以看出，有两个需要注意的细节：

- 不同于 `setTrustedCertificates`，重复调用 `useCertificateChain` 会清空之前的调用，即只有最后一次调用有效；
- 传入的证书链中证书的顺序很重要。

#### usePrivateKey

>API Doc

Sets the private key for a server certificate or client certificate.
A secure connection using this SecurityContext will use this key with the server or client certificate to sign and decrypt messages. 

 `usePrivateKey` 配置的是证书提供方所提供的证书对应的私钥。

看看其实现：

```dart
/// dart-lang-sdk/sdk/lib/_internal/vm/bin/secure_socket_patch.dart
class _SecurityContext extends NativeFieldWrapperClass1 implements SecurityContext {
  void usePrivateKeyBytes(List<int> keyBytes, {String? password})
      native "SecurityContext_UsePrivateKeyBytes";
}
```

在 Native 侧：

```c++
// dart-lang-sdk/runtime/bin/security_context.cc
void FUNCTION_NAME(SecurityContext_UsePrivateKeyBytes)(Dart_NativeArguments args) {
  SSLCertContext* context = SSLCertContext::GetSecurityContext(args);
  const char* password = SSLCertContext::GetPasswordArgument(args, 2);
  int status;
  {
    ScopedMemBIO bio(ThrowIfError(Dart_GetNativeArgument(args, 1)));
    EVP_PKEY* key = GetPrivateKey(bio.bio(), password);
    status = SSL_CTX_use_PrivateKey(context->context(), key);
    EVP_PKEY_free(key);
  }
  SecureSocketUtils::CheckStatus(status, "TlsException", "Failure in usePrivateKeyBytes");
}
```

`SSL_CTX_use_PrivateKey` 是 openssl 提供的能力：

> [SSL_CTX_use_PrivateKey](https://www.openssl.org/docs/man1.0.2/man3/SSL_CTX_use_PrivateKey.html)() adds **pkey** as private key to **ctx**.
>
> If a certificate has already been set and the private does not belong to the certificate an error is returned. To change a certificate, private key pair the new certificate needs to be set with SSL_use_certificate() or SSL_CTX_use_certificate() before setting the private key with SSL_CTX_use_PrivateKey() or SSL_use_PrivateKey().

也就是说， `usePrivateKey` 传入的私钥必须跟  `useCertificateChain` 中的证书相匹配，所以需要先  `useCertificateChain` 配置证书链，然后再  `usePrivateKey` 配置私钥。

#### 错误的 iOS note

在 `usePrivateKey` 和 `useCertificateChain` 的 API  Doc 还有针对 iOS 的注意事项：

 `usePrivateKey`  iOS note: 

>Only PKCS12 data is supported. It should contain both the private key and the certificate chain. On iOS one call to `usePrivateKey ` with this data is used instead of two calls to `useCertificateChain` and `usePrivateKey`.

 `useCertificateChain` iOS note:

>  As noted above, usePrivateKey does the job of both that call and this one. On iOS, this call is a no-op.

也就是说在 iOS 平台上，仅支持 PKCS12 格式的证书，并且配置私钥和配置证书链通过  `usePrivateKey` 一个 API 来完成。

但实际上，看代码发现这个 note 并不正确，目前的实现来看，已经没有平台的差别了，不管是 iOS 还是非 iOS 底层都是使用的 boringssl 来实现的。

切换到 boringssl 的实现是 Jun 7, 2017 这个 [review](https://codereview.chromium.org/2903743002) 修改实现的：

> Ported SecureSocket to use BoringSSL on OSX and iOS. This included registering a callback with BoringSSL using SSL_CTX_set_cert_verify_callback, which overrides the BoringSSL certificate verification process, in order to verify certificates against the system's root certificates and then proceed to let BoringSSL handle the rest of the SSL session. In addition, this change includes refactoring to share BoringSSL code that used on all platforms.

猜测可能是在修改 native 层实现的时候，并没有同步的更新 Dart 侧的 API Doc。

还有两个值得关注的地方：

- PEM or PKCS12

  虽然前面提到的 `SecurityContext`  的 API 都既支持 PEM 格式，又支持 PKCS12 格式，但是在 native 层实现中，都是先尝试用 PEM 格式读取，失败后再尝试用 PKCS12 格式。因此，建议调用这些 API 时，直接使用  PEM 格式以提升效率

-  `usePrivateKey` with  PKCS12

  API Doc 中说 `usePrivateKey` 提供 PKCS12 格式的证书，可以同时配置私钥和证书链。显然这是错误的，因为  `usePrivateKey`  的实现仅仅配置了私钥，PKCS12文件里面的证书链会被忽略。



#### 双向认证

双向认证即  two-way authentication 或 mutual authentication，或者在 TLS/SSL 场景下称为 Two-Way SSL，在 SSL 场景下指的是 client 和 server 两端以均提供 X509 证书的方式互相验证彼此的身份。

如果要支持双向认证，client 端就要像单向认证的 server 端一样，提供完整的证书链，并且还要有私钥。

显然，Dart 的 `SecurityContext` 支持双向认证，客户端使用 `usePrivateKey` 提供用以双向认证的私钥，用 `useCertificateChain` 提供用以双向认证的证书链即可。

双向认证的握手过程如下：

![SSL authentication handshake messages](https://www.codeproject.com/KB/IP/326574/1WaySSL.png)

1. Client sends `ClientHello `message proposing SSL options.
2. Server responds with `ServerHello `message selecting the SSL options.
3. Server sends `Certificate `message, which contains the server's certificate.
4. Server requests client's certificate in `CertificateRequest `message, so that the connection can be mutually authenticated.
5. Server concludes its part of the negotiation with `ServerHelloDone `message.
6. Client responds with `Certificate `message, which contains the client's certificate.
7. Client sends session key information (encrypted with server's public key) in `ClientKeyExchange `message.
8. Client sends a `CertificateVerify `message to let the server know it owns the sent certificate.
9. Client sends `ChangeCipherSpec `message to activate the negotiated options for all future messages it will send.
10. Client sends `Finished `message to let the server check the newly activated options.
11. Server sends `ChangeCipherSpec `message to activate the negotiated options for all future messages it will send.
12. Server sends `Finished `message to let the client check the newly activated options.

整个过程比较复杂，但其实 openssl 都帮我们完成，不需要太多的关心。除了第 4 步的一个细节。

第 4 步是 Server 通过  `CertificateRequest ` 向 Client 索要证书，发送这个 request 时，Server 可以把自己*信任的 CA 名字列表*一并发送给 Client， `SecurityContext` 提供了 `setClientAuthorities` 这个 API 来配置*信任的 CA 名字列表。*

##### setClientAuthorities

> Sets the list of authority names that a SecureServerSocket will advertise as accepted when requesting a client certificate from a connecting client.

他的实现：

```c++
void SSLCertContext::SetClientAuthoritiesBytes(Dart_Handle client_authorities_bytes, const char* password) {
  // skip
  status = SetClientAuthorities(context(), &bio, password);
  // skip
  SecureSocketUtils::CheckStatus(status, "TlsException", "Failure in setClientAuthoritiesBytes");
}

static int SetClientAuthorities(SSL_CTX* context, ScopedMemBIO* bio, const char* password) {
  // 尝试 PEM
  int status = SetClientAuthoritiesPEM(context, bio->bio());
      // 失败后尝试 PKCS12
      status = SetClientAuthoritiesPKCS12(context, bio, password);
  return status;
}

static int SetClientAuthoritiesPEM(SSL_CTX* context, BIO* bio) {
  while ((cert = PEM_read_bio_X509(bio, NULL, NULL, NULL)) != NULL) {
    // 通过 openssl 的  SSL_CTX_add_client_CA 实现
    status = SSL_CTX_add_client_CA(context, cert);
  }
  return SecureSocketUtils::NoPEMStartLine() ? status : 0;
}

static int SetClientAuthoritiesPKCS12(SSL_CTX* context, ScopedMemBIO* bio, const char* password) {
  int status = PKCS12_get_key_and_certs(&key, cert_stack.get(), &cbs, password);
  X509* ca;
  while ((ca = sk_X509_shift(cert_stack.get())) != NULL) {
    // 通过 openssl 的  SSL_CTX_add_client_CA 实现
    status = SSL_CTX_add_client_CA(context, ca);
  }
  return status;
}
```

最终都是通过 openssl 的 `SSL_CTX_add_client_CA` 实现的：

> [SSL_CTX_add_client_CA](https://www.openssl.org/docs/man1.1.0/man3/SSL_CTX_add_client_CA.html) 
>
> adds the CA name extracted from **cacert** to the list of CAs sent to the client when requesting a client certificate for **ctx**.

可见，`setClientAuthorities` 虽然传入的是 PEM 或 PKCS12 格式的证书，但是实际上只是取出里面的 CA name 用到双向认证的握手环节。

可以多次调用 `setClientAuthorities` 以传入多个 CA 证书，也可以一次调用的 PEM 或 PKCS12 文件中包含多个 CA 证书。

>  `setClientAuthorities` 有什么作用？

我不知道。或许可以在 Client 端根据 Server 的 `CertificateRequest ` 中的 Root CAs 的名字返回不同的 Client 证书。



## File to ByteArray

`SecurityContext` 提供的管理证书的 API，即可以接收文件，可以接收文件的 Byte Array。

如果使用文件，则在 API 的调用处会有一个阻塞的文件读取，因此最好在恰当的实际把文件读入到 Byte Array，然后传入 Byte Array。

```dart
List<int> bytes = File('path/to/cert').readAsBytesSync();
```

但是需要注意的是，`dart:io` 里面提供的 `File` 的 API 的 `path` 是基于所处平台的文件系统的。

如果是一个 command line Dart App，那么通过 `Platform.script` 就可以获取当前正在运行的 Dart Script 的 `URI`，然后就可以通过这个 URI 来定位证书文件的路径了。例如下面的文件结构：

```shell
.
├── certs
│   └── cert.pem
└── main.dart
```

就可以通过 `Platform.script.resolve('certs/cert.pem')` 获取到 `cert.pem` 这个证书文件的路径，然后就可以用 `dart:io`  的  `File` 进行读取了。

但是，如果项目是一个 Flutter App，一般会用 Assets Bundle 的方式来管理文件资源（[Adding assets and images](https://flutter.dev/docs/development/ui/assets-and-images)），证书文件如果放到 assets 目录中，只能通过 Assets Bundle 来访问，没法用 `File` 来读取。

或许可以考虑把证书文件放到平台的 apk 或 ipa 包里面，然后想办法用 `File` 来读取。不过不如直接通过 Assets Bundle 把文件读取出来转换成 Byte Array：

```dart
ByteData byteData = await rootBundle.load('assets/copy_airstar_stack.pem');
List<int> byteArray = byteData.buffer.asUint8List(byteData.offsetInBytes, byteData.lengthInBytes);
```

### 直接使用 Byte Array

与其在运行时想办法把文件读取到 Byte Array，不如直接把 Byte Array 以 `const` 的方式放到代码中，就像 [Dart从 NSS 拷贝 Root CAs](https://github.com/dart-lang/root_certificates) 那样。

Dart 的 print 不能输出太长的日志(貌似最多 1024 个字符)，需要把 Byte Array 分段输出到 console：

```dart
final byteArray = yourList.toString();
const chunk = 1000;
final length = byteArray.length;
var chunkCount = 0;
do {
  print(byteArray.substring(chunkCount * chunk, min((++chunkCount) * chunk, length)));
} while (chunkCount * chunk < length);
```

然后从控制台中每段每段的拷贝到代码中。麻烦。

或者直接使用 HEX dump 工具  `xxd` ，用shell 脚本自动完成：

```shell
#!/bin/bash
if [ -z "$1" ];then
	echo -e "\x1B[31mUsage: $0 path/to/your/binary\x1B[0m"
	exit 1
fi
if [ ! -f "$1" ];then
	echo -e "\x1B[31m\"$1\" NOT exist or NOT a file\x1B[0m"
	exit 1
fi
# hex dump | remove line break | prefix "0x" and append "," every 2 characters | format
xxd -p "$1" | tr -d \\n | sed 's/\(.\{2\}\)/0x\1,/g' | xargs printf "const cert = [%s];"
```

这样就方便多了，搭配 `pbcopy` (macOS) 或 `xclip` (Linux) 效果更好：`script.shell | pbcopy`

或许还应该还支持 `xxd` 的 reverse 操作，方便查看这个 Byte Array 到底是哪个证书文件。

## References

1. https://codereview.chromium.org/2903743002
2. https://api.dart.dev/stable/2.13.4/dart-io/SecurityContext-class.html
3. https://github.com/dart-lang/root_certificates
4. https://www.codeproject.com/Articles/326574/An-Introduction-to-Mutual-SSL-Authentication
5. https://en.wikipedia.org/wiki/Mutual_authentication




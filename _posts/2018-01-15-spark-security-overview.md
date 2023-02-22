---
layout: post
title: "[转载]Spark Security面面观"
date: 2018-01-15
author: "Jerry Shao"
header-img: "img/2018-01-15-spark-security-overview/data-security.jpg"
category: jerryshao
tags:
    - spark
    - cloud
    - security
---

### 背景

作为一款成熟的商业软件，安全往往鲜少被提及但又不可忽略，大数据软件也是如此。在生产环境中，对于一款成熟的大数据软件的考量，不仅需要考虑其功能完备性和性能，同时安全也是不可缺少的一环。为什么安全如此重要呢？

* 首先，商业环境通常是多租户环境，不同的用户/组对于不同的数据/应用有不同的安全考量。我们需要保证相应的用户不能做出超越权限的操作。
* 同时，分布式架构会将端口、数据暴露出去，如果没有相应的认证、加密措施，这会大大增加了匿名攻击的概率。

    举个简单的例子，Spark通过RPC通信协调`driver`和`executor`之间的任务分发，如果一个匿名用户通过伪造协议与`executor`进行通信，会产生什么后果呢？

    在一个可控的环境下（比如防火墙物理隔离）这样的风险相对可控。但是随着越来越多的云计算厂商提供EMR类似产品，这样的风险不可谓不存在。

* 再者，对于一个完整的大数据生态系统，安全往往是相互衔接的。如果中间有一款软件没有相应的支持，那么它就会成为整体的薄弱点，或者说会造成整体完备性的缺失。

    比方说，Hadoop系统的认证体系是建立在Kerberos基础上的，一款软件要融入Hadoop生态系统，它必定也要支持Kerberos认证，不然它是无法与其他系统进行交互的。

当然还有很多其他的原因，在这就不一一赘述了。

回到我们今天的话题上来，Spark作为大数据生态圈中的重要一环，许多厂商已经在其生产环境中大量部署了Spark集群，Spark本身在安全上面有哪些点值得我们关注呢？

## Spark Security

[Apache Spark](https://spark.apache.org/docs/latest/security.html)在其演化过程中，安全的支持也是逐渐添加和完善的。在Spark开发的早期，是没有任何安全上面的考量的，随着各大厂商在生产环境中的部署，对于安全的支持也是在逐步的完善。

总结前面提到的三个方面，对于Spark本身，我们需要考虑的三个方面是：

1. 权限管理
2. 数据/链路加密
3. 与其他安全系统交互

下文会就以上3点进行讨论，来看看在Spark中安全是如何实现的。

### 权限管理

上文所提到的第一个问题是**保证相应的用户不能做出超越权限的操作**，那么在Spark中如何做到这一点呢？

#### ACL

Spark支持基本的ACL(Access Control List)，通过配置不同用户的角色，能为不同的用户赋予不同的权限范围。

为了启动ACL机制，用户需要配置`spark.acls.enable`为`true`。然后将不同的用户名置于不同的配置中：

`spark.ui.view.acls`

`spark.modify.acls`

`spark.admin.acls`

这3个配置分别对应了可读、可改、和可管理三种不同的权限。集群管理者可将用户名置于相应的列表以赋予不同的权限。

对于大型集群来说，按用户来配置过于繁琐，Spark也可以支持按组进行配置，如：

`spark.ui.view.acls.groups`

`spark.modify.acls.groups`

`spark.admin.acls.groups`

那么进行了上述这些配置以后会带来什么变化呢？

* 首先，只有具有相应权限的用户才能在UI上浏览每个Spark应用的具体内容，包括配置、作业、任务等等。
* 其次，只要具有可改权限的用户才能在UI上停止作业。
* 最后，默认当前spark应用启动用户具有可读和可改的权限。

当前Spark的ACL主要用于UI权限的管理，以保证用户只能浏览相应权限的内容。对于HistoryServer，则有另一套类似的配置，如`spark.history.ui.acls.enable`，`spark.history.ui.admin.acls`，在这就不具体展开了。

#### Spnego Authentication

开启了ACL就能对UI进行权限隔离了吗？其实我们还只走了一半路。为了使web server（Spark内嵌Jetty）能够对用户进行隔离，首先Jetty需要知道是谁发起了HTTP请求，换句话说，只有用户的HTTP请求中包含了用户信息，UI和ACL才能根据用户信息进行正确的隔离。但是不幸的是，普通的HTTP请求是不包含用户信息。

为此需要为Jetty servlet加入authentication filter以获取认证用户的信息。在这之中，最常用的就是[Spnego authentication](https://en.wikipedia.org/wiki/SPNEGO).

Spnego是一套以Kerberos为基础的HTTP认证机制，只有经过Kerberos授权的HTTP请求才能被web server所接受。Hadoop生态系统下的各个组件的UI大都采用Spnego作为认证机制，如HDFS，YARN等。

为了使Spark UI能够使用Spnego认证，用户需要实现相应的authentication filter并将其添加到Jetty中。幸运的是Hadoop已经帮我们实现了相应的filter，只需将其配置就可使用。

```
spark.ui.filters=org.apache.hadoop.security.authentication.server.AuthenticationFilter

spark.org.apache.hadoop.security.authentication.server.AuthenticationFilter.params=type=kerberos,kerberos.pricipal=<kerberos-principal>,kerberos.keytab=<kerberos-keytab>
```

在这里，需要为Spark UI创建Kerberos principal和keytab。这样Spark UI就有了Spnego authentication的能力了，任何用户在发起HTTP请求之前必须先获得Kerberos tgt。使用curl的话，如：

```shell

$ kinit
$ curl --negotiate -u : <host>:<port>/<xxx>

```

经过Spnego认证的HTTP请求在其HTTP头部包含用户信息，Spark UI和ACL机制会从其中获取用户名并在相应的权限列表中进行比对。

#### LDAP and Others

当然，Spengo认证只是众多authentication中的一种，其他还有如[LDAP](https://www.ldap.com/getting-started-with-ldap)或是basic authentication等等。其中LDAP已经在[Hadoop 2.8](https://issues.apache.org/jira/browse/HADOOP-12082)中实现了，用户可以采用以上类似的配置实现LDAP认证。

同样，用户可以是实现并配置自己的authentication filter，只要依照servlet和Jetty的规范即可。

### 数据/链路加密

数据/链路加密是Spark安全中最为重要的一块，它主要是为了防止匿名用户通过端口获取报文，发送malicious数据，同时也需要防止用户伪造spilled数据（如shuffle数据）以破坏运行作业。下文就将数据和链路加密分别进行介绍。

#### 数据加密

在Spark中，往local disk上写的数据主要包含：

1. Shuffle数据，包括shuffle data和shuffle index file。
2. Shuffle spill数据，当内存无法容纳时向disk spill的数据。
3. BlockManager存储在disk的block数据。
4. 等等。。。

所有这些数据都是以二进制的方式将序列化（压缩后）的数据写入到local文件系统中。举例来说，比如匿名用户知道了Spark中shuffle数据的命名方式和映射规律，那么他就能伪造一份新的shuffle数据。为了防止此种情况的发生，我们需要对写到disk上的数据进行加密。

为此我们需要配置如下：

|Property                            |Default |Meaning|
|----------------------------------- |--------|-------|
|spark.io.encryption.enabled         |false	  |Enable IO encryption. Currently supported by all modes except Mesos. It's recommended that RPC encryption be enabled when using this feature.|
|spark.io.encryption.keySizeBits     |128     |	IO encryption key size in bits. Supported values are 128, 192 and 256.|
|spark.io.encryption.keygen.algorithm|HmacSHA1|	The algorithm to use when generating the IO encryption key. The supported algorithms are described in the KeyGenerator section of the Java Cryptography Architecture Standard Algorithm Name Documentation.|

当配置了这些选项以后，所有写入local disk的数据都会以加密方式写入。

#### 链路加密

链路加密，就是所谓的wire encryption，在Spark中主要包含两种类型的链路：

1. HTTP连接。Spark的live UI和History UI提供了WEB UI供用户浏览应用的具体细节。我们希望HTTP链路能够加密以防止匿名用户的劫持。
2. Binary连接。主要包含RPC链路、Shuffle链路和Block传输。在Spark中这一部分是由Netty实现的。我们希望所有的端到端的传输都是可信赖的，一个不被信赖的链接是无法发送数据的。

根据上面两种类型的链路，我们来看看应该如何配置Spark使链路能得到加密。

##### HTTPS/SSL

在Spark中，包含HTTP的链路分别有：

* Live UI，Spark应用的WEB UI。
* History UI，Spark history Server的WEB UI。
* Standalone UI，Spark Standalone cluster manager中master和worker node的WEB UI

为此Spark可以以统一的SSL配置来配置所有的HTTPS endpoint，也可以对每一个组件分别进行配置.

当用户配置了`spark.ssl.enabled`，这表示所有HTTPS endpoint都启动了SSL并使用相同的配置。用户也可以通过配置`spark.ssl.ui.xxxx`只启动和配置Live UI的SSL。当前支持的component包括`ui`，`standalone`和`historyServer`。

在配置之前，用户需要生成相应的keystore和truststore，可以通过使用Java keytool工具来生成相应的keystore和truststore，具体可以参考Oracle的[官方文档](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/keytool.html)。

|Property Name|	Default|	Meaning|
|-------------|--------|---------|
|spark.ssl.enabled|	false|	Whether to enable SSL connections on all supported protocols. When spark.ssl.enabled is configured, spark.ssl.protocol is required. All the SSL settings like spark.ssl.xxx where xxx is a particular configuration property, denote the global configuration for all the supported protocols. In order to override the global configuration for the particular protocol, the properties must be overwritten in the protocol-specific namespace. Use spark.ssl.YYY.XXX settings to overwrite the global configuration for particular protocol denoted by YYY. Example values for YYY include fs, ui, standalone, and historyServer. See SSL Configuration for details on hierarchical SSL configuration for services.|
|spark.ssl.[namespace].port|	None|	The port where the SSL service will listen on. The port must be defined within a namespace configuration; see SSL Configuration for the available namespaces. When not set, the SSL port will be derived from the non-SSL port for the same service. A value of "0" will make the service bind to an ephemeral port.|
|spark.ssl.enabledAlgorithms|	Empty|	A comma separated list of ciphers. The specified ciphers must be supported by JVM. The reference list of protocols one can find on this page. Note: If not set, it will use the default cipher suites of JVM.|
|spark.ssl.keyPassword|	None|	A password to the private key in key-store.|
|spark.ssl.keyStore|	None|	A path to a key-store file. The path can be absolute or relative to the directory where the component is started in.|
|spark.ssl.keyStorePassword|	None|	A password to the key-store.|
|spark.ssl.keyStoreType|	JKS|	The type of the key-store.|
|spark.ssl.protocol|	None|	A protocol name. The protocol must be supported by JVM. The reference list of protocols one can find on this page.|
|spark.ssl.needClientAuth|	false|	Set true if SSL needs client authentication.|
|spark.ssl.trustStore|	None|	A path to a trust-store file. The path can be absolute or relative to the directory where the component is started in.|
|spark.ssl.trustStorePassword|	None|	A password to the trust-store.|
|spark.ssl.trustStoreType|	JKS|	The type of the trust-store.|

需要注意的是：

* keystore和truststore是保存在本地文件系统中的，这意味着当你需要为standalone cluster manager启动SSL时，需要在所有的节点上生成和保存相应的keystore，truststore。
* 当前YARN并不支持proxy HTTPS request，所以在使用YARN的时候enable SSL会带来一定的问题。
* 当你在需要mutual authentication的时候才需要enable `spark.ssl.needClientAuth`。

##### SASL

说完了HTTPS/SSL之后，让我们来看看另一种类型的链路，Netty连接。在Spark中Netty连接也可以进行[SASL认证](https://en.wikipedia.org/wiki/Simple_Authentication_and_Security_Layer)。用户只需要进行如下的配置，所有的Netty链路，包括RPC和Block transfer，都会进行认证。

|Property Name|	Default|	Meaning|
|-------------|--------|---------|
|spark.authenticate|	false|	Whether Spark authenticates its internal connections. See spark.authenticate.secret if not running on YARN.|
|spark.authenticate.secret|	None|	Set the secret key used for Spark to authenticate between components. This needs to be set if not running on YARN and authentication is enabled.|
|spark.authenticate.enableSaslEncryption|	false|	Enable encrypted communication when authentication is enabled. This is supported by the block transfer service and the RPC endpoints.|
|spark.network.sasl.serverAlwaysEncrypt|	false|	Disable unencrypted connections for services that support SASL authentication.|

需要注意的是，当用户使用的不是YARN cluster manager运行Spark应用，用户需要配置`spark.authenticate.secret`。

另外，当前Spark SASL所使用的认证方式是[DIGEST-MD5](https://en.wikipedia.org/wiki/Digest_access_authentication)，它是一种比较弱的认证方式，容易被暴力破解，因此现在并不推荐使用（[Moving DIGEST-MD5 to Historic](https://tools.ietf.org/html/rfc6331)）。在Spark中提供了另一种较强的认证方式，可以通过配置开启：

|Property Name|	Default|	Meaning|
|-------------|--------|---------|
|spark.network.crypto.enabled|	false|	Enable encryption using the commons-crypto library for RPC and block transfer service. Requires spark.authenticate to be enabled.|
|spark.network.crypto.keyLength|	128|	The length in bits of the encryption key to generate. Valid values are 128, 192 and 256.|
|spark.network.crypto.keyFactoryAlgorithm|	PBKDF2WithHmacSHA1|	The key factory algorithm to use when generating encryption keys. Should be one of the algorithms supported by the javax.crypto.SecretKeyFactory class in the JRE being used.|
|spark.network.crypto.saslFallback|	true|	Whether to fall back to SASL authentication if authentication fails using Spark's internal mechanism. This is useful when the application is connecting to old shuffle services that do not support the internal Spark authentication protocol. On the server side, this can be used to block older clients from authenticating against a new shuffle service.|

### 与其他安全系统交互

在一个完整的生产环境集群中，Spark并不是唯一的系统，通常还需要与其他的调度、存储系统进行交互，如HDFS，YARN等。如果其他的系统都是需要安全认证的，Spark作为其他系统的使用方，如何进行交互呢？

每种系统的认证方式各不相同，没有一个普适的框架能够满足所有的认证方式，在这就以最常见的认证方式Kerberos来解释Spark是如何与其他Kerberized系统机型交互的。

首先每个Spark应用的提交者都需要通过Kerberos的认证，获取Kerberos的TGT。以HDFS为例，当Spark应用拿到TGT后就可以与HDFS进行交互，HDFS在通过验证后会给予Spark应用一个有效的Delegation Token来代表此用户，后续的交互都是采用Delegation Token验证的方式，而无需在进行Kerberos认证。

但是Delegation Token会过期，当它过期后就不再能够使用，为此Spark应用需要重新获得新的Delegation Token，获取的方式与上面介绍的一样（首先获得Kerberos TGT，通过Kerberos TGT获得Delegation Token）。当然TGT也会过期，过期后需要用户重新kinit，或是使用keytab和principal的方式让应用自己去更新TGT。

对于Kerberos和Delegation Token，在这就不做过多的介绍。关于Hadoop Security的设计，有一个非常好的[设计文档](http://www.carfield.com.hk/document/distributed/hadoop-security-design.pdf?)，对于分布式系统安全设计有非常大的指导意义。

## 总结

本文从总体上介绍了Spark中关于安全的各个方面，以及如何开启和配置这些安全机制。本文并未从原理和代码上阐述所有这些安全机制背后的原理以及实现，因为对于每一种安全机制都可以长篇大论赘述其由来。相反的是，相比于原理，怎么去使用它更为重要，毕竟绝大多数的用户/开发者不会涉及到里面真正的细节。

---
title: LoTW 不只确认 QSO：探索基于数字证书的业余无线电持牌人身份验证
date: 2026-11-15 17:00:00
tags: 
    - 业余无线电 
    - 密码学
---

# 引言
对于大部分业余无线电社区以及其他专为无线电爱好者（eg. [HamCQ](//hamcq.cn), [RadioID](//radioid.net)）提供服务平台而言，验证用户确实是持牌的HAM总是必要的。

而目前常见的方案主要是由平台的运营者（很多时候是志愿者）审核用户上传的执照，这种机制需要平台投入精力进行审核，用户也需要等待较长的审核时间，同时不能避免个人信息泄露的风险。
是否存在一种更为高效、安全的机制，可以在用户尽可能少披露个人信息的条件下实现对用户身份的验证？

# 信任锚
美国无线电转播联盟(ARRL)所提供的[Logbook Of The World(LoTW)](//lotw.arrl.org)是一个广为使用的电子通联日志确认系统，与传统的QSL卡片亦或[eQsl.cc](//eqsl.cc)等电子日志确认平台不同，LoTW引入了基于公钥密码学的数字签名机制，对操作员提交的日志进行签名，以此验证日志的来源和完整性，降低潜在的伪造通联记录的风险。

根据LoTW的[身份认证指引](https://lotw.arrl.org/lotw-help/authentication),每位用户必须通过ARRL的身份验证，才能获得与其呼号绑定的数字证书：

- 对于美国持牌操作员，ARRL 将验证明信片寄送至美国联邦通信委员会（FCC）执照数据库中登记的地址，申请人必须提供明信片上的验证码，才能完成首次证书申请。这一机制通过 FCC 的官方执照登记信息及申请人对登记地址的控制能力，建立呼号与申请人之间的可信联系。
- 对于美国以外的操作员，申请人原则上需要提交业余无线电操作许可文件，以及能够证明本人身份的其他官方文件。ARRL 将审核这些材料，或由愿意参与核验的 ARRL DXCC Card Checker 当面查验，并将核验结果提交给 LoTW 工作人员。只有审核通过后，ARRL 才会签发相应的呼号证书。

因此，LoTW的数字证书建立在ARRL对身份信息以及呼号使用资格进行核验的基础之上，为呼号与相应密码学身份之间的绑定提供了可信基础。。在[LoTW 系统介绍](//lotw.arrl.org/lotw/intro)中也明确指出，签发证书前的身份核验是防止用户提交虚假通联记录的重要措施。

在技术实现上,[Developer Information: Public-Key Encryption in LoTW](https://lotw.arrl.org/lotw-help/developer-pki)指出：

>LoTW uses public-key cryptography to authenticate QSOs submitted by each user. Public-key cryptography employs two separate but mathematically-related keys, one of which is private and one of which is public. Information encrypted with a private key can only be decrypted with its associated public key, proving that the holder of the private key was the source of the information.
>
>When the user directs TQSL to request a Callsign Certificate, TQSL generates a private key and a public key. It stores the private key in the user's computer, and sends the public key to the ARRL. The ARRL creates a Callsign Certificate that links the user's callsign with the public key, and emails the Callsign Certificate to the user in a file with a .tq6 filename extension.

>The LoTW Server decrypts each received QSO signature using your public key, and verifies its consistency with the received QSO -- thereby proving that you are the source of that QSO and that the QSO's details have not been changed.

由此，我们可以将ARRL视为本文构建的身份验证体系中的可信第三方，第三方平台可以复用这一既有的信任关系，通过验证用户对某一呼号对应的数字证书私钥的控制,进而推出用户享有使用该呼号的资格。

# Challenge-Response认证
在上一节，我们已经通过ARRL所建设的公钥基础设施（PKI）建立了呼号与数字证书间的可信绑定，但是为了达成我们的最终目的，还需要证明用户对于数字证书的控制。但直接让用户交出私钥显然不是一个好主意…… 因此，我们需要一种机制，使用户能够在不披露私钥的前提下，向第三方证明自己确实持有与该呼号证书匹配的私钥。

挑战响应认证（Challenge-Response Authentication）是一种常见的身份认证机制，其基本思想是由验证方生成一个不可预测的随机挑战（Challenge），要求被验证方使用某种只有自己掌握的凭据计算响应（Response），再由验证方检查响应是否正确。

对于本文所构建的系统而言，这一过程可以通过数字签名实现：
- 第三方平台生成一个随机挑战，并发送给用户。

- 用户使用 LoTW 呼号证书对应的私钥，对包含该挑战的认证消息进行数字签名。

- 用户将签名结果及对应的 LoTW 数字证书提交给平台。

- 平台验证证书的可信性，并使用证书中的公钥验证数字签名。

- 若证书和签名均有效，且挑战满足时效性等要求，则认证成功。

在这一过程中，私钥并没有脱离用户的控制，平台只需要获取数字证书、响应（即用户的数字签名）和原始挑战，即可验证用户是否拥有相应的密码学凭据。

与直接要求用户上传执照不同，本文所构建的系统将身份认证转化为一次密码学验证过程，既可以减少人工审核，也有望降低用户向多个平台重复披露个人身份资料的必要性。

# 原形实现
我们已经在理论层面上构建了一套用户到呼号的认证体系，但是这一体系是否可行，仍有待验证。

## 解析证书
使用 OpenSSL 解析由 TQSL 导出的 .p12 文件，可以确认该文件采用 `PKCS#12 容器格式`。在本次实验中，该容器包含`呼号证书`、`LoTW Production CA 中间证书`、`LoTW Root CA 根证书`，以及与呼号证书对应的加密私钥。
``` bash
> $ openssl pkcs12 -in <Callsign>.p12 -info -nokeys
Enter Import Password:
MAC: sha256, Iteration 2048
MAC length: 32, salt length: 8
PKCS7 Encrypted data: PBES2, PBKDF2, AES-256-CBC, Iteration 2048, PRF hmacWithSHA256
Certificate bag
Bag Attributes
    localKeyID: *
    friendlyName: TrustedQSL user certificate
subject=1.3.6.1.4.1.12348.1.1=<Callsign>, CN=*, emailAddress=*
issuer=C=US, ST=CT, L=Newington, O=American Radio Relay League, OU=Logbook of the World, CN=Logbook of the World Production CA, DC=arrl.org, emailAddress=lotw@arrl.org
-----BEGIN CERTIFICATE-----
......
-----END CERTIFICATE-----
Certificate bag
Bag Attributes: <No Attributes>
subject=C=US, ST=CT, L=Newington, O=American Radio Relay League, OU=Logbook of the World, CN=Logbook of the World Production CA, DC=arrl.org, emailAddress=lotw@arrl.org
issuer=C=US, ST=CT, L=Newington, O=American Radio Relay League, OU=Logbook of the World, CN=Logbook of the World Root CA, DC=arrl.org, emailAddress=lotw@arrl.org
-----BEGIN CERTIFICATE-----
......
-----END CERTIFICATE-----
Certificate bag
Bag Attributes: <No Attributes>
subject=C=US, ST=CT, L=Newington, O=American Radio Relay League, OU=Logbook of the World, CN=Logbook of the World Root CA, DC=arrl.org, emailAddress=lotw@arrl.org
issuer=C=US, ST=CT, L=Newington, O=American Radio Relay League, OU=Logbook of the World, CN=Logbook of the World Root CA, DC=arrl.org, emailAddress=lotw@arrl.org
-----BEGIN CERTIFICATE-----
......
-----END CERTIFICATE-----
PKCS7 Data
Shrouded Keybag: PBES2, PBKDF2, AES-256-CBC, Iteration 2048, PRF hmacWithSHA256
```

接下来可以尝试对证书及私钥进行分离，在提取私钥时openssl会要求设置私钥的密码，避免明文存储。
``` bash
> $ openssl pkcs12 -in <Callsign>.p12 -clcerts -nokeys -out <Callsign>.pem
Enter Import Password:

> $ openssl pkcs12 -in <Callsign>.p12 -nocerts -out <Callsign>.key
Enter Import Password:
Enter PEM pass phrase:
Verifying - Enter PEM pass phrase:
```

## 构造Challenge
在实际系统中，挑战应由服务器使用密码学安全随机数生成器产生，并与特定认证会话绑定。为了便于实验，我们直接通过 OpenSSL 生成一个 256 bit 的随机数, 并且规定如下的Challenge格式。
``` json
# <Callsign>.challenge
{
    "Protocol": "callsign-verify-v1",
    "Audience": "example.com",
    "Callsign": "<Callsign>",
    "Nonce": "ca97c04b572516a468e8a3d390291b4693e3a90b6cf3b604d3f70f9a77936e30"
}
```

## 数字签名
用户收到 Challenge 后，在本地使用 LoTW 私钥为认证消息生成数字签名。

使用openssl执行：
``` bash
> $ openssl dgst -sha256 \
    -sign <Callsign>.key \
    -out <Callsign>.response \
    <Callsign>.challenge
```
openssl 将读取私钥，并在需要时提示输入私钥保护密码。随后，它会计算挑战消息的 SHA-256 摘要，并使用相应的私钥签名算法生成数字签名。

最终得到的 `<Callsign>.response`即为我们需要的Response。


## 验证签名
首先从前面获得的`<Callsign>.pem`中提取公钥：
``` bash
> $ openssl x509 -in <Callsign>.pem -pubkey -noout > <Callsign>.pub
```
然后使用公钥对签名进行验证：
``` bash
> $ openssl dgst -sha256 \
  -verify <Callsign>.pub \
  -signature <Callsign>.response \
  <Callsign>.challenge
Verified OK
```
这说明 `<Callsign>.response` 是一个能够通过 `<Callsign>.pub` 中公钥验证的有效签名。

为了进一步验证签名与消息之间的绑定关系，我们还可以修改 `challenge.txt` 中的任意字符，然后重新执行验签命令。

在签名算法安全且实现正确的前提下，修改后的消息无法再通过原有签名的验证。

初次之外，我们还需要验证用户提交的证书确实是颁发给正在验证的呼号的：
``` bash
> $ openssl x509 -in <Callsign>.pem -noout -subject
subject=1.3.6.1.4.1.12348.1.1=<Callsign>, CN=*, emailAddress=*
```
关注`1.3.6.1.4.1.12348.1.1`，发现此项与我们正在验证的呼号一致。

由此，我们能够证明：只要用户能够调用 LoTW 证书对应的私钥进行签名，第三方就可以使用证书中的公钥验证其对指定挑战的响应，而不必获取用户的私钥。

## 验证证书可信度
不过，到这里我们的实验还只完成了 Challenge-Response 中的密码学验证，并没有完成真正意义上的身份认证。

这是因为，单纯验证签名只能证明响应方持有与某个公钥对应的私钥，却不能证明该公钥确实由 ARRL 绑定至相应的业余无线电呼号。

为了解决这一问题，第三方还需要验证证书的签发关系。

在标准 X.509 PKI 中，这通常需要验证证书链，并确认其最终能够追溯至预先信任的 CA。

通过查询[资料](https://www.qsl.net/kb9mwr/wapr/tcpip/lotw-vpn.html)发现，ARRL的根证书存储在`~/.tqsl/certs/root`路径下,使用openssl进行验证：
``` bash
> $ openssl x509 -in ~/.tqsl/certs/root -noout -text
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            b6:dc:09:11:16:4d:9c:51
        Signature Algorithm: sha512WithRSAEncryption
        Issuer: C=US, ST=CT, L=Newington, O=American Radio Relay League, OU=Logbook of the World, CN=Logbook of the World Root CA, DC=arrl.org, emailAddress=lotw@arrl.org
        Validity
            Not Before: Jun 28 12:28:35 2023 GMT
            Not After : Jun 25 12:28:35 2033 GMT
        Subject: C=US, ST=CT, L=Newington, O=American Radio Relay League, OU=Logbook of the World, CN=Logbook of the World Root CA, DC=arrl.org, emailAddress=lotw@arrl.org
......
```

然而，当我们利用CA证书进行验证时发现我们的证书并不受信任。
``` bash
> $ openssl verify -show_chain \
-CAfile ~/.tqsl/certs/root \
<Callsign>.pem
1.3.6.1.4.1.12348.1.1=<Callsign>, CN=*, emailAddress=*
error 20 at 0 depth lookup: unable to get local issuer certificate
error <Callsign>.pem: verification failed
```
结合我们对`<Callsign>.p12`的分析，发现在验证过程中缺少了`Logbook of the World Production CA`这张中间证书。

添加位于`~/.tqsl/certs/authorities`的中间证书后便可以顺利通过验证：
``` bash
> $ openssl verify -show_chain \
-CAfile ~/.tqsl/certs/root \
-untrusted ~/.tqsl/certs/authorities \
<Callsign>.pem
<Callsign>.pem: OK
Chain:
depth=0: 1.3.6.1.4.1.12348.1.1=<Callsign>, CN=*, emailAddress=* (untrusted)
depth=1: C=US, ST=CT, L=Newington, O=American Radio Relay League, OU=Logbook of the World, CN=Logbook of the World Production CA, DC=arrl.org, emailAddress=lotw@arrl.org (untrusted)
depth=2: C=US, ST=CT, L=Newington, O=American Radio Relay League, OU=Logbook of the World, CN=Logbook of the World Root CA, DC=arrl.org, emailAddress=lotw@arrl.org
```

## 证书吊销列表
据此，我们实现了一个呼号认证系统的原形，但是仍然存在一个问题：一张证书可能在到期前由于泄漏、遗失等原因提前被吊销。

LoTW的文档[Determining Whether a Callsign Certificate is Valid](https://lotw.arrl.org/lotw-help/certificatevalidity/?lang=en)指出:

>When a Callsign Certificate is issued for a Callsign, any previous Callsign Certificate for that Callsign is rendered invalid. LoTW will reject QSOs digitally signed by an invalid Callsign Certificate.
>
>Each Callsign Certificate has a unique serial number. LoTW maintains a list of your valid Callsign Certificates that includes the serial number for each. To verify that a Callsign Certificate in TQSL is valid, you can compare it's serial number with the serial number displayed by LoTW.

而在其他X.509体系中，通常使用`证书吊销列表（CRL）`实现对于证书吊销状态的检查。经过一番查找，发现LoTW同样提供了CRL。

我们使用CRL进行验证：
``` bash
> $ curl https://lotw.arrl.org/lotw/crl\?serial\=<Serial Number>
<LoTW_CRL>
  <Serial><Serial Number></Serial>
  <Status>Unrevoked</Status>
</LoTW_CRL>
```
而如果查询一张已经失效的证书，则会返回`Revoked`或`Superceded`等状态。

至此，我们完成了基于 LoTW 数字证书的呼号身份认证体系的核心可行性验证。

上述结果表明，在预先建立可信的 ARRL 根 CA 信任锚、正确验证证书身份属性及有效状态，并实现挑战随机数管理与防重放机制的前提下，复用 LoTW PKI 构建第三方呼号身份认证服务具有技术可行性。

# CMS封装
在上一节，我们已经实现了基于LoTW数字证书的呼号身份认证体系的原形。但是在验证过程需要用户分别上传证书和响应，看起来并不优雅。

几经检索，发现了一种封装形式：`CMS（Cryptographic Message Syntax，密码消息语法）` 是由 RFC 5652 定义的密码学消息封装标准，支持数字签名、签名验证、数据加密与解密等功能，广泛应用于基于公钥基础设施（PKI）的安全通信与身份认证。

我们尝试使用分离出的证书和私钥对Challenge进行签名并打包,获得`<Callsign>.response`：
``` bash
> $ openssl cms -sign -binary \
 -in <Callsign>.challenge \
 -signer <Callsign>.pem \
 -inkey <Callsign>.key \
 -certfile ~/.tqsl/certs/authorities \
 -outform DER \
 -nodetach \
 -md sha256 \
 -out <Callsign>.response
```

然后再次对响应进行验证：
``` bash
> $ openssl cms -verify \
-binary \
-inform DER \
-in <Callsign>.response \
-CAfile ~/.tqsl/certs/root \
-out verified-challenge.txt \
-signer verified-cert.pem
CMS Verification successful
```

在验证签名的同时分离出了用于签名的证书`verified-cert.pem`和`verified-challenge.txt`，我们便可以使用上文的方法对呼号和原始挑战进行验证。

于是，我们便以更优雅的方式实现了我们的功能。

# 不足之处
虽然我们已经达成了最初的目的，但是目前在原形设计和体系结构上仍存在一定的缺陷。

为了尽可能简单的进行验证，我们所构造的原形存在如下已知的缺陷：
- 为了经可能简单的构造挑战，我们并没有为Challenge设计有效期，也没有采取其他防止重放攻击的举措；
- 由于验证者和证明者处于同一环境，我们没有并关注中间证书的处理，而是直接使用了同一张位于`~/.tqsl/certs/authorities`的证书；
在正式实现中，中间证书应该在用户的p12中分离，并且随CMS传递至服务端；
- 我们直接将位于`~/.tqsl/certs/root`视为ARRL的根证书使用，而没有验证此证书的真实性。在正式实现中，应当对从其他可信途径获取根证书并加以验证。

其他缺陷：
- 本文所构建的体系需要用户使用openssl对证书进行分离、签名，或者安装第三方平台提供的签名工具进行自动化操作，仍旧不是很优雅。
（虽然说我们可以将Challenge伪装成一个ADIF日志的格式进行签名，但这样子很难说没有伪造通联日志的嫌疑，违反LoTW的政策）;
- 该体系并不能检测用户执照的等级（一些平台存在获取用户证书等级的需求）；
- 该体系不能判断用户执照的有效性（包括伪造，或注销等情况），这是由ARRL的审核机制决定的，其他平台通过上传执照的方式验证似乎同样无法避免。
（想要实现这个功能，似乎只能依靠各实体的无线电管理当局开放数据库）

# 结语

至此，我们基于 ARRL LoTW 现有的 PKI 基础设施，初步实现了一套面向业余无线电社区的呼号身份认证机制。通过复用 LoTW 签发的 X.509 数字证书，结合 Challenge-Response 协议与 CMS 签名封装，我们能够在减少披露个人信息的前提下，借助 ARRL 的证书信任链，建立一种高效的呼号身份认证机制。并且其安全性并不低于人工审核执照的方案。

这一方案的意义并不在于提出新的密码学算法，而在于探索现有信任基础设施的进一步利用。我们希望可以建立一个类似于 UNiDAYS 的统一身份认证平台，充分利用 LoTW 已经建立的、面向业余无线电操作员的证书签发与身份核验体系，为业余无线电社区提供一种统一、可信且可互操作的呼号身份认证服务。第三方网站、社区平台及其他服务提供者可以通过这一平台核验用户的呼号身份，而无须各自建立独立的执照审核流程。

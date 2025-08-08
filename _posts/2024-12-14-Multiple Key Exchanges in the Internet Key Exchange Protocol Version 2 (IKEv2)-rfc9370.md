---
title: Multiple Key Exchanges
description: 附加多次密钥交换 IKEv2-rfc9370 IKEv2
author: dead4ead
date: 2024-12-14 12:00:00 +0800
categories: [Secure, IPSec]
tags: [IKEv2, IPSec]
#pin: true
math: true
mermaid: true
---

Multiple Key Exchanges in the Internet Key Exchange Protocol Version 2 (IKEv2)

## Introduction
The security of the (EC)DH algorithms relies on the difficulty to solve a discrete logarithm problem in multiplicative (and, respectively, elliptic curve) groups when the order of the group parameter is large enough. While solving such a problem remains infeasible with current computing power, it is believed that general-purpose quantum computers will be able to solve this problem, implying that the security of IKEv2 is compromised.  
(EC)DH 算法的安全性依赖于当群参数的阶数足够大时求解乘法群（以及椭圆曲线群）中的离散对数问题的难度。 虽然以当前的计算能力解决这个问题仍然不可行，但相信通用量子计算机将能够解决这个问题，这意味着 IKEv2 的安全性受到了损害。  
It is essential to have the ability to perform one or more post-quantum key exchanges in conjunction with an (EC)DH key exchange so that the resulting shared key is resistant to quantum-computer attacks.  
必须能够结合 (EC)DH 密钥交换来执行一个或多个后量子密钥交换，以便生成的共享密钥能够抵抗量子计算机攻击。  

------
This document describes a method to perform multiple successive key exchanges in IKEv2.  
本文档描述了一种在 IKEv2 中执行多次连续密钥交换的方法。  

It is believed that the feature of using more than one post-quantum algorithm is important, as many of these algorithms are relatively new, and there may be a need to hedge the security risk with multiple key exchange data from several distinct PQC algorithms.  
人们认为，使用多个后量子算法的特征很重要，因为其中许多算法相对较新，并且可能需要通过来自几种不同的 PQC 算法的多个密钥交换数据来对冲安全风险。  

----
Then, before the IKE_AUTH exchange, one or more IKE_INTERMEDIATE exchanges are carried out, each of which contains an additional key exchange.
在 IKE_AUTH 交换之前，执行一次或多次 IKE_INTERMEDIATE 交换，每一次交换都包含一个附加的密钥交换。  
While this extension is primarily aimed at IKE SAs due to the potential fragmentation issue discussed above.  
虽然由于上面讨论的潜在碎片问题，此扩展主要针对 IKE SA，但它也适用于 CREATE_CHILD_SA 交换.

## Multiple Key Exchanges

### Design Overview
Most post-quantum key agreement algorithms are relatively new. Thus, they are not fully trusted. There are also many proposed algorithms that have different trade-offs and that rely on different hard problems. The concern is that some of these hard problems may turn out to be easier to solve than anticipated; thus, the key agreement algorithm may not be as secure as expected. A hybrid solution, when multiple key exchanges are performed and the calculated shared key depends on all of them, allows us to deal with this uncertainty by combining a classical key exchange with a post-quantum one, as well as leaving open the possibility of combining it with multiple post-quantum key exchanges.  
大多数后量子密钥协商算法都相对较新。 因此，他们并没有被完全信任。 还有许多提出的算法具有不同的权衡，并且依赖于不同的难题。 令人担忧的是，其中一些难题可能比预期更容易解决；因此，密钥协商算法可能不像预期的那么安全。 当执行多个密钥交换并且计算出的共享密钥取决于所有密钥交换时，混合解决方案使我们能够通过将经典密钥交换与后量子密钥交换相结合来处理这种不确定性，并保留组合的可能性它具有多个后量子密钥交换。  
Note that this document assumes that each key exchange method requires one round trip and consumes exactly one IKE_INTERMEDIATE exchange.For hypothetical future key exchange methods that require multiple round trips to complete, a separate document should define how such methods are split into several IKE_INTERMEDIATE exchanges.  
本文档假设每种密钥交换方法都需要一次往返，并且恰好消耗一次 IKE_INTERMEDIATE 交换。 对于假设的未来密钥交换方法需要多次往返才能完成，单独的文档应定义如何将此类方法拆分为多个 IKE_INTERMEDIATE 交换。  

------
To negotiate additional key exchanges, seven new Transform Types are defined. These transforms and Transform Type 4 share the same Transform IDs.  
为了协商额外的密钥交换，定义了七种新的转换类型。 这些转换和转换类型 4 共享相同的转换 ID。  

------
事实上，新定义的转换与转换类型 4 共享可能的转换 ID 的相同注册表，允许任何类型的附加密钥交换：后量子或经典 (EC)DH。 这种方法允许发生定义的密钥交换方法的任意组合。 这还允许 IKE 对等方在 IKE_SA_INIT 中执行单个后量子密钥交换，而无需额外的密钥交换，前提是 IP 分段不是问题并且不需要混合密钥交换。

IKE_SA_INIT 消息中的 SA 有效负载包括一个或多个新定义的转换，这些转换表示发起方所需的额外密钥交换策略。 响应方遵循通常的 IKEv2 协商规则：它选择每种类型的单个转换，并在 IKE_SA_INIT 响应消息中返回所有这些转换。

然后，如果协商了附加密钥交换，发起方和响应方将执行一次或多次 IKE_INTERMEDIATE 交换。 接下来，IKE_AUTH 交换对对等体进行身份验证并完成 IKE SA 建立。
```
Initiator                             Responder
---------------------------------------------------------------------
<-- IKE_SA_INIT (additional key exchanges negotiation) -->

<-- {IKE_INTERMEDIATE (additional key exchange)} -->
                         ...
<-- {IKE_INTERMEDIATE (additional key exchange)} -->

<-- {IKE_AUTH} -->
```

### Protocol Details
在最简单的情况下，发起者启动单个密钥交换（并且没有兴趣支持多个密钥交换），并且不关心 IKE_SA_INIT 消息可能的碎片（因为它选择的密钥交换足够小而不会碎片或发起者确信分片将通过 IP 分片或通过 TCP 传输来处理）。

在这种情况下，发起者使用转换类型 4（可能使用后量子算法）并包括发起者 KE 有效负载，执行单个密钥交换的 IKE_SA_INIT。 如果响应者接受该策略，它将使用 IKE_SA_INIT 响应进行响应，并且 IKE 照常继续。

如果发起者想要协商多个密钥交换，则发起者使用下面列出的协议行为。

#### IKE_SA_INIT Round: Negotiation
Seven new transform types("Additional Key Exchange (ADDKE) Transform Types") are defined:
```
0	Reserved		[RFC7296]
1	Encryption Algorithm (ENCR)	(IKE and ESP)	[RFC7296]
2	Pseudo-random Function (PRF)	(IKE)	[RFC7296]
3	Integrity Algorithm (INTEG)	(IKE, AH, optional in ESP)	[RFC7296]
4	Key Exchange Method (KE)	(IKE, optional in AH & ESP)	[RFC7296][RFC9370]
5	Extended Sequence Numbers (ESN)	(AH and ESP)	[RFC7296]
6	Additional Key Exchange 1 (ADDKE1)	(optional in IKE, AH, ESP)	[RFC9370]
7	Additional Key Exchange 2 (ADDKE2)	(optional in IKE, AH, ESP)	[RFC9370]
8	Additional Key Exchange 3 (ADDKE3)	(optional in IKE, AH, ESP)	[RFC9370]
9	Additional Key Exchange 4 (ADDKE4)	(optional in IKE, AH, ESP)	[RFC9370]
10	Additional Key Exchange 5 (ADDKE5)	(optional in IKE, AH, ESP)	[RFC9370]
11	Additional Key Exchange 6 (ADDKE6)	(optional in IKE, AH, ESP)	[RFC9370]
12	Additional Key Exchange 7 (ADDKE7)	(optional in IKE, AH, ESP)	[RFC9370]
```
They are interpreted as an indication of additional key exchange methods that peers agree to perform in a series of IKE_INTERMEDIATE exchanges following the IKE_SA_INIT exchange. The allowed Transform IDs for these transform types are the same as the IDs for Transform Type 4, so they all share a single IANA registry for Transform IDs.  
它们被解释为对等体同意在 IKE_SA_INIT 交换之后的一系列 IKE_INTERMEDIATE 交换中执行的附加密钥交换方法的指示。这些转换类型允许的转换 ID 与转换类型 4 的 ID 相同，因此它们都共享一个 IANA 转换 ID 注册表。  

----
Performed in an order of the values of their Transform Types.This is so that the key exchange negotiated using Additional Key Exchange i always precedes that of Additional Key Exchange i + 1.  
按照其转换类型值的顺序执行。 这样，使用附加密钥交换 i 协商的密钥交换始终先于附加密钥交换 i + 1。   
they all share a single registry for Transform IDs, namely "Transform Type 4 - Key Exchange Method Transform IDs".  
它们都共享一个转换 ID 注册表，即“转换类型 4 - 密钥交换方法转换 ID”。 所有密钥交换算法（经典的或后量子的）都应添加到此注册表中。  
 However, for the ADDKE Transform Types, the responder's choice MUST NOT contain duplicated algorithms (those with an identical Transform ID and attributes), except for the Transform ID of NONE.  
对于 ADDKE 转换类型，响应者的选择不得包含重复的算法（具有相同转换 ID 和属性的算法），转换 ID 为 NONE 的除外。  
If the responder is unable to select algorithms that are not duplicated for each proposed key exchange (either because the proposal contains too few choices or due to the local policy restrictions on using the proposed algorithms), then the responder MUST reject the message with an error notification of type NO_PROPOSAL_CHOSEN.   
如果响应者无法为每个提议的密钥交换选择不重复的算法（因为提议包含的选择太少，或者由于使用提议的算法的本地政策限制），则响应者必须 拒绝带有 NO_PROPOSAL_CHOSEN 类型错误通知的消息。

----
 The initiator's IKE_SA_INIT request message:
 PQ_KEM_*用于表示一些流行的后量子密钥交换方法尚未定义的变换 ID.
 ```
 SA Payload
   |
   +--- Proposal #1 ( Proto ID = IKE(1), SPI Size = 8,
         |            9 transforms,      SPI = 0x35a1d6f22564f89d )
         |
         +-- Transform ENCR ( ID = ENCR_AES_GCM_16 )
         |     +-- Attribute ( Key Length = 256 )
         |
         +-- Transform KE ( ID = 4096-bit MODP Group )
         |
         +-- Transform PRF ( ID = PRF_HMAC_SHA2_256 )
         |
         +-- Transform ADDKE2 ( ID = PQ_KEM_1 )
         |
         +-- Transform ADDKE2 ( ID = PQ_KEM_2 )
         |
         +-- Transform ADDKE3 ( ID = PQ_KEM_1 )
         |
         +-- Transform ADDKE3 ( ID = PQ_KEM_2 )
         |
         +-- Transform ADDKE5 ( ID = PQ_KEM_3 )
         |
         +-- Transform ADDKE5 ( ID = NONE )
 ```
The responder might return the following SA payload:
```
SA Payload
   |
   +--- Proposal #1 ( Proto ID = IKE(1), SPI Size = 8,
         |            6 transforms,      SPI = 0x8df52b331a196e7b )
         |
         +-- Transform ENCR ( ID = ENCR_AES_GCM_16 )
         |     +-- Attribute ( Key Length = 256 )
         |
         +-- Transform KE ( ID = 4096-bit MODP Group )
         |
         +-- Transform PRF ( ID = PRF_HMAC_SHA2_256 )
         |
         +-- Transform ADDKE2 ( ID = PQ_KEM_2 )
         |
         +-- Transform ADDKE3 ( ID = PQ_KEM_1 )
         |
         +-- Transform ADDKE5 ( ID = NONE )
```

----
如果发起方将任何 ADDKE 转换类型包含到 IKE_SA_INIT 交换请求消息中的 SA 有效负载中，则它必须还可以协商 IKE_INTERMEDIATE 交换的使用.
```
Initiator                          Responder
---------------------------------------------------------------------
HDR, SAi1(.. ADDKE*...), KEi, Ni,
N(INTERMEDIATE_EXCHANGE_SUPPORTED)    --->
                                   HDR, SAr1(.. ADDKE*...), KEr, Nr,
                                   [CERTREQ],
                           <---    N(INTERMEDIATE_EXCHANGE_SUPPORTED)

```

#### IKE_INTERMEDIATE Round: Additional Key Exchanges
```
Initiator                          Responder
---------------------------------------------------------------------
HDR, SK {KEi(n)}    -->
                            <--    HDR, SK {KEr(n)}
```
发起者在 KEi(n) 有效负载中发送密钥交换数据。 该消息受当前 SK_ei/SK_ai 密钥的保护。 符号“KEi(n)”表示来自发起方的第n个IKE_INTERMEDIATE KE有效负载；整数“n”从1开始连续。  
收到此消息后，响应方发回密钥交换有效负载 KEr(n)； “KEr(n)”表示来自响应方的第 n 个 IKE_INTERMEDIATE KE 有效负载。 与请求的保护方式类似，此消息受到当前 SK_er/SK_ar 密钥的保护。  

标准的DH交换的密钥材料计算如下：
```
SKEYSEED = prf(Ni | Nr, g^ir)
{SK_d | SK_ai | SK_ar | SK_ei | SK_er | SK_pi | SK_pr}
                   = prf+ (SKEYSEED, Ni | Nr | SPIi | SPIr)
```
增加额外DH交换的密钥材料计算如下：
```
SKEYSEED(n) = prf(SK_d(n-1), SK(n) | Ni | Nr)
```
SK(n) is the resulting shared secret. SK(n) 是最终的共享秘密.
```
{SK_d(n) | SK_ai(n) | SK_ar(n) | SK_ei(n) | SK_er(n) | SK_pi(n) |
 SK_pr(n)} = prf+ (SKEYSEED(n), Ni | Nr | SPIi | SPIr)
```

#### IKE_AUTH Exchange
 This exchange is the standard IKE exchange, as described in [RFC7296], with the modification of AUTH payload calculation described in [RFC9242].

#### CREATE_CHILD_SA Exchange

创建或重新生成子 SA 密钥时，对等方可以选择执行密钥交换以将新的熵添加到会话密钥中。 在 IKE SA 重新生成密钥的情况下，密钥交换是强制性的。 

The IKE_FOLLOWUP_KE exchange is introduced especially for transferring data of additional key exchanges following the one performed in the CREATE_CHILD_SA. Its Exchange Type value is 44.  
引入 IKE_FOLLOWUP_KE 交换是专门为了传输 CREATE_CHILD_SA 中执行的密钥交换之后的附加密钥交换的数据。 其交换类型值为 44。  

-----
After an IKE SA is created, the window size may be greater than one; thus, multiple concurrent exchanges may be in progress. Therefore, it is essential to link the IKE_FOLLOWUP_KE exchanges together with the corresponding CREATE_CHILD_SA exchange.   
多个并发交换可能同时进行，因此，必须将 IKE_FOLLOWUP_KE 交换与相应的 CREATE_CHILD_SA 交换链接在一起。  
A new status type notification called "ADDITIONAL_KEY_EXCHANGE" is introduced for this purpose. Its Notify Message Type value is 16441, and the Protocol ID and SPI Size are both set to 0.  
为此，引入了名为“ADDITIONAL_KEY_EXCHANGE”的新状态类型通知。  
The data associated with this notification is a blob meaningful only to the responder so that the responder can correctly link successive exchanges.  
与此通知关联的数据是仅对响应者有意义的 blob，以便响应者可以正确链接连续的交换。
```
Initiator                             Responder
---------------------------------------------------------------------
HDR(CREATE_CHILD_SA), SK {SA, Ni, KEi} -->
                          <--  HDR(CREATE_CHILD_SA), SK {SA, Nr, KEr,
                                   N(ADDITIONAL_KEY_EXCHANGE)(link1)}

HDR(IKE_FOLLOWUP_KE), SK {KEi(1),
 N(ADDITIONAL_KEY_EXCHANGE)(link1)} -->
                               <--  HDR(IKE_FOLLOWUP_KE), SK {KEr(1),
                                   N(ADDITIONAL_KEY_EXCHANGE)(link2)}

HDR(IKE_FOLLOWUP_KE), SK {KEi(2),
 N(ADDITIONAL_KEY_EXCHANGE)(link2)} -->
                               <--  HDR(IKE_FOLLOWUP_KE), SK {KEr(2),
                                   N(ADDITIONAL_KEY_EXCHANGE)(link3)}

HDR(IKE_FOLLOWUP_KE), SK {KEi(3),
 N(ADDITIONAL_KEY_EXCHANGE)(link3)} -->
                               <--  HDR(IKE_FOLLOWUP_KE), SK {KEr(3)}
```

-----
由于某些意外事件（例如重新启动），发起方可能会丢失其状态，忘记它正在执行其他密钥交换，并且永远不会启动剩余的 IKE_FOLLOWUP_KE 交换。 如果响应方在一段合理的时间后没有收到下一个预期的 IKE_FOLLOWUP_KE 请求，则必须优雅地处理这种情况并删除关联的状态。 大多数情况下5-20秒的等待时间是比较合适的。

----
```
In the case of an IKE SA rekey:
    SKEYSEED = prf(SK_d, SK(0) | Ni | Nr | SK(1) | ... SK(n))

In the case of a Child SA creation or rekey:
    KEYMAT = prf+ (SK_d, SK(0) | Ni | Nr | SK(1) |  ... SK(n))
```
In both cases, SK_d is from the existing IKE SA; SK(0), Ni, and Nr are the shared key and nonces from the CREATE_CHILD_SA, respectively; SK(1)...SK(n) are the shared keys from additional key exchanges.

## Sample Multiple Key Exchanges
### IKE_INTERMEDIATE Exchanges Carrying Additional Key Exchange Payloads
发起者提出三组附加密钥交换，响应方选择执行两次额外的密钥交换。
```
Initiator                     Responder
---------------------------------------------------------------------
HDR(IKE_SA_INIT), SAi1(.. ADDKE*...), --->
KEi(Curve25519), Ni, N(IKEV2_FRAG_SUPPORTED),
N(INTERMEDIATE_EXCHANGE_SUPPORTED)
    Proposal #1
    Transform ECR (ID = ENCR_AES_GCM_16,
                    256-bit key)
    Transform PRF (ID = PRF_HMAC_SHA2_512)
    Transform KE (ID = Curve25519)
    Transform ADDKE1 (ID = PQ_KEM_1)
    Transform ADDKE1 (ID = PQ_KEM_2)
    Transform ADDKE1 (ID = NONE)
    Transform ADDKE2 (ID = PQ_KEM_3)
    Transform ADDKE2 (ID = PQ_KEM_4)
    Transform ADDKE2 (ID = NONE)
    Transform ADDKE3 (ID = PQ_KEM_5)
    Transform ADDKE3 (ID = PQ_KEM_6)
    Transform ADDKE3 (ID = NONE)
                   <--- HDR(IKE_SA_INIT), SAr1(.. ADDKE*...),
                        KEr(Curve25519), Nr, N(IKEV2_FRAG_SUPPORTED),
                        N(INTERMEDIATE_EXCHANGE_SUPPORTED)
                        Proposal #1
                          Transform ECR (ID = ENCR_AES_GCM_16,
                                         256-bit key)
                          Transform PRF (ID = PRF_HMAC_SHA2_512)
                          Transform KE (ID = Curve25519)
                          Transform ADDKE1 (ID = PQ_KEM_2)
                          Transform ADDKE2 (ID = NONE)
                          Transform ADDKE3 (ID = PQ_KEM_5)

HDR(IKE_INTERMEDIATE), SK {KEi(1)(PQ_KEM_2)} -->
                   <--- HDR(IKE_INTERMEDIATE), SK {KEr(1)(PQ_KEM_2)}
HDR(IKE_INTERMEDIATE), SK {KEi(2)(PQ_KEM_5)} -->
                   <--- HDR(IKE_INTERMEDIATE), SK {KEr(2)(PQ_KEM_5)}

HDR(IKE_AUTH), SK{ IDi, AUTH, SAi2, TSi, TSr } --->
                      <--- HDR(IKE_AUTH), SK{ IDr, AUTH, SAr2,
                           TSi, TSr }
```
```
IKE_SA_INIT:SK_d、SK_a[i/r] 和 SK_e[i/r]  
IKE_INTERMEDIATE1:
    SKEYSEED(1) = prf(SK_d, SK(1) | Ni | Nr)  
    {SK_d(1) | SK_ai(1) | SK_ar(1) | SK_ei(1) | SK_er(1) | SK_pi(1) | SK_pr(1)} = prf+ (SKEYSEED(1), Ni | Nr | SPIi | SPIr)
    calc IntAuth_i1 and IntAuth_r1 by SK_pi(1) and SK_pr(1)
IKE_INTERMEDIATE2:
    SKEYSEED(2) = prf(SK_d(1), SK(2) | Ni | Nr)
    {SK_d(2) | SK_ai(2) | SK_ar(2) | SK_ei(2) | SK_er(2) | SK_pi(2) | SK_pr(2)} = prf+ (SKEYSEED(2), Ni | Nr | SPIi | SPIr)
    calc IntAuth_[i/r]2 by SK_p[i/r](2)
IKE_AUTH:
    calc IntAuth by IntAuth_[i/r]2
    calc InitiatorSignedOctets and ResponderSignedOctets by IntAuth
```
### Additional Key Exchange in the CREATE_CHILD_SA Exchange Only
发起者不建议使用额外的密钥交换来建立 IKE SA,响应方在其 IKE_SA_INIT 响应消息中包含 CHILDESS_IKEV2_SUPPORTED 通知。 发起方理解并支持此通知，与响应方交换修改后的 IKE_AUTH 消息，并立即通过其他密钥交换重新生成 IKE SA 密钥。 任何子 SA 都必须通过后续的 CREATED_CHILD_SA 交换来创建。  
```
Initiator                     Responder
---------------------------------------------------------------------
HDR(IKE_SA_INIT), SAi1, --->
KEi(Curve25519), Ni, N(IKEV2_FRAG_SUPPORTED)
                   <--- HDR(IKE_SA_INIT), SAr1,
                        KEr(Curve25519), Nr, N(IKEV2_FRAG_SUPPORTED),
                        N(CHILDLESS_IKEV2_SUPPORTED)

HDR(IKE_AUTH), SK{ IDi, AUTH  } --->
                   <--- HDR(IKE_AUTH), SK{ IDr, AUTH }

HDR(CREATE_CHILD_SA),
      SK{ SAi(.. ADDKE*...), Ni, KEi(Curve25519) } --->
  Proposal #1
    Transform ECR (ID = ENCR_AES_GCM_16,
                   256-bit key)
    Transform PRF (ID = PRF_HMAC_SHA2_512)
    Transform KE (ID = Curve25519)
    Transform ADDKE1 (ID = PQ_KEM_1)
    Transform ADDKE1 (ID = PQ_KEM_2)
    Transform ADDKE2 (ID = PQ_KEM_5)
    Transform ADDKE2 (ID = PQ_KEM_6)
    Transform ADDKE2 (ID = NONE)
                   <--- HDR(CREATE_CHILD_SA), SK{ SAr(.. ADDKE*...),
                        Nr, KEr(Curve25519),
                        N(ADDITIONAL_KEY_EXCHANGE)(link1) }
                          Proposal #1
                            Transform ECR (ID = ENCR_AES_GCM_16,
                                           256-bit key)
                            Transform PRF (ID = PRF_HMAC_SHA2_512)
                            Transform KE (ID = Curve25519)
                            Transform ADDKE1 (ID = PQ_KEM_2)
                            Transform ADDKE2 (ID = PQ_KEM_5)

HDR(IKE_FOLLOWUP_KE), SK{ KEi(1)(PQ_KEM_2), --->
N(ADDITIONAL_KEY_EXCHANGE)(link1) }
                  <--- HDR(IKE_FOLLOWUP_KE), SK{ KEr(1)(PQ_KEM_2),
                        N(ADDITIONAL_KEY_EXCHANGE)(link2) }

HDR(IKE_FOLLOWUP_KE), SK{ KEi(2)(PQ_KEM_5), --->
N(ADDITIONAL_KEY_EXCHANGE)(link2) }
                  <--- HDR(IKE_FOLLOWUP_KE), SK{ KEr(2)(PQ_KEM_5) }
```
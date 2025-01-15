---
title: IKEv2-rfc8784
description: Mixing Preshared Keys in the Internet Key Exchange Protocol Version 2 (IKEv2) for Post-quantum Security学习.
author: dead4ead
date: 2024-11-25 12:00:00 +0800
categories: [Secure, IPSec]
tags: [IKEv2, IPSec]
#pin: true
math: true
mermaid: true
---

Mixing Preshared Keys in the Internet Key Exchange Protocol Version 2 (IKEv2) for Post-quantum Security

## Introduction
 describes an extension of IKEv2 to allow it to be resistant to a quantum computer by using preshared keys.  
 描述了 IKEv2 的扩展，使其能够通过使用预共享密钥来抵抗量子计算机。

 If the preshared key has sufficient entropy and the Pseudorandom Function (PRF), encryption, and authentication transforms are quantum secure, then the resulting system is believed to be quantum secure -- that is, secure against classical attackers of today or future attackers with a quantum computer.  
 如果预共享密钥具有足够的熵，并且伪随机函数 (PRF)、加密和身份验证转换是量子安全的，则所得到的系统被认为是量子安全的——也就是说，可以安全地抵御当今的经典攻击者或未来的攻击者量子计算机。  

IKEv1 [RFC2409], when used with strong preshared keys, is not vulnerable to quantum attacks because those keys are one of the inputs to the key derivation function.  
IKEv1 [RFC2409] 在与强预共享密钥一起使用时不易受到量子攻击，因为这些密钥是密钥派生函数的输入之一。 

`IKEv1: SKEYID = prf(pre-shared-key, Ni_b | Nr_b)`  
`IKEv2: SKEYSEED = prf(Ni | Nr, g^ir) ` 

IKEv2的预共享密钥不用于密钥派生，用于身份验证。

This document describes a way to extend IKEv2 to have a similar property; assuming that the two end systems share a long secret key, then the resulting exchange is quantum secure.  
本文档描述了一种扩展 IKEv2 以具有类似属性的方法；假设两端系统共享一个长密钥，则最终的交换是量子安全的。

----
The general idea is that we add an additional secret that is shared between the initiator and the responder; this secret is in addition to the authentication method that is already provided within IKEv2. We stir this secret into the SK_d value, which is used to generate the key material (KEYMAT) for the Child Security Associations (SAs) and the SKEYSEED for the IKE SAs created as a result of the initial IKE SA rekey. This secret provides quantum resistance to the IPsec SAs and any subsequent IKE SAs. We also stir the secret into the SK_pi and SK_pr values; this allows both sides to detect a secret mismatch cleanly.  
总体思路是，我们添加一个在发起者和响应者之间共享的额外秘密；此秘密是对 IKEv2 中已提供的身份验证方法的补充。 我们将此秘密添加到 SK_d 值中，该值用于生成子安全关联 (SA) 的密钥材料 (KEYMAT) 以及由于初始 IKE SA 重新生成密钥而创建的 IKE SA 的 SKEYSEED。 此秘密为 IPsec SA 和任何后续 IKE SA 提供量子抵抗。 我们还将秘密搅入 SK_pi 和 SK_pr 值中；这允许双方干净地检测秘密不匹配。  

## Assumptions
We assume that each IKE peer has a list of Post-quantum Preshared Keys (PPKs) along with their identifiers (PPK_ID), and any potential IKE initiator selects which PPK to use with any specific responder.  
我们假设每个 IKE 对等方都有一个后量子预共享密钥 (PPK) 及其标识符 (PPK_ID) 列表，并且任何潜在的 IKE 发起方都会选择与任何特定响应方一起使用哪个 PPK。 此外，实现还有一个可配置标志，用于确定此 PPK 是否是强制的。  
假设每个节点上的 PPK 特定配置由以下元组组成：  
      ` Peer, PPK, PPK_ID, mandatory_or_not `  

## Exchanges

### IKE_SA_INIT
The initiator is configured to use a PPK with the responder (whether or not the use of the PPK is mandatory), then it MUST include a notification USE_PPK in the IKE_SA_INIT request message as follows:
```
Initiator                       Responder
------------------------------------------------------------------
HDR, SAi1, KEi, Ni, N(USE_PPK)  --->
```
发起方IKE_SA_INIT消息中包含USE_PPK通知载荷：**type=16435,无通知数据**.

The responder replies with the IKE_SA_INIT message, including a USE_PPK notification in the response:
```
Initiator                       Responder
------------------------------------------------------------------
                <--- HDR, SAr1, KEr, Nr, [CERTREQ,] N(USE_PPK)
```
响应方回应确认使用PPK.

----

双方基于PPK计算密钥：
```
 SKEYSEED = prf(Ni | Nr, g^ir)
 {SK_d' | SK_ai | SK_ar | SK_ei | SK_er | SK_pi' | SK_pr'}
                 = prf+ (SKEYSEED, Ni | Nr | SPIi | SPIr)

 SK_d  = prf+ (PPK, SK_d')
 SK_pi = prf+ (PPK, SK_pi')
 SK_pr = prf+ (PPK, SK_pr')
```
Note that the PPK is used in SK_d, SK_pi, and SK_pr calculations only during the initial IKE SA setup.  
请注意，PPK 仅在初始 IKE SA 设置期间用于 SK_d、SK_pi 和 SK_pr 计算,不用于IKE SA rekey, resumption等。  

### IKE_AUTH
The initiator then sends the IKE_AUTH request message, including the PPK_ID value as follows:  
```
Initiator                       Responder
------------------------------------------------------------------
HDR, SK {IDi, [CERT,] [CERTREQ,]
    [IDr,] AUTH, SAi2,
    TSi, TSr, N(PPK_IDENTITY, PPK_ID), [N(NO_PPK_AUTH)]}  --->
```
发起方IKE_AUTH消息中携带PPK_IDENTITY通知载荷:**type=16436,通知数据为PPK_ID**  
发送方在发送IKE_AUTH报文的时候才携带PPK_ID，此时还无法确认响应端是否支持该PPK_ID，因此发送端在发送IKE_AUTH报文的时候除了携带按PPK计算后的AUTH外，还会使用NO_PPK_AUTH通知载荷携带正常协商时计算出的AUTH。  
NO_PPK_AUTH:**type=16437,通知数据为no ppk时计算出的AUTH**  
IKE SA的加密SK_e和认证SK_a密钥计算不受PPK影响。

The responder then continues with the IKE_AUTH exchange (validating the AUTH payload that the initiator included) as usual and sends back a response, which includes the PPK_IDENTITY notification with no data to indicate that the PPK is used in the exchange:  
响应方像往常一样继续进行 IKE_AUTH 交换（验证发起方包含的 AUTH 有效负载），并发回响应，其中包括 PPK_IDENTITY 通知，但没有任何数据，以表明交换中使用了 PPK：
```
Initiator                       Responder
------------------------------------------------------------------
                           <--  HDR, SK {IDr, [CERT,]
                                AUTH, SAr2,
                                TSi, TSr, N(PPK_IDENTITY)}
```
## PPK
### PPK_ID Format
Both the initiator and the responder have a secret PPK value, with the responder selecting the PPK based on the PPK_ID that the initiator sends.  
发起方和响应方都有一个秘密 PPK 值，响应方根据发起方发送的 PPK_ID 选择 PPK。  
预计后续规范将扩展此技术以允许动态更改 PPK 值。 为了促进这样的扩展，我们指定发起者发送的 PPK_ID 将其第一个八位字节作为 PPK_ID 类型值。
- PPK_ID_OPAQUE (1), 本文档未指定 PPK_ID（以及 PPK 本身）的格式；假设发起者和响应者都可以相互理解。
- PPK_ID_FIXED (2), 在这种情况下，PPK_ID 和 PPK 的格式是固定的八位字节字符串； PPK_ID 的其余字节是配置值。 我们假设 PPK_ID 和 PPK 之间存在固定映射，该映射是在发起方和响应方本地配置的。 响应者可以使用PPK_ID来查找相应的PPK值。 并非所有实现都能够配置任意八位字节字符串；为了提高潜在的互操作性，建议在 PPK_ID_FIXED 情况下，将 PPK 和 PPK_ID 字符串限制为 base64 字符集 [RFC4648]。
  

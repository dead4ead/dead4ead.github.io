---
title: IKEv2-rfc9242
description: Intermediate Exchange in the Internet Key Exchange Protocol Version 2 (IKEv2)学习.
author: dead4ead
date: 2024-12-13 12:00:00 +0800
categories: [Secure, IPSec]
tags: [IKEv2, IPSec]
#pin: true
math: true
mermaid: true
---

Intermediate Exchange in the Internet Key Exchange Protocol Version 2 (IKEv2)  

## Introduction
One example of the need to transfer large amounts of data before an IKE SA is created is using the QC-resistant key exchange methods in IKEv2. Recent progress in quantum computing has led to concern that classical Diffie-Hellman key exchange methods will become insecure in the relatively near future and should be replaced with QC-resistant ones. Currently, most QC-resistant key exchange methods have large public keys.  
在创建 IKE SA 之前需要传输大量数据的一个示例是使用 IKEv2 中的抗 QC 密钥交换方法。 量子计算的最新进展引发了人们的担忧，即经典的 Diffie-Hellman 密钥交换方法在不久的将来将变得不安全，应该被抗 QC 的方法所取代。 目前，大多数抗QC密钥交换方法都具有较大的公钥。  
This specification describes a way to transfer a large amount of data in IKEv2 using UDP transport. For this purpose, the document defines a new exchange for IKEv2 called "Intermediate Exchange" or "IKE_INTERMEDIATE". One or more of these exchanges may take place right after the IKE_SA_INIT exchange and prior to the IKE_AUTH exchange.   
本规范描述了一种使用 UDP 传输在 IKEv2 中传输大量数据的方法。 为此，该文档为 IKEv2 定义了一种新的交换，称为“中间交换”或“IKE_INTERMEDIATE”。 这些交换中的一项或多项可能发生在 IKE_SA_INIT 交换之后、IKE_AUTH 交换之前。  
It is expected that the IKE_INTERMEDIATE exchange will only be used for transferring data that is needed to establish IKE SA and not for data that can be sent later when this SA is established.  
预计 IKE_INTERMEDIATE 交换将仅用于传输建立 IKE SA 所需的数据，而不用于传输此 SA 建立后可以稍后发送的数据。  

## Intermediate Exchange Details
### Support for Intermediate Exchange Negotiation
IKE_SA_INTI交互确认双方是否支持中间交换协商
```
Initiator                                 Responder
-----------                               -----------
HDR, SAi1, KEi, Ni,
[N(INTERMEDIATE_EXCHANGE_SUPPORTED)] -->
                                   <-- HDR, SAr1, KEr, Nr, [CERTREQ],
                                 [N(INTERMEDIATE_EXCHANGE_SUPPORTED)]
```
The INTERMEDIATE_EXCHANGE_SUPPORTED is a Status Type IKEv2 notification with Notify Message Type 16438. When it is sent, the Protocol ID and SPI Size fields in the Notify payload are both set to 0.  
### Using Intermediate Exchange
```
Initiator                                 Responder
-----------                               -----------
HDR, ..., SK {...}  -->
                                     <--  HDR, ..., SK {...}
```
The Intermediate Exchange is denoted as IKE_INTERMEDIATE; its Exchange Type is 43.  


### The IKE_INTERMEDIATE Exchange Protection and Authentication
保护和身份验证
#### Protection of IKE_INTERMEDIATE Messages
The keys SK_e[i/r] and SK_a[i/r] for the protection of IKE_INTERMEDIATE exchanges are computed in the standard fashion.  
用于保护 IKE_INTERMEDIATE 交换的密钥 SK_e[i/r] 和 SK_a[i/r] 以标准方式计算.  
Every subsequent IKE_INTERMEDIATE exchange uses the most recently calculated IKE SA keys before this exchange is started.   
每个后续 IKE_INTERMEDIATE 交换都使用此交换开始之前最近计算的 IKE SA 密钥。  
Once all the IKE_INTERMEDIATE exchanges are completed, the most recently calculated SK_e[i/r] and SK_a[i/r] keys are used for protection of the IKE_AUTH exchange and all subsequent exchanges.  
一旦所有 IKE_INTERMEDIATE 交换完成，最近计算的 SK_e[i/r] 和 SK_a[i/r] 密钥将用于保护 IKE_AUTH 交换和所有后续交换。  

#### Authentication of IKE_INTERMEDIATE Exchanges
The IKE_INTERMEDIATE messages must be authenticated in the IKE_AUTH exchange, which is performed by adding their content into the AUTH payload calculation.  
IKE_INTERMEDIATE 消息必须在 IKE_AUTH 交换中进行身份验证，这是通过将其内容添加到 AUTH 有效负载计算中来执行的。  
The requirement to support this behavior makes authentication challenging: it is not appropriate to add on-the-wire content of the IKE_INTERMEDIATE messages into the AUTH payload calculation, because implementations are generally unaware of which form these messages are received by peers.   
由于KE_INTERMEDIATE 消息可能会被分片，支持此行为的要求使身份验证具有挑战性：将 IKE_INTERMEDIATE 消息的在线内容添加到 AUTH 有效负载计算中是不合适的，因为实现通常不知道对等点接收这些消息的形式。  

认证方式做如下修改适配：
```
InitiatorSignedOctets = RealMsg1 | NonceRData | MACedIDForI | IntAuth
ResponderSignedOctets = RealMsg2 | NonceIData | MACedIDForR | IntAuth

IntAuth =  IntAuth_iN | IntAuth_rN | IKE_AUTH_MID

IntAuth_i1 = prf(SK_pi1,              IntAuth_i1A [| IntAuth_i1P])
IntAuth_i2 = prf(SK_pi2, IntAuth_i1 | IntAuth_i2A [| IntAuth_i2P])
IntAuth_i3 = prf(SK_pi3, IntAuth_i2 | IntAuth_i3A [| IntAuth_i3P])
...
IntAuth_iN = prf(SK_piN, IntAuth_iN-1 | IntAuth_iNA [| IntAuth_iNP])

IntAuth_r1 = prf(SK_pr1,              IntAuth_r1A [| IntAuth_r1P])
IntAuth_r2 = prf(SK_pr2, IntAuth_r1 | IntAuth_r2A [| IntAuth_r2P])
IntAuth_r3 = prf(SK_pr3, IntAuth_r2 | IntAuth_r3A [| IntAuth_r3P])
...
IntAuth_rN = prf(SK_prN, IntAuth_rN-1 | IntAuth_rNA [| IntAuth_rNP])
```
 IntAuth由三部分组成：IntAuth_iN、IntAuth_rN、IKE_AUTH_MID。   
IKE_AUTH_MID 块是来自第一轮 IKE_AUTH 交换的 IKE 标头的消息 ID 字段的值.  
The IntAuth_iN and IntAuth_rN chunks represent the cumulative result of applying the negotiated Pseudorandom Function (PRF) to all IKE_INTERMEDIATE exchange messages sent during IKE SA establishment by the initiator and the responder, respectively.  
IntAuth_iN 和 IntAuth_rN 块分别表示将协商的伪随机函数 (PRF) 应用于发起方和响应方在 IKE SA 建立期间发送的所有 IKE_INTERMEDIATE 交换消息的累积结果。  
IntAuth_[i/r]*A 块由从 IKE 标头的第一个八位位组（不包括前置的四个零八位位组，如果使用 ESP 数据包的 UDP 封装或 TCP 封装）到最后一个八位位组的八位位组序列组成加密有效负载的通用标头。  
IntAuth_[i/r]*P 块是明文形式的加密负载的内部负载，通过在名称中使用“P”后缀来强调。EEcr Pld指的是End Entity Certificate Request 载荷，也即IKE_SA_INIT消息对响应方回应的[CERTREQ]:证书请求载荷。
```
                     1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ ^ ^
|                       IKE SA Initiator's SPI                  | | |
|                                                               | | |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ I |
|                       IKE SA Responder's SPI                  | K |
|                                                               | E |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+   |
|  Next Payload | MjVer | MnVer | Exchange Type |     Flags     | H |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ d |
|                          Message ID                           | r A
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ | |
|                       Adjusted Length                         | | |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ v |
|                                                               |   |
~                 Unencrypted payloads (if any)                 ~   |
|                                                               |   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ ^ |
| Next Payload  |C|  RESERVED   |    Adjusted Payload Length    | | |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ | v
|                                                               | |
~                     Initialization Vector                     ~ E
|                                                               | E
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ c ^
|                                                               | r |
~             Inner payloads (not yet encrypted)                ~   P
|                                                               | P |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ l v
|              Padding (0-255 octets)           |  Pad Length   | d
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ |
|                                                               | |
~                    Integrity Checksum Data                    ~ |
|                                                               | |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ v
```
The prf calculations MUST be applied to whole messages only, before possible IKE fragmentation. This ensures that the IntAuth will be the same regardless of whether or not IKE fragmentation takes place. If the message was received in fragmented form, it MUST be reconstructed before calculating the prf as if it were received unfragmented.  
在可能的 IKE 分段之前，prf 计算必须仅应用于整个消息。 这确保了无论 IKE 分段是否发生，IntAuth 都是相同的。 如果消息以分段形式接收，则在计算 prf 之前必须对其进行重构，就好像它是未分段接收的一样。  

## Security Considerations
It is recommended that these specifications be defined in such a way that the responder would know (possibly via negotiation with the initiator) the exact number of these exchanges that need to take place.  
建议响应方知道（可能通过与发起方协商）需要发生的这些交换的确切数量.  
After the IKE_SA_INIT exchange is complete, it is preferred that both the initiator and the responder know the exact number of IKE_INTERMEDIATE exchanges they have to perform; it is possible that some IKE_INTERMEDIATE exchanges are optional and are performed at the initiator's discretion, but if a specification defines optional use of IKE_INTERMEDIATE, then the maximum number of these exchanges must be hard capped by the corresponding specification.  
在 IKE_SA_INIT 交换完成后，发起方和响应方最好都知道他们必须执行的 IKE_INTERMEDIATE 交换的确切数量；某些 IKE_INTERMEDIATE 交换可能是可选的，并且由发起者自行决定执行，但如果规范定义了 IKE_INTERMEDIATE 的可选使用，则这些交换的最大数量必须受到相应规范的硬性限制。

## Example of IKE_INTERMEDIATE Exchange
In this example, there is one IKE_SA_INIT exchange and two IKE_INTERMEDIATE exchanges, followed by the IKE_AUTH exchange to authenticate all initial exchanges.  
在此示例中，有一个 IKE_SA_INIT 交换和两个 IKE_INTERMEDIATE 交换，后跟 IKE_AUTH 交换以验证所有初始交换。  
HDR(xxx,MID=yyy)中的xxx表示交换类型，yyy表示用于该交换的消息ID。  

### 1. IKE_SA_INIT
```
Initiator                         Responder
-----------                       -----------
HDR(IKE_SA_INIT,MID=0),
SAi1, KEi, Ni,
N(INTERMEDIATE_EXCHANGE_SUPPORTED)  -->

                             <--  HDR(IKE_SA_INIT,MID=0),
                                  SAr1, KEr, Nr, [CERTREQ],
                                  N(INTERMEDIATE_EXCHANGE_SUPPORTED)
```
对等方计算 SK_* 并将其存储为 SK_*1。 SK_e[i/r]1 和 SK_a[i/r]1 将用于保护第一个 IKE_INTERMEDIATE 交换，SK_p[i/r]1 将用于其身份验证.

### 2. First IKE_INTERMEDIATE
```
Initiator                         Responder
-----------                       -----------
HDR(IKE_INTERMEDIATE,MID=1),
SK(SK_ei1,SK_ai1) {...}  -->

         <Calculate IntAuth_i1 = prf(SK_pi1, ...)>

                             <--  HDR(IKE_INTERMEDIATE,MID=1),
                                  SK(SK_er1,SK_ar1) {...}

         <Calculate IntAuth_r1 = prf(SK_pr1, ...)>
```
如果在完成此 IKE_INTERMEDIATE 交换后更新了 SK_*1 密钥（例如，作为新密钥交换的结果），则对等方将更新的密钥存储为 SK_*2；否则，它们使用 SK_*1 作为 SK_*2。 SK_e[i/r]2 和 SK_a[i/r]2 将用于保护第二个 IKE_INTERMEDIATE 交换，SK_p[i/r]2 将用于其身份验证。

### 3. Second IKE_INTERMEDIATE
```
Initiator                         Responder
-----------                       -----------
HDR(IKE_INTERMEDIATE,MID=2),
SK(SK_ei2,SK_ai2) {...}  -->

         <Calculate IntAuth_i2 = prf(SK_pi2, ...)>

                             <--  HDR(IKE_INTERMEDIATE,MID=2),
                                  SK(SK_er2,SK_ar2) {...}

         <Calculate IntAuth_r2 = prf(SK_pr2, ...)>
```
如果在完成第二次 IKE_INTERMEDIATE 交换后更新了 SK_*2 密钥（例如，作为新密钥交换的结果），则对等方将更新的密钥存储为 SK_*3；否则，他们使用 SK_*2 作为 SK_*3。 SK_e[i/r]3 和 SK_a[i/r]3 将用于保护 IKE_AUTH 交换，SK_p[i/r]3 将用于身份验证，SK_d3 将用于派生其他密钥（例如，对于子 SA）。

### 4. IKE_AUTH
```
Initiator                         Responder
-----------                       -----------
HDR(IKE_AUTH,MID=3),
SK(SK_ei3,SK_ai3)
{IDi, [CERT,] [CERTREQ,]
[IDr,] AUTH, SAi2, TSi, TSr}  -->
                             <--  HDR(IKE_AUTH,MID=3),
                                  SK(SK_er3,SK_ar3)
                                  {IDr, [CERT,] AUTH, SAr2, TSi, TSr}
```
在此示例中，发生了两次 IKE_INTERMEDIATE 交换；因此，SK_*3 密钥将用作 SK_* 密钥，以便在创建的 IKE SA 上下文中进行进一步的加密操作.
---
title: The Internet Key Exchange Protocol Version 2 (IKEv2)
description: IKEv2-rfc7296
author: dead4ead
date: 2024-11-20 12:00:00 +0800
categories: [Secure, IPSec]
tags: [IKEv2, IPSec]
#pin: true
math: true
mermaid: true
---

## 注意
DH猜想被拒绝时，发起方需要提供全套可接受的加密套件，因为拒绝消息未经身份验证，否则，攻击者可能会诱使发起端协商一个较弱的套件，而不是双方都喜欢的较强的套件。  

## 交互流程
The first two exchanges of messages establishing an IKE SA are called the IKE_SA_INIT exchange and the IKE_AUTH exchange; subsequent IKE exchanges are called either CREATE_CHILD_SA exchanges or INFORMATIONAL exchanges.   
- IKE_SA_INIT  
- IKE_AUTH  创建IKE SA和第一对IPSec SA
- CREATE_CHILD_SA 后续创建IPSec SA

In all cases, all IKE_SA_INIT exchanges MUST complete before any other exchange type, then all IKE_AUTH exchanges MUST complete, and following that, any number of CREATE_CHILD_SA and INFORMATIONAL exchanges may occur in any order.  
IKE_SA_INIT 先进行，再进行IKE_AUTH，最后可以进行任意数量的CREATE_CHILD_SA。  
- The first exchange of an IKE session, IKE_SA_INIT, negotiates security parameters for the IKE SA, sends nonces, and sends Diffie-Hellman values.第一对消息为IKE SA协商安全参数：nonce、DH等。
- The second exchange, IKE_AUTH, transmits identities, proves knowledge of the secrets corresponding to the two identities, and sets up an SA for the first (and often only) AH or ESP Child SA (unless there is failure setting up the AH or ESP Child SA, in which case the IKE SA is still established without the Child SA).  第二对消息传输身份信息，认证双方身份；同时生成第一对IPSec SA。
- The types of subsequent exchanges are CREATE_CHILD_SA (which creates a Child SA) and INFORMATIONAL (which deletes an SA, reports error conditions, or does other housekeeping).  后续CREATE_CHILD_SA消息生成后续IPSec SA。 

### The Initial Exchanges
IKE_SA_INIT & IKE_AUTH

    Initiator                         Responder
       -------------------------------------------------------- 
       HDR, SAi1, KEi, Ni       -->
                                <-- HDR, SAr1, KEr, Nr, [CERTREQ]
       HDR, SK {IDi, [CERT,] 
          [CERTREQ,] [IDr,] AUTH, 
    	  SAi2, TSi, TSr} -->
                                <-- HDR, SK {IDr, [CERT,] AUTH, 
        							SAr2, TSi, TSr}

SA_INIT交换加密套件、KE、随机值。  
AUTH中IDi声明自己的身份、可选的IDr指定与响应方哪个身份通信。


### The CREATE_CHILD_SA Exchange
- Either endpoint may initiate a CREATE_CHILD_SA exchange.任一端点都能发起 CREATE_CHILD_SA交换。
- The keying material for the Child SA is a function of SK_d established during the establishment of the IKE SA, the nonces exchanged during the CREATE_CHILD_SA exchange, and the Diffie-Hellman value (if KE payloads are included in the CREATE_CHILD_SA exchange). 子SA密钥材料为SK_d和CREATE_CHILD_SA交换的nonce和DH。  
- The responder sends a NO_ADDITIONAL_SAS notification to indicate that a CREATE_CHILD_SA request is unacceptable because the responder is unwilling to accept any more Child SAs on this IKE SA. This notification can also be used to reject IKE SA rekey. Some minimal implementations may only accept a single Child SA setup in the context of an initial IKE exchange and reject any subsequent attempts to add more.响应者可以发送NO_ADDITIONAL_SAS通知，指定不接受创建子SA请求，一种实现是可以只接受Initial Exchanges上下文的首个子SA。

#### Creating New Child SAs with the CREATE_CHILD_SA Exchange
CREATE_CHILD_SA  

    Initiator                         Responder  
    -------------------------------------------------------------  
        HDR, SK {SA, Ni, [KEi,]
               TSi, TSr}            -->  
        		                    <-- HDR, SK {SA, Nr, [KEr,] 
    	    						    TSi, TSr}  

- USE_TRANSPORT_MODE notification MAY be included in a request message that also includes an SA payload requesting a Child SA. It requests that the Child SA use transport mode rather than tunnel mode for the SA created.传输模式随SA载荷通知，如果响应方接受则回应notification of USE_TRANSPORT_MODE。  

#### Rekeying IKE SAs with the CREATE_CHILD_SA Exchange

    Initiator                         Responder
    ------------------------------------------------------
    HDR, SK {SA, Ni, KEi}     -->
                              <--  HDR, SK {SA, Nr, KEr}  

- A new initiator SPI is supplied in the SPI field of the SA payload.IKE SA重协商，使用新的SPI。  

#### Rekeying Child SAs with the CREATE_CHILD_SA Exchange

    Initiator                         Responder
    ------------------------------------------------------
    HDR, SK {N(REKEY_SA), 
             SA, Ni, [KEi,]
               TSi, TSr}      -->
                              <--  HDR, SK {SA, Nr, [KEr,] 
    						        TSi, TSr}  

- The REKEY_SA notification MUST be included in a CREATE_CHILD_SA exchange if the purpose of the exchange is to replace an existing ESP or AH SA. 如果交换的目的是替换现有ESP或AH SA，则必须在创建子SA交换中包含更新SA通知。  

### The INFORMATIONAL Exchange

    Initiator                         Responder  
    ------------------------------------------------------------  
    HDR, SK {[N,] [D,]
        [CP,] ...}  -->  
                                 <--  HDR, SK {[N,] [D,]
                                          [CP,] ...}  

- INFORMATIONAL exchanges MUST ONLY occur after the initial exchanges and are cryptographically protected with the negotiated keys.通知消息在初始交换之后，是被加密的。  
- Messages in an INFORMATIONAL exchange contain zero or more Notification, Delete, and Configuration payloads. 通知、删除和配置负载。  

#### Deleting an SA with INFORMATIONAL Exchanges
- ESP and AH SAs always exist in pairs, with one SA in each direction. When an SA is closed, both members of the pair MUST be closed (that is, deleted).ESP和AH一起删除。
- Normally, the response in the INFORMATIONAL exchange will contain Delete payloads for the paired SAs going in the other direction. 响应方通知删除另一个方向的SA。
- Deleting an IKE SA implicitly closes any remaining Child SAs negotiated under it. The response to a request that deletes the IKE SA is an empty INFORMATIONAL response.删除IKE SA会隐式删除子SA。
- Half-closed ESP or AH connections are anomalous, and a node with auditing capability should probably audit their existence if they persist.半关闭状态的链接是不正常的。  

## Calc Keying Material
### Generating Keying Material for the IKE SA
IKE_SA_INIT
```
SKEYSEED = prf(Ni | Nr, g^ir)
{SK_d | SK_ai | SK_ar | SK_ei | SK_er | SK_pi | SK_pr}
                   = prf+ (SKEYSEED, Ni | Nr | SPIi | SPIr)
```   
- SK_d used for deriving new keys for the Child SAs established with this IKE SA;    
- SK_ai and SK_ar used as a key to the integrity protection algorithm for authenticating the component messages of subsequent exchanges;    
- SK_ei and SK_er used for encrypting (and of course decrypting) all subsequent exchanges;   
- SK_pi and SK_pr, which are used when generating an AUTH payload.  

### Generating Keying Material for Child SAs
A single Child SA is created by the IKE_AUTH exchange, and additional Child SAs can optionally be created in CREATE_CHILD_SA exchanges. 
```
KEYMAT = prf+(SK_d, Ni | Nr)
KEYMAT = prf+(SK_d, g^ir (new) | Ni | Nr)
```
If multiple IPsec protocols are negotiated, keying material for each Child SA is taken in the order in which the protocol headers will appear in the encapsulated packet.如果协商了多个IPsec协议，则按照协议头在封装数据包中出现的顺序获取每个子SA的密钥材料。

### Rekeying IKE SAs Using a CREATE_CHILD_SA Exchange
```
SKEYSEED = prf(SK_d (old), g^ir (new) | Ni | Nr)
```

### Authentication of the IKE SA
The initiator's signed octets can be described as:  
```
   InitiatorSignedOctets = RealMessage1 | NonceRData | MACedIDForI
   GenIKEHDR = [ four octets 0 if using port 4500 ] | RealIKEHDR
   RealIKEHDR =  SPIi | SPIr |  . . . | Length
   RealMessage1 = RealIKEHDR | RestOfMessage1
   NonceRPayload = PayloadHeader | NonceRData
   InitiatorIDPayload = PayloadHeader | RestOfInitIDPayload
   RestOfInitIDPayload = IDType | RESERVED | InitIDData
   MACedIDForI = prf(SK_pi, RestOfInitIDPayload)
```
The responder's signed octets can be described as:  
```
   ResponderSignedOctets = RealMessage2 | NonceIData | MACedIDForR
   GenIKEHDR = [ four octets 0 if using port 4500 ] | RealIKEHDR
   RealIKEHDR =  SPIi | SPIr |  . . . | Length
   RealMessage2 = RealIKEHDR | RestOfMessage2
   NonceIPayload = PayloadHeader | NonceIData
   ResponderIDPayload = PayloadHeader | RestOfRespIDPayload
   RestOfRespIDPayload = IDType | RESERVED | RespIDData
   MACedIDForR = prf(SK_pr, RestOfRespIDPayload)
```
- pre-shared key：
    ```
    For the initiator:
        AUTH = prf( prf(Shared Secret, "Key Pad for IKEv2"),
                        <InitiatorSignedOctets>)
            
    For the responder:
        AUTH = prf( prf(Shared Secret, "Key Pad for IKEv2"),
                        <ResponderSignedOctets>)
    ```
- digital signature
    ```
    AUTH = Sign(PrivKey, SignedOctets)
    ```
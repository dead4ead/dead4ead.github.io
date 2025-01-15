---
title: GB/T 36968-2018 信息安全技术 IPSec VPN技术规范
description: Information security technology —Technical specification for IPSec VPN
author: dead4ead
date: 2025-01-14 12:00:00 +0800
categories: [Secure, IPSec]
tags: [IKE, IPSec]
#pin: true
math: true
mermaid: true
---

## 字段介绍
- IDi:发起方的标识载荷  
- IDr:响应方的标识载荷  
- SIGi:发起方的签名载荷  
- SIGr:响应方的签名载荷  
- CERT_sig_i:签名证书载荷  
- CERT_enc_i:加密证书载荷  
- MsgID:消息标识号  
- Ni:发起方的Nonce载荷  
- Nr:响应方的Nonce载荷  
- pub_i:发起方的公钥  
- pub_r:响应方的公钥  
- prv_i:发起方的私钥  
- prv_r:响应方的私钥  
- CKY-I:ISAKMP头中的发起方cookie  
- CKY-R:ISAKMP头中的响应方cookie  
  
## 一阶段 -- 主模式

```
     Initiator                          Responder  
    -----------                        -----------  
     HDR, SA            -->  
                        <--    HDR, SA, CERT_sig_r, CERT_enc_r  
     HDR, XCHi, SIGi    -->  
                        <--    HDR, XCHr, SIGr  
     HDR*, HASHi        -->  
                        <--    HDR*, HASHr
```
其中:  
`XCHi = Asymmetric_Encrypt(Ski,pub_r) | Symmetric_Encrypt(Ni,Ski) | Symmetric_Encrypt(IDi,Ski) | CERT_sig_i | CERT_enc_i`  
`XCHr = Asymmetric_Encrypt(Skr,pub_i) | Symmetric_Encrypt(Nr,Skr) | Symmetric_Encrypt(IDr,Skr)`  
`SIGi_b = Asymmetric_Sign(Ski_b | Ni_b | IDi_b | CERT_enc_i_b, priv_i)`  
`SIGr_b = Asymmetric_Sign(Skr_b | Nr_b | IDr_b | CERT_enc_r_b, priv_r)`  

密钥参数计算：  
`SKEYID = PRF(HASH(Ni_b | Nr_b), CKY-I | CKY-R)`  
`SKEYID_d = PRF(SKEYID, CKY-I | CKY-R | 0)`  
`SKEYID_a = PRF(SKEYID, SKEYID_d | CKY-I | CKY-R | 1)`  
`SKEYID_e = PRF(SKEYID, SKEYID_a | CKY-I | CKY-R | 2)`  
  SKEYID_e长度不够时，利用如下反馈及链接方法加以扩展：  
  K = K1 | K2 | K3 ...  
  K1 = PRF(SKEYID_e, 0)  
  K2 = PRF(SKEYID_e, K1)  
  K3 = PRF(SKEYID_e, K2)  
  ...  

其中：  
SKEYID_a是ISAKMP SA用来验证其消息完整性以及数据源身份的工作密钥。  
SKEYID_e是ISAKMP SA用来保护其消息机密性的工作密钥。  
SKEYID_d是用于会话密钥的产生。  

五六条报文由SKEYID_e加密，对称密码运算使用CBC模式，初始化向量IV计算方式如下：  
`IV = HASH(Ski_b | Skr_b)`  

为了鉴别交换，双方产生HASH计算：  
`HASHi = PRF(SKEYID, CKY_i | CKY-R | SAi_b | IDii_b )`  
`HASHr = PRF(SKEYID, CKY-R | CKY-I | SAi_b | IDir_b )`


## 二阶段 -- 快速模式
二阶段报文使用对称密码算法的CBC模式，IV计算如下：  
IV = HASH(第一阶段的最后一组密文 | MsgID)    
后续的IV是前一个消息的最后一组密文。  

```
     Initiator                        Responder  
    -----------                      -----------  
     HDR*, HASH_1, 
      SA, Ni [, IDci, IDcr ]   -->  
                               <--    HDR*, HASH_2, 
                                       SA, Nr [, IDci, IDcr ]  
     HDR*, HASH_3              -->  
```
`HASH_1 = PRF(SKEYID_a, MsgID | Ni_b | SA [ | IDi | IDr ])`  
`HASH_2 = PRF(SKEYID_a, MsgID | Ni_b | SA | Nr_b [ | IDi | IDr ])`  
`HASH_3 = PRF(SKEYID_a, 0 | MsgID | Ni_b | Nr_b)`  

最后，会话密钥材料： 
KEYMAT = PRF(SKEYID_d, protocol | SPI | Ni_b | Nr_b)  
KEYMAT = K1 | K2 | K3 ...  
其中：  
K1 = PRF(SKEYID_d, protocol | SPI | Ni_b | Nr_b)  
K2 = PRF(SKEYID_d, K1 | protocol | SPI | Ni_b | Nr_b)  
K3 = PRF(SKEYID_d, K2 | protocol | SPI | Ni_b | Nr_b)  
...    

单个SA协商产生两个安全联盟，一个入、一个出。每个SA的不同SPI保证了每个方向都有一个不同的KEYMAT。

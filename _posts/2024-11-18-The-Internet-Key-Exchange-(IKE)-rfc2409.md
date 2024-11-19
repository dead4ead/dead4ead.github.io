---
title: IKE-rfc2049
description: The Internet Key Exchange (IKE)-rfc2409学习.
author: dead4ead
date: 2024-11-18 12:00:00 +0800
categories: [Secure, IPSec]
tags: [IKE, IPSec]
#pin: true
math: true
mermaid: true
---

## 一阶段
### 注意
- 由ISAKMP基础文档可知，ISAKMP SA由发起者的cookie后跟响应者的cookie来标识——在第一阶段的交换中各方的角色决定了哪一个cookie是发起者的。  
- 当使用公共密钥加密来验证时，野蛮模式仍然提供身份保护。

### 协商参数
The following attributes are used by IKE and are negotiated as part of the ISAKMP Security Association.
- encryption algorithm 加密算法
- hash algorithm hash算法
- authentication method 认证方法
- information about a group over which to do Diffie-Hellman. DH组  

以上是强制且必须协商的。可选协商伪随机函数['prf']，当前文档中未定义prf函数，协商双方可私下使用属性值定义，若没有协商prf时，使用HMAC的hash算法作为伪随机函数。

### 算法计算
四种不同的认证方式：数字签名、公钥加密的两种认证方式、预共享密钥。  
计算SKEYID：
- 对于数字签名：`SKEYID = prf(Ni_b | Nr_b, g^xy)`
- 对于公共密钥加密：`SKEYID = prf(hash(Ni_b | Nr_b), CKY-I | CKY-R)`
- 对于共享密钥：`SKEYID = prf(pre-shared-key, Ni_b | Nr_b)`  

计算密钥材料：  
* `SKEYID_d = prf(SKEYID, g^xy | CKY-I | CKY-R | 0)`  
`SKEYID_a = prf(SKEYID, SKEYID_d | g^xy | CKY-I | CKY-R | 1)`  
`SKEYID_e = prf(SKEYID, SKEYID_a | g^xy | CKY-I | CKY-R | 2)`  

验证交互双方，首先计算HASH_I和HASH_R：
- `HASH_I = prf(SKEYID, g^xi | g^xr | CKY-I | CKY-R | SAi_b | IDii_b )`
- `HASH_R = prf(SKEYID, g^xr | g^xi | CKY-R | CKY-I | SAi_b | IDir_b )`  
* 对于使用数字签名，HASH_I和HASH_R是经过签名并验证；  
* 对于使用公钥加密验证或共享密钥验证，HASH_I和HASH_R直接验证交换

### 1 使用签名来验证的IKE第一阶段
- 主模式  

        Initiator                          Responder  
       -----------                        -----------  
        HDR, SA                     -->  
                                    <--    HDR, SA  
        HDR, KE, Ni                 -->  
                                    <--    HDR, KE, Nr  
        HDR*, IDii, [ CERT, ] SIG_I -->  
                                    <--    HDR*, IDir, [ CERT, ] SIG_R

- 野蛮模式

        Initiator                          Responder  
       -----------                        -----------  
        HDR, SA, KE, Ni, IDii       -->  
                                    <--    HDR, SA, KE, Nr, IDir,
                                                [ CERT, ] SIG_R
        HDR, [ CERT, ] SIG_I        -->
  
签名的数据——SIG_I或SIG_R是协商的数字签名算法分别应用到HASH_I或HASH_R所产生的结果。  
将HASH使用私钥加密，对应SIG_I，公钥通过证书[ CERT, ]带过去。

### 2 使用公共密钥加密的第一阶段验证
使用公共密钥加密来验证交换，发起者必须已经拥有响应者的公钥。当响应者有多个公钥时，发起者用来加密辅助信息的证书的hash值也作为第三个消息传递。  

- 主模式

        Initiator                        Responder  
       -----------                      -----------  
        HDR, SA                   -->  
                                  <--    HDR, SA  
        HDR, KE, [ HASH(1), ]
          <IDii_b>PubKey_r,
            <Ni_b>PubKey_r        -->  
                                         HDR, KE, <IDir_b>PubKey_i,
                                  <--            <Nr_b>PubKey_i  
        HDR*, HASH_I              -->  
                                  <--    HDR*, HASH_R  


- 野蛮模式

        Initiator                        Responder  
       -----------                      -----------  
        HDR, SA, [ HASH(1),] KE,
          <IDii_b>Pubkey_r,
           <Ni_b>Pubkey_r         -->  
                                         HDR, SA, KE, <IDir_b>PubKey_i,
                                  <--         <Nr_b>PubKey_i, HASH_R  
        HDR, HASH_I               -->  


### 3 使用修改过的公钥加密模式来进行第一阶段的验证
使用公钥加密来进行验证比签名验证（参看5.2节）有更大的优点。不幸的是，这需要4个公钥操作为代价——两个公钥加密和两个私钥解密。而修改过的验证模式保持了使用公钥加密来验证的优点，但只需要一半的公钥操作。  

- 主模式

        Initiator                        Responder  
       -----------                      -----------  
        HDR, SA                   -->  
                                  <--    HDR, SA  
        HDR, [ HASH(1), ]
          <Ni_b>Pubkey_r,
          <KE_b>Ke_i,
          <IDii_b>Ke_i,
          [<Cert-I_b>Ke_i]       -->  
                                         HDR, <Nr_b>PubKey_i,
                                              <KE_b>Ke_r,
                                  <--         <IDir_b>Ke_r,  
        HDR*, HASH_I              -->  
                                  <--    HDR*, HASH_R  

- 野蛮模式

        Initiator                        Responder  
       -----------                      -----------  
        HDR, SA, [ HASH(1),]
          <Ni_b>Pubkey_r,
          <KE_b>Ke_i, <IDii_b>Ke_i
          [, <Cert-I_b>Ke_i ]     -->  
                                         HDR, SA, <Nr_b>PubKey_i,
                                              <KE_b>Ke_r, <IDir_b>Ke_r,  
                                  <--         HASH_R  
        HDR, HASH_I               -->  

  - Nonce：使用对方公钥加密  
  - KE、身份[以及证书]：使用对称加密算法加密，对称密钥从nonce中衍生  
  - 对称密钥Ke：
    1. 先计算`Ne_i = prf(Ni_b, CKY-I) ; Ne_r = prf(Nr_b, CKY-R)`
    2. 再计算`Ke = K = K1 | K2 | K3 and K1 = prf(Ne_i, 0) ...` 

### 4 使用共享密钥的第一阶段协商
- 主模式

          Initiator                        Responder  
         ----------                       -----------  
          HDR, SA             -->  
                              <--    HDR, SA  
          HDR, KE, Ni         -->  
                              <--    HDR, KE, Nr  
          HDR*, IDii, HASH_I  -->  
                              <--    HDR*, IDir, HASH_R  

- 野蛮模式

        Initiator                        Responder  
       -----------                      -----------  
        HDR, SA, KE, Ni, IDii -->  
                              <--    HDR, SA, KE, Nr, IDir, HASH_R  
        HDR, HASH_I           -->  

## 二阶段
- The message ID in the ISAKMP header identifies a Quick Mode in progress for a particular ISAKMP SA which itself is identified by the cookies in the ISAKMP header.  
  在ISAKMP报头中的Message ID标识了快速模式正在进行中，而ISAKMP SA本身由ISAKMP报头中的cookie来标识。  
- The client identities are used to identify and direct traffic to the appropriate tunnel in cases where multiple tunnels exist between two peers and also to allow for unique and shared SAs with different granularities.  
  在双方之间有多个隧道存在的情况下，客户身份用于标识并指导流量进入对应的隧道，同时也用于支持不同粒度的唯一和共享SA。  

- 交互流程

        Initiator                        Responder  
       -----------                      -----------  
        HDR*, HASH(1), SA, Ni
          [, KE ] [, IDci, IDcr ] -->  
                                  <--    HDR*, HASH(2), SA, Nr
                                               [, KE ] [, IDci, IDcr ]  
        HDR*, HASH(3)             -->  

   `HASH(1) = prf(SKEYID_a, M-ID | SA | Ni [ | KE ] [ | IDci | IDcr )`  
   `HASH(2) = prf(SKEYID_a, M-ID | Ni_b | SA | Nr [ | KE ] [ | IDci | IDcr )`  
   `HASH(3) = prf(SKEYID_a, 0 | M-ID | Ni_b | Nr_b)`  

- 密钥材料计算  
   `KEYMAT = prf(SKEYID_d, protocol | SPI | Ni_b | Nr_b).`  
   `KEYMAT = prf(SKEYID_d, g(qm)^xy | protocol | SPI | Ni_b | Nr_b) `[PFS]``  
   [解密]InBound：OurKeymat-OurSPI  
   [加密]Outbound：PeerKeymat-PeerSPI  

- Using Quick Mode, multiple SA's and keys can be negotiated with one exchange.  
使用快速模式，多个SA和密钥可以使用一个交换来协商.

        Initiator                        Responder  
       -----------                      -----------  
        HDR*, HASH(1), SA0, SA1, Ni,
          [, KE ] [, IDci, IDcr ] -->  
                                  <--    HDR*, HASH(2), SA0, SA1, Nr,
                                            [, KE ] [, IDci, IDcr ]  
        HDR*, HASH(3)             -->  
        
     密钥材料和在单个SA的情况中一样的衍生出来。在这种情况下（两个SA负载的协商），结果将是四个安全联盟,每个方向两个。

## New Group Mode(NGM)
New Group Mode 是一个独立的 IKE 消息交换模式，不属于 IKE 的“阶段 1”或“阶段 2”。它通常发生在阶段 1 和阶段 2 之间，具体流程如下：
1. 阶段 1 建立安全关联（SA）：  
通过主模式或野蛮模式建立 IKE SA，双方完成认证并建立了初始的加密和完整性保护通道。
2. New Group Mode 协商新 DH 群：  
   - 在 IKE SA 的保护下，双方通过 NGM 消息交换协商新的 DH 参数（包括 p 和 g）。
   - NGM 的交换完成后，双方可以使用新群执行后续的 DH 密钥交换。
3. 阶段 2 使用新 DH 群：  
阶段 2 的 Quick Mode 或其他协商可以基于 NGM 协商的新群，生成更安全的会话密钥。  

## ISAKMP Informational Exchanges[ISAKMP信息交换]
Informational Exchanges 是一种独立的消息交换模式，专门用于传递与密钥管理或安全关联（SA）状态相关的辅助信息。它主要用来发送非关键但重要的信息，不涉及复杂的协商流程。  
Informational Exchanges 用于传递以下几类信息：
1. 错误通知：
   - 如果在 IKE 的协商过程中发生了错误（如算法不匹配或消息解析失败），一方可以通过 Informational Exchanges 发送错误通知。
   - 错误通知可以包括特定的错误代码和附加信息，例如某个特定参数无法支持。

2. 删除安全关联（SA）：
   - 用于通知对端终止或删除一个现有的 SA。删除操作通常是显式的，可以通过消息指明具体要删除的 SA。
   - 例如，当某一方不再需要使用某个 IPsec SA 时，它会通过 Informational Exchanges 通知对方。

3. 状态信息：
   - 用于传递状态更新信息，例如报告某些非致命的问题。
   - 例如，一方可以报告当前密钥即将到期，并建议进行重新协商。

4. 心跳（Keep-alive）机制：
   - Informational Exchanges 可用作轻量级的心跳机制，用于检测双方的通信连接是否正常。
   - 如果长时间没有其他流量，可以通过发送空消息来确认对方是否仍在线。

5. 其他通知或扩展：
   - 可以传递与协议扩展相关的附加信息，例如新特性支持或附加参数的使用。


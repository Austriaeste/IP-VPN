# IP-VPN設定の読み物

## 目次

- [はじめに](#はじめに)
- [IP-VPNとは](#ip-vpnとは)
- [主要技術要素](#主要技術要素)
- [構成例とデータフロー](#構成例とデータフロー)
- [RDとRT：IP重複とルーティング分離の仕組み](#rdとrtip重複とルーティング分離の仕組み)
- [MPLS LDPの動作概説](#mpls-ldpの動作概説)
- [RADIUS認証連携](#radius認証連携)
- [Cisco IOS/IOS-XE 設定例](#cisco-iosios-xe-設定例)
- [トラブルシュート例](#トラブルシュート例)
- [QoS設定例](#qos設定例)

---

## はじめに

この資料は、**IP-VPN (MPLS L3VPN中心)** の基礎から、詳細な設定、運用、さらにはトラブルシューティングまで、体系的にまとめたものです。

---

## IP-VPNとは

**IP-VPNは、閉域網を利用して顧客間の通信をセキュアかつ柔軟に接続するサービスです。**  
特に**MPLS L3VPN**がその中心となり、ルーティングレベルでの顧客ネットワークの分離を実現します。  
これにより、Quality of Service (QoS) の提供や、顧客間のIPアドレス重複の回避（VRFの利用）が可能になります。

---

## 主要技術要素

### MPLS

**MPLS (Multi-Protocol Label Switching)** は、パケットに「ラベル」を付与して転送することで、ルーティングテーブルの参照を最小限に抑え、高速な転送を可能にする技術です。

- **LSR (Label Switch Router):** ラベルを見てパケットを転送するルータです。  
- **LER (Label Edge Router):** ネットワークの境界に位置し、IPパケットにラベルを付与したり、ラベルを剥離したりします。PEルータがこれに該当します。

### VRF

**VRF (Virtual Routing and Forwarding)** は、ルータ上に複数の独立したルーティングテーブルと転送テーブルを論理的に作成する技術です。  
これにより、同じIPアドレス空間を持つ複数の顧客ネットワークが、互いに干渉することなく同じルータ上で共存できます。

### MP-BGP

**MP-BGP (Multi-Protocol BGP)** は、VPNv4アドレスファミリーをサポートするBGPの拡張です。

- **VPNv4アドレス:** 通常のIPv4アドレスに、後述する**RD (Route Distinguisher)** を付与した12バイトのアドレスです。  
- **RT (Route Target):** ルートのエクスポート/インポートを制御するための属性で、VPNルーティング情報のどのVRFにどのルートを配布するかを決定します。

---

## 構成例とデータフロー

### 構成図

```mermaid
graph TD

    %% Legend (凡例)
    LegendA[Customer A]
    LegendB[Customer B]
    LegendP[Provider NW - MPLS Core]

    %% 本構成
    subgraph Customer_A
        CE_A[Customer Edge A]
    end

    subgraph Customer_B
        CE_B[Customer Edge B]
    end

    subgraph Provider
        PE1[Provider Edge 1] --> P1_LSR[Provider LSR 1]
        P1_LSR --> P2_LSR[Provider LSR 2]
        P2_LSR --> PE2[Provider Edge 2]
    end

    CE_A -- カスタマールート交換 (e.g., EBGP) --> PE1
    PE2 -- カスタマールート交換 (e.g., EBGP) --> CE_B
````

### データフローの概要

1. **CEからPEへ (データプレーン):** Customer AのCEルータからPE1にIPパケットが到達します。PE1はVRF Aに割り当てられたインターフェースでパケットを受信します。
2. **PEにおけるラベル付与:** PE1は、VRF Aのルーティングテーブルを参照し、宛先IPアドレスに対応するVPNv4ルートから、**VPNラベル (内側ラベル)** と、PルータへのMPLS転送用の**トランスポートラベル (外側ラベル)** を付与します。
3. **Pルータでのラベルスワップ:** Pルータ（LSR）は、外側のトランスポートラベルのみを見てパケットを転送します。各Pルータは、受信したラベルを自身の転送テーブルに基づいて新しいラベルに交換（ラベルスワップ）し、次のホップへ転送します。
4. **最終PEでのラベル剥離:** PE2に到達する直前（Penultimate Hop Popping - PHP）で、通常、最後のPルータがトランスポートラベルを剥離します。PE2はVPNラベルのみを受信します。
5. **PEからCEへ (データプレーン):** PE2はVPNラベルに基づいて、該当するVRF Bのルーティングテーブルを参照し、パケットを通常のIPパケットとしてCE Bに転送します。

---

## RDとRT：IP重複とルーティング分離の仕組み

**IP-VPNにおいて、複数の顧客が同じIPアドレス空間を利用できるのは、RD (Route Distinguisher) とRT (Route Target) の巧妙な連携があるからです。**

### RD (Route Distinguisher)

RDは、**VPNv4アドレス**の一部として機能し、通常のIPv4アドレス（32ビット）の前に付与される64ビットの値です。これにより、異なるVRFで同じIPv4プレフィックスが使用されていても、BGP上ではそれぞれ異なるVPNv4ルートとして識別されます。

* **役割:** IPアドレスの重複を回避し、VRF間でルーティング情報を分離するための識別子。
* **例:**

  * Customer Aの`10.1.1.0/24`は`RD_A:10.1.1.0/24`となる。
  * Customer Bの`10.1.1.0/24`は`RD_B:10.1.1.0/24`となる。
    これにより、BGPはこれらを別々のルートとして認識します。

### RT (Route Target)

RTは、**VPNv4ルートのインポート/エクスポートを制御するためのコミュニティ属性**です。各VRFには、どのRTを持つルートを「インポート」し、どのRTを持つルートを「エクスポート」するかを設定します。

* **役割:** VPNv4ルートを特定のVRFに「注入」したり、「抽出」したりするためのタグ。これにより、顧客間のルート共有範囲を柔軟に制御できます。
* **インポート:** 特定のRTを持つVPNv4ルートを、そのVRFのルーティングテーブルに取り込みます。
* **エクスポート:** そのVRFからBGPに広告するVPNv4ルートに特定のRTを付与します。

### RDとRTの連携

1. CEからPEへ: CEからPEに広告されたIPv4ルートは、PEのVRFで受信されます。
2. VPNv4ルートへの変換 (PE): PEは、受信したIPv4ルートに自身のVRFに設定されたRDを付与し、VPNv4ルートに変換します。
3. RTのエクスポート (PE): そのVPNv4ルートには、VRFに設定されたエクスポートRTが付与されます。
4. MP-BGPによる広告: VPNv4ルートは、MP-BGPを介して他のPEルータに広告されます。
5. RTのインポート (PE): 受信側のPEルータは、広告されたVPNv4ルートのRTをチェックし、自身のVRFに設定されたインポートRTと一致するVPNv4ルートのみを、VRFのルーティングテーブルに取り込みます。
6. IPv4ルートへの変換 (PE): VRFに取り込まれたVPNv4ルートは、RDが剥がされ、通常のIPv4ルートとしてCEに広告されます。

この仕組みにより、同じIPアドレスを持つ顧客ネットワーク同士でも、RDとRTによってルーティング情報が適切に分離・共有され、セキュアな閉域網接続が実現されます。

---

## MPLS LDPの動作概説

**MPLS LDP (Label Distribution Protocol) は、MPLS網内でラベルを配布・交換するためのプロトコルです。**
これにより、Pルータ間でトランスポートラベル（外側ラベル）の経路情報が確立され、MPLSによる高速転送が可能になります。

### LDPセッションの確立

1. **Helloメッセージ交換:** LDPルータは、Helloメッセージをマルチキャストで送信し、LDPネイバーを発見します。
2. **TCPセッションの確立:** Helloメッセージ交換後、LDPネイバー間でTCPポート646番を利用してLDPセッションが確立されます。

### FECとラベルのマッピング

LDPは、**FEC (Forwarding Equivalence Class)** と**ラベル**をマッピングします。

1. **ローカルラベルの生成:** 各LDPルータは、自身のルーティングテーブルにあるIPプレフィックス（FEC）に対して、ローカルで有効なラベルを割り当てます。
2. **ラベルの配布:** ルータは、生成したFECとラベルのマッピング情報をネイバーに広告します。
3. **転送情報データベースの構築:** ルータは、ネイバーから受信したラベル情報を自身の「ラベル転送情報ベース (LFIB)」に格納します。

### LDPの動作によるMPLSパスの確立

LDPは、IGP (OSPF, EIGRPなど) で確立されたIP経路情報に基づいてラベルを配布し、Pルータ間でラベルスイッチパス (LSP) を確立します。

---

## RADIUS認証連携

### なぜ必要？

* **不正アクセス防止:** デバイスやVPN接続ユーザーの認証を中央管理し、不正アクセスを防止。
* **認証によるポリシー適用:** 認証結果に応じてアクセス権やQoSポリシーを適用。
* **アカウンティング:** 利用状況の記録・監査・課金に役立つ。

### RADIUS設定例（Ciscoルータ）

```cli
aaa new-model
aaa authentication login default group radius local
aaa authorization exec default group radius local

radius-server host {{RADIUS_SERVER_IP}} auth-port 1812 acct-port 1813 key {{SHARED_SECRET}} timeout 5 retransmit 3

line vty 0 4
    login authentication default
    transport input ssh
```

> **ポイント**
>
> * フォールバックにローカル認証を設定。
> * RADIUSサーバー情報と共有鍵の設定。
> * SSHアクセス推奨。
> * 疎通確認は`ping`や`debug radius authentication`で。

---

## Cisco IOS/IOS-XE 設定例

```cli
hostname {{HOSTNAME}}

interface {{PROVIDER_FACING_INTERFACE}}
 ip address {{IP_ADDRESS}} {{SUBNET_MASK}}
 ip ospf {{PROCESS_ID}} area {{AREA_ID}}
 mpls ip

ip vrf {{VRF_NAME_CUSTOMER_A}}
 rd {{ASN}}:{{VRF_ID_CUSTOMER_A}}
 route-target export {{ASN}}:{{VRF_ID_CUSTOMER_A}}
 route-target import {{ASN}}:{{VRF_ID_CUSTOMER_A}}

ip vrf {{VRF_NAME_CUSTOMER_B}}
 rd {{ASN}}:{{VRF_ID_CUSTOMER_B}}
 route-target export {{ASN}}:{{VRF_ID_CUSTOMER_B}}
 route-target import {{ASN}}:{{VRF_ID_CUSTOMER_B}}

interface {{CE_FACING_INTERFACE_CUSTOMER_A}}
 ip vrf forwarding {{VRF_NAME_CUSTOMER_A}}
 ip address {{CE_PE_IP_CUSTOMER_A}} {{CE_PE_SUBNET_MASK}}

router bgp {{ASN}}
 neighbor {{PEER_IP_LOOPBACK}} remote-as {{ASN}}
 neighbor {{PEER_IP_LOOPBACK}} update-source Loopback0
 address-family vpnv4
  neighbor {{PEER_IP_LOOPBACK}} activate
  neighbor {{PEER_IP_LOOPBACK}} send-community extended
 exit-address-family

 address-family ipv4 vrf {{VRF_NAME_CUSTOMER_A}}
  neighbor {{CE_IP_CUSTOMER_A}} remote-as {{CE_ASN_CUSTOMER_A}}
  neighbor {{CE_IP_CUSTOMER_A}} activate
  redistribute connected
 exit-address-family

 address-family ipv4 vrf {{VRF_NAME_CUSTOMER_B}}
  neighbor {{CE_IP_CUSTOMER_B}} remote-as {{CE_ASN_CUSTOMER_B}}
  neighbor {{CE_IP_CUSTOMER_B}} activate
  redistribute connected
 exit-address-family
```

---

## トラブルシュート例

### LDPセッションが張れない

* `show mpls ldp neighbor`でネイバー確認。
* `mpls ip` がインターフェースで有効か。
* IGP設定確認 (`show ip ospf neighbor` など)。
* TCPポート646のファイアウォール確認。

### VPNv4ルートが来ない

* `show bgp vpnv4 unicast all summary`でMP-BGPピア確認。
* RT設定ミス確認。
* RDの重複誤設定確認。
* BGPで`send-community extended`設定有無。

### CE-PE間BGPセッション不成立

* AS番号設定誤り。
* VRFの割り当て忘れ。
* ACL/NATのブロック確認。
* BGPネイバーIPのVRF内設定確認。

### Ping通らない

* VRFルーティングテーブル確認。
* ARP/MACテーブル確認。
* MPLS転送経路確認。
* 物理層/データリンク層確認。

---

## QoS設定例

```cli
class-map match-any VOICE_CLASS
 match dscp ef

class-map match-any VIDEO_CLASS
 match dscp af41

policy-map QOS_POLICY_OUT
 class VOICE_CLASS
  priority percent 30
 class VIDEO_CLASS
  bandwidth percent 20
  set dscp af41
 class class-default
  fair-queue
  random-detect

interface {{PROVIDER_FACING_INTERFACE}}
 service-policy output QOS_POLICY_OUT
```

---

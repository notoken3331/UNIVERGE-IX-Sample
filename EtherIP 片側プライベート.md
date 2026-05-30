## 想定構築ネットワーク
固定IPアドレス `100.200.111.222` には `rt1.example.jp` というドメインが設定されているものとします。

```
  RT1側ネットワーク
          ||
          ||
GE1.0 <主ネットワーク> 172.16.0.1/23
<<<============ RT1 ============>>> Tunnel11.0
GE0.1 PPPoE static 100.200.111.222     ||| 
          ||                           ||| 
          ||                           |||
[==== The Internet  ====]              ||| EtherIP L2 トンネル
          ||                           ||| IKEv2
[== Career Grade NAPT ==]              |||
          ||                           |||
          ||                           |||
GE0.0 DHCP Dynamic Private IP          |||
<<<============ RT2 ============>>> Tunnel11.0
GE1.0 <主ネットワークへL2接続> 172.16.0.2/23
          ||
          ||
  RT2側ネットワーク
```

## 設定例

### RT1
RT2側の出口IPアドレスに関しては「不定」のため、anyを設定している。
DDNS等を利用しても良いと思うが、これは出先のホテルとかで適当に使うことを想定しているので、anyとして何でも受け入れるようにしている

IPアドレスは `BVI1` 仮想インタフェースに設定し、 `GE1.0` 及び `Tunnel11.0` へ `bridge-group 1` でブリッジを行う。
こちら、GE1.0にIPを設定した場合、 `Tunnel11.0` 側から該当IPアドレスへのアクセスができなくなります。


```
! 事前共有鍵の定義
ikev2 authentication psk id keyid ROUTER1 key char RT1_PRE_SHARED_KEY_1111
ikev2 authentication psk id keyid ROUTER2 key char RT2_PRE_SHARED_KEY_2222


! EtherIP用プロファイルの作成
! 各自環境と要件に合わせて暗号強度を選択してください。
ikev2 profile for-etherip
  child-pfs off
  child-proposal enc aes-gcm-256-16 aes-cbc-256
  child-proposal integrity sha2-512
  dpd interval 10
  local-authentication psk id keyid ROUTER1
  sa-proposal enc aes-gcm-256-16 aes-cbc-256
  sa-proposal integrity sha2-512
  sa-proposal dh 3072-bit 2048-bit
  sa-proposal prf sha2-512
  ipsec-mode transport


! DHCPの設定
ip dhcp profile dhcp-local-network
  dns-server 172.16.0.1


! PPPoE認証情報の設定
ppp profile ppp_profile
  authentication myname PPPoE_USERNAME@DOMAIN
  authentication password PPPoE_USERNAME@DOMAIN PPPoE_PASSWORD


! PPPoEインターフェイスにIPSec関連の静的NAPT設定を入れる
interface GigaEthernet0.1
  encapsulation pppoe
  auto-connect
  ppp binding ppp_profile
  ip address ipcp
  ip tcp adjust-mss auto
  ip napt enable
  ip napt static GigaEthernet0.1 udp 500
  ip napt static GigaEthernet0.1 50
  ip napt static GigaEthernet0.1 udp 4500
  no shutdown


! 物理インターフェースにはIPを割り当てない
interface GigaEthernet1.0
  bridge-group 1
  no shutdown


! ブリッジインターフェース用で利用する仮想インターフェース `BVI1` にIPを設定する
interface BVI1
  ip address 172.16.0.1/23
  ip dhcp binding dhcp-local-network
  bridge-group 1
  no shutdown


!Tunnel インターフェイスの設定
interface Tunnel11.0
  description EhterIP_L2_TUNNEL
  tunnel mode ether-ip ipsec-ikev2
  no ip address
  bridge-group 1
  bridge ip tcp adjust-mss 1300
  bridge ipv6 tcp adjust-mss 1300
  ikev2 binding for-etherip
  ikev2 connect-type auto
  ikev2 ipsec pre-fragment
  ikev2 outgoing-interface GigaEthernet0.1
  ikev2 nat-traversal keepalive 10 force
  ikev2 peer any authentication psk id keyid ROUTER2
  no shutdown
```

<!-- 
#### if CGN Gateway IP にDDNS等でFQDNを設定する場合
CGN Gateway IP へ任意のDDNSアドレス等を設定する場合、上記の ikev2 peer 行を以下のように書き換えることができます。

* 注意事項
    * 同一Gateway IP から複数のトンネルを張る可能性がある場合は、以下の設定はうまく動かない可能性があります。
    * 同一Gateway IP に複数のDDNSによるFQDNが割り当てられた場合、意図しない挙動をする可能性があります。

```
ikev2 peer-fqdn-ipv4 DOMAIN_NAME authentication psk id keyid ROUTER2
```
-->

### RT2
GE0.0 にインターネット提供者からDHCPでプライベートIPアドレスが付与されることを想定している。

```
! 事前共有鍵の定義
ikev2 authentication psk id keyid ROUTER1 key char RT1_PRE_SHARED_KEY_1111
ikev2 authentication psk id keyid ROUTER2 key char RT2_PRE_SHARED_KEY_2222


! EtherIP用プロファイルの作成
! 各自環境と要件に合わせて暗号強度を選択してください。
ikev2 profile for-etherip
  child-pfs off
  child-proposal enc aes-gcm-256-16 aes-cbc-256
  child-proposal integrity sha2-512
  dpd interval 10
  local-authentication psk id keyid ROUTER2
  sa-proposal enc aes-gcm-256-16 aes-cbc-256
  sa-proposal integrity sha2-512
  sa-proposal dh 3072-bit 2048-bit
  sa-proposal prf sha2-512
  ipsec-mode transport


! ISPからDHCPでIPを受け取る設定
interface GigaEthernet0.0
  ip address dhcp receive-default
  ip tcp adjust-mss auto
  ip napt enable
  no shutdown


! 物理インターフェースにはIPを割り当てない
interface GigaEthernet1.0
  bridge-group 1
  no shutdown


! ブリッジインターフェース用で利用する仮想インターフェース `BVI1` にIPを設定する
interface BVI1
  ip address 172.16.0.2/23
  bridge-group 1
  no shutdown


!Tunnel インターフェイスの設定
interface Tunnel11.0
  tunnel mode ether-ip ipsec-ikev2
  no ip address
  bridge-group 1
  bridge ip tcp adjust-mss 1300
  bridge ipv6 tcp adjust-mss 1300
  ikev2 binding for-etherip
  ikev2 connect-type auto
  ikev2 ipsec pre-fragment
  ikev2 nat-traversal keepalive 10 force
  ikev2 peer-fqdn-ipv4 RT1.EXAMPLE.JP authentication psk id keyid ROUTER1
  no shutdown
```

## Tips:

### セキュリティに関して
共通ですので、こちらをご確認ください。

https://github.com/notoken3331/UNIVERGE-IX-Sample/blob/master/IPSec%20%E7%89%87%E5%81%B4%E3%83%97%E3%83%A9%E3%82%A4%E3%83%99%E3%83%BC%E3%83%88.md#tips

### MTUに関して
一部の通信に関して、MTUが1500を下回ると動作しないプロトコルが存在します。トンネル経由で何故かうまく行かない場合は多分それなので少し設定を弄る必要があります。

todo: 後で書く
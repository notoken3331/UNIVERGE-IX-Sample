## 想定構築ネットワーク
固定IPアドレス `100.200.111.222` には `rt1.example.jp` というドメインが設定されているものとします。
CGN出口IPアドレスに `rt2.example.jp` というDDNSドメインを設定するものとしています。

トンネル内IPアドレスに関しては /30 で作成しています。

```
  RT1側ネットワーク
          ||
          ||
GE1.0 172.16.11.1/24
<<<============ RT1 ============>>> Tunnel11.0 10.0.0.1/30
GE0.1 PPPoE static 100.200.111.222     ||| 
          ||                           ||| 
          ||                           |||
[==== The Internet  ====]              ||| IPSec
          ||                           ||| IKEv2
[== Career Grade NAPT ==]              |||
          ||                           |||
          ||                           |||
GE0.0 DHCP Dynamic Private IP          |||
<<<============ RT2 ============>>> Tunnel11.0 10.0.0.2/30
GE1.0 172.16.22.1/24
          ||
          ||
  RT2側ネットワーク
```

## 設定例

### RT1
RT2側の出口IPアドレスに関しては「不定」のため、anyを設定している。
DDNS等を利用しても良いと思うが、これは出先のホテルとかで適当に使うことを想定しているので、anyとして何でも受け入れるようにしている


```
! 事前共有鍵の定義
ikev2 authentication psk id keyid ROUTER1 key char RT1_PRE_SHARED_KEY_1111
ikev2 authentication psk id keyid ROUTER2 key char RT2_PRE_SHARED_KEY_2222


! static route (固定経路)設定
ip route 172.16.22.0/24 10.0.0.2


! EtherIP用プロファイルの作成
! 各自環境と要件に合わせて暗号強度を選択してください。
ikev2 profile for-ipsec-ike2
  child-pfs off
  child-proposal enc aes-gcm-256-16 aes-cbc-256
  child-proposal integrity sha2-512
  dpd interval 10
  local-authentication id keyid ROUTER1
  sa-proposal enc aes-gcm-256-16 aes-cbc-256
  sa-proposal integrity sha2-512
  sa-proposal dh 2048-bit 1024-bit
  sa-proposal prf sha2-512
  ipsec-mode tunnel


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
  ip address 172.16.11.1/24
  no shutdown


!Tunnel インターフェイスの設定
interface Tunnel11.0
  description ipsec-Router2
  tunnel mode ipsec-ikev2
  ip address 10.0.0.1/30
  ip mtu 1350
  ip tcp adjust-mss auto
  ikev2 binding for-ipsec-ike2
  ikev2 connect-type auto
  ikev2 dpd interval 10
  ikev2 ipsec pre-fragment
  ikev2 nat-traversal keepalive 10 force
  ikev2 negotiation-direction responder
  ikev2 outgoing-interface GigaEthernet0.1
  ikev2 peer-fqdn-ipv4 RT2.EXAMPLE.JP authentication psk id keyid ROUTER2
  no shutdown

```

### RT2
GE0.0 にインターネット提供者からDHCPでプライベートIPアドレスが付与されることを想定している。

```
! 事前共有鍵の定義
ikev2 authentication psk id keyid ROUTER1 key char RT1_PRE_SHARED_KEY_1111
ikev2 authentication psk id keyid ROUTER2 key char RT2_PRE_SHARED_KEY_2222


! static route (固定経路)設定
ip route 172.16.11.0/24 10.0.0.1


! EtherIP用プロファイルの作成
! 各自環境と要件に合わせて暗号強度を選択してください。
ikev2 profile for-ipsec-ike2
  child-pfs off
  child-proposal enc aes-gcm-256-16 aes-cbc-256
  child-proposal integrity sha2-512
  dpd interval 10
  local-authentication id keyid ROUTER2
  sa-proposal enc aes-gcm-256-16 aes-cbc-256
  sa-proposal integrity sha2-512
  sa-proposal dh 2048-bit 1024-bit
  sa-proposal prf sha2-512
  ipsec-mode tunnel


! ISPからDHCPでIPを受け取る設定
interface GigaEthernet0.0
  ip address dhcp receive-default
  ip tcp adjust-mss auto
  ip napt enable
  no shutdown


! 物理インターフェースにはIPを割り当てない
interface GigaEthernet1.0
  ip address 172.16.22.1/24
  no shutdown


!Tunnel インターフェイスの設定
interface Tunnel11.0
  description ipsec-Router1
  tunnel mode ipsec-ikev2
  ip address 10.0.0.2/30
  ip mtu 1350
  ip tcp adjust-mss auto
  ikev2 binding for-nec-ix
  ikev2 connect-type auto
  ikev2 dpd interval 10
  ikev2 ipsec pre-fragment
  ikev2 nat-traversal keepalive 10 force
  ikev2 negotiation-direction initiator
  ikev2 peer-fqdn-ipv4 RT1.EXAMPLE.JP authentication psk id keyid ROUTER1
  no shutdown
```

### Tips

#### トンネルにIPを振りたくない場合
トンネル内にIPアドレスを設定しない場合は、以下の通り設定することができる。
unnumbered 設定で 経路を Tunnel11.0 に設定する

RT1側

```
ip route 172.16.22.0/24 Tunnel11.0

interface Tunnel11.0
  ip unnumbered GigaEthernet1.0
```

RT2側

```
ip route 172.16.11.0/24 Tunnel11.0

interface Tunnel11.0
  ip unnumbered GigaEthernet1.0
```
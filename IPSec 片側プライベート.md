## 想定構築ネットワーク
固定IPアドレス `100.200.111.222` には `rt1.example.jp` というドメインが設定されているものとします。

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
事前共有鍵に関して、特段 `ikev2 peer any` を設定する場合は20桁以上の強固な鍵を設定することをおすすめします。

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
  sa-proposal dh 2048-bit
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
  ikev2 ipsec pre-fragment
  ikev2 nat-traversal keepalive 10 force
  ikev2 negotiation-direction responder
  ikev2 outgoing-interface GigaEthernet0.1
  ikev2 peer any authentication psk id keyid ROUTER2
  no shutdown

```

#### if CGN Gateway IP にDDNS等でFQDNを設定する場合
CGN Gateway IP へ任意のDDNSアドレス等を設定する場合、上記の ikev2 peer 行を以下のように書き換えることができます。が、特段理由がない限りanyを設定することをおすすめします。

* 注意事項
    * 同一Gateway IP から複数のトンネルを張る可能性がある場合は、以下の設定はうまく動かない可能性があります。
    * 同一Gateway IP に複数のDDNSによるFQDNが割り当てられた場合、意図しない挙動をする可能性があります。

```
ikev2 peer-fqdn-ipv4 DOMAIN_NAME authentication psk id keyid ROUTER2
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
  sa-proposal dh 2048-bit
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
  ikev2 ipsec pre-fragment
  ikev2 nat-traversal keepalive 10 force
  ikev2 negotiation-direction initiator
  ikev2 peer-fqdn-ipv4 RT1.EXAMPLE.JP authentication psk id keyid ROUTER1
  no shutdown
```

## Tips

### セキュリティに関して
* 事前共有鍵は、特段 `ikev2 peer any` を設定する場合、20桁以上の強固な鍵を設定することをおすすめします。
* セキュリティ強度に関して、特段インターネットVPNを構築する際は強固な暗号方式を使うことをおすすめします。
    * 詳細はコマンドリファレンスをご確認ください。
        https://www.necplatforms.co.jp/dl/ix-nrv/manual/crm/cli/security/cli_ikev2.html#ikev2-child-proposal-enc
    * 暗号化強度が低い順に以下のとおりとなります。
        1. null            -- NULL Algorithm
        1. 3des-cbc        -- Triple DES-CBC
        2. aes-cbc-128     -- AES-CBC (128 bits)
        3. aes-cbc-192     -- AES-CBC (192 bits)
        4. aes-cbc-256     -- AES-CBC (256 bits)
        5. aes-gcm-128-16  -- AES-GCM (128 bits key with 16 octets ICV)
        6. aes-gcm-256-16  -- AES-GCM (256 bits key with 16 octets ICV)
        * IX2105 (EoL済み機種) を利用する場合は、 `aes-gcm-256-16` が利用できませんので `aes-cbc-256` が最大強度となります。
        * null は暗号化しないという意味です。
    * 認証アルゴリズムに関しては強度が低い順に以下のとおりです。
        1. sha1            -- HMAC-SHA1-96
        2. sha2-256        -- HMAC-SHA2-256-128
        3. sha2-384        -- HMAC-SHA2-384-192
        4. sha2-512        -- HMAC-SHA2-512-256
    * 鍵交換プロトコルに関しては強度が低い順に以下のとおりです。
        1. 768-bit   -- DH Group 1
        1. 1024-bit  -- DH Group 2
        1. 1536-bit  -- DH Group 5
        1. 2048-bit  -- DH Group 14
        1. 3072-bit  -- DH Group 15
        * IX2105 (EoL済み機種) を利用する場合は、 `2048-bit` までの対応となります。

何にも考えず最高強度として設定するなら以下のようになります。

```
ikev2 profile for-ipsec-ike2
  child-proposal enc aes-gcm-256-16
  child-proposal integrity sha2-512
  sa-proposal enc aes-gcm-256-16
  sa-proposal integrity sha2-512
  sa-proposal dh 3072-bit
  sa-proposal prf sha2-512
```

IX2105 が含まれる場合は、以下のようにします

```
ikev2 profile for-ipsec-ike2
  child-proposal enc aes-gcm-256-16 aes-cbc-256
  child-proposal integrity sha2-512
  sa-proposal enc aes-gcm-256-16 aes-cbc-256
  sa-proposal integrity sha2-512
  sa-proposal dh 3072-bit 2048-bit
  sa-proposal prf sha2-512
```


### IPoE 固定IP (v6プラス固定IP, Xpass固定等) での設定
適時 `GigaEthernet0.1` を `Tunnel0.0` 等に読み替えてください。

```
interface Tunnel0.0
  ip napt enable
  ip napt static Tunnel0.0 udp 500
  ip napt static Tunnel0.0 50
  ip napt static Tunnel0.0 udp 4500

interface Tunnel11.0
  ikev2 outgoing-interface Tunnel0.0
```


### トンネルにIPを振りたくない場合
トンネル内にIPアドレスを設定しない場合は、以下の通り設定することができる。
unnumbered 設定で経路を Tunnel11.0 に設定する

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

### MTUに関して
一部の通信に関して、MTUが1500を下回ると動作しないプロトコルが存在します。トンネル経由で何故かうまく行かない場合は多分それなので少し設定を弄る必要があります。

todo: 後で書く

## トラブルシュート

### ping が通らない
ping を送信する際の送信元に対する経路がない場合、応答が帰ってこないことが多い

特段、以下の場合は CATV事業者からDHCPで受け取ったIPアドレスをソースとして送出してしまっているため、対向ルーターが返送先がデフォルトルートになってしまう。そのため、パケットが迷子になる

そのため、ソースアドレスを指定して ping を送信することを試してみてほしい

1. RT2 src: 10.16.224.3 dst: 10.0.0.1 でパケットを送出
2. RT1 が1で創出されたパケットを受け取る
3. RT1 は 10.16.224.3 へのルートが定義されていないため、デフォルトルートに向かってパケットを送出する
4. RT1 は GigabitEthernet0.1へパケットを送信しているため、 RT2 へ帰りパケットが帰ってこない

```
RT2 (config)# ping 10.0.0.1
PING 10.16.224.3 > 10.0.0.1 56 data bytes

--- 10.0.0.1 ping statistics ---
5 packets transmitted, 0 packets received, 100% packet loss
RT2 (config)# 
RT2 (config)# 
RT2 (config)# ping 10.0.0.1 source 10.0.0.2
PING 10.0.0.2 > 10.0.0.1 56 data bytes
64 bytes from 10.0.0.1: icmp_seq=0 ttl=64 time=10.613 ms
64 bytes from 10.0.0.1: icmp_seq=1 ttl=64 time=10.930 ms
64 bytes from 10.0.0.1: icmp_seq=2 ttl=64 time=11.149 ms
64 bytes from 10.0.0.1: icmp_seq=3 ttl=64 time=10.996 ms
64 bytes from 10.0.0.1: icmp_seq=4 ttl=64 time=10.854 ms

--- 10.0.0.1 ping statistics ---
5 packets transmitted, 5 packets received, 0% packet loss
round-trip (ms)  min/avg/max = 10.613/10.908/11.149
RT2 (config)#
```
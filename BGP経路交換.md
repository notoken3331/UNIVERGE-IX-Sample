## 想定構築ネットワーク

```
  RT1側ネットワーク
          ||
GE1.0 172.16.1.1/24 [RT1 から 広報したいネットワーク]
   < RT1 AS65001 >
Tunnel1.0 10.255.255.1/30
          ||
          ||
Tunnel1.0 10.255.255.2/30
   < RT2 AS65002 > 
GE1.0 172.16.2.1/24 [RT2 から 広報したいネットワーク]
          ||
  RT2側ネットワーク
```

## 設定項目

### AS関連
```
router bgp ${ルータの設定するAS番号}
  neighbor ${対抗ルータのIPアドレス} remote-as ${対抗ルータのAS番号}
  address-family ipv4 unicast
    network ${そのAS番号で広報するCIDR}
```

### I/F関連


- VPN で接続する場合、 CIDRの最小単位は /30 の 4IP アドレスである。それ以下でもそれ以上でも誰も幸せにならないため、設定してはいけない
- ルート交換用NWを構築する場合、必要なIPアドレス数でネットワークを構築する (推奨は /28(16IP)～/24(256IP) くらいのように思える)


```
interface Tunnel${VPNを接続するI/F名}
  ip address ${経路交換に利用するIPアドレス}/30
```


## RT1側設定
```
router bgp 65001
  neighbor  10.255.255.2 remote-as 65002
  address-family ipv4 unicast
    network 172.16.1.0/24

interface Tunnel1.0
  ip address 10.255.255.1/30
```


## RT2側設定
```
router bgp 65002
  neighbor  10.255.255.1 remote-as 65002
  address-family ipv4 unicast
    network 172.16.2.0/24

interface Tunnel1.0
  ip address 10.255.255.2/30
```

## RT1 側 設定確認
```
# ping 10.255.255.2
   対抗ルータへpingが通ることを確認する
# en
(config)# show ip bgp summary
BGP router ID 172.21.1.1, local AS number 64541
22 BGP AS-PATH entries

Neighbor         V    AS    MsgRcvd MsgSent Up/DownTime   State
 10.255.255.2    4    65002   1       1     00:01:00      ESTABLISHED

Total number of neighbors 1
```

## トラブルシュート
State が  ESTABLISHED になっていればOK。BGPの接続までだいたい2分程度かかるが、
その間は IDOL とか CONNECT とか ACTIVE とか表示されるが、問題はない。

2分待ってもBGPのネゴができない場合は、双方のルータの neighbor の設定及び
I/F のIPアドレスの確認をする。


| 事象 | 確認事項 |
| ---- | ---- |
| ping が通るのに、BGPがネゴれない | neighbor |
| ping が通らない                  | I/F の設定が間違っている |
| そもそもトンネルが上がっていない | VPN |


### TIPS VPNの状態を確認する方法

```
# show interfaces Tunnel1.0
Interface Tunnel11.0 is up ← トンネルは上がっているか？
    == 省略 ==
  Encapsulation TUNNEL:
    == 省略 ==
    Statistics:
      20718 packets input, 3645671 bytes, 0 errors    ← 相互にパケットは送受信できているか
      33431 packets output, 3307706 bytes, 0 errors
notoken-tokyo01(config)#
```
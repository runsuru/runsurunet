# Cisco C890ルータ構成自動化（Ansible + ChatGPT）

このリポジトリは、ネットワーク機器（主にCisco C890シリーズ）の構成を **Ansible** を使って自動化するPlaybookとRoleを管理するものです。  
構成や変数設計の大部分は、**ChatGPTを活用しながら作成**しました。

---

## 対象構成

本Playbook（`playbooks/c890_ebgp.yml`）は、以下のようなCisco C890ルータ2台（rt01 / rt02）のBGP構成を自動化対象としています。

- **BGP Peering**：AS65001（rt01）とAS65002（rt02）  
- **VLAN10でDHCP・SSHによる管理**  
- **ルートマップで受信ルートを制御**  
- **各種VLAN・インターフェース設定、ログ転送設定（Syslog）**

---

## 📷 構成図

例：

![rt01 - rt02 ネットワーク構成図](docs/rt01_rt02_network.png)

---

## ディレクトリ構成（`tree` コマンド）

```bash
tree
.
├── inventory/
│   ├── group_vars/
│   │   ├── cisco_c890.yml           # 共通設定（非機密）
│   │   └── cisco_c890_vault.yml     # 機密情報（Vault暗号化済み）
│   ├── host_vars/
│   │   ├── rt01.yml                 # 個別設定（IPなど）
│   │   └── rt02.yml
│   └── hosts.ini
├── playbooks/
│   └── c890_ebgp.yml                # 実行Playbook
├── roles/                           # Roleごとに処理分割
│   ├── common/
│   ├── vlan/
│   ├── interface/
│   ├── ip_prefix_list/
│   ├── route_map/
│   └── bgp/
```

---

## 変数の分離について

- 共通設定は `group_vars/cisco_c890.yml` に記載  
- 個別設定（IPアドレスなど）は `host_vars/rt01.yml` などに分離  
- 機密情報（SSHパスワード、enable passwordなど）は `group_vars/cisco_c890_vault.yml` にVaultで暗号化して管理

---

## ansible-vaultについて
パスワードが入ったファイルは、暗号化するためansible-vaultをしています。

### `cisco_c890_vault.yml` の中身（復号後）

ユーザ名：`runsuru`
パスワードは、`cisco123`になっています。

```yaml
ansible_user: runsuru
ansible_password: cisco123
enable_password: cisco123
peer_password: 045802150C2E1D1C5A
```

---

### 編集方法
パスワードは、`cisco123`で編集できます。
```bash
ansible-vault edit inventory/group_vars/cisco_c890_vault.yml
```

---

### Vaultパスワード自動指定（推奨）

```bash
echo 'cisco123' > ~/.vault_pass.txt
chmod 600 ~/.vault_pass.txt
```

---

## Playbook

### Playbook 実行の前に
ユーザ名：`runsuru`、パスワードは、`cisco123`でsshできる状態にしてください。

<details>
<summary>最低限必要な設定</summary>

```
conf t
hostname rt02
interface Vlan10
  ip address dhcp
  no shutdown

interface FastEthernet0
 switchport access vlan 10
 no shutdown

ip domain-name runsuru.local
crypto key generate rsa modulus 2048
username runsuru privilege 15 secret 5 $1$lZ1Y$se4163qx3dPgaBkASsalo1
ip ssh version 2
line vty 0 4
  login local
  transport input ssh
end
```
</details>


### Playbookの実行方法
```bash
ansible-playbook -i inventory/hosts.ini playbooks/c890_ebgp.yml   --vault-password-file ~/.vault_pass.txt
```

### オプション：

- `-l rt01` などで対象ホストを限定実行可能  
- `-k -u runsuru` を指定して明示ログインも可能（SSHパスワード方式の場合）

## 投入後のrt01とrt02のコンフィグ

BGPの状態
```
rt01#show ip bgp summary
BGP router identifier 1.1.1.1, local AS number 65001
BGP table version is 5, main routing table version 5
2 network entries using 288 bytes of memory
2 path entries using 160 bytes of memory
3/2 BGP path/bestpath attribute entries using 480 bytes of memory
1 BGP AS-PATH entries using 24 bytes of memory
0 BGP route-map cache entries using 0 bytes of memory
0 BGP filter-list cache entries using 0 bytes of memory
BGP using 952 total bytes of memory
BGP activity 3/1 prefixes, 3/1 paths, scan interval 60 secs

Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
192.168.12.2    4        65002     104     104        5    0    0 00:08:29        1
```

### rt01
<details>
<summary>rt01</summary>

```
Building configuration...

Current configuration : 2455 bytes
!
! Last configuration change at 12:05:29 UTC Wed Feb 6 2036 by runsuru
!
version 15.4
service timestamps debug datetime msec
service timestamps log datetime msec
no service password-encryption
!
hostname rt01
!
boot-start-marker
boot-end-marker
!
!
enable secret 5 $1$g9XX$AM5nD424ZcDOK9oiB3VIc.
!
no aaa new-model
!
!
!
!
!
!
!
!
!
!
!
!
!
!
!


!
!
!
!
no ip domain lookup
ip domain name runsuru.local
ip cef
no ipv6 cef
!
!
!
!
!
multilink bundle-name authenticated
!
!
!
!
!
!
!
cts logging verbose
license udi pid CISCO892-K9 sn FGL154123NR
!
!
vtp mode transparent
username runsuru privilege 15 secret 5 $1$lZ1Y$se4163qx3dPgaBkASsalo1
!
redundancy
!
!
!
!
!
vlan 10,20,30
!
!
!
!
!
!
!
!
!
!
!
interface Loopback0
 ip address 1.1.1.1 255.255.255.255
!
interface BRI0
 no ip address
 encapsulation hdlc
 shutdown
 isdn termination multidrop
!
interface FastEthernet0
 switchport access vlan 10
 no ip address
!
interface FastEthernet1
 no ip address
 shutdown
!
interface FastEthernet2
 no ip address
 shutdown
!
interface FastEthernet3
 no ip address
 shutdown
!
interface FastEthernet4
 no ip address
 shutdown
!
interface FastEthernet5
 no ip address
 shutdown
!
interface FastEthernet6
 no ip address
 shutdown
!
interface FastEthernet7
 no ip address
 shutdown
!
interface FastEthernet8
 no ip address
 shutdown
 duplex auto
 speed auto
!
interface GigabitEthernet0
 ip address 192.168.12.1 255.255.255.252
 duplex auto
 speed auto
!
interface Vlan1
 no ip address
!
interface Vlan10
 description "### for ssh
 ip address dhcp
!
router bgp 65001
 bgp log-neighbor-changes
 network 1.1.1.1 mask 255.255.255.255
 neighbor PEERS peer-group
 neighbor PEERS remote-as 65002
 neighbor PEERS description BGP peer group
 neighbor PEERS password 7 045802150C2E1D1C5A
 neighbor PEERS timers 5 15
 neighbor PEERS route-map RM_FROM_rt02 in
 neighbor 192.168.12.2 peer-group PEERS
!
ip forward-protocol nd
no ip http server
no ip http secure-server
!
!
ip ssh version 2
!
!
ip prefix-list FROM_rt02_ONLY seq 5 permit 2.2.2.2/32
logging host 10.1.100.250
!
route-map RM_FROM_rt02 permit 10
 match ip address prefix-list FROM_rt02_ONLY
 set local-preference 1000
!
!
!
control-plane
!
!
mgcp behavior rsip-range tgcp-only
mgcp behavior comedia-role none
mgcp behavior comedia-check-media-src disable
mgcp behavior comedia-sdp-force disable
!
mgcp profile default
!
!
!
!
!
!
!
!
line con 0
line aux 0
line vty 0 4
 login local
 transport input ssh
!
!
end
```
</details>

### rt02
<details>
<summary>rt02</summary>

```
Building configuration...

Current configuration : 2617 bytes
!
! Last configuration change at 03:10:04 UTC Thu Feb 7 2036 by runsuru
!
version 15.4
service timestamps debug datetime msec
service timestamps log datetime msec
no service password-encryption
!
hostname rt02
!
boot-start-marker
boot-end-marker
!
!
enable secret 5 $1$lS32$m2WD/ZywaEj1kNmokoTeS/
!
no aaa new-model
!
!
!
!
!
!
!
!
!
!
!
!
!
!
!


!
!
!
!
no ip domain lookup
ip domain name runsuru.local
ip cef
no ipv6 cef
!
!
!
!
!
multilink bundle-name authenticated
!
!
!
!
!
!
!
cts logging verbose
license udi pid CISCO892-K9 sn FGL15462182
!
!
vtp mode transparent
username runsuru privilege 15 secret 5 $1$lZ1Y$se4163qx3dPgaBkASsalo1
!
redundancy
!
!
!
!
!
vlan 2,10,20,30
!
vlan 101
 name OSPF
!
vlan 102
 name static
!
vlan 104
 name bgp02
!
vlan 110
 name test-1
!
vlan 111
 name test-2
!
vlan 901
 name rip
!
vlan 999
 name mgmt
!
!
!
!
!
!
!
!
!
!
!
interface Loopback0
 ip address 2.2.2.2 255.255.255.255
!
interface BRI0
 no ip address
 encapsulation hdlc
 shutdown
 isdn termination multidrop
!
interface FastEthernet0
 switchport access vlan 10
 no ip address
!
interface FastEthernet1
 no ip address
 shutdown
!
interface FastEthernet2
 no ip address
 shutdown
!
interface FastEthernet3
 no ip address
 shutdown
!
interface FastEthernet4
 no ip address
 shutdown
!
interface FastEthernet5
 no ip address
 shutdown
!
interface FastEthernet6
 no ip address
 shutdown
!
interface FastEthernet7
 no ip address
 shutdown
!
interface FastEthernet8
 no ip address
 shutdown
 duplex auto
 speed auto
!
interface GigabitEthernet0
 ip address 192.168.12.2 255.255.255.252
 duplex auto
 speed auto
!
interface Vlan1
 no ip address
!
interface Vlan10
 description "### for ssh
 ip address dhcp
!
router bgp 65002
 bgp log-neighbor-changes
 network 2.2.2.2 mask 255.255.255.255
 neighbor PEERS peer-group
 neighbor PEERS remote-as 65001
 neighbor PEERS description BGP peer group
 neighbor PEERS password 7 045802150C2E1D1C5A
 neighbor PEERS timers 5 15
 neighbor PEERS route-map RM_FROM_rt01 in
 neighbor 192.168.12.1 peer-group PEERS
!
ip forward-protocol nd
no ip http server
no ip http secure-server
!
!
ip ssh version 2
!
!
ip prefix-list FROM_rt01_ONLY seq 5 permit 1.1.1.1/32
logging host 10.1.100.250
!
route-map RM_FROM_rt01 permit 10
 match ip address prefix-list FROM_rt01_ONLY
 set local-preference 1000
!
!
!
control-plane
!
!
mgcp behavior rsip-range tgcp-only
mgcp behavior comedia-role none
mgcp behavior comedia-check-media-src disable
mgcp behavior comedia-sdp-force disable
!
mgcp profile default
!
!
!
!
!
!
!
!
line con 0
line aux 0
line vty 0 4
 login local
 transport input ssh
!
!
end
```
</details>



---

## 注意事項（Disclaimer）

⚠️ 本Playbookの使用によって発生した損害・障害などについて、作者は一切の責任を負いません。  
⚠️ *Use at your own risk. The author is not responsible for any damage or failure caused by using this Playbook.*


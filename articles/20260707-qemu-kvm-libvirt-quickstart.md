---
title: "QEMU/KVM + libvirt 仮想化クイックガイド"
emoji: "🧰"
type: "tech"
topics: ["linux", "libvirt", "kvm", "qemu", "仮想化"]
published: false
---

## はじめに

最近、仮想化について学習しています。
しかし、専用の書籍などが見当たらずとっつきづらい印象があります。
そこで、まず仮想マシン（以下VM）を1台起動することを目的としたクイックガイドを書いてみました。

ゲストOSにはUbuntu 26.04を利用します。そのため、VMを開発環境として利用することもできるはずです。

ちなみに、クイックガイドといいながらVM起動への最短経路ではありません。
しかし、VMを作って終わりにならないよう、実際に使えるスペックのVMを用意できるように意識しています！

秘密基地を作るような感覚で楽しんでもらえたらうれしいです。

## 検証環境

ホストOSはLinuxです。
CPUはx86-64アーキテクチャです。ARMアーキテクチャのPCでは、この記事の手順にしたがうとVM作成が失敗します。

Windowsの方はWSL2でも大丈夫です。
ただし、複数のディストリビューションを同時に起動しているとネットワーク関連の不具合が発生したため、1つだけ起動した状態にしておくことをおすすめします。

私は以下のホストOSで実践しました。

- Ubuntu 26.04
  - libvirt: 12.0.0
  - QEMU: QEMU emulator version 10.2.1
- Fedora 44
  - libvirt: 12.0.0
  - QEMU: QEMU emulator version 10.2.2

## KVMが使えるか確認する

まず、ホストとなるコンピュータのCPUがハードウェア仮想化支援機能（Intel VT-x / AMD-V）を提供しているか確認します。

```bash
$ egrep -o 'vmx|svm' /proc/cpuinfo | sort -u
vmx
```

`vmx`（Intel）か`svm`（AMD）のいずれかが出力されれば問題ありません。
何も出力されない場合は、BIOS/UEFIで仮想化支援機能が無効になっている可能性がありますので、有効化してください（申し訳ありませんが、ここは省略します…）。

## パッケージをインストールする

仮想化に必要なパッケージをインストールします。

```bash
# Ubuntu 26の場合
sudo apt update
sudo apt install qemu-system-x86 libvirt-daemon-system libvirt-clients virtinst ovmf

# Fedora 44の場合
sudo dnf check-update
sudo dnf install @virtualization
```

インストール後、libvirtdを有効化し、自分を`libvirt`・`kvm`グループに追加します。

```bash
sudo systemctl enable --now libvirtd
sudo usermod -aG libvirt,kvm $USER
```

ここで、所属グループの変更を反映させるためホストOSから一度ログアウトし、再ログインしてください。
再ログイン後、所属グループを確認します。

```bash
$ id -nG
# Ubuntu 26の場合
adm cdrom sudo dip plugdev users libvirt kvm

# Fedora 44の場合
wheel libvirt kvm
```

## 作業環境を整備する

まず、以下のコマンドによって、virshの接続モードをシステム全体（`qemu:///system`）に設定します。

```bash
echo 'export LIBVIRT_DEFAULT_URI="qemu:///system"' >> ~/.bashrc
source ~/.bashrc
virsh uri   # qemu:///system になっているか確認
```

その後、作業ディレクトリを作成し、移動します。
以降のコマンド例は、すべてこのディレクトリの中で実行する想定です（任意のディレクトリで構いません）。

```bash
mkdir -p ~/qemu-kvm-quickstart
cd ~/qemu-kvm-quickstart
```

## ネットワークを確認する

VM用のネットワークは自由に定義できます。
今回は、libvirtパッケージインストール時に定義される`default`というNATネットワークを使いましょう。
`default`ネットワークは、自動的に`active`かつ`autostart`の状態になっています。
念のため確認しておきましょう。

```bash
$ virsh net-list --all
 Name      State    Autostart   Persistent
--------------------------------------------
 default   active   yes         yes
```

もし`State`が`inactive`になっていた場合は、起動しましょう。

```bash
$ virsh net-start default
Network default started

$ virsh net-autostart default
Network default marked as autostarted
```

## ストレージプールを作る

VMのディスクイメージを管理する置き場所として、libvirtの「ストレージプール」を作成します。
ここでは`default`という名前で、`/var/lib/libvirt/images`ディレクトリを使うプールを作成します。

> 先ほどの`default`ネットワークとは違い、この`default`プールはパッケージをインストールしただけでは自動的に作られません。
> `/var/lib/libvirt/images`ディレクトリ自体はパッケージ側で用意されますが、それを指すプールの定義は`virt-install`などがVM作成時にプール未指定の場合に遅延生成する仕組みになっています。

```bash
$ virsh pool-define-as default dir --target /var/lib/libvirt/images
Pool default defined

$ virsh pool-build default
Pool default built

$ virsh pool-start default
Pool default started

$ virsh pool-autostart default
Pool default marked as autostarted
```

```bash
$ virsh pool-list --all
 Name      State    Autostart
-----------------------------
 default   active   yes
```

各コマンドの意味は本記事末尾の付録にまとめています。

## Ubuntu 26.04 LTSクラウドイメージを取得してプールに登録する
### Ubuntu 26.04 LTSクラウドイメージを取得する

動作確認用のゲストOSには、検証環境のホストOSと合わせてUbuntu 26.04 LTSの公式クラウドイメージを使います。Ubuntu Cloud Imagesから直接ダウンロードします。

> クラウドイメージとはクラウド環境や仮想環境での起動を前提としたディスクイメージです。
> インストーラーを介してセットアップするのではなく、そのままVMのディスクとして使える状態で配布されています。
> ファイルの実サイズが 819 MiBほどあります！

```bash
$ curl -L -O https://cloud-images.ubuntu.com/releases/26.04/release/ubuntu-26.04-server-cloudimg-amd64.img
```

ダウンロードしたファイルの情報を確認します。

```bash
$ qemu-img info ubuntu-26.04-server-cloudimg-amd64.img
image: ubuntu-26.04-server-cloudimg-amd64.img
file format: qcow2
virtual size: 3.5 GiB (3758096384 bytes)
disk size: 819 MiB
cluster_size: 65536
Format specific information:
    compat: 1.1
    compression type: zlib
    lazy refcounts: false
    refcount bits: 16
    corrupt: false
    extended l2: false
Child node '/file':
    filename: ubuntu-26.04-server-cloudimg-amd64.img
    protocol type: file
    file length: 819 MiB (859024384 bytes)
    disk size: 819 MiB
```
クラウドイメージには、あらかじめ組み込まれたユーザーやパスワードがありません。
初回起動時に「cloud-init」という仕組みによって、ユーザー作成やホスト名設定などを行う前提で作られています。

### cloud-initのシードイメージを作る

cloud-initはそれ単体でかなり奥が深いものです。しかし、ここでは「VMの初期設定を行う仕組み」という理解で問題ありません。

ローカル環境でcloud-initを実現するには、次の2つの設定ファイルが必要です。

- `user-data`: ユーザー名やパスワードなどを記述するファイル
- `meta-data`: VMインスタンスの識別情報を記述するファイル

この2つのファイルをもとに**シードイメージ**と呼ばれるファイルを作成し、VMにCD-ROMとして渡すことで、VMの初期設定が行われる仕組みです。
シード=seed（種）の意味ですね。

まず`user-data`ファイルを作成し、以下の内容を貼付します。一行目の#cloud-configを消すと識別してくれないので注意してください。
ここで設定した`ubuntu`ユーザーとパスワードで、この後VMにSSH接続します。

`user-data`:
```yaml
#cloud-config
hostname: quickstart-vm
users:
  - name: ubuntu
    plain_text_passwd: 'ChangeMe123!'
    lock_passwd: false
    sudo: ALL=(ALL) NOPASSWD:ALL
    shell: /bin/bash
    # 実運用ではパスワードの代わりにSSH公開鍵を登録するのが安全です
    # ssh_authorized_keys:
    #   - ssh-ed25519 AAAA... your-key-comment
chpasswd:
  expire: false
ssh_pwauth: true
```

続いて、`meta-data`ファイルを作成し、以下の内容を貼付します:

`meta-data`:
```yaml
instance-id: quickstart-vm
local-hostname: quickstart-vm
```

この2つのファイルをもとに、シードイメージを作成します。
`cloud-localds`コマンドを使います。

```bash
# Ubuntu の場合
$ sudo apt install cloud-image-utils

# Fedora の場合
$ sudo dnf install cloud-utils

$ cloud-localds seed.img user-data meta-data
```

`seed.img`というファイルが同ディレクトリに作成されているかと思います。

```bash
$ ls .
meta-data  seed.img  ubuntu-26.04-server-cloudimg-amd64.img  user-data
```

このファイルが、シードイメージです。

### ディスクイメージをプールに登録する

ここまでの手順で、ディスクイメージとシードイメージのファイルを用意できました。
これらのファイルを「ボリューム」としてlibvirtのストレージプールに登録します。
「ボリューム」は、VMに差し込むSSDだとイメージするとわかりやすいかもしれません。

手順としては、以下の通りです。

1. ボリュームを`default`プールに作成する。→ SSDを用意する。
2. ボリュームにディスクイメージの中身を流し込む。→ SSDにOSをインストールする。

ちなみに、これからqcow2形式のボリュームを作成する際に論理的なサイズを指定しますが、現段階ではディスクイメージの物理的なサイズである 819 MiBしか消費されません。
qcow2形式のボリュームは、使った分だけ実ディスクを消費するからです。

```bash
# ボリュームを`default`プールに作成する
$ virsh vol-create-as default ubuntu-2604.qcow2 3758096384 --format qcow2
Vol ubuntu-2604.qcow2 created

# ボリュームにディスクイメージの中身を流し込む
$ virsh vol-upload --pool default ubuntu-2604.qcow2 ubuntu-26.04-server-cloudimg-amd64.img
```

クラウドイメージはルートファイルシステムの空き容量が小さいため、開発環境として利用する場合には容量が足りないかもしれません。
`vol-resize`でボリューム容量を10 GiBまで拡張しておきます。


```bash
# ボリュームを 10 GiBまで対応可能にする
$ virsh vol-resize --pool default ubuntu-2604.qcow2 10G
Size of volume 'ubuntu-2604.qcow2' successfully changed to 10 GiB
```

`default`プールにボリュームが登録できたか、確認します。

```bash
$ virsh vol-list default
 Name                Path
--------------------------------------------------------------
 ubuntu-2604.qcow2   /var/lib/libvirt/images/ubuntu-2604.qcow2

$ virsh vol-info --pool default ubuntu-2604.qcow2
Name:           ubuntu-2604.qcow2
Type:           file
Capacity:       10.00 GiB
Allocation:     819.24 MiB
```

`vol-create-as`で確保したボリューム容量（`Capacity`）は`vol-resize`によって10 GiBに拡張されている一方、ホスト上で実際に消費しているサイズ（`Allocation`）はディスクイメージのサイズ程度です。

シードイメージ（`seed.img`）も同様にプールへ登録しておきます。

```bash
$ virsh vol-create-as default seed.img $(stat -c%s seed.img) --format raw
Vol seed.img created

$ virsh vol-upload --pool default seed.img seed.img
```

登録できたか確認します。

```bash
$ virsh vol-list default
 Name                Path
----------------------------------------------------------------
 seed.img            /var/lib/libvirt/images/seed.img
 ubuntu-2604.qcow2   /var/lib/libvirt/images/ubuntu-2604.qcow2
```

## VMを定義して起動する

いよいよ、VMを作成していきます。

まずはVMの定義です。
VMの定義とは、どれくらいリソースを割り当てるか（CPUやメモリなど）、どんなデバイスを利用するか（ストレージディスクやNICなど）、その他もろもろ細かく指定することです。

VMの定義にはXMLファイルを使います。Domain XML（ドメインXML）などと呼びます。

今回は私が以下のドメインXMLを用意しました。モダンな構成をとりいれています。
自作PCを組むようなイメージで、ここを凝るのは面白いと思います。

`quickstart-vm.xml`:

```xml
<domain type='kvm'>
  <name>quickstart-vm</name>
  <memory unit='KiB'>2097152</memory>
  <currentMemory unit='KiB'>2097152</currentMemory>
  <vcpu placement='static'>2</vcpu>
  <cpu mode='host-passthrough' check='none'></cpu>
  <os firmware='efi'>
    <type arch='x86_64' machine='q35'>hvm</type>
    <firmware>
      <feature enabled='yes' name='secure-boot'/>
      <feature enabled='yes' name='enrolled-keys'/>
    </firmware>
    <boot dev='hd'/>
  </os>
  <features>
    <acpi/>
    <apic/>
    <smm state='on'/>
  </features>
  <pm>
    <suspend-to-mem enabled='no'/>
    <suspend-to-disk enabled='no'/>
  </pm>
  <devices>
    <emulator>/usr/bin/qemu-system-x86_64</emulator>
    <disk type='file' device='disk'>
      <driver name='qemu' type='qcow2' discard='unmap' detect_zeroes='unmap'/>
      <source file='/var/lib/libvirt/images/ubuntu-2604.qcow2'/>
      <target dev='vda' bus='virtio'/>
    </disk>
    <disk type='file' device='cdrom'>
      <driver name='qemu' type='raw'/>
      <source file='/var/lib/libvirt/images/seed.img'/>
      <target dev='sda' bus='sata'/>
      <readonly/>
    </disk>
    <interface type='network'>
      <source network='default'/>
      <model type='virtio'/>
    </interface>
    <serial type='pty'>
      <target type='isa-serial' port='0'>
        <model name='isa-serial'/>
      </target>
    </serial>
    <console type='pty'>
      <target type='serial' port='0'/>
    </console>
    <channel type='unix'>
      <target type='virtio' name='org.qemu.guest_agent.0'/>
    </channel>
    <rng model='virtio'>
      <backend model='random'>/dev/urandom</backend>
    </rng>
    <memballoon model='virtio'/>
  </devices>
</domain>
```

メモリ2048 MiB・vCPU 2という構成は、Ubuntuをフル機能で動かすための最低限の目安です。もっと追加してもいいと思います（メモリ 8192 MiB・vCPU 4くらいがおすすめです）。

devicesタグに囲まれた部分が重要です。
上から順に、ディスクイメージ、シードイメージ、ネットワークが、今まで用意してきたものに設定されています。
この設定によって、ホストOS上の`qemu-system-x86_64`プロセスがこれらのデバイスのふりをして、VMに仮想デバイスを提供してくれるようになるのです。

他にもいくつかこだわりがあります。あまり気にしなくていいです。
- `cpu`には`host-passthrough`を指定する。
- マシンタイプは`q35`とする。マシンタイプとはいわゆるチップセットのこと。
- 起動方式はBIOSではなくUEFIとする。
- Secure Bootを有効化する。
などなど

このXMLを定義し、起動します。

```bash
$ virsh define quickstart-vm.xml
Domain 'quickstart-vm' defined from quickstart-vm.xml

$ virsh start quickstart-vm
Domain 'quickstart-vm' started
```

起動できたか確認します。

```bash
$ virsh list --all
 Id   Name            State
--------------------------------
 1    quickstart-vm   running

$ virsh domiflist quickstart-vm
 Interface   Type      Source    Model    MAC
-------------------------------------------------------------
 vnet0       network   default   virtio   52:54:00:63:a9:95
```

`default`ネットワークにはDHCPサーバーが立っているので、VM起動後しばらくするとIPアドレスが割り当てられます。

```bash
$ virsh net-dhcp-leases default
 Expiry Time           MAC address         Protocol   IP address           Hostname        Client ID or DUID
-----------------------------------------------------------------------------------------------------------------------------------------------------
 2026-07-08 13:49:37   52:54:00:63:a9:95   ipv4       192.168.122.155/24   quickstart-vm   ff:32:39:f9:b5:00:02:00:00:ab:11:10:18:02:d2:4d:b5:29:08
```

`virsh domiflist`で見たMACアドレスにIPアドレスが割り当てられていれば、VMは正常にネットワークへ接続できています。

VMが起動しました！🎉

※もしかしたら、`virsh start quickstart-vm`で以下のエラーが出た方がいるかもしれません。

```bash
$ virsh start quickstart-vm
error: Failed to start domain 'quickstart-vm'
error: Cannot get interface MTU on 'virbr0': No such device
```

その際は、ネットワークを再起動してから再度お試しください。

```bash
virsh net-destroy default
virsh net-start default
ip a   # virbr0 が出現するか確認
virsh start quickstart-vm
```

## VMにSSH接続する

最後に、先ほど確認したIPアドレスへSSH接続して中身を覗いてみます。
ユーザー名・パスワードは、cloud-initの`user-data`で指定した`ubuntu`/`ChangeMe123!`です。

起動処理に数十秒かかるので、起動直後は`Connection refused`になることがあります。
少し待ってからリトライしましょう。

```bash
$ ssh ubuntu@192.168.122.155
The authenticity of host '192.168.122.155 (192.168.122.155)' can't be established.
ED25519 key fingerprint is: SHA256:g6c+lQvDECzxfmOAHmdjwhQROiaqq7q+Gwh9FIswJyI
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.122.155' (ED25519) to the list of known hosts.
ubuntu@192.168.122.155's password:
Welcome to Ubuntu 26.04 LTS (GNU/Linux 7.0.0-27-generic x86_64)

ubuntu@quickstart-vm:~$ ip -4 addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: ens2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    altname enp0s2
    altname enx52540063a995
    inet 192.168.122.155/24 metric 100 brd 192.168.122.255 scope global dynamic ens2
       valid_lft 3426sec preferred_lft 3426sec
```

ゲストOSのシェルプロンプトが`ubuntu@quickstart-vm`となっていますね。
これは、`user-data`の`hostname: quickstart-vm`が適用されたためです。

このように、VMの名前とゲストOS内のホスト名を一致させると管理しやすいです。

これでVMの起動・SSH接続まで一通り確認できました。

## 後片付け

検証が終わりました。
VMの操作には`virsh start, shutdown, destroy`などのコマンドが利用できます。

VMを操作する:
```bash
# 停止中のVMを起動する
virsh start quickstart-vm

# VMを正常にシャットダウンする、数十秒かかることもある
virsh shutdown quickstart-vm

# VMを強制停止する、電源ケーブルを抜くイメージ
virsh destroy quickstart-vm

# 現在の状態を確認する
virsh list --all
```

VMとボリュームを削除したい方は以下のコマンドをご利用ください。
`default`プール自体はlibvirtの標準的なプールとして他のVM作成でも使われるため、削除せずに残しています。

VMとボリュームを削除する:
```bash
virsh destroy quickstart-vm
virsh undefine quickstart-vm
virsh vol-delete --pool default ubuntu-2604.qcow2
virsh vol-delete --pool default seed.img
rm ubuntu-26.04-server-cloudimg-amd64.img seed.img user-data meta-data
```

## まとめ

とりあえずVM起動まで実践してみましたが、あとの楽しみ方は自由です。
新しいVMを作成したい場合「Ubuntu 26.04 LTSクラウドイメージを取得してプールに登録する」の章からの手順を繰り返すことでVMを作成できると思います。

さまざまなOSや、異なるVM定義など、ぜひ試してみてください。
私もいろいろ試しているところです！

## 付録：ストレージプール・ボリューム関連コマンドの意味

本編で使ったストレージプール・ボリューム関連のコマンドを、まとめて解説します。

### ストレージプールとストレージボリューム

libvirtは、VMのディスクの置き場所を「ストレージプール」と「ストレージボリューム」という2つの階層に分けて管理しています。

- **ストレージプール（Storage Pool）**: VMのディスクファイルを格納する「置き場所」です。本編ではディレクトリ型のプールとして`/var/lib/libvirt/images`を`default`という名前で登録しました。
- **ストレージボリューム（Storage Volume）**: プールの中に配置される、VM 1台分のストレージです。本編の`ubuntu-2604.qcow2`がこれにあたります。

プールはボリュームの置き場所、ボリュームは各VMのストレージ、という関係です。

### 各コマンドの意味

- `virsh pool-define-as <name> dir --target <path>`: プールを定義するだけで、ディレクトリはまだ作られません
- `virsh pool-build <name>`: 定義したプールの実体となるディレクトリを実際に作成します（既存ディレクトリの場合はパーミッションの確認のみ）
- `virsh pool-start <name>`: プールを使える状態（`active`）にします
- `virsh pool-autostart <name>`: ホスト（libvirtd）起動時に自動でプールを開始するようにします
- `virsh pool-list --all`: 定義済みのプール一覧と状態（`active`/`inactive`）、`autostart`設定を確認します
- `virsh vol-create-as <pool> <name> <size> --format qcow2`: プール内に指定サイズの空のボリュームを確保します
- `virsh vol-upload --pool <pool> <name> <file>`: ローカルファイルの中身をボリュームに転送します
- `virsh vol-resize --pool <pool> <name> <size>`: ボリュームの容量（`Capacity`）を変更します。qcow2形式ならデータを保持したまま拡張できます
- `virsh vol-list <pool>` / `virsh vol-info --pool <pool> <name>`: ボリューム一覧や、容量・実使用量を確認します

### raw形式とqcow2形式の違い

VMのディスクファイル形式にはいくつか種類がありますが、本編で使った`qcow2`（QEMU Copy-On-Write 2）と、もっとも単純な`raw`形式の違いを押さえておくと理解が深まります。

- **raw形式**: 構造が単純でオーバーヘッドが少なく高速ですが、指定したサイズ（例: 10 GiB）を最初からホストのディスク上でそのまま消費します
- **qcow2形式**: シンプロビジョニングに対応しており、実際に書き込まれたデータの分だけホスト側のファイルサイズが増えていきます。本編の`vol-info`で`Capacity`（論理サイズ）と`Allocation`（実際の消費量）が一致していなかったのはこのためです。スナップショット機能もqcow2の大きな利点です

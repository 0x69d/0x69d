---
title: "Claude Desktop on Linux (beta)からVMにSSH接続する"
emoji: "🔗"
type: "tech"
topics: ["linux", "libvirt", "claude", "claudecode", "仮想化"]
published: true
---

## はじめに

Claude Desktop on Linux (beta)がリリースされ、Ubuntuで使えるようになりました。
Claude DesktopのClaude Codeは使いやすくて、かなり好きです。

今回は、QEMU/KVM上に立てたVMに対して、Claude DesktopからSSH接続してみます。

IPアドレス指定ではなく、VMの名前でSSH接続できるようにすることが今回のポイントです。

## 検証環境

- ホストOS: Ubuntu 26.04
- Claude Desktop: 1.18286.2

## Claude Desktop on Linux (beta)をインストールする

公式ドキュメント「[Claude Desktop on Linux (beta)](https://code.claude.com/docs/ja/desktop-linux)」の手順に沿って、Claude DesktopをAnthropicのaptリポジトリからインストールします。

Anthropicの署名キーを取得し、aptリポジトリを登録します。

```bash
sudo curl -fsSLo /usr/share/keyrings/claude-desktop-archive-keyring.asc https://downloads.claude.ai/claude-desktop/key.asc
echo "deb [arch=amd64,arm64 signed-by=/usr/share/keyrings/claude-desktop-archive-keyring.asc] https://downloads.claude.ai/claude-desktop/apt/stable stable main" | sudo tee /etc/apt/sources.list.d/claude-desktop.list
```

パッケージをインストールします。

```bash
sudo apt update && sudo apt install claude-desktop
```

ターミナルから`claude-desktop`を実行し、サインインします。

`dpkg -l claude-desktop`でインストール済みバージョンを確認できます。

```bash
$ dpkg -l claude-desktop
ii  claude-desktop 1.18286.2    amd64        Desktop application for Claude.ai
```

## SSH鍵認証のVMを作成する

Claude Desktopの「SSHホストを追加」機能では、SSH鍵による認証が前提となっているようです。
そのため、VMへのアクセスはcloud-initを利用してSSH鍵認証のみ有効化しておきます。

VM作成の基本的な流れは前回記事「[QEMU/KVM + libvirt 仮想化クイックガイド](https://zenn.dev/0x69d/articles/20260707-qemu-kvm-libvirt-quickstart)」と同じです。
クラウドイメージのダウンロードなど前提となる手順がまだの方は、先にこちらをご参照ください。

まず、接続用のSSH鍵を作成します。

```bash
ssh-keygen -t ed25519
```

デフォルトのパス（`~/.ssh/id_ed25519`）のまま保存します。すでに同名の鍵がある場合は、上書きせずそのまま既存の鍵を利用して大丈夫です。

続いて、SSH鍵認証のVMを作成します。
今回は`ubuntu2604-001`という名前にします。

`user-data`:
```yaml
#cloud-config
hostname: ubuntu2604-001
users:
  - name: ubuntu
    lock_passwd: true
    sudo: ALL=(ALL) NOPASSWD:ALL
    shell: /bin/bash
    ssh_authorized_keys:
      - ssh-ed25519 AAAA... your-key-comment
ssh_pwauth: false
```

`ssh_authorized_keys`には、`cat ~/.ssh/id_ed25519.pub`で表示される公開鍵の内容をそのまま貼り付けてください。
パスワードは設定せず`ssh_pwauth: false`でパスワード認証自体を無効化します。

続いて`meta-data`も作成します。

`meta-data`:
```yaml
instance-id: ubuntu2604-001
local-hostname: ubuntu2604-001
```

`user-data`と`meta-data`からシードイメージを作成します。

```bash
cloud-localds ubuntu2604-001-seed.img user-data meta-data
```

ディスクイメージとシードイメージを、前回記事と同じ要領で`default`ストレージプールにボリュームとして登録します。

```bash
# ディスクボリュームを作成し、クラウドイメージの中身を流し込む
virsh vol-create-as default ubuntu2604-001.qcow2 3758096384 --format qcow2
virsh vol-upload --pool default ubuntu2604-001.qcow2 ubuntu-26.04-server-cloudimg-amd64.img
virsh vol-resize --pool default ubuntu2604-001.qcow2 10G

# シードイメージのボリュームを作成し、アップロードする
virsh vol-create-as default ubuntu2604-001-seed.img $(stat -c%s ubuntu2604-001-seed.img) --format raw
virsh vol-upload --pool default ubuntu2604-001-seed.img ubuntu2604-001-seed.img
```

ドメインXMLも前回記事のものを転用します。`name`と、2つの`disk`の`source file`を`ubuntu2604-001`用に変更しているだけで、それ以外は同じ構成です。

`ubuntu2604-001.xml`:
```xml
<domain type='kvm'>
  <name>ubuntu2604-001</name>
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
      <source file='/var/lib/libvirt/images/ubuntu2604-001.qcow2'/>
      <target dev='vda' bus='virtio'/>
    </disk>
    <disk type='file' device='cdrom'>
      <driver name='qemu' type='raw'/>
      <source file='/var/lib/libvirt/images/ubuntu2604-001-seed.img'/>
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
    <graphics type='vnc' port='-1' autoport='yes' listen='127.0.0.1'/>
    <video>
      <model type='virtio'/>
    </video>
    <memballoon model='virtio'/>
  </devices>
</domain>
```

このXMLを定義し、起動します。

```bash
virsh define ubuntu2604-001.xml
virsh start ubuntu2604-001
```

これで`ubuntu2604-001`という名前のVMが、SSH鍵認証で接続できる状態になりました。

## VMをドメイン名で名前解決できるようにする

デフォルトでは、VMへSSH接続するにはIPアドレスを指定する必要があります。
ここでは、libvirtが提供するNSSモジュールを使い、VMの名前（domain XMLの`<name>`）でそのまま名前解決できるようにします。

まず、`libnss-libvirt`パッケージをインストールします。

```bash
sudo apt install libnss-libvirt
```

続いて、`/etc/nsswitch.conf`の`hosts:`行に`libvirt`、`libvirt_guest`を追加します。

```
hosts:          files mdns4_minimal [NOTFOUND=return] mymachines libvirt libvirt_guest dns
```

> 内容が環境によって異なるかもしれませんが、`dns`の直前に追加しておけば問題ありません。

`getent hosts`コマンドで、VM名からIPアドレスを解決できるか確認します。

```bash
$ getent hosts ubuntu2604-001
192.168.122.165 ubuntu2604-001
```

> VMが起動していてIPアドレスが割り当て済みである必要があります。

## Claude DesktopでSSHホストを追加する

Claude Desktop内のClaude Codeを開きましょう。
「ローカル」と書かれたボタンをクリックすると、「SSH」の項目に「SSHホストを追加...」と表示されています。

![Claude Desktopの環境選択メニュー](/images/20260709-claude-desktop-linux-vm-ssh/claude-desktop-01.png)

クリックすると「SSH接続を追加」というダイアログが開くので、以下のように入力します。

- 名前: `ubuntu2604-001`
- SSHホスト: `ubuntu@ubuntu2604-001`
- SSHポート: `22`
- IDファイル（秘密鍵）: `~/.ssh/id_ed25519`

![SSH接続を追加ダイアログ](/images/20260709-claude-desktop-linux-vm-ssh/claude-desktop-02.png)

「SSH接続を追加」をクリックすると接続が完了し、「ローカル」だったボタンが`ubuntu2604-001`に変更されているはずです。

![接続後の環境タブ](/images/20260709-claude-desktop-linux-vm-ssh/claude-desktop-03.png)

実際にタスクを投げてみると、VM上でClaude Codeが動作していることがわかります。

![VM上でのClaude Code実行例](/images/20260709-claude-desktop-linux-vm-ssh/claude-desktop-04.png)

成功です。🎉

## まとめ

VMを作成し、Claude Desktopから名前解決とSSH鍵認証だけで接続できるようになりました。
これで、IPアドレスの確認やパスワード入力なしに、VMにすぐアクセスできます。

## 参考

- [Claude Desktop on Linux (beta)](https://code.claude.com/docs/ja/desktop-linux)
- [QEMU/KVM + libvirt 仮想化クイックガイド](https://zenn.dev/0x69d/articles/20260707-qemu-kvm-libvirt-quickstart)

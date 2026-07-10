---
title: "Claude Desktop on Linux (beta)からVMにSSH接続する"
emoji: "🔗"
type: "tech"
topics: ["linux", "libvirt", "claude", "claude code", "仮想化"]
published: false
---

## はじめに

Claude Desktop on Linux (beta)がリリースされ、UbuntuでもデスクトップアプリからClaudeを使えるようになりました。
Claude DesktopのClaude Codeは使いやすくて、かなり好きです。

今回は、このClaude DesktopのClaude Codeから、VMにSSH接続してみます。

前回記事「[QEMU/KVM + libvirt 仮想化クイックガイド](https://zenn.dev/0x69d/articles/20260707-qemu-kvm-libvirt-quickstart)」で作成した`quickstart-vm`を利用する前提で進めます。

VM作成がまだの方は、先にそちらをご参照ください。
IPアドレス指定ではなく、VMの名前（`quickstart-vm`）でSSH接続できるようにすることが今回のポイントです。

## 検証環境

- ホストOS: Ubuntu 26.04
- Claude Desktop: <!-- TODO: バージョンを記載 -->
- VM: `quickstart-vm`（前回記事で作成したもの）

## Claude Desktop on Linux (beta)をインストールする

公式ドキュメント「[Claude Desktop on Linux (beta)](https://code.claude.com/docs/ja/desktop-linux)」の手順に沿って、Anthropicのaptリポジトリからインストールします。

<!-- TODO: インストール手順・確認結果を記載 -->

## VMをドメイン名で名前解決できるようにする

デフォルトでは、VMへSSH接続するにはIPアドレスを指定する必要があります。
ここでは、libvirtが提供するNSSモジュールを使い、VMの名前（domain XMLの`<name>`）でそのまま名前解決できるようにします。

まず、`libnss-libvirt`パッケージをインストールします。

```bash
sudo apt install libnss-libvirt
```

続いて、`/etc/nsswitch.conf`の`hosts:`行に`libvirt`、`libvirt_guest`を追加します。

<!-- TODO: nsswitch.confの編集前後の差分を記載 -->

`getent hosts`コマンドで、VM名からIPアドレスを解決できるか確認します。

```bash
$ getent hosts quickstart-vm
```

<!-- TODO: 実行結果を記載 -->

> `libvirt_guest`モジュールはVMのDHCPリース情報を参照するため、VMが起動していてIPアドレスが割り当て済みでないと名前解決できません。

## Claude DesktopのターミナルからVMにSSH接続する

Claude Desktopの統合ターミナルを開き、先ほど名前解決できるようになったVM名を指定してSSH接続します。

```bash
ssh ubuntu@quickstart-vm
```

<!-- TODO: 接続結果を記載 -->

IPアドレスを覚えたり調べ直したりする必要がなくなり、VM名だけで接続できるようになりました。

## まとめ

<!-- TODO: まとめを記載 -->

## 参考

- [Claude Desktop on Linux (beta)](https://code.claude.com/docs/ja/desktop-linux)
- [QEMU/KVM + libvirt 仮想化クイックガイド](https://zenn.dev/0x69d/articles/20260707-qemu-kvm-libvirt-quickstart)

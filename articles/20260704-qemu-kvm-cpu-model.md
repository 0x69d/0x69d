---
title: "QEMU/KVMのCPUモデルについて"
emoji: "⚙️"
type: "tech"
topics: ["qemu", "kvm", "libvirt", "linux", "仮想化"]
published: true
---

QEMU/KVM + libvirtによる仮想環境構築ツールを勉強がてら開発しています。

仮想環境でOpenCodeを動かそうとしたときに、実行環境であるBunが起動しませんでした。
原因はlibvirtのドメイン定義でCPUモデルを指定しておらず、デフォルトにフォールバックされていたことでした。

このデフォルトのCPUモデルがそもそも非推奨とされているみたいなので、初めてですが記事にして周知します！
libvirtのドメイン定義について、多少の知識があることを前提としています。

## 検証環境

- ホストOS: Ubuntu 26.04
- libvirt: 12.0.0
- QEMU: QEMU emulator version 10.2.1

## CPUモデルとは

先に進む前に、この記事で繰り返し出てくる「CPUモデル」という言葉を定義しておきます。

QEMUは仮想マシンにCPUを提供する際、実際のホストCPUをそのまま見せるのではなく、「どの命令セット（CPUID機能フラグ）をゲストに公開するか」を定義した設定を見せます。
この設定に名前を付けたものが「CPUモデル」です。

たとえば`Haswell`という名前のCPUモデルを指定すると、ゲストにはHaswell世代のIntel CPUが持つ機能フラグ一式（AVX2やFMAなど）が見えるようになります。
QEMUの`-cpu`オプション、libvirtの`<cpu>`要素はどちらもこのCPUモデルを指定するためのものです。

重要なのは、CPUモデルは「物理的にどのCPUを使うか」ではなく「ゲストにどの機能があるように見せるか」を決めるものだという点です。

## 現象
VMは正常に起動するが、ゲストOSにインストールしたOpenCodeを実行すると異常終了する。

## 原因究明まで

`opencode run`を実行したところ、エラーで異常終了しました。

```
$ opencode run "Hello"
Bun v1.3.14 (0d9b296a) Linux x64 (baseline)
Features: jsc no_avx2 no_avx standalone_executable
RSS: 7.52GB | Peak: 4.06GB | Commit: 7.52GB

CPU lacks AVX support. Please consider upgrading to a newer CPU.
panic(main thread): Segmentation fault at address 0x4EC55800000
oh no: Bun has crashed. This indicates a bug in Bun, not your code.
```

Bun自身が`CPU lacks AVX support`と教えてくれています。
CPU機能の欠如が原因のようです。
（余談ですが、`no_avx`と検知していながらこのエラーで落ちる件はBun側の既知の不具合としてもIssueが上がっているようです。今回はCPUモデル設定の話が本題なので深追いはしません。）

念のためホストのCPUフラグを確認しました。ホストのCPUは`Intel(R) Core(TM) i7-10610U CPU @ 1.80GHz`で、AVX・AVX2には対応しています。

```
$ grep -o 'avx[0-9a-z_]*' /proc/cpuinfo | sort -u
avx
avx2
```
しかし、ゲストでは…。

```
$ grep -o 'avx[0-9a-z_]*' /proc/cpuinfo | sort -u
$
（返事がない）
```

ゲストのドメイン定義を確認すると…。

```
$ virsh dumpxml opencode-test | grep -A5 "<cpu"
<cpu mode='custom' match='exact' check='full'>
  <model fallback='forbid'>qemu64</model>
  <feature policy='require' name='x2apic'/>
  <feature policy='require' name='hypervisor'/>
  <feature policy='require' name='lahf_lm'/>
  <feature policy='disable' name='svm'/>
```

`qemu64`が指定されていました。
そもそもVM作成時に`<cpu>`要素自体を定義していませんでした！

つまり、VM作成時にCPUモデルを指定していなかった結果として、ゲストのCPUモデルがlibvirtのデフォルトである`qemu64`にフォールバックしており、ホストが持っている機能がゲストに伝わっていなかったのです。

## QEMUのデフォルトCPUモデルのあれこれ

そもそも、なぜ`qemu64`だとAVXが使えないのでしょうか。

先ほども書いたとおり、CPUモデルとはCPUIDとしてゲストに公開する機能フラグ一式に名前をつけたものです（QEMUでは`-cpu`オプション、libvirtでは`<cpu>`要素ですね）。
`qemu64`や`kvm64`は、その中でも「引数なし・`<cpu>`要素なし」のときに使われるデフォルトのモデルにあたります。

[公式ドキュメント](https://qemu-project.gitlab.io/qemu/system/qemu-cpu-models.html)にはこう書かれています。

> The default CPU models are designed such that they can run on all hosts.

つまり「どんなホストでも確実に動くこと」を最優先した設計で、Intel・AMD、新しいCPU・古いCPUを問わず、とにかく起動できることを保証するためのモデルです。

その代償として、公開される機能はかなり古いです。

同じドキュメントに、こんな警告もありました。

> The default CPU models will, however, leave the guest OS vulnerable to various CPU hardware flaws, so their use is strongly discouraged.

「強く非推奨」と書かれています。
デフォルトのままだと機能が古いだけでなく、CPUの脆弱性対策も入らない、ということのようです。

BunはbaselineビルドであってもSSE4.2を要求するので、`qemu64`では実行できませんでした。

## ちょっと脱線：x86-64のマイクロアーキテクチャレベル（v1〜v4）

調べていく中で、`x86-64-v1`〜`v4`という表記をよく見かけました。

2020年に、AMD・Intel・Red Hat・SUSEが共同で、x86-64アーキテクチャのCPUについて、v1をベースラインとして、その上に3段階のレベル（v2・v3・v4）を定義したそうです。
CPUの世代ごとに「どこまでの命令セットが使えるか」を整理しています。

| レベル | 目安となる世代 | 追加される主な機能 |
|---|---|---|
| v1 | 2003年ベースライン（AMD K8／Intel Prescott） | CMOV, MMX, SSE, SSE2 |
| v2 | Nehalem（2008年頃） | SSE4.2, POPCNTなど |
| v3 | Haswell（2013年頃） | AVX, AVX2, FMAなど |
| v4 | Skylake-X／AMD Genoaなど（2017年頃〜） | AVX-512 |

`qemu64`/`kvm64`はちょうどこの`v1`に相当します。
Bunの要求スペックは、標準ビルドが`v3`（Haswell相当）、baselineビルドでは`v2`（Nehalem相当）でした。
つまりデフォルトのCPUモデルとは2段階分もギャップがあったことになります。

なお、これらのレベルは包含関係（v1 ⊂ v2 ⊂ v3 ⊂ v4）になっているので、「うちのCPUはどのレベルまで対応しているんだろう」というときは、まずホストのCPU世代を確認すれば、だいたいの見当がつきそうです。

## 最終的な対応とCPU割り当て

まず`<cpu mode='host-passthrough'/>`を指定したところ、問題なく動作することを確認できました。

```diff
+  <cpu mode='host-passthrough'/>
```

対応としてはこれで十分ですが、「せっかくなら一般的な構成に寄せておきたい」と思い、選択肢を整理しました。
QEMU/KVM + libvirtにおけるCPUモデルの指定には大きく4段階あるようです。

- デフォルト（`qemu64`など、非推奨）
- 名前付きモデルを明示（例：`Haswell`、`Nehalem`）
- `host-model`（libvirtがホストに近いモデルを自動選択）
- `host-passthrough`（ホストCPUをそのまま渡す）

`virsh domcapabilities`コマンドによって、ホストOSで`usable`なモデルを確認できます。
CPU以外にも色んな情報が出力されて、有益です。

参考までに、私の環境で`usable='yes'`だったモデルの一部は以下の通りです。
（実際の出力はXMLタグ込みで100行近くになるため、モデル名だけ抜き出しています。かなり古いモデルも大量に出てくるので、それらは省略しています。）

```
$ virsh domcapabilities | grep "usable='yes'"

Nehalem-v1 / Nehalem-v2
Westmere-v1 / Westmere-v2
SandyBridge-v1 / SandyBridge-v2
IvyBridge-v1 / IvyBridge-v2
Haswell-v2 / Haswell-v4
Broadwell-v2 / Broadwell-v4
Skylake-Client-v3 / Skylake-Client-v4
```
名前付きモデルを手動で選ぶなら、この一覧の中から必要な機能を満たすものを選定する必要があります。
しかし今回はそこまでの厳密さを求めていたわけではなかったので、最終的には`host-model`を採用しました。ホストに近い名前付きモデルを自動選択してくれるため、名前付きモデルの手動選定のような保守の手間が発生しません。

```diff
-  <cpu mode='host-passthrough'/>
+  <cpu mode='host-model'/>
```

host-model適用後、ゲストとホスト双方で/proc/cpuinfoのmodel nameを確認しました。
```
# ゲスト
$ grep -m1 '^model name' /proc/cpuinfo
model name      : Intel Core Processor (Skylake, IBRS)

# ホスト
$ grep -m1 '^model name' /proc/cpuinfo
model name      : Intel(R) Core(TM) i7-10610U CPU @ 1.80GHz
```

ゲストにはSkylakeとして見えていますが、ホストの実機はComet Lake世代のi7-10610Uです。
QEMUにはComet Lake専用の名前付きモデルが存在しないため、libvirtが機能セット的に一番近いSkylake系列を自動選定した、という結果になります。

opencode runも問題なく実行できました。
```
$ opencode run "Hello"
（正しい応答が返り、exit code 0で終了）
```

よかったです。


## まとめ
QEMUのCPUモデルは公式ドキュメントに一覧がまとまっています。

- [QEMU / KVM CPU model configuration](https://qemu-project.gitlab.io/qemu/system/qemu-cpu-models.html)

今回のように他ホストへの移植性を考慮しなくてよい単一ホストの環境であれば、`host-passthrough`もしくは`host-model`を選んでおくのがよさそうです。
逆に複数の異なる世代のホストにまたがる環境や、もしくはゲストの構成を固定して長期間安定させたい場合などは、`virsh domcapabilities`で`usable`なモデルを確認した上で、名前付きモデルを明示的に選ぶのがよいかもしれません。

今回、CPUから提供される機能など、低レベルなハードウェアに関するあれこれについて自分が無知なことを痛感しました…。
CPU含め、各種ハードウェアのアーキテクチャの歴史って面白いんですけど覚えきれませんね。

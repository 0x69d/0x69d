# Welcome
ITエンジニアです。前職は介護士でした。
仕事ではLiDARセンサーなどを利用して、物理世界とデジタルを繋ぐシステム開発をしています。
個人的にはLinuxにまつわるさまざまな技術（仮想化、ネットワークなど）に興味があり、手を動かしながら学んでいます。

## About Me
- プログラミング言語
  - Python
  - Go（学習中）
- インフラ・仮想化
  - Linux / QEMU・KVM / libvirt / Kubernetes
  - Prometheus / Grafana
- クラウド
  - AWS

## Certifications
- 基本情報技術者
- LinuC Level 2
- AWS Certified Solutions Architect - Associate
- さくらのクラウド検定 ベーシック

## Projects
QEMU/KVM + libvirtを使い、VPSのような仮想サーバー基盤を個人で作っています。宣言的な定義からVMを払い出すコントロールプレーンと、その上で動くネットワーク・DNS・Kubernetesのアプライアンスで構成しています。

- **[mini-vps-platform](https://github.com/0x69d/mini-vps-platform)**
  - YAML / JSONの宣言からlibvirt domainを払い出す、単一ホスト向けのVMコントロールプレーン。CLIとWeb APIの2つの入口、nwfilterによるパケットフィルタ、静的IP割当、Prometheus / Grafanaによる監視まで実装しています。（Python / FastAPI / Typer）
- **[minivps-router-appliance](https://github.com/0x69d/minivps-router-appliance)**
  - 相互に分離された3つのネットワークセグメント間をルーティングするルータVM。IP forwarding + nftablesによるファイアウォールをゲスト内に構成し、ゴールデンイメージとして再現可能な形でビルドします。
- **[minivps-dns-appliance](https://github.com/0x69d/minivps-dns-appliance)**
  - 内部ドメイン`minivps.internal`の権威DNSと、内部向け再帰リゾルバを提供するDNS VM。BIND9のTSIGで保護した動的更新により、mini-vps-platformがVM作成時にA・PTRレコードを自動登録します。
- **[minivps-k8s-appliance](https://github.com/0x69d/minivps-k8s-appliance)**
  - control-plane 1台 + worker N台のKubernetesクラスタを、kubeadmベースで宣言的に構築・削除するアプライアンス。他のアプライアンスと違いkubeadm initが出力するjoin tokenを別VMへ配る必要があるため、ゴールデンイメージに加えてAnsibleによる複数VM間のオーケストレーションを組み合わせています。

## OSS Contribution
- **[Stellio Context Broker](https://github.com/stellio-hub/stellio-context-broker)**
  - スマートシティ等で用いられるデータ連携プラットフォームの基盤。Docker環境におけるデータ消失のバグを特定し、データの永続化を担保する修正を行いました。([PR #1642](https://github.com/stellio-hub/stellio-context-broker/pull/1642))

## Contact
- [GitHub](https://github.com/0x69d)
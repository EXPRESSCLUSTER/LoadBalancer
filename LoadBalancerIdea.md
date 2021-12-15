# The planning of Load Balancer SL
## ロードバランサとは

クライアントからのアクセスを複数ノードに振り分けるための装置。

### ロードバランサのタイプ
- ラウンドロビン（Round-robin）
	- 事前に登録された複数のサーバに対して、順番にリクエストを割り当てていくアルゴリズム。
- 加重ラウンドロビン（Weighted Round-robin）
	- サーバの処理能力に応じて重み付けをした上で、能力に応じてリクエストを割り当てていくアルゴリズム。
- 最小レスポンス（Least response time）
	- サーバの負荷に応じて動的にリクエストを割り当てていくアルゴリズムの一種。応答速度が短い順に割り当てを行う。

最近では ADC (Application Delivery Controller) というロードバランサ + α の機能を持つ装置が使われることもある。
- TCPマルチプレクシング/TCPバッファリング
	- ADC が TCP セッション確立処理を肩代わりしてサーバの負荷を軽減する。
	また、サーバ応答を記録し、代理応答する。
- HTTPトラフィックの圧縮
	- 転送速度をアップする。
- 送信データのキャッシュ
	- リクエストが多いコンテンツをキャッシュして、レスポンス速度を向上する。
- SSLアクセラレーション
	- SSL を ADC が代行する。
- DDoS攻撃への防御
- ファイアウォール

## Safekit とは

SafeKit は、負荷分散、同期データ レプリケーション、フェールオーバーを提供する。
- ネットワーク機器、複製された SAN ストレージ、エンタープライズ エディションの OS や DB のコストを節約できる。

https://www.evidian.com/ja/products/high-availability-software-for-application-clustering/

### LB 機能 (farm)

#### LB クラスタ構築方法
- Windows
	- https://www.evidian.com/products/igh-availability-software-for-application-clustering/indows-load-balancing-failover/
- Linux
	- https://www.evidian.com/products/igh-availability-software-for-application-clustering/linux-load-balancing-failover/

LB クラスタは 2-node 以上も可。LB クラスタと DB クラスタは同居も可能。
- https://www.evidian.com/products/high-availability-software-for-application-clustering/clustering-software-load-balancing-mirroring/

Apache と IIS の連携モジュール有

start_both, stop_both (スクリプト) で起動開始を制御
	
#### デモ動画
- https://www.youtube.com/watch?v=5k_KetKxo8w&amp;t=289s
1. Safekit をインストールしたノードにブラウザからアクセス
1. もう片方のノードの IP アドレスを入力
1. タイプ (farm) を選択
1. LB の仮想 IP と待ち受けポート番号を設定
1. 構築完了
1. ノード障害を発生させ、全ての TCP セッションが生き残ったノードを介していることを示す (デモ用に作成したページっぽい)
- farm と mirror の連携手順については説明なし。
	- 他に mirror との連携機能で何か強みがあるかも。

### ミラーリング 機能 (mirror)

バイトレベル ファイルレプリケーション (同期ミラーと非同期ミラー)

DR 構成 OK

VM レプリケーションも可 (Hyper-V と KVM) (最大 25 VM)
- https://www.evidian.com/ja/products/high-availability-software-for-application-clustering/vm-ha-vs-application-ha/

Active - Standby
- https://www.evidian.com/products/high-availability-software-for-application-clustering/clustering-software-load-balancing-mirroring/
- Active - Active の構成例も書かれているが、ECX で言うところの「2 つの FOG」のような構成になっている。

### 強み
- Web アプリの N-tiers 構造を、同じソフトウェアを入れてタイプ (farm or mirror) を選択するだけで構築できる。
- 同じ画面で farm と mirror クラスタを一元管理できる。
- Apache と IIS の連携モジュールがあり、選択するだけですぐに LB 連携を設定できる。
- LB 機能を提供しつつ、アプリケーションのフェイルオーバもできる。
	- Safekit では、Apache が Active - Active で実行され、リクエストを複数ノードに割り振れる。

### 不明点
- 仮想 IP アドレスを一つしか設定していないが、実際にはどちらのノードに付いているのか？
	- もし片方に付いているならリクエストをもう片方に転送する機能があるはず。
	- デモ動画では LB (Safekit) と Apache が同居しているが、本来は LB と Apache を 2 台ずつに分けるべきかも。
- 割り振りアルゴリズム (ラウンドロビン、加重ラウンドロビン、最小レスポンス) を詳細に設定できるのか？
- ミラーリングは Active - Standby だが、本当にロードバランスしてるの？

## Load Balancer SL with ECX

- 構成
	- LB を入れた 2 台のサーバでクラスタを組む。
	- 別に Apache をインストールしたサーバを 2 台用意する。

- LB として用いるソフトウェア
	- Windows
		- Windows NLB
	- Linux
		- 何かしらの OSS
		- Apache の LB 機能を使う (この場合はサーバは 2 台のみ？)
			- https://www.ritolab.com/entry/128

- 詰まりそうなポイント
	- フェイルオーバ時の LB の設定の引き継ぎ	
	- まずは Apache は Static なコンテンツを提供することとする
	- 次のステップは Apache のコンテンツをクラスタ内で同期
	- さらに DB 側へのアクセスもロードバランサしようとすると Active - Active な構成が必要
		- Windows なら DFS レプリケーション、Linux は？
	
- テスト
	- TCP セッションがどちらの Apache サーバと繋がってるか確認する用の Web ページを用意する
		- HTTP ヘッダ？の IP アドレスを取得して HTML で表示すればいけそう

## メモ

- 透過的であるか？
	- アクセス先である Apache 等のアプリケーションは、ロードバランサ装置の IP アドレスではなくクライアントの IP アドレスを認識できるか？
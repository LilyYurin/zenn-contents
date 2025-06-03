---
title: "Ubuntu 24.04 LTS にCeforeを導入する方法"
emoji: "💻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Cefore,"ICN","CCN","Ubuntu"]
published: false
---

## はじめに

動画配信や情報共有の効率化を目指す次世代ネットワーク技術「CCN（Content-Centric Networking）」の実装の1つであり、**[NICT](https://www.nict.go.jp/)** によりープンソースプラットフォームにて開発が行われている **[Cefore](https://cefore.net/)**。本記事では、Ubuntu環境にCeforeを導入し、起動の確認までの手順をまとめます。
:::message
本記事は作成時点において最新であるCefore-version0.11.0を元に動作確認を行っています。
バージョンによっては異なる操作が必要となる可能性があるため、詳しくは[Github repository](https://github.com/cefore/cefore/)のreadmeを参照ください。
:::

## 対象読者

- CCN/ICNに興味がある方
- Ubuntu環境で新たにCeforeを導入する方
- コンテンツ指向ネットワークに関心がある方

(私自身、まだまだCeforeの内部にまで詳しいわけではないので事実と異なっている情報などあればご指摘お願いします)

## Ceforeとは？
Cefore（Content-Centric Networking for Future Internet）は、情報通信研究機構（NICT）によって開発された、**情報指向ネットワーク（Information-Centric Networking, ICN）** の実装のひとつです。
従来のインターネットが採用している **IPベースのネットワーク（ホスト指向型）** とは異なるアプローチを採用していている点が特徴です。

### 従来ネットワークとの違い
| 観点        | 従来のIPネットワーク | 情報指向ネットワーク   |
| --------- | ----------- | -------------------- |
| 通信の単位     | ホスト（IPアドレス） | コンテンツ名（名前付きデータ）      |
| データの要求方法  | 送信元から宛先へ直接  | コンテンツ名に基づく要求         |
| キャッシュ     | 原則、エンドツーエンド | 中継ノードでキャッシュ可能        |
| 通信の柔軟性    | 中継点に依存しがち   | 任意のノードから取得可能（マルチソース） |

### Ceforeの主な特徴
- **Publisher / Consumer モデル**
Ceforeでは、コンテンツを提供する側をPublisher、 コンテンツを要求する側をConsumerと呼ぶ。

- **名前付きデータ転送**
パケットには送信元・宛先のIPアドレスではなく、取得したいデータの名前（例: ccnx:/user1/example/file.txt）を使用してやり取りを行う。

- **ネットワーク内キャッシュ**
一度取得されたデータは中継ノードにキャッシュされ、次回以降はネットワークの近い場所から高速に取得が可能となる。

- **IoTやマルチキャスト環境に適応**
複数のノードから同一のデータを取得するなど、ネットワーク効率が重視される環境に適している。



このようにCeforeは、**誰から取得するか**ではなく**何を取得するか**に焦点を当てた新しいネットワークのあり方を模索するプロジェクトとなっています。


## 環境
動作検証を行った環境は以下の通りです。


| OS        | CPU | メモリ |
| --------- | ----------- | -------------------- |
| Ubuntu 24.04 LTS | Intel® Core™ i7-14700 × 28 | 32GB |

## Ceforeインストール手順

### 1. 依存パッケージのインストール

```bash:bash
sudo apt update
sudo apt install -y build-essential automake libtool libssl-dev git
```

### 2. Ceforeソースの取得

```bash:bash
git clone https://github.com/cefore/cefore.git
cd cefore
```
:::message
Ceforeの最新バージョン以外を使用したい場合は、Githubにて公開されている[こちら](https://github.com/cefore/cefore/releases)より過去のバージョンをダウンロードし、次の処理へと進んでください
:::

### 3. インストール先の指定（任意）
インストールディレクトリを変更したい場合は、環境変数 CEFORE_DIR を設定します。
デフォルトは `/usr/local` です。

```bash:bash
export CEFORE_DIR=/opt/cefore
```

### 4. autoconf / automake の実行

```bash:bash
autoconf
automake
```

### 5. configureの実行
最低限の構成で構わない場合は、オプションなしで`configure`を実行します。

```bash:bash
./configure
```

もしローカルキャッシュを利用する`csmgr`や、デバッグ機能を有効にしたい場合は以下のように指定してください。

```bash:bash
./configure --enable-csmgr --enable-cache --enable-debug
```

### 6. make & install の実行
標準では`/usr/local/`にインストールが行われます。

```bash:bash
make
sudo make install
```

### 6. ldconfigの実行

```bash:bash
sudo ldconfig
```

## 動作確認

Ceforeには、CCNの **Interest/Dataパケットの処理** や、 **FIB（Forwarding Information Base）**  および **PIT（Pending Interest Table）** の管理を行う`cefnetd`と、 **Content Store（CS）** 機能を提供する`csmgrd`が存在するため、順に動作確認を行います。
:::message
`csmgrd`を使用するためにはインストール時の`configure`の実行の際に`--enable-csmgr --enable-cache`を指定する必要があります。
:::
:::message
`csmgrd` の設定を行わない場合、`cefnetd`はコンテンツを受信するConsumerとしてのみ動作することとなります。
:::

###  cefnetd デーモンの起動・確認・停止
動作確認手順は以下の通りです

```bash:bash
# cefnetd を起動
$ sudo cefnetdstart

# ステータス確認
$ cefstatus

# cefnetd を停止
$ sudo cefnetdstop
```

#### 実際の画面

![cefnetd動作確認](/images/article_01/image-1.jpg)

このように表示されていれば`cefnetd`は正常に動作しています。

### csmgrd デーモンの設定・起動・確認・停止

#### cefnetd.conf の設定：csmgrd の有効化
`csmgrd`を利用するには、`cefnetd`側の設定ファイルである `/usr/local/cefore/cefnetd.conf`（※インストールパスの `CEFORE_DIR` に応じて適宜変更）にて、CSを利用する設定（CS_MODE=2）に変更する必要があります。 

```conf:/usr/local/cefore/cefnetd.conf
...
# Content Store used by cefnetd
#  0 : No Content Store
#  1 : Use cefnetd's Local cache
#  2 : Use external Content Store (use csmgrd)
#  3 : Use external Content Store (use conpubd)
#CS_MODE=0
CS_MODE=2
...
```

これにより、`cefnetd` は外部キャッシュマネージャである `csmgrd` を利用するようになります。

#### csmgrd.conf の設定：キャッシュ方式の指定（任意）

また、同ディレクトリ内に存在する`csmgrd.conf`中の設定では、使用するキャッシュの種類を `CACHE_TYPE` により指定できます。

```
...
# Type of CS space used by csmgrd.
#  filesystem : UNIX filesystem
#  memory     : Memory
#CACHE_TYPE=filesystem
CACHE_TYPE=memory
...
```

`CACHE_TYPE=memory` を指定すると、キャッシュがRAM上に保持され、ディスクベースの `filesystem` よりも高速な読み書きが可能となります。

#### csmgrd と cefnetd の起動順序について
`csmgrd`を利用する場合、`cefnetd` は起動時に `csmgrd` へ接続を行うため、必ず `csmgrd` を先に起動する必要があります。

以下の順序で起動と確認を行うことができます。

```bash:bash
# csmgrd を起動
$ sudo csmgrdstart

# cefnetd を起動
$ sudo cefnetdstart

# ステータス確認
$ csmgrstatus

# csmgrd を停止
$ sudo csmgrdstop
```

この順序を守らない場合、cefnetd が csmgrd に接続できず、外部キャッシュ機能が正しく動作しないことがあります。

#### 実際の画面
![csmgrd動作確認](/images/article_01/image-2.jpg)

このように表示されていれば`csmgrd`は正常に動作しています。

また、`csmgrd`が正常に動作している場合、`cefstatus`にて表示される`Cache Mode`の欄が` Excache`となっていることが確認できると思います。
ここからも、`cefnetd`上で`csmgrd`が正常に動作していることが確認できます。
![csmgrd動作中のcefstatus](/images/article_01/image-3.jpg)


## おわりに

今回は、Ubuntu 24.04 LTSにおけるCeforeの導入手順を紹介しました。
今後はPublisher/Consumer間で実際にファイルのやり取りを行う方法までを記事にできればと思います。
## 補足：Dockerを利用して構築を行う

本記事ではUbuntu 24.04 LTS上でCeforeをソースからビルドして導入する方法を解説しましたが、Dockerを利用することでより簡単に構築が行えます。
Dockerイメージと設定済みのスクリプトを作成したので、もしよければ下記GitHubリポジトリよりご利用ください。

https://github.com/LilyYurin/cefore-docker


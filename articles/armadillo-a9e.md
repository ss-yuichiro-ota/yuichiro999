---
title: "Armadillo A9E の開発手順と Flask サーバー構築"
emoji: "🦔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["armadillo", "flask", "python", "Linux", "iot", "armadilloa9e",]
published: false
published_at: 2025-05-27 09:00
publication_name: "secondselection"
---

## はじめに

今回は、アットマークテクノ製の組み込みLinuxボードである**Armadillo-A9E**を使って、
シンプルなWebアプリケーションの実装と動作環境構築までの流れをまとめます。
この記事では以下の内容を解説します。

- Visual Studio Code（以下、VSCode）を使用したArmadilloの開発の流れ
- VSCodeの拡張機能であるArmadillo Base OS Development Environment（以下、ABOSDE）のコマンドの説明
- swuファイルの作成方法とアップデート方法

## Armadilloについて

[Armadillo](https://armadillo.atmark-techno.com/about)とは、アットマークテクノが提供するArmプロセッサ・Linux OS搭載のIoTゲートウェイおよびCPUボードの製品シリーズです。
ユーザーが開発したアプリケーションをボード本体に書き込むことで、FA機器・監視システム・スマートデバイスなど幅広い用途に活用できます。
また、試作開発した内容をそのまま量産製造できる産業用途向けの製品となっています。

### Armadillo-A9E について

今回扱う**Armadillo-A9E**は、ARM Cortex-A9コアを搭載したアットマークテクノ製の組み込みLinuxボードです。
従来製品と比べCPU処理能力が向上しており、より幅広い用途への採用が可能です。
また、無線LANやBluetoothを搭載したモデルもあり、**Armadillo Base OS**に対応しています。

![Armadillo A9E](/images/armadillo-a9e/armadillo-a9e.png)

- 主なスペック
  - CPU: ARM Cortex-A9 (i.MX6系)
  - OS: Armadillo Base OS
  - インタフェース : Ethernet, USB, シリアル(RS-485) , 入出力接点
  - 拡張インタフェースによりGPIO, I2C, CANなどが使用可能

### Armadillo Base OS（ABOS）について

[Armadillo Base OS(ABOS)](https://armadillo.atmark-techno.com/guide/armadillo-base-os)は、アットマークテクノが提供するArmadillo専用のOSディストリビューションです。
軽量な**Alpine Linux**をベースとしており、コンテナ管理機能やソフトウェアアップデート機能を搭載しています。コンテナ上でアプリケーションを動作させることで、OSとアプリを分離した安全なアップデートが実現できます。

## 開発環境の準備

本記事では以下のツールを使用します。

- **ATDE（Atmark Techno Development Environment）**
- **Visual Studio Code（VSCode）**
- **VSCodeの拡張機能：ABOSDE**

### ATDE（Atmark Techno Development Environment）とは

ATDEはアットマークテクノが提供する、Armadillo開発向けのVMです。
DebianベースのVM上にクロスコンパイル環境が整っており、Armadillo上で動作するソフトウェアをPC上で開発できます。

![ATDE画面](/images/armadillo-a9e/atde.png)

### 準備物

開発にあたり必要なものは以下のとおりです。
なお、ATDE環境の構築やVSCodeのインストール方法については本記事では割愛します。

- Armadillo-A9E本体
- microSDカード（初期インストールディスク書き込み用）
- USB Type-A - Type-Cケーブル - ➀
- LANケーブル - ➁
- 電源ケーブル - ➂
- DIPスイッチ（ArmadilloA9E搭載）- ➃
- ATDE環境（VSCodeおよびABOSDEインストール済み）

![接続構成](/images/armadillo-a9e/system.drawio.png)

### 初期インストールの実施

[Armadillo Base OSのインストールディスクイメージ](https://armadillo.atmark-techno.com/resources/software/armadillo-iot-a9e/disc-image)をダウンロードし、圧縮ファイルを解凍してイメージファイルを取り出します。このイメージファイルをWin32_Disk_Imagerなどのソフトウェアを用いてmicroSDカードに書き込みます。

![Win32 Disk Imager](/images/armadillo-a9e/sd_imager.png)

イメージを書き込んだmicroSDカードをArmadillo A9Eに挿入し、以下の手順で初期インストールを行います。

1. DIPスイッチをmicroSDカードブート用に変更します。
2. 電源ケーブルを差し込むと初期インストールが開始されます。SDカードから書き込んでいる間はSYSLEDが点滅します。
3. 1分ほど待ち、SYSLEDが消灯したら電源ケーブルを取り外します。
4. DIPスイッチを元の設定に戻し、microSDカードを取り外します。
5. 再度電源ケーブルを差し込み、Armadilloが起動すれば初期インストール完了となります。

![SDブート時](/images/armadillo-a9e/sd_install1.drawio.png)

### ABOS Webにアクセスする

**ABOS Web**とは、Armadillo Base OSを搭載した組み込み機器の設定をブラウザから簡単に行えるGUIツールです。

初期インストール完了後、Armadillo A9Eと同じネットワークに接続されたPCのブラウザで
`https://armadillo.local:58080`にアクセスします。
初回アクセス時はパスワードの設定が求められます。

![ABOS Webログイン](/images/armadillo-a9e/abos-web-login.png)

パスワード設定後、以下のような管理画面が表示されます。

![ABOS Web管理画面](/images/armadillo-a9e/abos-web.png)

ABOS Webでは以下の操作が行えます。

- ネットワーク設定（IPアドレス・DNS・プロキシなど）
- コンテナの管理（起動・停止・設定）
- 時刻設定
- swuファイルのインストール（ソフトウェアアップデート）
- VPNの設定
- ログの確認
- 再起動および電源オフ

:::message

ネットワーク設定でIPアドレスを固定しておくと次回以降は`https://<設定したIPアドレス>:58080`でABOS Webにアクセスできます。

:::

## ATDE環境内のVSCode上での操作方法について

**※ 以降の手順は、ATDE環境内にインストールされたVSCode上での操作を前提とします。**

### 拡張機能ABOSDEのインストール

以下の手順でVSCodeの拡張機能であるABOSDEをインストールします。

1. ATDE上でVSCodeを開きます。
2. 左メニューの拡張機能アイコンをクリックします。
3. 検索欄に「ABOSDE」と入力し、インストールボタンをクリックします。

![拡張機能ABOSDE](/images/armadillo-a9e/abosDE1.png)

インストール後、左メニューにABOSDEのアイコンが追加されます。
アイコンをクリックすると、以下のようなABOSDE EXPLORERが表示されます。

![ABOSDEメニュー](/images/armadillo-a9e/abosDE2.png)

### MONITOR機能

MONITOR機能を使うと、同一ネットワーク上のArmadilloを検出し、SWUのインストールやSSH接続などの操作をGUIから行えます。

ABOSDE EXPLORERの「MONITOR」→「Scanning Armadillo on the network」をクリックとスキャンが開始されます。スキャン後、同じネットワークに接続しているArmadilloの一覧が表示されます。

表示されたArmadillo名の横にある各アイコンから、以下の操作が行えます。

| アイコン | 操作 |
| --- | --- |
| ↑ | SWU のインストール |
| 🌐 | ABOS Web をブラウザで開く |
| 📋 | ssh_config に IP を設定 |
| … | 名前の変更・再起動・電源オフ など |

MONITOR:  
![MONITOR1](/images/armadillo-a9e/MONITOR.png)

### initial_setup.swuファイルの作成

`initial_setup.swu`はArmadilloの初回セットアップ時に必ずインストールする初期設定ファイルです。このファイルをインストールすることで、以下の初期設定が行えます。

- 証明書のコモンネーム（会社や製品が分かる任意の名称）
- 証明書の鍵のパスワード（swuファイル作成時に使用）
- アップデートイメージの暗号化の有無
- アットマークテクノ製イメージのインストール可否
- rootパスワード
- atmarkユーザのパスワード
- Base OSの自動アップデートの有無
- abos-webユーザのパスワード

:::message

**なぜinitial_setup.swuが必要か**

出荷状態のArmadilloは誰でもアップデートファイルを書き込める状態になっています。
`initial_setup.swu`をインストールすることでArmadillo固有の署名が書き込まれ、
署名が一致する場合のみアップデートできるようになります。
実運用では第三者による改ざんを防ぐため、インストールしておく必要があります。

:::

### COMMON PROJECT COMMAND機能

ABOSDE EXPLORERの「COMMON PROJECT COMMAND」→「Generate Initial Setup Swu」をクリックします。

COMMON PROJECT COMMAND:  
![initial_setup1](/images/armadillo-a9e/initial_setup1.png)

クリックするとターミナルが起動し、証明書のパスワードなどの初期設定が行えます。
![initial_setup2](/images/armadillo-a9e/initial_setup2.png)

実行後、`~/mkswu`配下に以下のファイルが生成されます。

- initial_setup.swu：初期設定をArmadilloに書き込むSWUファイル
- initial_setup.desc：SWUファイルを作成するための書式ファイル
- mkswu.conf：mkswuのコンフィグファイル
- swupdate.key：鍵情報
- swupdate.pem：署名情報

:::message alert

これらのファイルを消去すると同じ署名でアップデートできなくなります。
必ずバックアップを取っておいてください。

:::

#### initial_setup.swu を Armadillo にインストール

`~/mkswu` に作成された `initial_setup.swu` をArmadilloにインストールします。
インストールには、前節で紹介したABOSDE EXPLORERのMONITORにある矢印アイコン、またはABOS Webを使用します。以下は、ABOS Webでのインストールの流れを示しています。

![abos-initial-swu-install1](/images/armadillo-a9e/abos-initial-swu-install1.png)

インストール中：
![abos-initial-swu-install2](/images/armadillo-a9e/abos-initial-swu-install2.png)

インストール完了：
![abos-initial-swu-install3](/images/armadillo-a9e/abos-initial-swu-install3.png)

### ABOSDEによるプロジェクトの作成

次にアプリケーション本体を開発していきます。
ABOSDEでプロジェクトを作成すると、以下の3つが自動生成されます。

- アプリケーション本体のソースコード
- .confファイル（コンテナ起動設定）
- Dockerfile（コンテナイメージの定義）

これらをカスタマイズしてプロジェクトをビルドすることで、アプリケーションのコンテナイメージを含んだSWUファイルである`development.swu`または`release.swu`が生成されます。
今回は例として、Debian 12ベースのコンテナを作成します。Flaskを使って「Hello, Armadillo A9E」と表示するシンプルなWebアプリケーションを作成します。

#### CREATE NEW PROJECT機能

ABOSDE EXPLORERの「CREATE NEW PROJECT」→「A9E」→「Python New Project」をクリックします。
クリック後、「プロジェクトを配置するディレクトリ」と「プロジェクト名（コンテナ名）」を設定します。

CREATE NEW PROJECT：

![CREATE NEW PROJECTの画面](/images/armadillo-a9e/create_project2.png)

Python New Project：

![自動生成されたプロジェクトの画面](/images/armadillo-a9e/create_project1.png)

#### アプリケーション本体の作成

今回はFlaskを使用するので、まず`requirements.txt`に以下を記載します。

:::details requirements.txt

```txt
flask
```

:::

次に、アプリケーションのメインスクリプトである`main.py`を以下のように記述します。
ルート`/`にアクセスすると「Hello, Armadillo A9E」を返すシンプルなWebアプリケーションです。

:::details main.py

```python
from flask import Flask

app = Flask(__name__)

@app.route("/")
def hello():
    return "Hello, Armadillo A9E"

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

:::

#### コンテナ設定の作成

コンテナ設定`app.conf`にはコンテナのマウント設定やポートマッピングなどを記述します。
今回の例では、デフォルトから変更するのは1箇所のみで、Flask用にポートマッピングの設定を追加します。

:::details app.conf

```conf
set_image localhost/{{PROJECT}}:latest
add_volumes /var/app/rollback/volumes/{{PROJECT}}:/vol_app:ro
add_volumes /var/app/volumes/{{PROJECT}}:/vol_data
add_volumes /sys:/sys
add_args -it
add_args -p 5000:5000
add_armadillo_env
set_command python_launch
```

:::

#### コンテナイメージの作成

コンテナイメージでは、ベースOS・aptパッケージ・Pythonパッケージ・実行コマンドなどを指定します。
今回の例では、Debian 12のスリム版イメージ（`bookworm-slim`）をベースに使用します。slimイメージは最小限のパッケージのみを含む軽量版です。

:::message

本番環境では、再現性・安全性のためにパッケージのバージョンを固定することを推奨します。

:::

:::message alert

Debian 12（bookworm）以降、システム全体への`pip install`はデフォルトで禁止されています。
`--break-system-packages`オプションを追加しないとインストールが失敗するため、指定してください。

:::

:::details Dockerfile

```Dockerfile
ARG ARCH
ARG VERSION_CODENAME
FROM docker.io/${ARCH}/debian:bookworm-slim
LABEL version="2.0.0"

ARG VERSION_CODENAME
COPY resources/etc/apt resources_${VERSION_CODENAME}/etc/apt /etc/apt/

ARG PACKAGES
RUN apt-get update && apt-get upgrade -y \
    && apt-get install -y --no-install-recommends ${PACKAGES} \
    && apt-get clean

ARG PRODUCT
COPY resources [r]esources_${VERSION_CODENAME} [r]esources_${PRODUCT} resources_python /

ARG PIP_OPTION
RUN python3 -m pip --no-cache-dir install ${PIP_OPTION} --break-system-packages -r requirements.txt

RUN useradd -m -u 1000 atmark
```

:::

### development.swuファイルの作成

アプリケーション本体・コンテナイメージ・コンテナ設定の作成が完了したら、OPENED PROJECT機能を使って`development.swu`を作成します。
`development.swu`は開発用にArmadilloへアプリを書き込むためのファイルで、SSHが自動的に有効化されます。

:::message

`release.swu`は本番用のファイルです。内容は`development.swu`と同じですが、セキュリティの観点からSSHは無効化されています。
今回は開発・検証目的のため`development.swu`を使用します。

:::

#### OPENED PROJECT機能

ABOSDE EXPLORERの「OPENED PROJECT」→「Setup environment」は、SSH接続に必要な鍵を生成する初期設定です。
`development.swu`はSSHを自動有効化するため、`development.swu`を作成する前にクリックしておく必要があります。

SSH接続鍵の生成後、ABOSDE EXPLORERの「OPENED PROJECT」→「Generate development swu」をクリックします。

ABOSDE EXPLORER：

![ABOSDE EXPLORERの画面](/images/armadillo-a9e/development_swu1.png)

「Generate development swu」をクリックすると、以下の処理が実施されます。

1. Dockerfileをもとにコンテナイメージをビルド
2. ビルドしたコンテナイメージ・app.conf・アプリ一式をまとめて`development.swu`ファイルとして生成
3. sshdの有効化処理をswuに組み込む（development.swuのみ）

ビルドが成功すると、証明書の鍵のパスワードが求められます。

### development.swuファイルのインストール

`development.swu`のインストールは`initial_setup.swu`と同じ手順で実施します。
ABOSDE EXPLORERのMONITORの矢印アイコンまたはABOS Web上でswuファイルをインストールします。
インストール後、自動的に再起動され、アプリケーションが起動します。

## アプリの動作確認

PCのブラウザのURL入力欄に「http://<ArmadilloのIPアドレス>:5000」と入力すると、`Hello, Armadillo A9E`と表示されることを確認できます。

![ブラウザでの動作確認画面](/images/armadillo-a9e/development_swu3.drawio.png)

## おわりに

今回は、**VSCode**の拡張機能である**ABOSDE**を使ってArmadillo A9E上にシンプルなWebアプリケーションを構築する方法をまとめました。

従来のArmadilloに比べてアプリケーションの構築が容易になっており、swuファイルによるアップデートも一度流れを把握してしまえばスムーズに進めることができます。

一方で、実際に製品開発する場合は以下の点をルール化する必要があると感じました。

- swuファイルの管理方法とセキュリティ上の注意点（署名・保管場所・配布経路など）
- コンテナイメージのバージョン固定など、依存関係の管理ルール

今後はセンサ類を取り付けて計測・制御するWebアプリケーションを作成したいと考えています。この記事がArmadilloの開発を始める方の参考になれば幸いです。

## 参考資料

- [Armadillo IoT A9E 製品マニュアル - 開発の基本](https://manual.atmark-techno.com/armadillo-iot-a9e/armadillo-iotg-a9e_product_manual_ja-1.5.0/ch03.html)

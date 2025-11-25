---
title: "Pythonで実装する安全なメール送信：STARTTLS・SMTPSの違い"
emoji: "🐡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ssl", "tls", "starttls", "smtps", "python"]
published: false
published_at: 2025-12-01 06:00
publication_name: "secondselection"
---

## 🏁 はじめに

Pythonでメール送信の暗号化処理を実装する機会があったので、  
以前からなんとなく理解していた **SSL/TLS** や **SMTP** について改めて整理しました。

メール送信の通信方式は主に次の3つがあります。

| 方式 | 概要 |
| ---- | ---- |
| SMTP | 平文通信（暗号化なし）でメールを送信する標準的なプロトコル。 |
| STARTTLS | SMTP通信の途中からTLS暗号化に切り替える方式。 |
| SMTPS | SMTP通信開始時からSSL/TLSで暗号化された状態で接続する方式。 |

現在ではセキュリティの観点から、暗号化されない**SMTP**だけの使用は推奨されていません。
一般的には**SMTP+STARTTLS**または**SMTPS**が利用されています。

:::message
この記事はこんな方におすすめです。

- Pythonでメール送信を実装したい初学者の方
- メールサーバー設定やTLS関連のエラーで悩んでいる方  

:::

## 🔐 そもそも「SSL/TLS」ってなに？

### SSL/TLSの概要

**SSL**(**Secure Sockets Layer**)/**TLS**(**Transport Layer Security**)は、クライアントとサーバー間の通信を暗号化し、安全な接続を確立するためのプロトコルです。

>　SSLは現在は廃止となっており、TLSがその後継となります。  
> ただし「SSL」という言葉が慣習的に残っており、現在でも混同して使われています。

### SSL/TLSの主な目的

　暗号化されていない通信では、通信経路上で個人情報や機密情報が第三者に盗聴されたり、内容を改ざんされたりするリスクがあります。
SSL/TLSはこのような通信経路上の攻撃（盗聴・改ざん・なりすまし）を防ぐことを目的とした技術です。

## 📡 メール送信の「SMTP」

**SMTP**(**Simple Mail Transfer Protocol**)は、電子メールを送信・転送するための通信プロトコルです。
送信元のメールクライアントから送信元のメールサーバーへメールを送信する際や、送信元のメールサーバーから送信先のメールサーバーへメールを送信する際に利用されます。
> SMTPはメール通信の基盤となる技術ですが、暗号化する機能は備えていません。
そのため、安全にやり取りするにはSSL/TLS（例：STARTTLSやSMTPS）による暗号化を併用する必要があります。

## ⚙️ メールの暗号化「STARTTLS」

### STARTTLSの概要

**STARTTLS** は、暗号化されないSMTPを途中から **SSL/TLS暗号化通信** に切り替える仕組みです。

> SMTPの「拡張コマンド」の一つで、暗号化を“後から開始”できます。

:::message
ちなみに以下の主要なメールサービスでは、標準でSTARTTLSに対応しています。

- Gmail
- Outlook.com / Microsoft 365
- iCloud Mail
- Yahoo!メール

:::

### STARTTLSの仕組み

実際にSTARTTLSがどういった順番で処理しているのか調べてみました。

1. 送信元のメールクライアントは後述するSMTPSと呼ばれる暗号化された通信で送信元のメールサーバーにメールを送ります。

2. 送信元のメールサーバーから送信先のメールサーバーに対して、SMTPの拡張機能(SSL/TLSを使用するための機能)をサポートしていることを伝える**EHLOコマンド**と呼ばれるコマンドを送信します。

3. 送信先のメールサーバーからEHLOへの応答が返却され、SSL/TLSに対応しているか判定します。

4. 送信元のメールサーバーから送信先のメールサーバーに対して、STARTTLSコマンドを実行します。相手側がTLS非対応の場合は暗号化せず、SMTPの通信を継続します。

5. 送信先のメールサーバーから正常の応答が返却されるとTLS接続が確立されます。

6. 以降の通信が暗号化されます。

![alt text](/images/starttls_and_smtps/starttls.drawio.png)

### ⚠️ 注意点

- STARTTLSコマンドを実行するまでは **暗号化されていません。**
- 相手がTLS非対応なら暗号化されないまま送信されるため、**セキュリティリスクがあります。**

### 強制TLSと日和見（ひよりみ）TLS

TLSは**強制TLS**と**日和見（ひよりみ）TLS**というのがあり、どちらも主にメールの送受信設定で使用される概念です。

#### 強制TLS

送信先と送信元の間で、必ずTLSによる暗号化通信を確立します。もし、確立できないのであれば、メールは送信されずエラーとなります。

#### 日和見（ひよりみ）TLS

送信先がTLSに対応していれば暗号化通信を試みます。送信先がTLSに対応していなくても、メールの送信は可能ですが、暗号化されていないため、当然セキュリティ上のリスクが発生します。  

STARTTLSは送信先がTLSに対応していなくてもメールを送信可能であるため、日和見TLSということになります。送信先がTLSに対応していなくてもメールを送信できるメリットはありますが、暗号化されていないメールにはセキュリティ上のリスクが発生してしまうのがデメリットと言えるでしょう。

## 🔒　メールの暗号化「SMTPS」

### SMTPSの概要

**SMTPS** (**SMTP over SSL/TLS**)は、通信開始時から **SSL/TLS** による暗号化接続を確立してメールを送信する方式です。主に**送信元のメールクライアントと送信元のメールサーバー間**の通信で使用されます。

> SMTPSは暗号化通信専用のポート（一般的に **465番**）を使用する必要があります。  

![alt text](/images/starttls_and_smtps/smtps.drawio.png)

### SMTPSのサンプルプログラム

Pythonを使用する場合、標準ライブラリ**smtplib**に含まれる**SMTP_SSL**クラスを利用するとSMTPサーバーにSSL/TLS接続するためのオブジェクトを作ることができます。これによって最初から安全な（暗号化された）通信経路を確保できます。

```python
import os
from email.mime.text import MIMEText
from smtplib import SMTP_SSL
from dotenv import load_dotenv

# 例: .env ファイルに以下を記載
# MAIL_USER=abc@testmail.com
# MAIL_PASS=xxxx xxxx xxxx xxxx
# MAIL_TO=someone@example.com
# SMTP_HOST=smtp.example.com

# .env 読み込み
load_dotenv()

# "test"というプレーンテキストメールを"UTF-8"で作成
mail = MIMEText("test", "plain", "utf-8")
mail["Subject"] = 'メールの件名'
mail["From"] = os.getenv("MAIL_USER")
mail["To"] = os.getenv("MAIL_TO")
mail['Date'] = "メールの日付" # 省略可

def send_mail(msg: MIMEText):
  """
  SMTP_SSLでメール送信
  """
  host: str = os.getenv("SMTP_HOST") # メール送信サーバーを指定
  port: int = 465 # 専用ポート(一般的に465)を指定
  user = os.getenv("MAIL_USER")
  password = os.getenv("MAIL_PASS")

  with SMTP_SSL(host, port) as server:
      server.login(user, password) # ログイン
      server.send_message(msg)  # メール送信

if __name__ == "__main__":
    send_mail(mail)
```

## まとめ

- **SMTP**は電子メールを送信・転送するための通信プロトコルですが、単独では暗号化する機能はありません。

- **STARTTLS**はまず**送信元のメールサーバーが送信先のメールサーバーに対してSTARTTLSに対応しているか確認**します。対応していれば、暗号化してメールを送信します。対応していなければ、暗号化せずメールを送信します。送信先のメールサーバーがSTARTTLSに対応していなくてもメールを送信できますが、セキュリティ上のリスクは発生してしまいます。

- **SMTPS**は、通信の開始時から暗号化を行う方式で、**送信元のメールクライアントと送信元のメールサーバーの通信**に主に利用されます。最初からSSL/TLSで保護されるため、**比較的安全性が高い**です。一方で、専用ポートを使用する必要があります。

- 利用するメールサーバーやセキュリティ要件に合わせて選択することが大切です。

## 参考リンク・文献

@[card](https://medium-company.com/smtps-starttls-%E9%81%95%E3%81%84/)

@[card](https://zenn.dev/shimakaze_soft/articles/9601818a95309c)

@[card](https://jp.globalsign.com/ssl/about/)

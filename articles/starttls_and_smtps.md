---
title: "Pythonで実装する安全なメール送信：STARTTLS・SMTPSの違い"
emoji: "🐡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["SSL", "TLS", "STARTTLS", "SMTPS", "Python"]
published: false
published_at: 2025-11-28 09:00
publication_name: "secondselection"
---

## 🏁 はじめに

Pythonでメール送信の暗号化処理を実装する機会があったので、  
以前からなんとなく理解していた **SSL/TLS** や **SMTP** について改めて整理しました。

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

　通信の暗号化が行われていない場合、何者かに不正アクセスされ、データを改ざんされる可能性があります。また、暗号化されていない通信を第三者が覗き見ることで、個人情報が盗まれてしまうリスクもあります。SSL/TLSを利用することでこれらのリスクを防ぐことができます。

## 📡 メール送信の「SMTP」

**SMTP**(**Simple Mail Transfer Protocol**)は、電子メールを送信・転送するための通信プロトコルです。送信者のメールサーバーから受信者のメールサーバーへメールを届けるまで管理する役割があります。
> SMTPはメールの送受信に欠かせない技術となりますが、**暗号化する機能はない** ので、前述したSSL/TLSなどで暗号化する必要があります。

## ⚙️ メールの暗号化「STARTTLS」

### STARTTLSの概要

**STARTTLS** は、平文通信のSMTPを途中から **SSL/TLS暗号化通信** に切り替える仕組みです。

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

1. はじめにメールクライアントとメールサーバー間でTCPのコネクションを確立します。

2. メールクライアントからメールサーバーに対して、SMTPの拡張機能(SSL/TLSを使用するための機能)をサポートしていることを伝える**EHLOコマンド**と呼ばれるコマンドを送信します。

3. メールサーバーからEHLOへの応答が返却され、SSL/TLSに対応しているか判定します。

4. メールクライアントからメールサーバーに対してSTARTTLSコマンドを実行します。相手側がTLS非対応の場合は暗号化せず、SMTPの通信を継続します。

5. メールサーバーから正常の応答が返却されるとTLS接続が確立されます。

6. 以降の通信が暗号化されます。

![alt text](/images/starttls_and_smtps/starttls.drawio.png)

### ⚠️ 注意点

- STARTTLSコマンドを実行するまでは **暗号化されていません。**
- 相手がTLS非対応なら平文のまま送信されるため、**セキュリティリスクがあります。**

### 強制TLSと日和見TLS

TLSは**強制TLS**と**日和見TLS**というのがあり、どちらも主にメールの送受信設定で使用される概念です。

#### 強制TLS

メールクライアントとメールサーバー間で、必ずTLSによる暗号化通信を確立します。もし、確立できないのであれば、メールは送信されずエラーとなります。

#### 日和見TLS

通信先のメールサーバーがTLSに対応していれば暗号化通信を試みます。通信先のメールサーバーがTLSに対応していなくても、メールの送信は可能ですが、暗号化されていないため、当然セキュリティ上のリスクが発生します。  

STARTTLSは受信側がTLSに対応していなくてもメールの送信が可能のため、日和見TLSということになります。受信側がTLSに対応していなくてもメールを送信できるメリットはありますが、暗号化されていないメールにはセキュリティ上のリスクが発生してしまうのがデメリットと言えるでしょう。

### STARTTLSのサンプルプログラム

Pythonを使用する場合、標準ライブラリの**smtplib**と**MIMEText**を利用することでSMTPを使用してメールを送信するための機能を実装できます。smtplibに**SMTP**というクラスが存在するため、SMTPサーバーに接続するためのオブジェクトを作れます。ただし、前述したとおり、SMTPだけでは暗号化されていないため、処理の途中で**starttlsメソッド**を呼び出し、TLS暗号化モードに切り替える必要があります。

```python
import os
from email.mime.text import MIMEText
from smtplib import SMTP
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
mail["Subject"] = "メールの件名"
mail["From"] = os.getenv("MAIL_USER", "example@test.com")
mail["To"] = os.getenv("MAIL_TO")
mail["Date"] = "メールの日付"  # 省略可

def send_mail(msg: MIMEText):
    """
    STARTTLSでメール送信
    """
    host: str = os.getenv("SMTP_HOST")   # メール送信サーバーを指定
    port: int = 587  # STARTTLS用のポート
    user = os.getenv("MAIL_USER")
    password = os.getenv("MAIL_PASS")

    with SMTP(host, port) as server:
        server.starttls()  # TLS 暗号化
        server.login(user, password)  # ログイン
        server.send_message(msg)  # メール送信

if __name__ == "__main__":
    send_mail(mail)
```

## 🔒　メールの暗号化「SMTPS」

### SMTPSの概要

**SMTPS** (**SMTP over SSL/TLS**)は、通信開始時から **SSL/TLS** による暗号化接続を確立してメールを送信する方式です。最初から暗号化されているため、**STARTTLS** と比べて **より高い安全性** を確保できます。

> 一方で、SMTPSは暗号化通信専用のポート（一般的に **465番**）を使用する必要があります。  
> また、送信先のメールサーバーがSMTPSに対応していない場合、**メールが届かない** といったデメリットも存在します。

![alt text](/images/starttls_and_smtps/smtps.drawio.png)

### SMTPSのサンプルプログラム

Pythonを使用する場合、標準ライブラリ**smtplib**に含まれる**SMTP_SSL**クラスを利用するとSMTPサーバーにSSL接続するためのオブジェクトを作ることができます。これによって最初から安全な（暗号化された）通信経路を確保できます。

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

- **STARTTLS**は送信先のメールサーバーがTLSに対応しているか確認し、対応していれば、暗号化してメールを送信します。対応していなければ、暗号化せずメールを送信します。送信先のメールサーバーがTLSに対応していなくてもメールを送信できますが、セキュリティ上のリスクは発生してしまいます。

- **SMTPS**は最初から暗号化通信し、比較的安全性が高いメールの送信方法です。しかし、専用ポートを準備する必要があります。また、送信先のメールサーバーがSMTPSに対応している必要があります。

- どちらの方式にもメリットとデメリットがあるため、利用するメールサーバーやセキュリティ要件に合わせて選択することが大切です。

## 参考リンク・文献

@[card](https://medium-company.com/smtps-starttls-%E9%81%95%E3%81%84/)

@[card](https://zenn.dev/shimakaze_soft/articles/9601818a95309c)

@[card](https://jp.globalsign.com/ssl/about/)

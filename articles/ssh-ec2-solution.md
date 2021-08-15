---
title: "公開鍵設定なし・public ip無し・ポート開放なしのインスタンスにIAM権限だけでsshできるようにした"
emoji: "🔖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws", "ec2", "ssh"]
published: false
---

> 追記: この記事で説明している仕組みと全く同じ内容が既に紹介されています。
> https://qiita.com/sisisin/items/1cb6b79f729f1173eaba
>
> また、こちらの方は既に[golang で同様の CLI ツール](https://github.com/youyo/awssh)を実装されています。
> https://blog.youyo.io/posts/created-awssh-command-that-can-login-to-ec2-instance-securely/
>
> `ssh_ec2`は薄い shell script であり軽量で依存が少なく改変可能なこと、ProxyCommand に指定することで通常の ssh コマンドと統合して使用できることが新規性です。

# TL;DR

キーペア・SSH 公開鍵設定してなくてプライベートサブネットにあってポート開放もしてないインスタンスに IAM 権限だけで ssh できる仕組みを思いついたのでユーティリティ化しました。
ただの超薄い shellscript なので適当にコピペして改変して使っても OK です。
https://github.com/moajo/ssh_ec2

# 使い方

`ssh_ec2` はただの shellscript なので、適当に PATH の通る場所に配置すれば OK です。

基本的には`.ssh/config`に以下のように設定しておくだけで使えます。

```
host i-* mi-*
    ProxyCommand sh -c "ssh_ec2 %r %h --send-key-only && aws ssm start-session --target %h --document-name AWS-StartSSHSession --parameters 'portNumber=%p'"
```

(ちなみに`%r`はあまり使われているところを見ませんが、`sshしようとしているユーザ名`が展開されます。)

インスタンスには以下のように接続できます。

```sh
# ssh [ユーザ名]@[インスタンスID]
ssh ec2-user@i-xxxxxxx
```

簡単！！

最新の ubuntu server の AMI をそのまま起動したらすぐに ssh できます。
事前にキーペアをダウンロードしてきたり、逆に公開鍵を登録したりする必要はありません。
ポートの開放も public ip アドレスの設定も不要です。

---

既に SessionManager を使っていて`.ssh/config`を書き換えたくないようなケースもあると思います。
その場合は`.ssh/config`を以下のようにしたまま

```
host i-* mi-*
    ProxyCommand sh -c "aws ssm start-session --target %h --document-name AWS-StartSSHSession --parameters 'portNumber=%p'"
```

`ssh`の代わりに`ssh_ec2`コマンドを使います

```sh
# ssh_ec2 [ユーザ名] [インスタンスID]
ssh_ec2 ec2-user i-xxxxxxx
```

(@がなくなってるのは引数のパースが面倒だったからです・・・すいません・・・:pray:)

(`--send-key-only` を外すと、キーを転送したあと追加で `ssh "$USER_NAME@$INSTANCE_ID"` を実行します。)

# 何が嬉しいか

EC2 インスタンスに ssh するときは、通常そのインスタンスを public サブネットに置くか、さもなければ踏み台インスタンスを確保しておく必要があります。
しかし、インスタンスを public サブネットに置くことはセキュリティ上のリスク要因であり、最低限にしたいものです。
また、セキュリティグループでは ssh に使う(デフォルトでは 22 番)ポートを開放する必要がありますが、この点も sshd の設定が脆弱だとセキュリティリスクになります。できればポートも開放したくないですよね？
更に、キーペアの管理は非常に面倒であり、しばしば管理者から DM で鍵を送ってもらう等の人力オペレーションに依存しています。正直やりたくないです。

このツールを使うとこのあたりの問題が**全部**解決します。

# 仕組み

### SessionManager

https://docs.aws.amazon.com/ja_jp/systems-manager/latest/userguide/session-manager.html
AWS には[SSM SessionManager](https://docs.aws.amazon.com/ja_jp/systems-manager/latest/userguide/session-manager.html)というサービスがあります。
これはインスタンス内に SSM エージェントを立てておくと、ポート開放等していなくても SSM の API からインスタンスにセッションを張ることができるものです。(SSM エージェントは最新の ubuntu/AmazonLinux の AMI には最初から入っています)
また、 [SSM でトンネルを作ってその中で SSH することができます](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-getting-started-enable-ssh-connections.html)。
これを使うと、インスタンスを private subnet に配置してポートを閉じた状態でも、インスタンスに ssh することが可能になります。
このときの権限は IAM で管理できる(`ssm:StartSession`を許可すればいい)ので、通常の ssh よりも安全な構成を維持することができます。

一方で、**公開鍵の設定は事前にやっておく必要があるので、鍵管理の問題は解決しません。**
公開鍵と IAM で権限が 2 重管理になってしまうのもイケてないポイントです。

### InstanceConnect

https://docs.aws.amazon.com/ja_jp/AWSEC2/latest/UserGuide/Connect-using-EC2-Instance-Connect.html
AWS には [EC2 Instance Connect](https://docs.aws.amazon.com/ja_jp/AWSEC2/latest/UserGuide/Connect-using-EC2-Instance-Connect.html)というサービスもあります。
こちらはインスタンス内(正確にはインスタンスメタデータ)に 60 秒間有効な一時的な SSH 公開鍵を push してくれるサービスです。
こちらも事前にインスタンス内に `ec2-instance-connect` をインストールしておく必要がありますが、最新の ubuntu/AmazonLinux には(たぶん)最初から入っています。
必要な権限は `ec2-instance-connect:SendSSHPublicKey` です。
これを使うと鍵管理をから脱却できるので、運用が単純化できます。
一方で、**接続自体は通常の ssh を使うので、インスタンスを public subnet に配置してポートを開放する必要があります。**
この点、SessionManager と一長一短です。

> ちなみに[ec2instanceconnectcli](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-connect-set-up.html#ec2-instance-connect-install-eic-CLI)という公式ツールもありますが、Python 依存なのと、SessionManager との連携が考慮されていない点が欠点です。

### SessionManager+InstanceConnect

よく見るとこの 2 つのサービスは相補的なので、組み合わせたらだいたいすべての問題が解決します。
やっていることは、 `InstanceConnectで一時公開鍵をpushし、SessionManagerのトンネル下でsshする`だけです。
どちらも IAM で権限管理できます。インスタンス内には一切公開鍵を配置する必要がなくなります。

# まとめ

組み合わせたら便利！
(そもそも EC2 使う運用をできるだけ避けたいところではある)

# 関連記事

https://qiita.com/sisisin/items/1cb6b79f729f1173eaba  
https://blog.youyo.io/posts/created-awssh-command-that-can-login-to-ec2-instance-securely/

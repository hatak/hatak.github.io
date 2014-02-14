---
title: レプリケーションが追いつかないときに試すこと
author: hatak
layout: post
permalink: /2011/12/07/9407
categories:
  - MySQL
tags:
  - mysql
---

&#8220;[MySQL Casual Advent Calendar 2011][1]&#8221; 7 日目を担当させていただく、hatak ([@hisashi][2]) です。 普段はモバイルゲームのインフラをメインにみているのですが、今回はそんな業務で経験したことを基に記事を書かせていただきます。 カジュアルすぎる内容かもしれませんが、お付き合いいただければと思います。

<!--more-->

## MySQL のレプリケーション

MySQL のレプリケーションは、安定稼働やバックアップ、負荷分散などの目的に利用できる優れた機能です。 bin-log (バイナリログ) を利用して Master サーバから Slave サーバに更新を伝播させ、データの複製を行います。Slave サーバでは、2 つのスレッドが動作しています。

* IO_THREAD &#8211; Master から送られてきたデータを受け取り、relay-log (リレーログ) として書き出す
* SQL_THREAD &#8211; relay-log を読み出し、DB を更新する

##  遅延の調べ方

&#8220;SQL_THREAD&#8221; による遅延の場合は、Slave サーバで &#8220;SHOW SLAVE STATUS&#8221; コマンドを実行することで確認ができます。

```
hatak@dbslave> SHOW SLAVE STATUS G\
*************************** 1. row ***************************
Slave_IO_State: Waiting for master to send event
Master_Host: 192.168.12.2
Master_User: replicator
Master_Port: 3306
Connect_Retry: 60
Master_Log_File: mysql-bin.012863
Read_Master_Log_Pos: 205295676
Relay_Log_File: mysqld-relay-bin.026640
Relay_Log_Pos: 75468325
Relay_Master_Log_File: mysql-bin.012863
Slave_IO_Running: Yes
Slave_SQL_Running: Yes
Replicate_Do_DB:
Replicate_Ignore_DB:
Replicate_Do_Table:
Replicate_Ignore_Table:
Replicate_Wild_Do_Table:
Replicate_Wild_Ignore_Table:
Last_Errno: 0
Last_Error:
Skip_Counter: 0
Exec_Master_Log_Pos: 205295676
Relay_Log_Space: 205296082
Until_Condition: None
Until_Log_File:
Until_Log_Pos: 0
Master_SSL_Allowed: No
Master_SSL_CA_File:
Master_SSL_CA_Path:
Master_SSL_Cert:
Master_SSL_Cipher:
Master_SSL_Key:
Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
Last_IO_Errno: 0
Last_IO_Error:
Last_SQL_Errno: 0
Last_SQL_Error:
Replicate_Ignore_Server_Ids:
Master_Server_Id:100
1 row in set (0.00 sec)
```

ここに示されている &#8220;Seconds\_Behind\_Master&#8221; の値が、「現在 SQL_THREAD が実行しているクエリの実行時刻」と「Slave サーバが保持しているリレーログの時刻」の差となり、遅延を表しています。

&#8220;IO_THREAD&#8221; による遅延の場合は、Master からのバイナリログが受信しきっていないため、Master における &#8220;SHOW MASTER STATUS&#8221; の結果も参考にする必要があります。

```
hatak@dbmaster> SHOW MASTER STATUS G\
*************************** 1. row ***************************
File: mysql-bin.012863
Position: 205295676
Binlog_Do_DB:
Binlog_Ignore_DB:
1 row in set (0.00 sec)
```

この結果を基に、どれくらいずれているかを見なければなりません。

Master と Slave で &#8220;File&#8221; と &#8220;Master\_Log\_File&#8221;、&#8221;Position&#8221; と &#8220;Read\_Master\_Log\_Pos&#8221; をそれぞれ比較し、どの程度転送が遅れているかをチェックします。 &#8220;IO\_THREAD&#8221; に起因した遅延の場合は、サーバの処理というよりはネットワーク帯域の問題である可能性が高いと思います。

今回は &#8220;SQL_THREAD&#8221; による遅延を想定してまとめていきます。

### Master の更新をブロックする

レプリケーションで送られてくるクエリの流量が多すぎる場合、つまり Master の更新が激しすぎて追いつかないケースでは、そもそも Master での更新を止めてしまうという方法があります。

これは、MASTER\_POS\_WAIT() 関数を利用することで実現できます。

手順については、[MySQL5.1 リファレンスマニュアル][3]の FAQ 項目内に「[レプリケーションが追いつくまでマスタの更新をブロックする方法][4]」として紹介されています。

この方法では同期化をコントロールすることで追いつかせることができますが、実際にサービス運用中のサーバではなかなか使いづらいところもあります。

### Slave のパフォーマンスを調整する

このとき、レプリケーションで伝播するクエリは全て直列化されるため、更新が激しい場合はどうしても遅れてしまうことがあります。 DiskI/O への負荷が高いとき &#8220;innodb-flush-log-at-trx-commit&#8221; の値を変更することで、ディスクへのフラッシュを減らすことができます。このパラメータではログバッファからログファイルへの書き込み、およびディスクへのフラッシュをコントロールすることができます。

| 設定値 | ログバッファのファイルへの書き込み | ディスクへのフラッシュ | 備考 |
| ------ | ---------------------------------- | ---------------------- | ---- |
| 0 | 毎秒 | ログファイル上 | |
| 1 | コミット時 | ログファイル上 | デフォルト値 |
| 2 | コミット時 | 毎秒 | |


この設定値によるパフォーマンス向上度合いは、経験的には効果の大きい順に 0 > 2 > 1 の順と思っています。

my.cnf に記述し起動時に適用することもできますが、 mysqld の再起動をせずに変更・反映が可能です。

```sql
SELECT @@innodb_flush_log_at_trx_commit;
SET GLOBAL innodb_flush_log_at_trx_commit = 2;
```

この設定値はパフォーマンス向上の代わりに、信頼性を犠牲にします。プロセスが突然落ちた場合などにディスクにフラッシュされていないデータをロストする可能性がありますので、状況に応じて（あるいは追いつかせるまでの間だけなど）の使用に抑えることが良いかと思います。

## まとめ

MySQL のレプリケーションが追いつかない場合、プロセスの再起動を行わずに簡単に試せて効果の期待できる方法をまとめてみました。

「実践ハイパフォーマンスMySQL」や「エキスパートのためのMySQLトラブルシューティングガイド」などの書籍でもわかりやすく紹介されていますので、ぜひご参照ください。

このほか、サーバ上で不要なデーモン(cpuspeed など)が動いていないかチェックする、ionice(I/O スケジューラを cfq にする必要があります) で mysqld が優先的に DiskI/O を使えるようにするなどサーバ側でもできることはありそうです。

間違っているところや、他にもこんな方法がある、などございましたらぜひお聞かせください。

明日は [@kamipo][5] さんです！

 [1]: http://mysql-casual.org/2011/11/mysql-casual-advent-calendar-2011.html "MySQL Casual Advent Calendar 2011"
 [2]: http://twitter.com/hisashi "@hisashi"
 [3]: http://dev.mysql.com/doc/refman/5.1/ja/ "MySQL5.1 リファレンスマニュアル"
 [4]: http://dev.mysql.com/doc/refman/5.1/ja/replication-faq.html#qandaitem-5-4-4-1-4 "レプリケーションが追いつくまでマスタの更新をブロックする方法"
 [5]: http://twitter.com/kamipo "@kamipo"

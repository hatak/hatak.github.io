---
title: Homebrew で mroonga ストレージエンジンを入れる
date: 2012-10-24 00:00:00 +0900
author: hatak
layout: post
permalink: /2012/10/24/17562
categories:
  - instruction
tags:
  - groonga
  - mac
  - mecab
  - mroonga
  - mysql
---

ローカルマシン (MaxOSX 10.8.2) に MySQL 5.5 + mroonga の環境を構築してたのですが、いろいろ嵌まったのでメモしておきます。

<!--more-->

## MeCab

mroonga で TokenMecab トークナイザーを利用するためには MeCab が必要です。 Homebrew で IPA 辞書と合わせてインストールします。

    $ brew install mecab mecab-ipadic
    $ mecab -D
    filename: /usr/local/Cellar/mecab/0.994/lib/mecab/dic/ipadic/sys.dic
    version: 102
    charset: utf8
    type: 0
    size: 392126
    left size: 1316
    right size: 1316
    $ mecab 今日は雨です
    今日 名詞,副詞可能,*,*,*,*,今日,キョウ,キョー
    は 助詞,係助詞,*,*,*,*,は,ハ,ワ
    雨 名詞,一般,*,*,*,*,雨,アメ,アメ
    です 助動詞,*,*,*,特殊・デス,基本形,です,デス,デス
    EOS

brew で mecab-ipadic 入れるとデフォルトで UTF-8 になるようになっています。mecab コマンドでインタラクティブに形態素解析してみて、文字化けせずに出力されれば問題なしです。

もし、文字化けしてる場合は Homebrew 周りのどこかに問題があります。brew doctor などで調べていきます。

    $ brew unlink mecab mecab-ipadic
    $ brew uninstall mecab mecab-ipadic
    $ brew doctor

私の場合、10.8 にアップデートする以前に Homebrew で入れた libiconv が中途半端に残ってたことでおかしくなっていたようでした。

## groonga

MeCab の次は groonga をインストールします。

    $ brew install groonga
    $ groonga --version
    groonga 2.0.7 [darwin12.2.0,x86_64,utf8,match-escalation-threshold=0,nfkc,msgpack,zlib,lzo,kqueue]
    configure options: < '--prefix=/usr/local/Cellar/groonga/2.0.7' '--with-zlib' 'CC=cc' 'CXX=c++' 'PKG_CONFIG_PATH=/usr/local/lib/pkgconfig:/usr/local/share/pkgconfig:/usr/local/Library/Homebrew/pkgconfig'>

インストール後にチェックをして、TokenMecab が使えるか確認します。 mecab と書かれていない場合は使えません。groonga の configure オプションでは mecab はデフォルト有効と書かれており、mecab-config を自動で探してくれることになっているのですが、パスが通ってないなどがあるのかもしれません。 直接 brew edit で Fomula を書き換えて再度インストールしてしまいます。

    diff --git a/Library/Formula/groonga.rb b/Library/Formula/groonga.rb
    index 1c4e8cf..45a7a2b 100644
    --- a/Library/Formula/groonga.rb
    +++ b/Library/Formula/groonga.rb
    @@ -8,9 +8,11 @@
       depends_on 'pkg-config' => :build
       depends_on 'pcre'
       depends_on 'msgpack'
    +  depends_on 'mecab'
    +  depends_on 'mecab-ipadic'
    
       def install
    -    system "./configure", "--prefix=#{prefix}", "--with-zlib"
    +    system "./configure", "--prefix=#{prefix}", "--with-zlib", "-with-mecab", "-with-mecab-config=/usr/local/bin/mecab-config"
         system "make install"
       end
     end

これでOK。

    $ brew install groonga
    $ groonga --version
    groonga 2.0.7 [darwin12.2.0,x86_64,utf8,match-escalation-threshold=0,nfkc,mecab,msgpack,zlib,lzo,kqueue]
    configure options: < '--prefix=/usr/local/Cellar/groonga/2.0.7' '--with-zlib' '--with-mecab' '--with-mecab-config=/usr/local/bin/mecab-config' 'CC=cc' 'CXX=c++' 'PKG_CONFIG_PATH=/usr/local/lib/pkgconfig:/usr/local/share/pkgconfig:/usr/local/Library/Homebrew/pkgconfig'>

## mroonga

次は mroonga をインストールします。Homebrew には内包されていませんが、mroonga のリポジトリに Fomula があるのでこちらを使います。

    $ brew install https://raw.github.com/mroonga/Homebrew/master/mroonga.rb --use-Homebrew-mysql

もし、MySQL を Homebrew で入れていない場合 (オフィシャルバイナリを利用している場合など) は、同一バージョンのソースを別途ダウンロードして引数に渡します。また、パーミッションの関係で最後の make install で失敗するので、plugin ディレクトリの所有グループを変更してパーミッションも変えておきます。

    $ sudo chgrp staff /usr/local/mysql/lib/plugin
    $ sudo chmod g+w /usr/local/mysql/lib/plugin
    $ brew install https://raw.github.com/mroonga/Homebrew/master/mroonga.rb --with-mysql-source=/usr/local/src/mysql-5.5.28 --with-mysql-config=/usr/local/mysql/bin/mysql_config

嵌まりながら MySQL を Homebrew 管理とオフィシャルバイナリの両方で試してみたのですが、基本的には Homebrew で揃えた方が便利かなと思いました。

オフィシャルバイナリの特徴をまとめてみるとこんな感じです。

* 長所
    * OS の設定画面で起動/停止、あるいは自動起動設定が行える
    * インストールウィザードが付属しているためインストールが容易
* 短所
    * 言語バインディングを利用するときに DYLD\_LIBRARY\_PATH にパスが通っていないとエラーになることがある 
        * 逆にパスを通していると Homebrew でエラーになることがある
    * データやログなどのパーミッションがユーザ権限ではなく root になってしまう
    * 起動/停止を行う際にパスワードが必要なのが面倒

個人的には、複数のユーザで利用している mac でユーザ共通に自動起動させたい（もしくはどうしてもオフィシャルバイナリを使いたい）という条件でなければ Homebrew で良いのかなと思いました。

## 動作確認

MySQL Server が起動していれば、mroonga の Fomula の中でプラグインと関数の追加も行われています。停止していた場合は起動後に追加のコマンドを入力します。

    $ mysql -uroot -e 'INSTALL PLUGIN mroonga SONAME "ha_mroonga.so"; CREATE FUNCTION last_insert_grn_id RETURNS INTEGER SONAME "ha_mroonga.so"; CREATE FUNCTION mroonga_snippet RETURNS STRING SONAME "ha_mroonga.so";'

これで mroonga ストレージエンジンが使えるようになります。

    root@localhost[test]> SHOW ENGINES;
    +-------+---+----------------------+-----+--+----+
    | Engine | Support | Comment | Transactions | XA | Savepoints |
    +-------+---+----------------------+-----+--+----+
    | FEDERATED | NO | Federated MySQL storage engine | NULL | NULL | NULL |
    | MRG_MYISAM | YES | Collection of identical MyISAM tables | NO | NO | NO |
    | MyISAM | YES | MyISAM storage engine | NO | NO | NO |
    | BLACKHOLE | YES | /dev/null storage engine (anything you write to it disappears) | NO | NO | NO |
    | CSV | YES | CSV storage engine | NO | NO | NO |
    | MEMORY | YES | Hash based, stored in memory, useful for temporary tables | NO | NO | NO |
    | mroonga | YES | CJK-ready fulltext search, column store | NO | NO | NO |
    | InnoDB | DEFAULT | Supports transactions, row-level locking, and foreign keys | YES | YES | YES |
    | PERFORMANCE_SCHEMA | YES | Performance Schema | NO | NO | NO |
    | ARCHIVE | YES | Archive storage engine | NO | NO | NO |
    +-------+---+----------------------+-----+--+----+
    10 rows in set (0.01 sec)
    
    root@localhost[test]> SHOW PLUGINS;
    +---------+----+-------+-----+---+
    | Name | Status | Type | Library | License |
    +---------+----+-------+-----+---+
    | binlog | ACTIVE | STORAGE ENGINE | NULL | GPL |
    | mysql_native_password | ACTIVE | AUTHENTICATION | NULL | GPL |
    | mysql_old_password | ACTIVE | AUTHENTICATION | NULL | GPL |
    | CSV | ACTIVE | STORAGE ENGINE | NULL | GPL |
    | MEMORY | ACTIVE | STORAGE ENGINE | NULL | GPL |
    | MyISAM | ACTIVE | STORAGE ENGINE | NULL | GPL |
    | MRG_MYISAM | ACTIVE | STORAGE ENGINE | NULL | GPL |
    | ARCHIVE | ACTIVE | STORAGE ENGINE | NULL | GPL |
    | BLACKHOLE | ACTIVE | STORAGE ENGINE | NULL | GPL |
    | FEDERATED | DISABLED | STORAGE ENGINE | NULL | GPL |
    | InnoDB | ACTIVE | STORAGE ENGINE | NULL | GPL |
    | INNODB_TRX | ACTIVE | INFORMATION SCHEMA | NULL | GPL |
    | INNODB_LOCKS | ACTIVE | INFORMATION SCHEMA | NULL | GPL |
    | INNODB_LOCK_WAITS | ACTIVE | INFORMATION SCHEMA | NULL | GPL |
    | INNODB_CMP | ACTIVE | INFORMATION SCHEMA | NULL | GPL |
    | INNODB_CMP_RESET | ACTIVE | INFORMATION SCHEMA | NULL | GPL |
    | INNODB_CMPMEM | ACTIVE | INFORMATION SCHEMA | NULL | GPL |
    | INNODB_CMPMEM_RESET | ACTIVE | INFORMATION SCHEMA | NULL | GPL |
    | INNODB_BUFFER_PAGE | ACTIVE | INFORMATION SCHEMA | NULL | GPL |
    | INNODB_BUFFER_PAGE_LRU | ACTIVE | INFORMATION SCHEMA | NULL | GPL |
    | INNODB_BUFFER_POOL_STATS | ACTIVE | INFORMATION SCHEMA | NULL | GPL |
    | PERFORMANCE_SCHEMA | ACTIVE | STORAGE ENGINE | NULL | GPL |
    | partition | ACTIVE | STORAGE ENGINE | NULL | GPL |
    | mroonga | ACTIVE | STORAGE ENGINE | ha_mroonga.so | GPL |
    +---------+----+-------+-----+---+
    24 rows in set (0.01 sec)
    
    root@localhost[test]> CREATE TABLE diaries (
        -> id INT PRIMARY KEY AUTO_INCREMENT,
        -> content VARCHAR(255),
        -> FULLTEXT INDEX (content) COMMENT 'PARSER "TokenMecab"'
        -> ) ENGINE = mroonga COMMENT = 'engine "innodb"' DEFAULT CHARSET utf8;
    Query OK, 0 rows affected (0.50 sec)
    
    root@localhost[test]> SHOW CREATE TABLE diaries;
    +---+--------------------------------------------------------------------------------------+
    | Table | Create Table |
    +---+--------------------------------------------------------------------------------------+
    | diaries | CREATE TABLE `diaries` (
    `id` int(11) NOT NULL AUTO_INCREMENT,
    `content` varchar(255) DEFAULT NULL,
    PRIMARY KEY (`id`),
    FULLTEXT KEY `content` (`content`) COMMENT 'PARSER "TokenMecab"'
    ) ENGINE=mroonga DEFAULT CHARSET=utf8 COMMENT='engine "innodb"' |
    +---+--------------------------------------------------------------------------------------+
    1 row in set (0.00 sec)

warning がでなければ問題なく動作しています。念のため mysql.err や groonga.log といったログにも目を通しておきます。

これで mroonga が使えるようになりました。使ってみた印象としては、文節を考慮してくれる高速な LIKE というイメージです。ちょっとした全文検索であれば、手軽に使えるレベルで良いかと思いました。
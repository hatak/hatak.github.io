---
title: CentOS + Nginx で WordPress を構築
author: hatak
layout: post
permalink: /2011/05/12/178
categories:
  - 構築手順
tags:
  - centos
  - nginx
---
WordPress を CentOS + Nginx で動作させたくなったので、思い立ってやってみました。

<!--more-->

# EPEL からのパッケージ導入

CentOS なので、アプリケーションは yum でまとめて管理したいと思います。本来ならば、CentOS のオフィシャルに存在しないパッケージは RPM を自前で作ったほうがよいのかも知れませんが、EPEL に stable （けど少し古い） のパッケージがあるのでこれを利用します。

``` console
$ wget http://download.fedora.redhat.com/pub/epel/5/x86_64/epel-release-5-4.noarch.rpm
$ sudo rpm -ivh epel-release-5-4.noarch.rpm
```

今回は x86_64 の環境であるため、他のアーキテクチャのパッケージが入らないように yum.conf に設定をしてしまいます。

``` console
$ echo "exclude=*.i686 *.i386" >> /etc/yum.conf
```

必要となるパッケージをまとめてインストールします。

``` console
$ sudo yum install nginx spawn-fcgi mysql mysql-server php53 php53-mysql php53-mbstring
```

そしてこれらがインストールされました。（依存パッケージは割愛してます）

*   nginx-0.8.53-1.el5
*   spawn-fcgi-1.6.3-1.el5
*   mysql-5.0.77-4.el5_5.5
*   mysql-server-5.0.77-4.el5_5.5
*   php53-5.3.3-1.el5_6.1
*   php53-mysql-5.3.3-1.el5_6.1
*   php53-mbstring-5.3.3-1.el5_6.1

# spawn-fcgi 設定

spawn-fcgi はポートを指定して起動する方式を利用します。 起動引数を設定できるので、ここで pid ファイルや起動ユーザなどと一緒に指定します。

``` bash /etc/sysconfig/spawn-fcgi
OPTIONS="-u nginx -g nginx -a 127.0.0.1 -p 8080 -S -M 0600 -C 5 -F 1 -P /var/run/spawn-fcgi.pid -- /usr/bin/php-cgi
```

daemon が起動してエラーが出なければ問題ありません。

``` console
$ sudo /sbin/service spawn-fcgi start
$ sudo /sbin/chkconfig spawn-fcgi on
```

# Nginx 設定

設定ファイルに WordPress 公開のための Location を追加します。 うまく起動しない場合、

``` nginx
server {
    listen       80;
    server_name  blog.example.com;

    location / {
        root   /var/www/wordpress;
        index  index.html index.htm index.php;
        if (-f $request_filename) {
            expires 14d;
            break;
        }
        if (!-e $request_filename) {
            rewrite ^(.+)$  /index.php?q=$1 last;
        }
    }

    location ~ .php$ {
        root           html;
        fastcgi_pass   127.0.0.1:8080;
        fastcgi_index  index.php;
        fastcgi_param  SCRIPT_FILENAME  /var/www/wordpress/$fastcgi_script_name;
        include        fastcgi_params;
    }
}
```

そして、Nginx を起動します。

``` console
$ sudo /sbin/service nginx start
$ sudo /sbin/chkconfig nginx on
```

# MySQL 設定

daemon を起動して初期化します。

``` console
$ sudo /sbin/service mysqld start
$ sudo /sbin/chkconfig mysqld on
$ sudo mysql_install_db
```

``` mysql
CREATE DATABASE wordpress;
GRANT ALL PRIVILEGES ON wordpress.* TO `wordpress`@'localhost' IDENTIFIED BY "password";
FLUSH PRIVILEGES;
```

実際に接続できるか、ローカル上で動作テストを行います。 ログインできれば問題ありません。

``` console
$ mysql -u wordpress -p wordpress
```

# WordPress インストール

今回は /var/www/wordpress 以下に日本語版をそのままインストールします。

``` console
$ wget http://ja.wordpress.org/wordpress-3.1.2-ja.tar.gz
$ sudo mkdir /var/www/wordpress
$ sudo tar zxvf ~/wordpress-3.1.2.tar.gz
```

あとは流れにあわせ、[WordPress のインストール手順][1] に沿って設定を行うと利用可能となります。

 [1]: http://ja.wordpress.org/install/

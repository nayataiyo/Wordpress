- ## wordpressを作成しよう!
- ### mentaユーザ作成する（リモートマシン）
   ```
     [root@vagrant vagrant]# useradd menta
   ```
   ```
     [root@vagrant vagrant]# cat /etc/passwd
   ```
   - mentaユーザーが作成できるのを確認
　 <br> menta:x:1002:1002::/home/menta:/bin/bash </br>
   - visudoでnopassword設定をする
   <br> ```[root@vagrant vagrant]# visudo```</br>
     以下を追記して保存
　 <pre>menta ALL=(ALL) NOPASSWD:ALL</pre>

- ### mentaユーザでのssh認証
   - ホストマシンで認証鍵の作成
 　<br> ```$ ssh-keygen -t rsa -b 2048```</br>
   - リモートマシンに接続
 　<br> ```$ vagrant ssh ``` </br>
   - 設定ファイルの変更
  <br>```[vagrant@vagrant ~]$ sudo su```</br>
　<br>```[root@vagrant vagrant]# vim /etc/ssh/sshd_config```</br>
  　</pre>
   - 以下の設定になっていること確認する
  　<pre>
     PubkeyAuthentication yes
     PermitRootLogin yes
     AuthorizedKeysFile .ssh/authorized_keys
     PasswordAuthentication no</br> 
  　</pre>
   - sshd.serviceの再起動
　　<br>  ```systemctl restart sshd.service```  </br>
   - ホストマシンにてssh接続設定
  　<br>```$ vim ~/.ssh/config```</br>
  　<pre>
      Host dev01
      Hostname 192.168.50.4
      User menta
      Port 22
      IdentityFile ~/.ssh/id_rsa ←作成した秘密鍵のパス
      ForwardAgent yes
  　</pre>
- ### リモートマシンに公開鍵を配置 
  　
   <br> ```$ cat ~/.ssh/id_rsa.pub ```</br>
   
  　</pre>
   - リモートマシンにvagrantで接続後mentaホームディレクトリに公開鍵ファイル移動
   <br>``` $ ssh  vagrant@192.168.50.4```</br>
  　<br>```[vagrant@vagrant ~]$ sudo su```</br>
  　<br>```[root@vagrant vagrant]# vi /home/menta/.ssh/authorized_keys```</br>
  　<br>```[root@vagrant vagrant]# chown -R menta:menta /home/menta/.ssh/authorized_keys```</br>
  　<br>```[root@vagrant vagrant]# chown -R menta:menta .ssh/```</br>
  ※上記コマンドで接続できない場合ファイヤーウォール設定、アクセス権限を見直す
   - ホストマシンからmentaユーザで仮想マシンへ接続
    <br> ```$ ssh -i ~/.ssh/id_rsa menta@192.168.50.4``` </br>
 - ### Nginxのバーチャルホスト設定(Nginxの設定ファイル)
    <br>※仮想マシンの環境にもう既にnginx導入済み(192.168.50.4でのブラウザ接続設定完了済み)</br>
    <br>```[menta@vagrant ~]$ sudo su```</br>
    <br>```[root@vagrant menta]# vi /etc/nginx/nginx.conf```</br>
      以下を追記して保存
<pre>
    user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

include /usr/share/nginx/modules/*.conf;

events {
    worker_connections 1024;
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';   

    access_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 4096;

    include             /etc/nginx/mime.types;
    include /etc/nginx/sites-enabled/*;　　　　　←追記箇所
    default_type        application/octet-stream;

    include /etc/nginx/conf.d/*.conf;
    
    server {
        listen       80;
        listen       [::]:80;
        server_name  192.168.50.4;
        root         /usr/share/nginx/html;

        include /etc/nginx/default.d/*.conf;

        error_page 404 /404.html;
        location = /404.html {
        }

        error_page 500 502 503 504 /50x.html;
        location = /50x.html {
        }
    }

}
</pre>

   - バーチャルホスト用のドキュメントルート＋テキスト作成

<br>```[root@vagrant menta]# mkdir /var/www/dev.menta.me ```</br>
 <br> ```[root@vagrant menta]# vi /var/www/dev.menta.me/index.html```</br>
 <br> ```[root@vagrant menta]# chown -R nginx:nginx /var/www/dev.menta.me``` </br>
  <br>```[root@vagrant menta]# chown -R nginx:nginx /var/www/dev.menta.me/index.html ```</br>

   - バーチャルホスト用設定ファイルの設定＋必要ディレクトリの作成
  <br>```[root@vagrant menta]# mkdir /etc/nginx/conf.d``` ←ない場合は作成</br>
 <br> ```[root@vagrant menta]# vi /etc/nginx/conf.d/dev.menta.me.conf```</br> 
以下を追記して保存
<pre>
   server {
    listen 80;
    server_name dev.menta.me;

    root /var/www/dev.menta.me;
    index index.html;

    location / {
    try_files $uri $uri/ =404;

     }
    }  

</pre>
- /etc/hostsで名前解決の設定（etc/hostsで設定することで最優先で名前解決される）
 <br>```[root@vagrant menta]# vi /etc/hosts```</br>
  以下を追記して保存
<pre>
   192.168.50.4  dev.menta.me
</pre>
  - ### ログフォーマットを変更する。
 <br>```[root@vagrant menta]# vi /etc/nginx/nginx.conf```</br>
 
 以下を追記して保存＋必要のない設定は消す
 <pre>
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

include /usr/share/nginx/modules/*.conf;
events {
 worker_connections 1024;
 }
http {
↓追記
    log_format custom_log '
               [nginx] time:$time_iso8601 
               server_addr:$server_addr 
               host:$host 
               method:$request_method 
               reqsize:$request_length 
               uri:$request_uri 
               query:$query_string 
               status:$status 
               size:$body_bytes_sent 
               referer:$http_referer 
               ua:$http_user_agent 
               forwardedfor:$http_x_forwarded_for 
               reqtime:$request_time 
               apptime:$upstream_response_time';
↓追記
    gzip on;
    gzip_types text/plain text/css text/html application/json application/javascript application/xml text/xml application/x-javascript text/javascript image/svg+xml;
    gzip_min_length 1000;
    gzip_comp_level 6;

    include /etc/nginx/conf.d/*.conf;

}
※もともと記載されていたlog_formatは削除
 </pre>

<br>```[root@vagrant menta]# vi /etc/nginx/conf.d/dev.menta.me.conf```</br>
以下にも追記
<pre>
server {
    listen 80;
    server_name dev.menta.me;

    root /var/www/dev.menta.me/;
    index index.php index.html ;

    access_log /var/log/nginx/dev.menta.me.log custom_log;
    error_log /var/log/nginx/error.log2;

location ~ \.php$ {
     include        fastcgi_params;
     fastcgi_pass   unix:/var/run/php-fpm/www.sock;
     fastcgi_index  index.php;
     fastcgi_param  SCRIPT_FILENAME $document_root$fastcgi_script_name;
}
}                                                                                                     
</pre>

  - User-Agent を DotBot に変更してテスト

<br>```[root@vagrant menta]# curl -A "Mozilla/5.0 (compatible; DotBot/1.2; +https://opensiteexplorer.org/dotbot; help@moz.com)" http://dev.menta.me </br>  
<br>```[root@vagrant menta]# systemctl restart nginx.service```</br>
<br>ブラウザから　[http://dev.menta.me/](http://dev.menta.me/)　にアクセスしてみる</br>
<br>※ログのフォーマットが変更されているのかも確認</br>
 - ### phpのインストールと設定
<br>```[root@vagrant menta]# amazon-linux-extras enable php8.2```</br>
<br>```[root@vagrant menta]# yum remove php* -y``` </br>
<br>```[root@vagrant menta]# yum install php php-cli php-fpm php-mysqlnd php-pdo php-gd php-mbstring php-xml php-json php-curl -y ```</br>

<br>```[root@vagrant menta]# amazon-linux-extras install epel```　←amazonlinux環境　（eprlがない場合）</br>
<br>```[root@vagrant menta]# yum install epel-release```</br>
<br>```[root@vagrant menta]# yum install -y https://rpms.remirepo.net/enterprise/remi-release-7.rpm```</br>
<br>```[root@vagrant menta]# yum-config-manager --enable remi-php83```　←有効化</br>
<br>```[root@vagrant menta]# # vim /etc/yum.repos.d/remi-php83.repo```</br>
<pre>
[remi-php83]
name=Remi's PHP 8.3 RPM repository for Enterprise Linux 7 - $basearch       
#baseurl=http://rpms.remirepo.net/enterprise/7/php83/$basearch/
#mirrorlist=https://rpms.remirepo.net/enterprise/7/php83/httpsmirror        
mirrorlist=http://cdn.remirepo.net/enterprise/7/php83/mirror
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-remi
priority=1
</pre>
- wordpress環境で必要なパッケージ群の依存関係の解消
<br>```yum -y install libzip5 libsodium krb5-devel openssl-devel gcc gcc-c++ libxml2-devel libtool oniguruma5php jbigkit gd gd-devel libwmf libraqm gd3php　　←openssl-devel、libxml2-devel、oniguruma5php、gcc と gcc-c++、libtool```gd と gd-develは必要その他は拡張機能レベル</br>

<br>```yum -y install php php-cli php-fpm php-mysqlnd php-pdo php-gd php-mbstring php-xml php-json php-curl -y ```</br>

<br>```[root@vagrant libgd-2.2.5]# php -v```</br>
<pre>
PHP 8.3.8 (cli) (built: Jun  4 2024 14:53:17) (NTS gcc x86_64)
Copyright (c) The PHP Group
Zend Engine v4.3.8, Copyright (c) Zend Technologies
    with Zend OPcache v8.3.8, Copyright (c), by Zend Technologies
</pre>
- ソケット通信の設定
  <br>``` [root@vagrant menta]# vim /etc/php-fpm.d/www.conf ``` </br>
以下に修正して保存
<pre>  
[www] ←必須   
group = nginx
user = nginx

listen = /run/php-fpm/www.sock
listen.owner = nginx
listen.group = nginx
listen.mode = 0660
</pre>

  dev.menta.me.confにも以下を追記して保存
<pre>
server {
    listen 80;
    server_name dev.menta.me;

    root /var/www/dev.menta.me/;
    index index.php index.html ;　←追記　 index.phpを追記している

   location / {
    try_files $uri $uri/ =404;
 }

↓追記　　
   
location ~ \.php$ {
     include        fastcgi_params;
     fastcgi_pass   unix:/var/run/php-fpm/www.sock;
     fastcgi_index  index.php;
     fastcgi_param  SCRIPT_FILENAME $document_root$fastcgi_script_name;
                                            }
}
</pre>

<br>[global] セクションは、php-fpm 全体の動作に関わる設定を行います。</br>
<br>[www] などの各プールセクションは、特定のワーカーグループ（プール）ごとの設定を行います。複数のプールを定義することで、異なるアプリケーションやサイトごとに異なる設定を適用できます。</br>


- ### php-fpmでのチューニング
<br>```[root@vagrant menta]# htop```</br>
<br>※自分のマシンの環境に合わせて適宜調整する</br>
<br>```[root@vagrant menta]# vim /etc/php-fpm.d/www.conf```</br>
<pre>
pm = static   ←初期状態はdynamicに設定されているがオーバーヘッドが発生する点から推奨されていない
pm.max_children = 10 　←作成される子プロセスの最大数
pm.start_servers = 10　←起動時に作成される子プロセス数
pm.min_spare_servers = 10　←アイドル状態のサーバプロセス数の最小値
pm.max_spare_servers = 10　←アイドル状態のサーバプロセス数の最大値
pm.process_idle_timeout = 10s;　←各子プロセスが再起動するまでに実行するリクエスト数
pm.max_requests = 100　←各子プロセスが再起動するまでに実行するリクエスト数
php_admin_value[memory_limit] = 256M　←phpのメモリ利用制限
request_terminate_timeout = 180　←タイムアウトの設定
</pre>
static=自分で設定した子プロセスの最大数で固定されるので起動時のオーバーヘッドが生じない
<br>dynamic=同時接続数が増えてプロセス数が足りなくなった時だけ子プロセスの最大数を超えて追加で起動されるのでオーバーヘッドの懸念が生じる</br>

<br>```[root@vagrant menta]# ab -c 10 -n 100 http://192.168.50.4/```</br>
<pre>
This is ApacheBench, Version 2.3 <$Revision: 1913912 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking 192.168.50.4 (be patient).....done


Server Software:        nginx/1.22.1
Server Hostname:        192.168.50.4
Server Port:            80

Document Path:          /
Document Length:        615 bytes

Concurrency Level:      10
Time taken for tests:   0.016 seconds ←テストに要した時間
Complete requests:      100　←成功したリクエスト数
Failed requests:        0　←失敗したリクエスト数
Total transferred:      84800 bytes
HTML transferred:       61500 bytes
Requests per second:    6190.42 [#/sec] (mean)　←秒間さばけるリクエスト数
Time per request:       1.615 [ms] (mean)　←各リクエストがサーバーによって完了するまでにかかった時間の平均値
Time per request:       0.162 [ms] (mean, across all concurrent requests)　←並行して行われた前リクエストにわたる平均応答時間
Transfer rate:          5126.44 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    1   0.5      0       2
Processing:     0    1   0.5      1       3
Waiting:        0    1   0.5      0       2
Total:          0    1   0.6      1       3
WARNING: The median and mean for the initial connection time are not within a normal deviation
        These results are probably not that reliable.
ERROR: The median and mean for the waiting time are more than twice the standard
       deviation apart. These results are NOT reliable.

Percentage of the requests served within a certain time (ms)
  50%      1
  66%      2
  75%      2
  80%      2
  90%      2
  95%      2
  98%      3
  99%      3
 100%      3 (longest request)
</pre>
以下のコマンドの用途としては認証情報が必要となるサイトでの検証などに用いられるのでcookieを付与している。
<br>```ab -c 10 -t 100 -C "sessionid=sessionidtoken; hoge=example" http://192.168.50.4/　```←例として作成。「sessionidtoken」と「example」は2つのcookieを指定しているということであり、Name=Valueで指定することができる。</br>
<br>```各cookie情報は　右クリック→検証→Application→Cookiesで確認することができる```</br>

- ### MySQLのインストールと設定

<br>```[root@vagrant menta]# yum install -y https://dev.mysql.com/get/mysql80-community-release-el7-3.noarch.rpm```</br>
<br>```[root@vagrant menta]# rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2023```　←gpgキーのバージョン変更により名前が変更されている場合があるので注意</br>
<br>```[root@vagrant menta]# rpm -qa gpg-pubkey*```</br>
- インストールを可能状態に設定
<br>```[root@vagrant menta]# vim /etc/yum.repos.d/mysql-community.repo```</br>
 
<pre>
 
[mysql80-community]
name=MySQL 8.0 Community Server
baseurl=https://repo.mysql.com/yum/mysql-8.0-community/el/7/x86_64/   ←amazonlinuxはredhat7として扱うらしい
enabled=1　　　　←有効化
gpgcheck=1　　　　
gpgkey=https://repo.mysql.com/RPM-GPG-KEY-mysql　　　←ダウンロード先のリンクを指定
</pre>
- MySQLインストール
<br>```[root@vagrant menta]# yum clean all```</br>
<br>```[root@vagrant menta]# yum makecache```</br>
<br>```[root@vagrant menta]# yum install -y mysql-community-server```</br>

<br>```[root@vagrant menta]# sudo systemctl start mysqld```</br>
<br>```[root@vagrant menta]# sudo systemctl enable mysqld```</br>
- 設定ファイルの変更

<br>```[root@vagrant menta]#　vim /etc/my.cnf```</br>
- 文字コードの設定とログの出力先を指定
  
<pre>
[mysqld]
character-set-server=utf8mb4　　　
collation-server=utf8mb4_unicode_ci

↓ログの出力先を指定
log-error=/var/log/mysql/error.log
slow_query_log=1
slow_query_log_file=/var/log/mysql/slow.log
long_query_time=2
</pre>

<br>```[root@vagrant menta]#systemctl restart mysqld ```</br>
- データベースの設定

<br>```[root@vagrant menta]#　cat /var/log/mysqld.log | grep 'temporary password'```　　←mysqlの初期パスワードを確認</br>　
<br>```[root@vagrant menta]# mysql -u root -p```</br>
<pre>
mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY 'Kokop660---';　←パスワードポリシーがあるのでできだけ複雑に
mysql> CREATE DATABASE wordpress_db CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;　　　文字コード設定
mysql> CREATE USER 'menta'@'localhost' IDENTIFIED BY 'Kokop660---';　ユーザーIDパスワード設定
mysql> GRANT ALL PRIVILEGES ON wordpress_db.* TO 'menta'@'localhost';　　　権限設定
GRANT PROCESS ON *.* TO 'username'@'localhost';　　プロセス設定（usernameのところは自環境の場合mentaで設定）
mysql> FLUSH PRIVILEGES;　　　更新
mysql>EXIT;　抜ける
</pre>

- wordpressのソースコードをダウンロード
<br>```[root@vagrant menta]# cd /var/www/dev.menta.me```</br>
<br>```[root@vagrant menta]# chown -R nginx:nginx /var/www/dev.menta.me```</br>
<br>```[root@vagrant menta]# chmod -R 755 /var/www/dev.menta.me```</br>
<br>```[root@vagrant menta]# wget https://wordpress.org/latest.tar.gz```</br>
<br>```[root@vagrant menta]# tar -xzvf latest.tar.gz```</br>
<br>```[root@vagrant menta]# rm -rf wordpress latest.tar.gz```</br>
- データベースとの接続設定
<br>```[root@vagrant dev.menta.me]# cp wordpress/wp-config-sample.php wordpress/wp-config.php```  ←sampleが入っている状態だと読み込まないので名前を変更してコピー</br>
<br>```[root@vagrant dev.menta.me]# chown nginx:nginx /var/www/dev.menta.me/wordpress/wp-config.php```</br>
<br>```[root@vagrant ~]# vim /var/www/dev.menta.me/wordpress/wp-config.php```</br>
<pre>
define( 'DB_NAME', 'wordpress_db' );

/** Database username */
define( 'DB_USER', 'menta' );

/** Database password */
define( 'DB_PASSWORD', 'Kokop660---' );

/** Database hostname */
define( 'DB_HOST', 'localhost' );

/** Database charset to use in creating database tables. */
define( 'DB_CHARSET', 'utf8mb4' );

/** The database collate type. Don't change this if in doubt. */
define( 'DB_COLLATE', '' );
</pre>
- データベースのバックアップシェルスクリプトの作成
パスワード読み込み時のシェルスクリプトを作成
<br>```[root@vagrant menta]# vim /home/menta/set_db_env.sh```</br>
<pre>
 #!/bin/bash

## 認証情報を環境変数として設定
export DB_USER='ユーザー名'
export DB_PASSWORD='パスワード'  
</pre>
<br>```[root@vagrant menta]# chmod 700 /home/menta/set_db_env.sh```</br>

<br>```[root@vagrant menta]# vim wordpress.sh```</br>
<pre>
#!/bin/bash

#エラー発生時に処理を止める
set -e

#環境変数にユーザー名とパスワードをセットする
source /home/menta/set_db_env.sh

# バックアップを保存するディレクトリ
 BACKUP_DIR="/path/to/backup/directory"
#
# # MySQLの設定
 DB_NAME="wordpress_db"

# # 日付形式 (YYYY-MM-DD)
 DATE=$(date +"%Y-%m-%d")

# # バックアップファイル名
 BACKUP_FILE="$BACKUP_DIR/$DB_NAME-$DATE.sql"

# # 古いバックアップを削除するための日数 (例: 5日)
 DAYS_TO_KEEP=5

# # ディレクトリが存在しない場合は作成
 mkdir -p $BACKUP_DIR

# # MySQLデータベースのバックアップを作成
mysqldump --single-transaction --databases "$DB_NAME"  > $BACKUP_FILE   
# # 古いバックアップを削除
OLD_DATE=`date +%Y%m%d --date "5 days ago"` ←””オプションらしい
DUMP=${DIR}/mysqldump.${DATE}.sql
OLD_DUMP=${DIR}/mysqldump.${OLD_DATE}.sql

rm -f ${OLD_DUMP}





# # バックアップ完了メッセージ
 echo "Backup of $DB_NAME completed and old backups deleted."
</pre>

スクリプトの実行
``` [root@vagrant ~]# ./wordpress.sh```
<br>Backup of wordpress_db completed and old backups deleted.</br>

- ### ワードプレスを表示させる
- 各サービス再起動 

<br>```[root@vagrant menta]# systemctl restart nginx.service```</br>
<br>```[root@vagrant menta]# systemctl restart php-fpm.service```</br>
<br>```[root@vagrant menta]# systemctl restart mysqld.service```</br>
    
[http://dev.menta.me/wordpress](http://dev.menta.me/wordpress)にアクセス










    
  
  

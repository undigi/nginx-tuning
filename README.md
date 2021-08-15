# nginx-tuning

Nginxのチューニングポイントまとめ

## OSの設定

### sysctl

- fs.file-max
  - 確認->設定

    ```sh
    sysctl fs.file-max
    -> fs.file-max = 1850054
    sysctl fs.file-max=320000
    -> fs.file-max = 320000
    sysctl fs.file-max | sudo tee /etc/sysctl.d/file-max.conf
    -> fs.file-max = 320000
    ```

### limit

- systemd
  - `LimitNOFILE`の設定をする。`infinity`で無限になる
  - fs.file-maxとlimitで小さい方の値が採用される

    ```sh
    mkdir /etc/systemd/system/nginx/service.d
    cat << EOC > /etc/systemd/system/nginx/service.d
    [Services]
    LimitNOFILE=655360
    EOC
    ```

## nginxの設定

詳細は`nginx/nginx.conf`や`nginx/conf.d/default.conf`に記載

### workerの設定

- worker_connections
  - workerごとのコネクション数
  - eventsコンテキストに書く
- worker_processes
  - workerのプロセス数
  - mainコンテキストに書く
- worker_rlimit_nofile
  - 同時に利用できるファイルディスクリプタ数に関するOSによる制限値
  - workerごとの値になる
  - ファイル、クライアントからの接続、バックエンドへの接続も含まれる
  - mainコンテキストに書く

### logの設定

- access_log
  - gzipの設定を入れるので最初は`$gzip_ratio`も入れておく
  - 処理時間がわかるように`$request_time`も入れておく
  - 最終的にはoffにする
- error_log
  - 最終的にはoffにする

### sendfileの設定

- sendfile
  - ファイルの中身をネットワークに転送するシステムコール
  - コピーやユーザープロセスの処理が省略されて性能が向上する
  - HTTPSのWebサーバでは使うことができない
  - httpコンテキストかserverコンテキストに書く
- tcp_nopush
  - sendfileシステムコールはファイルの内容とHTTPヘッダの送信が別れてしまい性能劣化してしまう
  - tcp_nopushはヘッダのみ送信されることを防ぐ

### file cacheの設定

- open_file_cache
  - 特定のファイルへのアクセスが集中する場合に有効
  - ファイルを閉じる処理を遅らせて、ファイルディスクリプタやフィアルサイズ、ファイルの更新時刻をcacheすることで、次のアクセス時にファイルを開く処理を省略する
  - serverコンテキストで書く

### ioの設定

使わないと思う。。

- aio
  - ファイルへのI/Oを非同期にできる
  - serverコンテキストに書く
- directio
  - キャッシュに載せなくなり、直接ファイルやりとりをする
  - aioと合わせて利用することが多い
  - serverコンテキストに書く

### 帯域の設定

- limit_rate
  - 1つのコネクションが使える帯域を指定する
  - これをすることで1つのユーザーからのアクセスが多い場合に制限することができる
  - serverコンテキストで書く
- limit_rate_after
  - limit_rateが動作するまでのサイズを指定する
  - serverコンテキストで書く

### cache, bufferの設定

- proxy_cache_path
  - バックエンドに処理を流さずにキャッシュに溜め込んだデータを使うようになる
  - httpコンテキストに記載する
  - 引数は以下の通り
    - keys_zone
      - ゾーン名とサイズを指定できる
      - ゾーンの容量1MBあたり約8000個のキーを保持できる
    - levels
      - キャッシュディレクトリの構造を指定できる
    - inactive
      - キャッシュがアクセスされなくなってから捨てられるまでの時間
    - max_size
      - キャッシュ全体のサイズ
      - キャッシュファイルを書き出してから古いフィアルを削除するという処理になる
- proxy_cache
  - ゾーン名を指定し、記述したコンテキストが対象となる部分でコンテンツをキャッシュする。
  - locationコンテキストで書く
- proxy_no_cache
  - CookieやAuthorizationヘッダーで使わない時などに使う
  - 空文字列もしくは「0」でない場合、キャッシュにコンテンツを保存しないようになる
  - locationコンテキストで書く
- proxy_cache_bypass
  - 空文字列もしくは「0」でない場合、キャッシュからレスポンスを返さないようになる
  - locationコンテキストで書く
- proxy_cache_key
  - デフォルトで`$scheme$proxy_host$request_uri`にMD5をかけたものが使われる
  - locationコンテキストに書く
- proxy_buffering
  - bufferとはデータの転送効率をあげるためにレスポンスのデータをまとめておくための領域
  - バックエンドから送られてきたデータを一時的にメモリやディスクのバッファに貯めつつ、クライアントに送信するという動作が効率的
  - バッファリングをするかどうかを指定する。デフォルトはon
  - locationコンテキストに書く
- proxy_buffer_size
  - レスポンスの先頭部分に使われるメモリ上のバッファサイズを指定する。デフォルトは4kB
  - locationコンテキストに書く
- proxy_buffers
  - `proxy_buffer_size`の次に使われるメモリ上のバッファの数とサイズを指定。デフォルト`8 4k`で、4kBのバッファが8個用意される
  - locationコンテキストに書く
- proxy_max_temp_file_size
  - 一時ファイルに配置されるバッファのサイズ
  - `proxy_buffer_size`と`proxy_buffer_size`の全てのバッファが使われてまだデータがある場合にファイルのバッファが使われる
  - ファイルのバッファが使われたとき、ログレベルで`WARN`が出る
  - ログに`WARN`が出るのを見ながら、`proxy_buffers`などをチューニングしていくのがポイント
  - デフォルトで1GB
  - locationコンテキストに書く
- proxy_temp_file_write_size
  - 一時ファイルノバファに一度に書き出されるデータのサイズ。デフォルトで`proxy_buffer_size + proxy_buffers`
  - locationコンテキストに書く
- proxy_busy_buffers_size
  - レスポンス送信中の状態にできるバッファのサイズで、クライアントが多くのデータを受信しきらないうちに次の送信を始めないように制限する。デフォルトで`proxy_buffer_size + proxy_buffers`
  - locationコンテキストに書く
- その他けっこう重要なこと
  - コンテンツの更新について
    - コンテンツが更新されたかどうかの判断材料として`ETag`ヘッダや`Last-Modifired`ヘッダがある
    - `ETag`ヘッダには更新を判定できる文字列を返し、その文字列が次にクライアントから`If-None-Match`ヘッダとして渡される。
    - `Last-Modifired`は、コンテンツの最終雨更新時刻を示し、クライアントが`If-Modifired-Since`ヘッダでクライアントの知っている最終更新時刻を渡すことで、Webサーバは最終更新時刻以降にコンテンツを更新したかどうかを判定できるようになる
  - expiresディレクティブの活用
    - `Expires`と`Cache-Control`ヘッダは`expires`ディレクティブを使って制御できる
    - ファイルタイプなどによっても制御できる
  - ngx_cache_purgeモジュールの活用
    - データの変更が起きるとcacheのせいで古いデータを返してしまうことがよくある
    - [この記事](https://qiita.com/zaru/items/c41072e29b9550c2e6a8)を参考に、`ngx_cache_purge`を使うといい
    - 例えば、postでデータ登録が発生→アプリケーション実行の中でcacheをリフレッシュなど
    - 簡単にはpost起きたらもうcacheを全て削除してしまうなど
    - 試してないけどインストール方法

        ```sh
        wget http://nginx.org/download/nginx-$NGINX_VERSION.tar.gz
        wget https://github.com/nginx-modules/ngx_cache_purge/archive/$NGX_CACHE_PURGE_VERSION.tar.gz -O ngx_cache_purge-$NGX_CACHE_PURGE_VERSION.tar.gz
        tar -xf nginx-$NGINX_VERSION.tar.gz
        mv nginx-$NGINX_VERSION nginx
        rm nginx-$NGINX_VERSION.tar.gz
        tar -xf ngx_cache_purge-$NGX_CACHE_PURGE_VERSION.tar.gz
        mv ngx_cache_purge-$NGX_CACHE_PURGE_VERSION ngx_cache_purge
        rm ngx_cache_purge-$NGX_CACHE_PURGE_VERSION.tar.gz

        BASE_CONFIGURE_ARGS=`nginx -V 2>&1 | grep "configure arguments" | cut -d " " -f 3-`
        /bin/sh -c "./configure ${BASE_CONFIGURE_ARGS} --add-module=/tmp/ngx_cache_purge"
        make && make install
        ```

### 転送量の設定

gzipによる圧縮では、nginxはファイルから読み込んだデータを圧縮して送信する。  
そのため、プロセスのメモリ常にファイルのデータを読み込まない`sendfile`でのデータ送信はできなくなる。  
つまり、`gzip`と`sendfile`は両立しない。両方設定した場合は、`sendfile`が無効になる。  
あらかじめ圧縮したファイルを用意してgzip_staticを使って、sendfileと共存することもできる

- gzip
  - gzipエンコードに対応する
  - httpコンテキスト、serverコンテキストに書く
- gzip_comp_level
  - 圧縮レベルを指定する。1は高速で圧縮率が低くなり、9は高圧縮で処理が遅くなる。
  - httpコンテキスト、serverコンテキストに書く
- gzip_types
  - 圧縮対象となるMIMEタイプを追加する。
  - httpコンテキスト、serverコンテキストに書く
- gzip_min_length
  - 圧縮対象となる最小データサイズを設定する。デフォルトは20バイト
  - httpコンテキスト、serverコンテキストに書く
- gzip_static
  - あらかじめ圧縮したファイルを使うかどうかを設定する
  - `on`にすると圧縮済みファイルがある場合はその内容を使う。
  - `always`にするとクライアントの`Accept-Encording`ヘッダの値に関わらず、必ず圧縮済ファイルを使う
  - httpコンテキスト、serverコンテキストに書く
- gunzip
  - httpコンテキスト、serverコンテキストに書く

### 負荷分散の設定

- upstream
  - 重み付け

    ```conf
    upstream backend {
        server 192.168.1.10:80 weight=2;
        server 192.168.1.11:80 weight=1;
    }
    ```

  - 接続元IPアドレスによるハッシュ

    ```conf
    upstream backend {
        hash $remote_addr
        server 192.168.1.10:80;
        server 192.168.1.11:80;
    }
    ```
  
  - 最小コネクション数による負荷分散

    ```conf
    upstream backend {
        least_conn;
        server 192.168.1.10:80;
        server 192.168.1.11:80;
    }
    ```

### httpsの設定

- keepalive_timeout
  - tcpコネクションしたあと、そのコネクションを使い回す最大時間
  - サーバ側とクライアント側のうち期限が切れれば切断される。デフォルト65秒
  - httpコンテキストかserverコンテキストに書く
- keepalive_requests
  - 一つの接続でKeep-Aliveを使って処理するリクエストの最大数。デフォルト100
  - httpコンテキストかserverコンテキストに書く
- ssl_session_timeout
  - セッションIDを保持する時間の設定。デフォルト5分
  - クライアント側が1日のことが多いので、長くても1日に設定する
  - serverコンテキストに書く
- ssl_session_cache
  - セッションIDの保存先の情報。デフォルトは`none`で無効になっている
  - `shared:SSL:50m`とすると、SSLという名前のキャッシュで50MBの容量を使うという意味になる
  - serverコンテキストに書く
- ssl_session_ticket
  - サーバー側で共有鍵を暗号化したものをチケット情報としてクライアント側に送り、再接続時にはクライアント側からのClient Helloで送られたチケット情報をサーバ側で複合してセッションを再開させる方式
  - `on`か`off`を指定する。
  - serverコンテキストに書く
- ssl_session_ticket_key
  - ssl_session_ticketの鍵を保持する秘密鍵を置いておく場所を書く
  - ↓の様な感じで48バイトデータを簡単に作成する

    ```sh
    openssl rand 48 > ticket.key
    ```
  
  - serverコンテキストに書く
- http2
  - https通信を使う前提。
  - 通信が多重化されていて基本的に性能が向上する
  - listenディレクティブに`ssl`に続けて記載する
  - serverコンテキストに書く

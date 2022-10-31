Fluentdのシステム設定
==============================

## Fluentdのシステム設定
`system` ディレクティブを設定しない場合の各種挙動は以下の通り。

```text:プロセス名/worker数
$ docker compose exec fluentd ps
PID   USER     TIME  COMMAND
    1 fluent    0:00 tini -- /bin/entrypoint.sh fluentd
    6 fluent    0:00 {fluentd} /usr/bin/ruby /usr/bin/fluentd --config /fluentd
   15 fluent    0:00 /usr/bin/ruby -Eascii-8bit:ascii-8bit /usr/bin/fluentd --c
   22 fluent    0:00 ps
```

`system`ディレクティブを設定した場合の挙動は以下の通り。

```
$ docker compose exec fluentd ps
PID   USER     TIME  COMMAND
    1 fluent    0:00 tini -- /bin/entrypoint.sh fluentd
    7 fluent    0:00 {fluentd} supervisor:Data collector
   16 fluent    0:00 {ruby} worker:Data collector0
   17 fluent    0:00 {ruby} worker:Data collector1
   18 fluent    0:00 {ruby} worker:Data collector2
   19 fluent    0:00 {ruby} worker:Data collector3
   20 fluent    0:00 {ruby} worker:Data collector4
   21 fluent    0:00 {ruby} worker:Data collector5
   22 fluent    0:00 {ruby} worker:Data collector6
   23 fluent    0:00 {ruby} worker:Data collector7
   24 fluent    0:00 ps
```

`root_dir`パラメータについて、コンテナ上ではfluentユーザで動作するため /var/run/fluentd などを指定することができない。
該当ディレクトリが存在せず、fluentユーザの権限ではディレクトリが作成できないため。
コンテナ上では設定ファイルを /fluentd/ 以下に配置しているので、ここを指定するとよさそう。
これにより、/fluentd/ 以下にworker別ディレクトリが作成される。

```
$ docker compose exec fluentd ls /fluentd/
etc      plugins  worker1  worker3  worker5  worker7
log      worker0  worker2  worker4  worker6
```

シグナル・PRCはホストからのcurlまたはコンテナコマンドでのシグナル送信で動作確認できる。

```
$ curl localhost:24226/api/config.getDump
{"conf":"<ROOT>\n  <system>\n    process_name Data collector\n    workers 8\n    root_dir /fluentd\n    log_level debug\n    suppress_config_dump false\n    rpc_endpoint 0.0.0.0:24226\n    enable_get_dump true\n    <log>\n      format json\n      time_format %Y-%m-%d %H:%M:%S %z\n      rotate_age 10\n      rotate_size 100k\n    </log>\n  </system>\n  <source>\n    @type forward\n    @id input1\n    @label @mainstream\n    port 24224\n  </source>\n  <filter **>\n    @type stdout\n  </filter>\n  <label @mainstream>\n    <match docker.**>\n      @type file\n      @id output_docker1\n      path \"/fluentd/log/docker.*.log\"\n      symlink_path \"/fluentd/log/docker.log\"\n      append true\n      time_slice_format %Y%m%d\n      time_slice_wait 1m\n      time_format %Y%m%dT%H%M%S%z\n      <buffer time>\n        timekey_wait 1m\n        timekey 86400\n      </buffer>\n      <inject>\n        time_format %Y%m%dT%H%M%S%z\n      </inject>\n    </match>\n    <match **>\n      @type file\n      @id output1\n      path \"/fluentd/log/data.*.log\"\n      symlink_path \"/fluentd/log/data.log\"\n      append true\n      time_slice_format %Y%m%d\n      time_slice_wait 10m\n      time_format %Y%m%dT%H%M%S%z\n      <buffer time>\n        timekey_wait 10m\n        timekey 86400\n      </buffer>\n      <inject>\n        time_format %Y%m%dT%H%M%S%z\n      </inject>\n    </match>\n  </label>\n</ROOT>\n","ok":true}

$ docker compose kill -s USR2
[+] Running 1/0
 ⠿ Container sec03-fluentd-1  Killed

$ docker compose logs -f
...
sec03-fluentd-1  | {"time":"2022-10-30 09:17:59 +0000","level":"debug","message":"fluentd supervisor process got SIGUSR2"}
sec03-fluentd-1  | {"time":"2022-10-30 09:17:59 +0000","level":"info","message":"Reloading new config"}
sec03-fluentd-1  | {"time":"2022-10-30 09:17:59 +0000","level":"warn","message":"[output_docker1] 'time_format' specified without 'time_key', will be ignored"}
sec03-fluentd-1  | {"time":"2022-10-30 09:17:59 +0000","level":"warn","message":"[output1] 'time_format' specified without 'time_key', will be ignored"}
...
```

## Fluentdのコマンドラインオプション
docker fluentdはtiniおよびentrypoint.shにて起動しているが特別なことはしていない。
追加のコマンドライン引数を指定する場合はcompose.yamlファイルにcommand命令で指定すればよい。

## 設定変更の反映
composeファイルでコンテナの全再起動したければ docker compose restart、
シグナルはdocker compose kill -s <シグナル> で動作する。

## よくわからなかったこと
### system.log の適用範囲
system.log.formatにてjson指定したが、2種類のログ出力フォーマットが混在している。
一部のログは時刻やログレベルがjson化されておらずログメッセージのみjson化されている。また時刻フォーマットも適用されていない。
一部のログは時刻やログレベルも含めてjson化されている

```
sec03-fluentd-1  | 2022-10-29 10:19:23.198162453 +0000 fluent.info: {"port":24224,"bind":"0.0.0.0","message":"[input1] listening port port=24224 bind=\"0.0.0.0\""}
sec03-fluentd-1  | 2022-10-29 10:19:23.202381812 +0000 fluent.info: {"pid":21,"ppid":8,"worker":4,"message":"starting fluentd worker pid=21 ppid=8 worker=4"}
sec03-fluentd-1  | {"time":"2022-10-29 10:19:23","level":"warn","message":"no patterns matched tag=\"fluent.info\"","worker_id":4}
sec03-fluentd-1  | 2022-10-29 10:19:23.202866814 +0000 fluent.debug: {"instance":2180,"stage_size":0,"queue_size":0,"message":"[output1] buffer started instance=2180 stage_size=0 queue_size=0"}
sec03-fluentd-1  | {"time":"2022-10-29 10:19:23","level":"warn","message":"no patterns matched tag=\"fluent.debug\"","worker_id":4}
sec03-fluentd-1  | 2022-10-29 10:19:23.203631731 +0000 fluent.debug: {"message":"[output1] flush_thread actually running"}
sec03-fluentd-1  | 2022-10-29 10:19:23.203860947 +0000 fluent.debug: {"message":"[output1] enqueue_thread actually running"}
sec03-fluentd-1  | {"time":"2022-10-29 10:19:23","level":"warn","message":"no patterns matched tag=\"fluent.info\"","worker_id":7}
sec03-fluentd-1  | {"time":"2022-10-29 10:19:23","level":"warn","message":"no patterns matched tag=\"fluent.debug\"","worker_id":4}
sec03-fluentd-1  | 2022-10-29 10:19:23.204320511 +0000 fluent.debug: {"instance":2140,"stage_size":0,"queue_size":0,"message":"[output_docker1] buffer started instance=2140 stage_size=0 queue_size=0"}
sec03-fluentd-1  | 2022-10-29 10:19:23.198793609 +0000 fluent.debug: {"message":"[output1] enqueue_thread actually running"}
sec03-fluentd-1  | 2022-10-29 10:19:23.199129283 +0000 fluent.debug: {"message":"[output1] flush_thread actually running"}
sec03-fluentd-1  | 2022-10-29 10:19:23.199258976 +0000 fluent.debug: {"message":"[output_docker1] flush_thread actually running"}
sec03-fluentd-1  | 2022-10-29 10:19:23.204845726 +0000 fluent.info: {"port":24224,"bind":"0.0.0.0","message":"[input1] listening port port=24224 bind=\"0.0.0.0\""}
sec03-fluentd-1  | 2022-10-29 10:19:23.204943392 +0000 fluent.debug: {"message":"[output_docker1] flush_thread actually running"}
sec03-fluentd-1  | 2022-10-29 10:19:23.205303927 +0000 fluent.debug: {"message":"[output_docker1] enqueue_thread actually running"}
sec03-fluentd-1  | 2022-10-29 10:19:23.199886819 +0000 fluent.debug: {"message":"[output_docker1] enqueue_thread actually running"}
sec03-fluentd-1  | {"time":"2022-10-29 10:19:23","level":"warn","message":"no patterns matched tag=\"fluent.debug\"","worker_id":4}
sec03-fluentd-1  | {"time":"2022-10-29 10:19:23","level":"warn","message":"no patterns matched tag=\"fluent.debug\"","worker_id":7}
sec03-fluentd-1  | 2022-10-29 10:19:23.207018404 +0000 fluent.info: {"worker":4,"message":"fluentd worker is now running worker=4"}
sec03-fluentd-1  | 2022-10-29 10:19:23.200556338 +0000 fluent.info: {"worker":7,"message":"fluentd worker is now running worker=7"}
```


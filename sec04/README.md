設定ファイルの書式と構造
====================

## リテラル
リテラルの細かな仕様は意識したことなかったけど、意識しておくと設定ファイルの理解や書き間違いが避けられそう。

## パラメータ
レコードキーが馴染みがないので確認が必要。
自分が普段利用している設定にはレコードキーを用いたパラメータは存在しなかった。
以降の章でレコードキーは登場するのでここでは検証しない。

## ディレクティブ
同一ディレクティブ内のパラメータは後勝ちで、matchディレクティブのイベント処理は先勝ち、
filterディレクティブやlabelディレクティブはすべてのイベントに対応するなど、
パラメータやディレクティブによって動作が異なるのは、注意が必要。

labelディレクティブにおける `@FLUENT_LOG` の説明について、 `<match fluent.**>` より `<label @FLUENT_LOG>` の方がわかりやすいと説明がある。
一方で、fluent v1.15.1で動作確認すると、 `<match fluent.**>` は非推奨なので `<label @FLUENT_LOG>` を使えと、より強い表現をしている。
わかりやすさだけでなく、より積極的に `<label @FLUENT_LOG>` を利用した方がよさそう。

```
{"time":"2022-11-06 12:34:18 +0000","level":"warn","message":"define <match fluent.**> to capture fluentd logs in top level is deprecated. Use <label @FLUENT_LOG> instead"}
```

## 予約語パラメータ
予約語が@付きで表現されること、@labelパラメータのラベル名には@付きで定義することはすぐ忘れて混同するので注意が必要。

@idパラメータを利用したときの設定ファイルの不備による警告例は以下の通り。
それぞれ、`output_docker1` と `output1` に関する不備であるとわかりやすいので、@idパラメータを上手く活用するとわかりやすいことがわかる。

```
{"time":"2022-11-06 12:34:18 +0000","level":"warn","message":"[output_docker1] 'time_format' specified without 'time_key', will be ignored"}
{"time":"2022-11-06 12:34:18 +0000","level":"warn","message":"[output1] 'time_format' specified without 'time_key', will be ignored"}
```


## YAMLによる設定ファイル記述
以下の手順で設定ファイルの検証を行う。

```
$ docker run -it --rm -v $(pwd)/fluent.yaml:/fluent.yaml fluent/fluentd:v1.15.1-1.0 --dry-run -c fluent.yaml
fluentd --dry-run -c fluent.yaml
{"time":"2022-11-06 13:21:12 +0000","level":"info","message":"parsing config file is succeeded path=\"fluent.yaml\""}
{"time":"2022-11-06 13:21:12 +0000","level":"info","message":"gem 'fluentd' version '1.15.1'"}
{"time":"2022-11-06 13:21:12 +0000","level":"info","message":"starting fluentd-1.15.1 as dry run mode ruby=\"3.1.2\""}
{"time":"2022-11-06 13:21:12 +0000","level":"info","message":"using configuration file: <ROOT>\n  <system>\n    process_name \"Data collector\"\n    workers 8\n    root_dir \"/fluentd\"\n    log_level debug\n    suppress_config_dump false\n    rpc_endpint 0.0.0.0:24226\n    enable_ge_dump true\n    <log>\n      format json\n      time_format \"%Y-%m-%d %H:%M:%S %z\"\n      rotate_age 10\n      rotate_size 100k\n    </log>\n  </system>\n  <source>\n    @type forward\n    @label @mainstream\n    @id input1\n    port 24224\n  </source>\n  <label @FLUENT_LOG>\n    <match **>\n      @type stdout\n    </match>\n  </label>\n  <label @mainstream>\n    <match docker.**>\n      @type file\n      @id output_docker1\n      path \"/fluentd/log/docker.*.log\"\n      symlink_path \"/fluentd/log/docker.log\"\n      append true\n      time_slice_format %Y%m%d\n      time_slice_wait 1m\n      time_format %Y%m%dT%H%M%S%z\n      time_key timestamp\n      <buffer time>\n        timekey_wait 1m\n        timekey 86400\n      </buffer>\n      <inject>\n        time_key timestamp\n        time_format %Y%m%dT%H%M%S%z\n      </inject>\n    </match>\n    <match **>\n      @type file\n      @id output1\n      path \"/fluentd/log/data.*.log\"\n      symlink_path \"/fluentd/log/data.log\"\n      append true\n      time_slice_format %Y%m%d\n      time_slice_wait 10m\n      time_format %Y%m%dT%H%M%S%z\n      time_key timestamp\n      <buffer time>\n        timekey_wait 10m\n        timekey 86400\n      </buffer>\n      <inject>\n        time_key timestamp\n        time_format %Y%m%dT%H%M%S%z\n      </inject>\n    </match>\n  </label>\n</ROOT>"}
{"time":"2022-11-06 13:21:12 +0000","level":"info","message":"finished dry run mode"}
```

ファイル名として、 `fluentd.yaml` および `fluentd.conf` とあるが、
3章などでは `fluent.conf` として登場していること、[公式ドキュメント](https://docs.fluentd.org/configuration/config-file-yaml)も `fluent.yaml` として説明記載があることから、 `fluent.yaml` が正式名称と考えてよさそう。

ディレクティブのタグについて、`$tag`で指定することは書籍中でも明記されているが、labelディレクティブのみ `$tag` ではなく `$name` で指定する。
設定ファイルとしては登場するが説明ないので注意が必要。
間違えて `$tag` を指定した場合エラーメッセージは出力されるがわかりにくい(`Missing symbol argument on <label> directive`)ので注意が必要。

```
config:
  - label:
      $name: '@mainstream'
      config:
...

# $nameではなく$tagを指定した場合のエラーメッセージ

{"time":"2022-11-06 12:31:46 +0000","level":"error","message":"config error file=\"fluent.yaml\" error_class=Fluent::ConfigError error=\"Missing symbol argument on <label> directive\""}
```

YAML形式では、Conf形式では不要だった文字列についてもシングルクオートでエスケープする必要がある点に注意。

- labelパラメータ名 (予約語パラメータとは異なり、`$mainstream` ではなく `@mainstream` のように @ のままである点に注意)
- filterタグ名 (`**`の場合のみ。 `docker.**` のように他の文字列を含む場合は不要)
- % を含む文字列 (time_formatなど)

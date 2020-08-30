## 終わった後の感想

ISUCONに触るのは初めてのこころみで、何よりとても楽しかったです。今まであまり知らなかったnginxをさわるキッカケが出来たり、ずっとやりたいと思っていたMySQLやLinuxパフォーマンス分析ツールの練習ができるなど、やってよかったと思います。

それで今回取り組んだ一人ISUCONの内容自体は、スムーズではなかったですね…生来の手の遅さと合わせて作戦の失敗で当初のボトルネックを解決できず、第2のボトルネックへと移行できませんでした。
06:25まではやや遅いながらも当初のボトルネックの原因特定までいき、そこまではよかったのですが、その後4時間もかけてエラーや自分のしこんだソースコードのバグと戦ううちにどんどん時間が溶けてなくなりました。

途中計算の結果をスクリプトを使ってDBに格納する作戦だったのですが、よく考えれば単純計算であきらかに時間が足りないのにそれに気づいてなかったです
> 計算すれば一件あたり5秒だとしても、7500件x5秒かかるので、どう考えても時間が足りないのですけどんな単純なことも見落とすものですね…

事前準備では色々なツールを素振りしていて、JavaのFlame Graph https://github.com/brendangregg/FlameGraph も準備していたのに使うところまでいけませんでした。
まる2日以上事前準備にかけて、自分でAzureのプロビジョニングを行った関係上、必然的に一部ソースコードを「見てしまった」のですが、それでも全然時間足りなかったですね。上位入賞者は普段の仕事でそもそも慣れた作業を更に素早くこなす訓練を事前に行っているのだと思います。

## 開始時刻

2020/08/30 日曜日 03:30 一人Isucon6予選開始

## 事前準備

[Isuconブログの記事](http://isucon.net/archives/48680557.html)[README.md](./README.md)と[provisiningディレクトリ](./provisioning/)の中身を参考にAzureのセットアップをしました。

> 今回は、予選環境にAzureを使いましたが、AzureではVMイメージの直接の提供が難しい代わりに、Deploy to Azure Buttonという便利な仕組みがあり、それを利用しました。

残念ながらDeploy to Azure Buttonでエラーが出てしまいました。 http://isucon6q.example.com/ などのリソースがシャットダウンされたことが原因と思われますが、エラーを解決するより手動でAzureをセットアップする方が早いと思ったので、手動セットアップをしています。

![VMの構成](images/vms.png)

本来であればprovisiningディレクトリにあるように4台のAzure VMが使われたようですが:

- portalはwebサイトをホストし、ベンチマーカーのスケジューリングやスコア記録に使われた
- qualifierは用途不明、おそらくコンテストあとの追試用？

からimageとbenchという2つのVMだけ準備し、それぞれの上で走るプロセスは以下のようにしました。

![プロセスの構成](images/procs.png)

benchとworkerはどちらもGoプロセスで、強調して動くのですが「workerがportalのwebサイトとやりとりし、workerを起動する」というしくのため、workerがなくてもbenchさえあれば手動で走らせてスコアを取得できます。

Azureの"deploy from template"でデプロイを試みたところprovisioningディレクトリ以下にあるAnsibleプレイブックが結構失敗している模様...(VMの/var/log/cloud-init-output.log)
各deploy.jsonのcustomdataをbase64でデコードして確認したところ、init.shを実行していただけなので、手動でinit.shと同様の作業をおこないました。

## 03:30 初回ベンチマーク実行 

```
bench -target "http://10.0.1.4"
2020/08/30 03:29:28 start pre-checking
2020/08/30 03:29:58 pre-check finished and start main benchmarking
2020/08/30 03:30:28 benchmarking finished
{"pass":true,"score":0,"success":269,"fail":26,"messages":["Response code should be 200, got 500, data:  (GET /)","Response code should be 302, got 500, data:  (POST /login)","Response code should be 302, got 500, data: アイドルメーカー・エッジ (POST /keyword)","Response code should be 302, got 500, data: 北消防 署 (POST /keyword)","Response code should be 400, got 500, data: 2001年宇宙の旅 (POST /keyword)","Response code should be 400, got 500, data: 岩原裕二 (POST /keyword)","Response code should be 400, got 500, data: 高橋留美子 (POST /keyword)","Response code should be 403, got 500, data:  (POST /login)","リクエスト がタイムアウトしました (GET /)"]}
```

pre-checkingは成功、successも269あるので、isuda、 isutar、 isupamは正しく動いているようですが、スコアはゼロ点スタートですね。一人ISUCON予選前に約束していたように、作業時間は、初回のベンチマークがクラウド上で走ってから最大8時間とのことなので同日午前11:30までですね。ここまでの作業でGoのbenchを見てしまったり、provisioningディレクトリの中身からVMとプロセスの構成を知るなど、若干カンニング気味ではあります…。isuda/isutarに関してはほぼ中身はみてないですが。

## 03:50 初回ベンチマークの分析と分析ツールのセットアップ

このisuda/isutarアプリケーションは以下のようなページを返すようです。

![トップページ](images/top.png)

初回は`Response code should be 200, got 500`のように500系エラーを返していたようですが、何度かベンチマーカーを走らせた結果がこちら、リクエストがタイム・アウトしました、という現象が頻発しているようです。

```
{"pass":true,"score":0,"success":210,"fail":39,"messages":["リクエストがタイムアウトしました (GET /)","リクエストがタイムアウトしました (GET /keyword/オートマチック限定免許)","リクエストがタイムアウトしました (GET /keyword/サミュエル・ゴーズミット)","リクエストがタイムアウトしました (GET /keyword/南蟹谷村)","リクエストがタイムアウトしました (GET /keyword/国道138号)","リクエストがタイムアウトしました (GET /keyword/菅山かおる)","リクエストがタイムアウトしま した (GET /keyword/輪状甲状筋)","リクエストがタイムアウトしました (GET /keyword/黒田倫弘)","リクエストがタイムアウトしました (POST /keyword)","リクエストが タイムアウトしました (POST /login)"]}
```

どうもタイム・アウトするほど遅いページがある？ブラウザからトップページにアクセスした動作も10秒以上かかってからようやくトップページが表示されるなど、だいぶ重たい印象です。


### 追加ツールのインストール

https://github.com/tkuchiki/alp/blob/69333d16570d17ef1b9c0820cf0ab7c786a616df/README.ja.md
ここでHTTPのエンドポイントごとの統計をとるためにalpをインストールします。上の方法を参考にnginx.confを書き換えて、/etc/nginx/nginx-isucon6.confに置きます。

うむ、おそいですね。10秒以上かかっているページがたくさんあります。

```
isucon@isucon6-webapp:~$ cat /var/log/nginx/access.log | alp ltsv

+-------+-----+-----+-----+-----+-----+--------+-------------------------------------+--------+--------+---------+--------+--------+--------+--------+--------+------------+------------+-------------+------------+
| COUNT | 1XX | 2XX | 3XX | 4XX | 5XX | METHOD |                 URI                 |  MIN   |  MAX   |   SUM   |  AVG   |   P1   |  P50   |  P99   | STDDEV | MIN(BODY)  | MAX(BODY)  |  SUM(BODY)  | AVG(BODY)  |
+-------+-----+-----+-----+-----+-----+--------+-------------------------------------+--------+--------+---------+--------+--------+--------+--------+--------+------------+------------+-------------+------------+
|     1 |   0 |   1 |   0 |   0 |   0 | GET    | /keyword/宮津天橋立インターチェンジ |  2.705 |  2.705 |   2.705 |  2.705 |  2.705 |  2.705 |  2.705 |  0.000 |   5738.000 |   5738.000 |    5738.000 |   5738.000 |
|     1 |   0 |   0 |   0 |   1 |   0 | GET    | /keyword/国道138号                  | 15.005 | 15.005 |  15.005 | 15.005 | 15.005 | 15.005 | 15.005 |  0.000 |      0.000 |      0.000 |       0.000 |      0.000 |
|     1 |   0 |   0 |   0 |   0 |   1 | GET    | /keyword/鳥取中央農業協同組合       |  0.204 |  0.204 |   0.204 |  0.204 |  0.204 |  0.204 |  0.204 |  0.000 |  37062.000 |  37062.000 |   37062.000 |  37062.000 |
|     1 |   0 |   0 |   0 |   1 |   0 | GET    | /keyword/オートマチック限定免許     | 15.001 | 15.001 |  15.001 | 15.001 | 15.001 | 15.001 | 15.001 |  0.000 |      0.000 |      0.000 |       0.000 |      0.000 |
|     1 |   0 |   0 |   0 |   1 |   0 | GET    | /keyword/黒田倫弘                   | 14.997 | 14.997 |  14.997 | 14.997 | 14.997 | 14.997 | 14.997 |  0.000 |      0.000 |      0.000 |       0.000 |      0.000 |
|     1 |   0 |   0 |   0 |   1 |   0 | GET    | /keyword/サミュエル・ゴーズミット   | 14.997 | 14.997 |  14.997 | 14.997 | 14.997 | 14.997 | 14.997 |  0.000 |      0.000 |      0.000 |       0.000 |      0.000 |
|     1 |   0 |   1 |   0 |   0 |   0 | GET    | /keyword/605年                      |  4.644 |  4.644 |   4.644 |  4.644 |  4.644 |  4.644 |  4.644 |  0.000 |   2236.000 |   2236.000 |    2236.000 |   2236.000 |
|     1 |   0 |   1 |   0 |   0 |   0 | GET    | /keyword/692年                      |  4.956 |  4.956 |   4.956 |  4.956 |  4.956 |  4.956 |  4.956 |  0.000 |   3816.000 |   3816.000 |    3816.000 |   3816.000 |
|     1 |   0 |   1 |   0 |   0 |   0 | GET    | /keyword/宇都宮信房                 |  4.224 |  4.224 |   4.224 |  4.224 |  4.224 |  4.224 |  4.224 |  0.000 |   3407.000 |   3407.000 |    3407.000 |   3407.000 |
|     1 |   0 |   1 |   0 |   0 |   0 | GET    | /keyword/伊藤里奈                   |  5.624 |  5.624 |   5.624 |  5.624 |  5.624 |  5.624 |  5.624 |  0.000 |   6090.000 |   6090.000 |    6090.000 |   6090.000 |
|     1 |   0 |   0 |   1 |   0 |   0 | GET    | /css/main.css                       |  0.428 |  0.428 |   0.428 |  0.428 |  0.428 |  0.428 |  0.428 |  0.000 |      0.000 |      0.000 |       0.000 |      0.000 |
|     1 |   0 |   1 |   0 |   0 |   0 | GET    | /keyword/東洋英和女学院大学         |  5.624 |  5.624 |   5.624 |  5.624 |  5.624 |  5.624 |  5.624 |  0.000 |   8685.000 |   8685.000 |    8685.000 |   8685.000 |
|     1 |   0 |   0 |   1 |   0 |   0 | GET    | /js/star.js                         |  0.212 |  0.212 |   0.212 |  0.212 |  0.212 |  0.212 |  0.212 |  0.000 |      0.000 |      0.000 |       0.000 |      0.000 |
|     1 |   0 |   1 |   0 |   0 |   0 | GET    | /keyword/浅間山                     |  1.505 |  1.505 |   1.505 |  1.505 |  1.505 |  1.505 |  1.505 |  0.000 |  44562.000 |  44562.000 |   44562.000 |  44562.000 |
|     1 |   0 |   0 |   0 |   1 |   0 | GET    | /keyword/源仲綱                     | 13.692 | 13.692 |  13.692 | 13.692 | 13.692 | 13.692 | 13.692 |  0.000 |      0.000 |      0.000 |       0.000 |      0.000 |
|     2 |   0 |   2 |   0 |   0 |   0 | GET    | /keyword/平尾山                     |  1.740 |  2.261 |   4.001 |  2.001 |  2.261 |  2.261 |  2.261 |  0.261 |   6365.000 |   6365.000 |   12730.000 |   6365.000 |
|     2 |   0 |   2 |   0 |   0 |   0 | GET    | /keyword/日本列島ほっと通信         |  2.896 |  3.213 |   6.109 |  3.054 |  3.213 |  3.213 |  3.213 |  0.159 |  23315.000 |  23315.000 |   46630.000 |  23315.000 |
|     2 |   0 |   0 |   0 |   2 |   0 | GET    | /keyword/南蟹谷村                   |  1.292 | 15.000 |  16.292 |  8.146 | 15.000 | 15.000 | 15.000 |  6.854 |      0.000 |      0.000 |       0.000 |      0.000 |
|     2 |   0 |   0 |   0 |   2 |   0 | GET    | /keyword/菅山かおる                 |  4.284 | 15.001 |  19.285 |  9.643 | 15.001 | 15.001 | 15.001 |  5.358 |      0.000 |      0.000 |       0.000 |      0.000 |
|     2 |   0 |   0 |   0 |   2 |   0 | GET    | /keyword/輪状甲状筋                 | 11.752 | 15.001 |  26.753 | 13.377 | 15.001 | 15.001 | 15.001 |  1.624 |      0.000 |      0.000 |       0.000 |      0.000 |
|     2 |   0 |   2 |   0 |   0 |   0 | GET    | /keyword/ウーズ                     |  4.476 |  4.644 |   9.120 |  4.560 |  4.476 |  4.476 |  4.476 |  0.084 |   2273.000 |   2273.000 |    4546.000 |   2273.000 |
|     2 |   0 |   2 |   0 |   0 |   0 | GET    | /keyword/毛細管現象                 |  4.072 |  5.496 |   9.568 |  4.784 |  5.496 |  5.496 |  5.496 |  0.712 |   5002.000 |   5002.000 |   10004.000 |   5002.000 |
|     2 |   0 |   2 |   0 |   0 |   0 | GET    | /keyword/内田修平                   |  5.408 |  5.908 |  11.316 |  5.658 |  5.908 |  5.908 |  5.908 |  0.250 |   6528.000 |   6528.000 |   13056.000 |   6528.000 |
|     2 |   0 |   2 |   0 |   0 |   0 | GET    | /keyword/イギリス政府               |  6.264 |  6.628 |  12.892 |  6.446 |  6.628 |  6.628 |  6.628 |  0.182 |   7554.000 |   7554.000 |   15108.000 |   7554.000 |
|     2 |   0 |   1 |   0 |   0 |   1 | GET    | /initialize                         |  0.088 |  0.352 |   0.440 |  0.220 |  0.088 |  0.088 |  0.088 |  0.132 |     15.000 |  33562.000 |   33577.000 |  16788.500 |
|     3 |   0 |   0 |   0 |   3 |   0 | GET    | /css/scalate/errors.css             |  0.064 |  0.072 |   0.204 |  0.068 |  0.068 |  0.068 |  0.064 |  0.003 |    306.000 |    306.000 |     918.000 |    306.000 |
|    15 |   0 |  14 |   1 |   0 |   0 | GET    | /js/jquery.min.js                   |  0.212 |  1.180 |   7.580 |  0.505 |  0.344 |  0.964 |  0.752 |  0.275 |  86351.000 |  86351.000 | 1208914.000 |  80594.267 |
|    15 |   0 |  14 |   1 |   0 |   0 | GET    | /js/bootstrap.min.js                |  0.228 |  1.264 |   8.104 |  0.540 |  0.336 |  0.872 |  0.616 |  0.261 |  28631.000 |  28631.000 |  400834.000 |  26722.267 |
|    15 |   0 |  14 |   1 |   0 |   0 | GET    | /img/star.gif                       |  0.224 |  0.708 |   7.236 |  0.482 |  0.356 |  0.656 |  0.592 |  0.168 |     93.000 |     93.000 |    1302.000 |     86.800 |
|    15 |   0 |  14 |   1 |   0 |   0 | GET    | /css/bootstrap-responsive.min.css   |  0.208 |  1.064 |   7.324 |  0.488 |  0.304 |  0.520 |  0.632 |  0.211 |  16849.000 |  16849.000 |  235886.000 |  15725.733 |
|    18 |   0 |  18 |   0 |   0 |   0 | GET    | /favicon.ico                        |  0.068 |  1.056 |   7.608 |  0.423 |  0.088 |  0.248 |  0.620 |  0.263 |   1092.000 |   1092.000 |   19656.000 |   1092.000 |
|    29 |   0 |  28 |   1 |   0 |   0 | GET    | /css/bootstrap.min.css              |  0.220 |  1.132 |  15.192 |  0.524 |  0.300 |  1.076 |  0.548 |  0.240 | 106015.000 | 106015.000 | 2968420.000 | 102359.310 |
|    30 |   0 |   3 |   0 |  24 |   3 | GET    | /                                   |  0.068 | 15.001 | 395.717 | 13.191 |  0.068 | 15.001 | 15.001 |  4.501 |      0.000 |  83571.000 |  349885.000 |  11662.833 |
|    38 |   0 |   0 |  10 |  28 |   0 | POST   | /keyword                            |  0.360 |  3.001 |  40.874 |  1.076 |  0.616 |  0.664 |  0.716 |  0.720 |      0.000 |      5.000 |     125.000 |      3.289 |
|    69 |   0 |   0 |  49 |  13 |   7 | POST   | /login                              |  0.132 |  3.000 |  56.851 |  0.824 |  0.164 |  0.624 |  0.828 |  0.723 |      0.000 |  35641.000 |  230371.000 |   3338.710 |
+-------+-----+-----+-----+-----+-----+--------+-------------------------------------+--------+--------+---------+--------+--------+--------+--------+--------+------------+------------+-------------+------------+
```

それから、これを参考にMySQLのslow-query-logを設定し...

https://www.youtube.com/watch?v=noFn2sgQiNw

```
mysql> SET GLOBAL slow_query_log='ON';
mysql> SHOW GLOBAL VARIABLES LIKE '%slow_query%';
+---------------------+----------------------------------------+
| Variable_name       | Value                                  |
+---------------------+----------------------------------------+
| slow_query_log      | ON                                     |
| slow_query_log_file | /var/lib/mysql/isucon6-webapp-slow.log |
+---------------------+----------------------------------------+

mysql> SET GLOBAL long_query_time=0;
mysql> select @@global.long_query_time;
+--------------------------+
| @@global.long_query_time |
+--------------------------+
|                 0.000000 |
+--------------------------+
1 row in set (0.00 sec)
```

...pt-query-digestも入れておきます。

https://gist.github.com/yuuki/aef3b7c91f23d1f02aaa266ebe858383


```
sudo cat /var/lib/mysql/isucon6-webapp-slow.log | pt-query-digest
cat: less: No such file or directory
Reading from STDIN ...

# 510ms user time, 0 system time, 35.21M rss, 102.94M vsz
# Current date: Sat Aug 29 19:33:37 2020
# Hostname: isucon6-webapp
# Files: STDIN
# Overall: 1.95k total, 21 unique, 19.45 QPS, 2.89x concurrency __________
# Time range: 2020-08-29T19:14:19 to 2020-08-29T19:15:59
# Attribute          total     min     max     avg     95%  stddev  median
# ============     ======= ======= ======= ======= ======= ======= =======
# Exec time           289s    39us      3s   149ms      2s   490ms    89us
# Lock time          153ms       0    95ms    78us    93us     2ms       0
# Rows sent          1.34M       0   6.94k  723.52   6.63k   2.00k       0
# Rows examine       2.85M       0  13.88k   1.50k  13.78k   4.20k       0
# Query size       130.25k      13  16.41k   68.57   65.89  454.82   31.70

# Profile
# Rank Query ID                       Response time  Calls R/Call V/M   It
# ==== ============================== ============== ===== ====== ===== ==
#    1 0x28BC6892F47C29FC6C987DBB9... 287.5218 99.3%   198 1.4521  0.38 SELECT entry
# MISC 0xMISC                           1.9607  0.7%  1747 0.0011   0.0 <20 ITEMS>

# Query 1: 2 QPS, 2.90x concurrency, ID 0x28BC6892F47C29FC6C987DBB94BEC091 at byte 357136
# This item is included in the report because it matches --limit.
# Scores: V/M = 0.38
# Time range: 2020-08-29T19:14:20 to 2020-08-29T19:15:59
# Attribute    pct   total     min     max     avg     95%  stddev  median
# ============ === ======= ======= ======= ======= ======= ======= =======
# Count         10     198
# Exec time     99    288s   214ms      3s      1s      3s   746ms      2s
# Lock time     11    17ms    48us   545us    85us   108us    46us    76us
# Rows sent     99   1.34M   6.93k   6.94k   6.94k   6.63k       0   6.63k
# Rows examine  94   2.68M  13.87k  13.88k  13.88k  13.78k       0  13.78k
# Query size     9  12.96k      67      67      67      67       0      67
# String:
# Databases    isuda
# Hosts        localhost
# Users        root
# Query_time distribution
#   1us
#  10us
# 100us
#   1ms
#  10ms
# 100ms  ##################################
#    1s  ################################################################
#  10s+
# Tables
#    SHOW TABLE STATUS FROM `isuda` LIKE 'entry'\G
#    SHOW CREATE TABLE `isuda`.`entry`\G
# EXPLAIN /*!50100 PARTITIONS*/
SELECT * FROM entry
        ORDER BY CHARACTER_LENGTH(keyword) DESC\G
```        


ツールのインストールにだいぶ手間取ってしまいました。まだベンチマークの分析は続きますが、いったんここでgit commit

## 5:00 CPUとメモリどちらがボトルネックか調査

[Netflix - Linux Performance Analysis in 60,000 Milliseconds](https://netflixtechblog.com/linux-performance-analysis-in-60-000-milliseconds-accc10403c55) に従って、ベンチマークを再び走らせながらリソースの利用を見ていきます。

topコマンドから、PID=45710のjavaプロセス=isudaとPID=45627のmysqldがCPUを奪い合っているようですね。メモリSwapは発生しておらず、ボトルネックはCPUである可能性が高く、isudaプロセスがなにか重たいことをしているのかもしれません。
    
```
> top
top - 19:49:25 up 21:09,  2 users,  load average: 4.07, 0.98, 0.44
Tasks: 125 total,   2 running,  71 sleeping,   0 stopped,   0 zombie
%Cpu(s): 96.3 us,  2.2 sy,  0.0 ni,  1.3 id,  0.0 wa,  0.0 hi,  0.2 si,  0.0 st
KiB Mem :  4017072 total,   577560 free,  2452560 used,   986952 buff/cache
KiB Swap:        0 total,        0 free,        0 used.  1216348 avail Mem

   PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
 45710 isucon    20   0 3508996 1.176g  19336 S 169.0 30.7   3:18.21 java
 45627 mysql     20   0 1791972 460344  30912 S  21.3 11.5   0:59.63 mysqld
 45774 isucon    20   0 3440488 466436  17680 S   3.0 11.6   0:12.63 java
 13222 isucon    20   0   14492  10516   4912 S   2.7  0.3   0:08.11 isupam_linux
  1822 root      20   0  222728  22232   7136 S   1.0  0.6   2:09.59 python3
 45228 www-data  20   0   24548   3136   1908 S   0.7  0.1   0:00.07 nginx
 45229 www-data  20   0   24524   3144   1908 S   0.3  0.1   0:00.07 nginx
     1 root      20   0   37880   5812   3968 S   0.0  0.1   0:04.34 systemd
     2 root      20   0       0      0      0 S   0.0  0.0   0:00.01 kthreadd
     4 root       0 -20       0      0      0 I   0.0  0.0   0:00.00 kworker/0:0H
```

vmstatをみてもswapは発生していません。csつまりcontext switchの値も低いので、スレッド切り替えが忙しいタイプの詰まり方じゃなさそうです。(一般的な話では、context switchは数百万あってもアプリケーションによっては通常運転の場合もあったはず。)

```
vmstat 1
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
32  0      0 598848 164136 793832    0    0     0   132   90 1733 97  3  0  0  0
 2  0      0 613512 164144 793484    0    0     8   224  131 1823 97  2  1  0  0
19  0      0 606584 164148 793568    0    0     0   108   99 1651 95  3  3  0  0
```

というわけで、isudaを中心に更に詳しく見ていき、なぜそれが複数のエンドポイントのレスポンスにおいて10秒以上かかるほど重くなっているのか見ていきましょう。

## 5:15 isudaのなにが遅いのかを調査

さて、nginxのログから統計を再びとって、AVGで10秒以上かかっているエンドポイントをいくつか適当に抜き出してくるとこんなかんじです。/keyword/***とトップページ / のGETが遅いようです。

```
+-------+-----+-----+-----+-----+-----+--------+-----------------------------------------+--------+--------+----------+--------+--------+--------+--------+--------+------------+------------+--------------+------------+
| COUNT | 1XX | 2XX | 3XX | 4XX | 5XX | METHOD |                   URI                   |  MIN   |  MAX   |   SUM    |  AVG   |   P1   |  P50   |  P99   | STDDEV | MIN(BODY)  | MAX(BODY)  |  SUM(BODY)   | AVG(BODY)  |
+-------+-----+-----+-----+-----+-----+--------+-----------------------------------------+--------+--------+----------+--------+--------+--------+--------+--------+------------+------------+--------------+------------+
|     1 |   0 |   0 |   0 |   1 |   0 | GET    | /keyword/サミュエル・ゴーズミット                 | 14.997 | 14.997 |   14.997 | 14.997 | 14.997 | 14.997 | 14.997 |  0.000 |      0.000 |      0.000 |        0.000 |      0.000 |
|     1 |   0 |   0 |   0 |   1 |   0 | GET    | /keyword/黒田倫弘                        | 14.997 | 14.997 |   14.997 | 14.997 | 14.997 | 14.997 | 14.997 |  0.000 |      0.000 |      0.000 |        0.000 |      0.000 |
|     1 |   0 |   0 |   0 |   1 |   0 | GET    | /keyword/オートマチック限定免許                | 15.001 | 15.001 |   15.001 | 15.001 | 15.001 | 15.001 | 15.001 |  0.000 |      0.000 |      0.000 |        0.000 |      0.000 |
|     1 |   0 |   0 |   0 |   1 |   0 | GET    | /keyword/国道138号                       | 15.005 | 15.005 |   15.005 | 15.005 | 15.005 | 15.005 | 15.005 |  0.000 |      0.000 |      0.000 |        0.000 |      0.000 |
|     1 |   0 |   0 |   0 |   1 |   0 | GET    | /keyword/源仲綱                          | 13.692 | 13.692 |   13.692 | 13.692 | 13.692 | 13.692 | 13.692 |  0.000 |      0.000 |      0.000 |        0.000 |      0.000 |
|    91 |   0 |  13 |   0 |  62 |  16 | GET    | /                                       |  0.068 | 15.004 | 1086.669 | 11.941 |  0.068 | 15.001 |  3.252 |  5.254 |  37993.000 |  85012.000 |  1303529.000 |  14324.495 |
```

しかしなぜかkeywordのPOSTはそんなに遅くありません。平均1秒

```
+-------+-----+-----+-----+-----+-----+--------+-----------------------------------------+--------+--------+----------+--------+--------+--------+--------+--------+------------+------------+--------------+------------+
| COUNT | 1XX | 2XX | 3XX | 4XX | 5XX | METHOD |                   URI                   |  MIN   |  MAX   |   SUM    |  AVG   |   P1   |  P50   |  P99   | STDDEV | MIN(BODY)  | MAX(BODY)  |  SUM(BODY)   | AVG(BODY)  |
+-------+-----+-----+-----+-----+-----+--------+-----------------------------------------+--------+--------+----------+--------+--------+--------+--------+--------+------------+------------+--------------+------------+
|   108 |   0 |   0 |  30 |  69 |   9 | POST   | /keyword                                |  0.160 |  3.001 |  109.339 |  1.012 |  0.616 |  0.676 |  1.484 |  0.604 |      5.000 |  23087.000 |   176609.000 |   1635.269 |
```

[keyword](https://github.com/richardimaoka/isucon6-qualify/blob/f100d0df3266b606de6200e5c0e3af4d2fc5ac86/webapp/scala/isuda/src/main/scala/isuda/Web.scala#L163L177)も[トップページ](https://github.com/richardimaoka/isucon6-qualify/blob/f100d0df3266b606de6200e5c0e3af4d2fc5ac86/webapp/scala/isuda/src/main/scala/isuda/Web.scala#L37L65))もGETはDBから読み込んで、テンプレートエンジンであるmustacheに渡しています。DBのクエリにはLIMITがかかっているのでムダなことをやっているわけではなさそう。
となると、mustacheがものすごく重い…？

[keywordのPOST](https://github.com/richardimaoka/isucon6-qualify/blob/f100d0df3266b606de6200e5c0e3af4d2fc5ac86/webapp/scala/isuda/src/main/scala/isuda/Web.scala#L71L96))はmustacheを使っておらず、そしてmustacheを使っているGETに比べ速いですね。

pt-query-digestを見ても、全クエリのなかの99.3%を占める最上位のクエリが…


```
# Profile
# Rank Query ID                           Response time   Calls R/Call V/M
# ==== ================================== =============== ===== ====== ===
#    1 0x28BC6892F47C29FC6C987DBB94BEC091 1067.9435 99.3%   702 1.5213  0.40 SELECT entry
#    2 0x83E24179728A838A89D66B9E4EC2F9BE    2.4206  0.2%    76 0.0318  0.02 SELECT entry
#    3 0x640CB72830F8ED000C85264D3C24C18B    1.4059  0.1%    76 0.0185  0.01 SELECT entry
#    4 0x4A7E975E688B9FA80A8B82E36A960BF8    0.6220  0.1%     3 0.2073  0.01 TRUNCATE
#    5 0x43CF265C9501E65D782FEBA311BC59ED    0.4833  0.0%   702 0.0007  0.00 SELECT star
#    6 0x53A8D707516DC33FFEBFBA3FAAD53B0D    0.4699  0.0%    13 0.0361  0.01 INSERT UPDATE entry
```

avgで2秒程度しかかかっていないので、レスポンスタイム10秒以上というのを考えると、現在のボトルネックがDBではないことがうかえます。

```
# Query 1: 0.23 QPS, 0.35x concurrency, ID 0x28BC6892F47C29FC6C987DBB94BEC091 at byte 896443
# This item is included in the report because it matches --limit.
# Scores: V/M = 0.40
# Time range: 2020-08-29T19:14:20 to 2020-08-29T20:05:06
# Attribute    pct   total     min     max     avg     95%  stddev  median
# ============ === ======= ======= ======= ======= ======= ======= =======
# Count         10     702
# Exec time     99   1068s   177ms      3s      2s      3s   781ms      2s
# Lock time     19    56ms    41us   653us    80us   103us    39us    76us
# Rows sent     99   4.76M   6.93k   6.94k   6.94k   6.63k    0.00   6.63k
# Rows examine  94   9.51M  13.87k  13.88k  13.88k  13.78k       0  13.78k
# Query size    13  45.93k      67      67      67      67       0      67
# String:
# Databases    isuda
# Hosts        localhost
# Users        root
# Query_time distribution
#   1us
#  10us
# 100us
#   1ms
#  10ms
# 100ms  ##############################
#    1s  ################################################################
#  10s+
# Tables
#    SHOW TABLE STATUS FROM `isuda` LIKE 'entry'\G
#    SHOW CREATE TABLE `isuda`.`entry`\G
# EXPLAIN /*!50100 PARTITIONS*/
SELECT * FROM entry
        ORDER BY CHARACTER_LENGTH(keyword) DESC\G
```        

mustache周りを改善すれば、レスポンスタイムが向上してスコアが上がりそうですね。取り掛かってみましょう。

## 6:25 最初のボトルネック特定？

どうやら、ボトルネックはmustacheというよりは以下の問題だったようです。GET /keyword のたびによばれてるこれが長大なStringに対して正規表現を走らせているっぽいです。
1リクエストごとにDBのentryテーブルの中身全体7000件 x 正規表現になるので、これは重い…

```scala
  def htmlify(content: String): String = {
    val entries = DB.readOnly { implicit session =>
      sql"""
        SELECT * FROM entry
        ORDER BY CHARACTER_LENGTH(keyword) DESC
      """.map(asEntry).list.apply()
    }
    val regex =
      entries.map(e => Pattern.quote(e.keyword)).mkString("(", "|", ")").r
    val hashBuilder = Map.newBuilder[String, String]
    val escaped = regex.replaceAllIn(content, m => {
      val kw = m.group(1)
      val hash = s"isuda_${sha1Hex(kw)}"
      hashBuilder += kw -> hash
      hash
    }).htmlEscaped
    hashBuilder.result.foldLeft(escaped) { case (content, (kw, hash)) =>
      val url = s"/keyword/${kw.uriEncoded}"
      val link = s"""<a href="$url">${kw.htmlEscaped}</a>"""
      content.replaceAllLiterally(hash, link)
    }.replaceAllLiterally("\n", "<br />\n")
  }
```

とりあえずこの部分を全部消去してhtmlifyが"aaaaaa"という固定文字を返すようにしたら、

```scala
 def htmlify(content: String): String = {
    "aaaaaaa"
  }
```

スコア200くらいになりました。いや固定Stringはダメだろうと思うけどwwwでも後で考えます。とりあえずゼロスコア脱出。

```
bench -target "http://10.0.1.4"
2020/08/30 06:46:44 start pre-checking
{"pass":false,"score":221,"success":88,"fail":2,"messages":["keyword: \"平尾山\" に \"浅間山\" からのリンクがありません (GET /keyword/浅間山)","keyword: \"弓長九天\" に \"11年\" へのリンクがありません (GET /keyword/弓長九天)"]}
```

## 7:10 htmlifyのさらなる分析

nginxのaccess.logより、全てのレスポンスが極端に速くなったことはわかりますが、当然ながらHTMLの中身が不正になっているのでスコアがあがっていないようです。

```
> cat /var/log/nginx/access.log | alp ltsv
+-------+-----+-----+-----+-----+-----+--------+-----------------------------------+-------+-------+-------+-------+-------+-------+-------+--------+------------+------------+-------------+------------+
| COUNT | 1XX | 2XX | 3XX | 4XX | 5XX | METHOD |                URI                |  MIN  |  MAX  |  SUM  |  AVG  |  P1   |  P50  |  P99  | STDDEV | MIN(BODY)  | MAX(BODY)  |  SUM(BODY)  | AVG(BODY)  |
+-------+-----+-----+-----+-----+-----+--------+-----------------------------------+-------+-------+-------+-------+-------+-------+-------+--------+------------+------------+-------------+------------+
|     1 |   0 |   1 |   0 |   0 |   0 | GET    | /initialize                       | 0.236 | 0.236 | 0.236 | 0.236 | 0.236 | 0.236 | 0.236 |  0.000 |     15.000 |     15.000 |      15.000 |     15.000 |
|     1 |   0 |   1 |   0 |   0 |   0 | GET    | /keyword/129年                    | 0.124 | 0.124 | 0.124 | 0.124 | 0.124 | 0.124 | 0.124 |  0.000 |   1636.000 |   1636.000 |    1636.000 |   1636.000 |
|     1 |   0 |   0 |   1 |   0 |   0 | GET    | /logout                           | 0.220 | 0.220 | 0.220 | 0.220 | 0.220 | 0.220 | 0.220 |  0.000 |      0.000 |      0.000 |       0.000 |      0.000 |
|     1 |   0 |   1 |   0 |   0 |   0 | GET    | /keyword/浅間山                   | 0.068 | 0.068 | 0.068 | 0.068 | 0.068 | 0.068 | 0.068 |  0.000 |   1684.000 |   1684.000 |    1684.000 |   1684.000 |
|     2 |   0 |   2 |   0 |   0 |   0 | GET    | /keyword/平尾山                   | 0.124 | 0.224 | 0.348 | 0.174 | 0.124 | 0.124 | 0.124 |  0.050 |   1684.000 |   1684.000 |    3368.000 |   1684.000 |
|     2 |   0 |   2 |   0 |   0 |   0 | GET    | /keyword/目黒川                   | 0.136 | 0.192 | 0.328 | 0.164 | 0.136 | 0.136 | 0.136 |  0.028 |   1680.000 |   1680.000 |    3360.000 |   1680.000 |
|     3 |   0 |   0 |   2 |   1 |   0 | POST   | /keyword                          | 0.316 | 0.452 | 1.112 | 0.371 | 0.316 | 0.316 | 0.344 |  0.059 |      0.000 |      5.000 |       5.000 |      1.667 |
|     6 |   0 |   6 |   0 |   0 |   0 | GET    | /                                 | 0.188 | 0.652 | 3.048 | 0.508 | 0.544 | 0.428 | 0.652 |  0.161 |   5115.000 |   5639.000 |   31959.000 |   5326.500 |
|     7 |   0 |   0 |   5 |   2 |   0 | POST   | /login                            | 0.148 | 0.264 | 1.620 | 0.231 | 0.148 | 0.236 | 0.252 |  0.035 |      0.000 |      0.000 |       0.000 |      0.000 |
|     8 |   0 |   8 |   0 |   0 |   0 | GET    | /favicon.ico                      | 0.140 | 0.552 | 3.176 | 0.397 | 0.352 | 0.552 | 0.496 |  0.131 |   1092.000 |   1092.000 |    8736.000 |   1092.000 |
|     8 |   0 |   8 |   0 |   0 |   0 | GET    | /js/jquery.min.js                 | 0.160 | 0.560 | 3.296 | 0.412 | 0.400 | 0.560 | 0.412 |  0.129 |  86351.000 |  86351.000 |  690808.000 |  86351.000 |
|     8 |   0 |   8 |   0 |   0 |   0 | GET    | /css/bootstrap-responsive.min.css | 0.080 | 0.544 | 3.224 | 0.403 | 0.404 | 0.544 | 0.388 |  0.135 |  16849.000 |  16849.000 |  134792.000 |  16849.000 |
|     8 |   0 |   8 |   0 |   0 |   0 | GET    | /img/star.gif                     | 0.160 | 0.592 | 3.488 | 0.436 | 0.432 | 0.592 | 0.396 |  0.120 |     93.000 |     93.000 |     744.000 |     93.000 |
|     8 |   0 |   8 |   0 |   0 |   0 | GET    | /js/bootstrap.min.js              | 0.144 | 0.548 | 3.120 | 0.390 | 0.360 | 0.548 | 0.356 |  0.109 |  28631.000 |  28631.000 |  229048.000 |  28631.000 |
|    11 |   0 |  10 |   0 |   1 |   0 | POST   | /stars                            | 0.052 | 0.348 | 1.688 | 0.153 | 0.188 | 0.348 | 0.072 |  0.089 |     15.000 |    289.000 |     439.000 |     39.909 |
|    16 |   0 |  16 |   0 |   0 |   0 | GET    | /css/bootstrap.min.css            | 0.092 | 0.552 | 6.192 | 0.387 | 0.364 | 0.552 | 0.092 |  0.124 | 106015.000 | 106015.000 | 1696240.000 | 106015.000 |
+-------+-----+-----+-----+-----+-----+--------+-----------------------------------+-------+-------+-------+-------+-------+-------+-------+--------+------------+------------+-------------+------------+
```

エラーを細かく見てみるとこのように「リンクがない」といわれていて

```
["keyword: \"129年\" に \"129年\" へのリンクがありません (GET /keyword/129年)","keyword: \"平尾山\" に \"浅間山\" からのリンクがありません (GET /keyword/浅間山)"]
```

つまりベンチマーカーは「リンクがない→不正なHTMLである→低スコアを出す」という仕組みになっていると予想しているのですが、じゃあその「リンクがない」が何を意味するのかもう少し詳しく考えます。GET /keyword はDBのentryテーブルを加工して(ハイパー)リンクがついたHTMLとして返しているのですが、そのHTMLを生成するメインコントテンツとなっているのが、データベースentryテーブルのdescriptionカラムに入っているプレーンテキストです。そしてプレーンテキストの中の単語に、その単語が別のエントリとして用意されていればハイパーリンクを付けることが期待されているようです。はてなキーワードみたいなものですね！

```
mysql> select * from entry limit 1;
+----+-----------+---------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------------+---------------------+
| id | author_id | keyword | description                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | updated_at          | created_at          |
+----+-----------+---------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------------+---------------------+
|  1 |         1 | 248年   |


== 他の紀年法 ==

* 干支 : 戊辰
* 日本
** 神功皇后摂政48年
** 皇紀908年
* 中国
** 魏 : 正始9年
** 蜀 : 延熙11年
** 呉 : 赤烏11年
* 朝鮮
** 高句麗 : 東川王22年、中川王元年
** 新羅 : 沾解王2年
** 百済 : 古尓王15年
** 檀紀2581年
* 仏滅紀元 : 791年
* ユダヤ暦 : 4008年 - 4009年

== できごと ==
*日本神話の天照大神の岩戸隠れのエピソードは、この年日本で見られた日食をモチーフにしたものとする解釈がある。

== 誕生 ==

== 死去 ==
*卑弥呼

*                                                                                                                                             | 2016-09-11 00:47:39 | 2016-09-11 00:47:39 |
+----+-----------+---------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------------+---------------------+
1 row in set (0.00 sec)
```

ベンチマーカーはリンクがちゃんと貼れているかをチェックしているので、以下のいずれかの方法をとると良さそうです。

- 1) ベンチマーカーがどうやってリンクを調べているかを把握して、不正なHTMLを返しつつ、ベンチマーカーロジックの抜け道をくぐるようにスコアへの影響は防ぐ
- 2) リンク作成済みのHTML化テキストをデータベースに予め保存しておく

2)の方がISUCON的には正攻法ですかね…？

とりあえず2の方針でテーブルにhtmlifyカラムを足してみます。

mysql> ALTER TABLE entry ADD COLUMN htmlify MEDIUMTEXT;
Query OK, 0 rows affected (0.12 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> describe entry;
+-------------+-----------------+------+-----+---------+----------------+
| Field       | Type            | Null | Key | Default | Extra          |
+-------------+-----------------+------+-----+---------+----------------+
| id          | bigint unsigned | NO   | PRI | NULL    | auto_increment |
| author_id   | bigint unsigned | NO   |     | NULL    |                |
| keyword     | varchar(191)    | YES  | UNI | NULL    |                |
| description | mediumtext      | YES  |     | NULL    |                |
| updated_at  | datetime        | NO   |     | NULL    |                |
| created_at  | datetime        | NO   |     | NULL    |                |
| htmlify     | mediumtext      | YES  |     | NULL    |                |
+-------------+-----------------+------+-----+---------+----------------+


## 10:00

苦労して上記2)の方針でコードを書き換え、

- entryテーブルにhtmlifyという作成済みHTMLを保存するカラムを追加
- GET keywordが来たときに
  - htmlifyが保存されていればそれを返す
  - htmlifyが空ならhtmlifyを生成して返すと同時にDBに保存(GETでDB保存は気持ち悪いですが…)

というコードを書いていて、ロジックの下記間違え対処や、バルクでcurlをはしらせて予めhtmlifyを保存する対応などをしていたのですが…何故か下記のエラーが出るように。
このエラー、mysql-connector-jのバージョンが5.xでmysql-serverのバージョンが8.0のときによく出るエラーなんですが、今回どちらも8.0だし、直前まで動いていたのになぜ…。

```
Unable to load authentication plugin 'caching_sha2_password'.
java.sql.SQLException: Unable to load authentication plugin 'caching_sha2_password'.


at com.mysql.cj.jdbc.exceptions.SQLError.createSQLException(SQLError.java:695)
at com.mysql.cj.jdbc.exceptions.SQLError.createSQLException(SQLError.java:663)
at com.mysql.cj.jdbc.exceptions.SQLError.createSQLException(SQLError.java:653)
at com.mysql.cj.jdbc.exceptions.SQLError.createSQLException(SQLError.java:638)
at com.mysql.cj.jdbc.exceptions.SQLError.createSQLException(SQLError.java:606)
at com.mysql.cj.jdbc.exceptions.SQLError.createSQLException(SQLError.java:624)
at com.mysql.cj.jdbc.exceptions.SQLError.createSQLException(SQLError.java:620)
at com.mysql.cj.jdbc.exceptions.SQLExceptionsMapping.translateException(SQLExceptionsMapping.java:78)
at com.mysql.cj.jdbc.ConnectionImpl.createNewIO(ConnectionImpl.java:1663)
at com.mysql.cj.jdbc.ConnectionImpl.<init>(ConnectionImpl.java:662)
at com.mysql.cj.jdbc.ConnectionImpl.getInstance(ConnectionImpl.java:352)
at com.mysql.cj.jdbc.NonRegisteringDriver.connect(NonRegisteringDriver.java:221)
at java.sql.DriverManager.getConnection(DriverManager.java:664)
at java.sql.DriverManager.getConnection(DriverManager.java:247)
at org.apache.commons.dbcp2.DriverManagerConnectionFactory.createConnection(DriverManagerConnectionFactory.java:77)
at org.apache.commons.dbcp2.PoolableConnectionFactory.makeObject(PoolableConnectionFactory.java:256)
at org.apache.commons.pool2.impl.GenericObjectPool.create(GenericObjectPool.java:868)
at org.apache.commons.pool2.impl.GenericObjectPool.borrowObject(GenericObjectPool.java:435)
at org.apache.commons.pool2.impl.GenericObjectPool.borrowObject(GenericObjectPool.java:363)
at org.apache.commons.dbcp2.PoolingDataSource.getConnection(PoolingDataSource.java:134)
at scalikejdbc.Commons2ConnectionPool.borrow(Commons2ConnectionPool.scala:48)
at scalikejdbc.DB$.readOnly(DB.scala:173)
```

エラーが出始める直前のスコア。Successは大幅に増えましたが、Failも増えたせいなのかスコアはゼロに戻ってしまいました…

```
{"pass":true,"score":0,"success":639,"fail":112,"messages":["Response code should be 200, got 500, data:  (GET /)","Response code should be 200, got 500, data:  (GET /keyword/613年)","Response code should be 400, got 500, data: 上座部仏教 (POST /keyword)","keyword: \"909年\" に \"10年\" へのリンクがありません (GET /keyword/909年)","keyword: \"917年\" に \"1003年\" へのリンクがありません (GET /keyword/917年)","keyword: \"Avex tune\" に \"13年\" へのリンクがありません (GET /keyword/Avex tune)","keyword: \"NYAOS\" に \"NYAOS\" へのリンクがありません (GET /keyword/NYAOS)","keyword: \"ノーンブワラムプー県\" に \"11年\" へのリンクがありません (GET /keyword/ノーンブワラムプー県)","keyword: \"ムニホヴォ・フラジシチェ\" に \"279年\" へのリンクがありません (GET /keyword/ムニホヴォ・フラジシチェ)","keyword: \"リトルシニア\" に \"10年\" へのリンクがありません (GET /keyword/リトルシニア)","keyword: \"個人協賛競 走\" に \"16年\" へのリンクがありません (GET /keyword/個人協賛競走)","keyword: \"出雲国風土記\" に \"11年\" へのリンクがありません (GET /keyword/出雲国 風土記)","keyword: \"博多南線\" に \"10年\" へのリンクがありません (GET /keyword/博多南線)","keyword: \"原ノ町駅\" に \"10年\" へのリンクがありません (GET /keyword/原ノ町駅)","keyword: \"和ジラ\" に \"5年\" へのリンクがありません (GET /keyword/和ジラ)","keyword: \"国際標準化機構\" に \"947年\" へのリン クがありません (GET /keyword/国際標準化機構)","keyword: \"岡山県道270号清音真金線\" に \"岡山県道270号清音真金線\" へのリンクがありません (GET /keyword/岡山県道270号清音真金線)","keyword: \"岩手県道220号氏子橋夕顔瀬線\" に \"45年\" へのリンクがありません (GET /keyword/岩手県道220号氏子橋夕顔瀬線)","keyword: \"島ひとみ\" に \"11年\" へのリンクがありません (GET /keyword/島ひとみ)","keyword: \"島根県道249号八重垣神社八雲線\" に \"島根県の県道一覧\" へのリンクがありません (GET /keyword/島根県道249号八重垣神社八雲線)","keyword: \"当用漢字\" に \"21年\" へのリンクがありません (GET /keyword/当用漢字)","keyword: \"志度駅\" に \"10年\" へのリンクがありません (GET /keyword/志度駅)","keyword: \"札内駅\" に \"12年\" へのリンクがありません (GET /keyword/札内駅)","keyword: \"東茨城郡\" に \"11年\" へのリンクがありません (GET /keyword/東茨城郡)","keyword: \"橋野真依子\" に \"4年\" へのリンクがありません (GET /keyword/橋野真依子)","keyword: \"河辺駅\" に \"11年\" へのリンクがありません (GET /keyword/河辺駅)","keyword: \"第一共和国 (オーストリア)\" に \"918年\" へ のリンクがありません (GET /keyword/第一共和国 (オーストリア))","keyword: \"豊橋閣日進禅寺\" に \"新川停留場\" からのリンクがありません (GET /keyword/新 川停留場)","keyword: \"魚津市農業協同組合\" に \"10年\" へのリンクがありません (GET /keyword/魚津市農業協同組合)","starがついていません (GET /)","タカラジェンヌ に 937年 へのリンクがありません (GET /)","リクエストがタイムアウトしました (GET /)","リクエストがタイムアウトしました (GET /keyword/495年)","リクエストがタイムアウトしました (GET /keyword/岩内郡)","リクエストがタイムアウトしました (POST /keyword)","リクエストがタイムアウトしました (POST /login)","リクエストがタイムアウトしました (POST /stars)","切通駅 に 10年 へのリンクがありません (GET /)","劇団カムカムミニキーナ に 10年 へのリンクがありませ ん (GET /)","吉富町 に 17年 へのリンクがありません (GET /)","済海寺 に 2年 へのリンクがありません (GET /)"]}
```


## 10:30 バグ修正

以前のコミットまでさかのぼって、エラーを消し、さらに最新のコミットに戻ると`Unable to load authentication plugin 'caching_sha2_password'.`が消えました。なんだったんだ…
そして、他のエラー、htmlifyカラムがnullのときに起こるエラーを無理やりねじ伏せたところ狙い通りに速く返せるキーワードは早く返し、そうでないものは以前の正規表現を使ってHTMLをオンメモリで生成、という動作になりました！
エラーが全てリクエストタイムアウトなので狙い通りなのですが、スコアが上がらない…そして、事前に全てのhtmlifyカラムをpopulateするバルクHTTPリクエストが残り一時間だと間に合わなさそう…


```
2020/08/30 10:31:20 start pre-checking
2020/08/30 10:31:25 pre-check finished and start main benchmarking
2020/08/30 10:32:20 benchmarking finished
{"pass":true,"score":0,"success":838,"fail":58,"messages":["リクエストがタイムアウトしました (GET /)","リクエストがタイムアウトしました (GET /keyword/城野駅 (北九州高速鉄道))","リクエストがタイムアウトしました (GET /keyword/潜水指定船)","リクエストがタイムアウトしました (GET /keyword/祥興帝)","リクエス トがタイムアウトしました (POST /keyword)","リクエストがタイムアウトしました (POST /stars)"]}
```


## 10:50 Cannot get a connection? 

まだまだisudaプロセスのCPU仕様がボトルネックになっている(おそらくhtmlifyの正規表現？)なので、残り時間を考えてスコア改善は難しそうですね。

```
top - 01:51:49 up 1 day,  3:11,  3 users,  load average: 4.82, 9.38, 6.79
Tasks: 135 total,   1 running,  75 sleeping,   4 stopped,   0 zombie
%Cpu(s): 96.0 us,  1.8 sy,  0.0 ni,  1.7 id,  0.0 wa,  0.0 hi,  0.5 si,  0.0 st
KiB Mem :  4017072 total,   741640 free,  2399780 used,   875652 buff/cache
KiB Swap:        0 total,        0 free,        0 used.  1233796 avail Mem

   PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
119110 isucon    20   0 3521904 1.158g  19192 S 131.2 30.2   3:40.97 java
 45627 mysql     20   0 1794620 486196  14788 S  31.2 12.1  19:52.11 mysqld
 45774 isucon    20   0 3448856 412960  13932 S  26.2 10.3   2:34.92 java
 13222 isucon    20   0   14492   8796   2608 S   6.0  0.2   0:27.95 isupam_linux
 54672 www-data  20   0   24536   3188   1932 S   0.3  0.1   0:03.76 nginx
     1 root      20   0  119808   5508   3632 S   0.0  0.1   0:05.59 systemd
     2 root      20   0       0      0      0 S   0.0  0.0   0:00.01 kthreadd
     4 root       0 -20       0      0      0 I   0.0  0.0   0:00.00 kworker/0:0H
```    


そもそも重い処理(1ページ辺り数秒)を7500ものキーワードに対して走らせ、それをDBに格納するというのは完全に作戦ミスでした。
計算すれば一件あたり5秒だとしても、7500件x5秒かかるので、どう考えても時間が足りないのですけどんな単純なことも見落とすものですね…

なんどかベンチマーカーを走らせるとsuccessの数は増えてきているものの、格納済みのhtmlifyに当たる確率が低いためか、failも多くスコアはあがらないですね。

```
{"pass":true,"score":0,"success":970,"fail":87,"messages":["Response code should be 200, got 404, data:  (POST /stars)","Response code should be 200, got 500, data:  (GET /)","Response code should be 200, got 500, data:  (GET /keyword/1008年)","Response code should be 200, got 500, data:  (GET /keyword/エッセンス)","Response code should be 200, got 500, data:  (GET /keyword/池田政員)","Response code should be 302, got 500, data:  (POST /login)","Response code should be 302, got 500, data: トイズ (POST /keyword)","Response code should be 302, got 500, data: 作曲賞 (POST /keyword)","Response code should be 302, got 500, data: 普門院 (POST /keyword)","Response code should be 302, got 500, data: 空印寺 (POST /keyword)","starがついていません (GET /)","リ クエストがタイムアウトしました (GET /)","リクエストがタイムアウトしました (POST /keyword)","大前駅 は既に表示されています (GET /)"]}
```

でもタイムアウトだけじゃなくHTTP 500が増えてきた…なんだろうこれは？


うむむむ、GC overhead limit exceeded…

```
> journalctl -u isuda.scala.service  
Aug 30 01:46:06 isucon6-webapp java[110422]: java.lang.OutOfMemoryError: GC overhead limit exceeded
Aug 30 01:46:06 isucon6-webapp java[110422]:         at java.util.Arrays.copyOf(Arrays.java:3332)
Aug 30 01:46:06 isucon6-webapp java[110422]:         at java.lang.StringCoding.safeTrim(StringCoding.java:89)
Aug 30 01:46:06 isucon6-webapp java[110422]:         at java.lang.StringCoding.access$100(StringCoding.java:50)
Aug 30 01:46:06 isucon6-webapp java[110422]:         at java.lang.StringCoding$StringDecoder.decode(StringCoding.java:154)
Aug 30 01:46:06 isucon6-webapp java[110422]:         at java.lang.StringCoding.decode(StringCoding.java:193)
Aug 30 01:46:06 isucon6-webapp java[110422]:         at java.lang.String.<init>(String.java:426)
Aug 30 01:46:06 isucon6-webapp java[110422]:         at com.mysql.cj.util.StringUtils.toString(StringUtils.java:1695)
Aug 30 01:46:06 isucon6-webapp java[110422]:         at com.mysql.cj.result.StringValueFactory.createFromBytes(StringValueFactory.java:129)
Aug 30 01:46:06 isucon6-webapp java[110422]:         at com.mysql.cj.result.StringValueFactory.createFromBytes(StringValueFactory.java:48)
Aug 30 01:46:06 isucon6-webapp java[110422]:         at com.mysql.cj.protocol.a.MysqlTextValueDecoder.decodeByteArray(MysqlTextValueDecoder.java:134)
Aug 30 01:46:06 isucon6-webapp java[110422]:         at com.mysql.cj.protocol.result.AbstractResultsetRow.decodeAndCreateReturnValue(AbstractResultsetRow.java:133)
Aug 30 01:46:06 isucon6-webapp java[110422]:         at com.mysql.cj.protocol.result.AbstractResultsetRow.getValueFromBytes(AbstractResultsetRow.java:241)
Aug 30 01:46:06 isucon6-webapp java[110422]:         at com.mysql.cj.protocol.a.result.TextBufferRow.getValue(TextBufferRow.java:132)
Aug 30 01:46:06 isucon6-webapp java[110422]:         at com.mysql.cj.jdbc.result.ResultSetImpl.getString(ResultSetImpl.java:850)
Aug 30 01:46:06 isucon6-webapp java[110422]:         at com.mysql.cj.jdbc.result.ResultSetImpl.getString(ResultSetImpl.java:863)
Aug 30 01:46:06 isucon6-webapp java[110422]:         at org.apache.commons.dbcp2.DelegatingResultSet.getString(DelegatingResultSet.java:267)
Aug 30 01:46:06 isucon6-webapp java[110422]:         at org.apache.commons.dbcp2.DelegatingResultSet.getString(DelegatingResultSet.java:267)
Aug 30 01:46:06 isucon6-webapp java[110422]:         at scalikejdbc.DBConnectionAttributesWiredResultSet.getString(DBConnectionAttributesWiredResultSet.scala:227)
Aug 30 01:46:06 isucon6-webapp java[110422]:         at scalikejdbc.Binders$$anonfun$51.apply(Binders.scala:151)
Aug 30 01:46:06 isucon6-webapp java[110422]:         at scalikejdbc.Binders$$anonfun$51.apply(Binders.scala:151)
Aug 30 01:46:06 isucon6-webapp java[110422]:         at scalikejdbc.Binders$$anon$2.apply(Binders.scala:35)
Aug 30 01:46:06 isucon6-webapp java[110422]:         at scalikejdbc.WrappedResultSet$$anonfun$get$2.apply(WrappedResultSet.scala:469)
Aug 30 01:46:06 isucon6-webapp java[110422]:         at scalikejdbc.WrappedResultSet.wrapIfError(WrappedResultSet.scala:26)
Aug 30 01:46:06 isucon6-webapp java[110422]:         at scalikejdbc.WrappedResultSet.get(WrappedResultSet.scala:469)
Aug 30 01:46:06 isucon6-webapp java[110422]:         at isuda.Web$.asEntry(Web.scala:311)
Aug 30 01:46:06 isucon6-webapp java[110422]:         at isuda.Web$$anonfun$43$$anonfun$44.apply(Web.scala:236)
Aug 30 01:46:06 isucon6-webapp java[110422]:         at isuda.Web$$anonfun$43$$anonfun$44.apply(Web.scala:236)
Aug 30 01:46:06 isucon6-webapp java[110422]:         at scala.collection.TraversableLike$$anonfun$map$1.apply(TraversableLike.scala:234)
Aug 30 01:46:06 isucon6-webapp java[110422]:         at scala.collection.TraversableLike$$anonfun$map$1.apply(TraversableLike.scala:234)
Aug 30 01:46:06 isucon6-webapp java[110422]:         at scalikejdbc.ResultSetTraversable$$anonfun$foreach$1.apply(ResultSetTraversable.scala:21)
Aug 30 01:46:06 isucon6-webapp java[110422]:         at scalikejdbc.ResultSetTraversable$$anonfun$foreach$1.apply(ResultSetTraversable.scala:18)
Aug 30 01:46:06 isucon6-webapp java[110422]:         at scalikejdbc.LoanPattern$class.using(LoanPattern.scala:18)
Aug 30 01:46:44 isucon6-webapp java[110422]: 01:46:44.633 [qtp134310351-97] ERROR s.StatementExecutor$$anon$1 - SQL execution failed (Reason: GC overhead limit exceeded):
Aug 30 01:46:44 isucon6-webapp java[110422]:    SELECT * FROM entry ORDER BY CHARACTER_LENGTH(keyword) DESC
Aug 30 01:46:46 isucon6-webapp java[110422]: 01:46:46.350 [qtp134310351-90] WARN  o.e.jetty.servlet.ServletHandler - Error for /keyword/???
Aug 30 01:46:46 isucon6-webapp java[110422]: java.lang.OutOfMemoryError: GC overhead limit exceeded
Aug 30 01:46:46 isucon6-webapp java[110422]: 01:46:46.863 [qtp134310351-47] WARN  o.e.jetty.servlet.ServletHandler - Error for /keyword/????????
Aug 30 01:46:46 isucon6-webapp java[110422]: java.lang.OutOfMemoryError: GC overhead limit exceeded
Aug 30 01:47:19 isucon6-webapp java[110422]: 01:47:19.551 [qtp134310351-13] ERROR s.StatementExecutor$$anon$1 - SQL execution failed (Reason: GC overhead limit exceeded):
Aug 30 01:47:19 isucon6-webapp java[110422]:    SELECT * FROM entry ORDER BY CHARACTER_LENGTH(keyword) DESC
Aug 30 01:47:20 isucon6-webapp java[110422]: 01:47:20.282 [qtp134310351-79] WARN  o.e.jetty.servlet.ServletHandler - Error for /keyword/???
Aug 30 01:47:20 isucon6-webapp java[110422]: java.lang.OutOfMemoryError: GC overhead limit exceeded
Aug 30 01:47:23 isucon6-webapp java[110422]: 01:47:23.018 [qtp134310351-100] WARN  o.e.jetty.servlet.ServletHandler - Error for /keyword/????????????????
Aug 30 01:47:23 isucon6-webapp java[110422]: java.lang.OutOfMemoryError: GC overhead limit exceeded
Aug 30 01:47:23 isucon6-webapp java[110422]: 01:47:23.247 [qtp134310351-51] WARN  o.e.jetty.servlet.ServletHandler - Error for /keyword/????
Aug 30 01:47:23 isucon6-webapp java[110422]: java.lang.OutOfMemoryError: GC overhead limit exceeded
```

そしてこれはnginxの統計。同じエンドポイントでMIN(htmlify保存済みと予想)とMAXの差が大きく開いてしまいました。

```
+-------+-----+------+------+-----+-----+--------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+--------+--------+----------+--------+--------+--------+--------+--------+------------+------------+---------------+------------+
| COUNT | 1XX | 2XX  | 3XX  | 4XX | 5XX | METHOD |                                                                                                                                                                                              URI                                                                                                                                                                                              |  MIN   |  MAX   |   SUM    |  AVG   |   P1   |  P50   |  P99   | STDDEV | MIN(BODY)  | MAX(BODY)  |   SUM(BODY)   | AVG(BODY)  |
+-------+-----+------+------+-----+-----+--------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+--------+--------+----------+--------+--------+--------+--------+--------+------------+------------+---------------+------------+
|     7 |   0 |    3 |    0 |   0 |   4 | GET    | /keyword/1066\xE5\xB9\xB4                                                                                                                                                                                                                                                                                                                                                                     |  0.056 | 51.134 |   68.575 |  9.796 |  0.852 |  0.852 |  1.940 | 17.204 |   4344.000 |  20627.000 |     95540.000 |  13648.571 |
|     7 |   0 |    3 |    0 |   0 |   4 | GET    | /keyword/1065\xE5\xB9\xB4                                                                                                                                                                                                                                                                                                                                                                     |  0.056 | 51.222 |   66.054 |  9.436 |  0.808 |  0.792 |  1.932 | 17.237 |   2968.000 |  20627.000 |     91412.000 |  13058.857 |
|     7 |   0 |    3 |    0 |   0 |   4 | GET    | /keyword/1064\xE5\xB9\xB4                                                                                                                                                                                                                                                                                                                                                                     |  0.056 | 50.958 |   64.850 |  9.264 |  0.840 |  0.824 |  2.040 | 17.144 |   2740.000 |  20627.000 |     90728.000 |  12961.143 |
|     7 |   0 |    3 |    0 |   0 |   4 | GET    | /keyword/1062\xE5\xB9\xB4                                                                                                                                                                                                                                                                                                                                                                     |  0.052 | 49.650 |   78.759 | 11.251 |  0.840 |  0.916 | 16.401 | 16.541 |   4096.000 |  21365.000 |     95534.000 |  13647.714 |
|     7 |   0 |    3 |    0 |   0 |   4 | GET    | /keyword/1061\xE5\xB9\xB4                                                                                                                                                                                                                                                                                                                                                                     |  0.052 | 49.738 |   62.346 |  8.907 |  0.872 |  0.848 |  2.256 | 16.750 |   2813.000 |  20627.000 |     90947.000 |  12992.429 |
|     7 |   0 |    3 |    0 |   0 |   4 | GET    | /keyword/1060\xE5\xB9\xB4                                                                                                                                                                                                                                                                                                                                                                     |  0.056 | 51.978 |   68.039 |  9.720 |  0.916 |  0.768 |  2.328 | 17.511 |   2860.000 |  20627.000 |     91088.000 |  13012.571 |
```


## 11:24 regxのオンメモリキャッシュ

これはきいた！！こっちを先にやっておきべきでした。スコアがまであがりました。

```
 bench -target "http://10.0.1.4"
2020/08/30 11:25:32 start pre-checking
2020/08/30 11:25:36 pre-check finished and start main benchmarking
2020/08/30 11:26:32 benchmarking finished
{"pass":true,"score":3337,"success":2096,"fail":30,"messages":["keyword: \"CCE\" に \"海浜幕張駅\" からのリンクがありません (GET /keyword/海浜幕張駅)","keyword: \"G15\" に \"三菱・ランサー\" からのリンクがありません (GET /keyword/三菱・ランサー)","keyword: \"イドリース\" に \"1099年\" からのリンクがありません (GET /keyword/1099年)","keyword: \"ウーズ\" に \"共通祖先\" からのリンクがありません (GET /keyword/共通祖先)","keyword: \"トイズ\" に \"Sugar (韓国の音楽グループ)\" からのリンクがありません (GET /keyword/Sugar (韓国の音楽グループ))","keyword: \"レフラー\" に \"ロベルト・コッホ\" から のリンクがありません (GET /keyword/ロベルト・コッホ)","keyword: \"井上敏夫\" に \"国会議員一覧\" からのリンクがありません (GET /keyword/国会議員一覧)","keyword: \"八田小学校\" に \"梅迫駅\" からのリンクがありません (GET /keyword/梅迫駅)","keyword: \"加藤彰\" に \"日本の映画監督一覧\" からのリンク がありません (GET /keyword/日本の映画監督一覧)","keyword: \"北海道の再開発の一覧\" に \"帯広駅\" からのリンクがありません (GET /keyword/帯広駅)","keyword: \"北消防署\" に \"南森町駅\" からのリンクがありません (GET /keyword/南森町駅)","keyword: \"南蟹谷村\" に \"所属郡を変更した町村一覧\" からのリ ンクがありません (GET /keyword/所属郡を変更した町村一覧)","keyword: \"普門院\" に \"新白岡駅\" からのリンクがありません (GET /keyword/新白岡駅)","keyword: \"枇杷島橋\" に \"名鉄一宮線\" からのリンクがありません (GET /keyword/名鉄一宮線)","keyword: \"空印寺\" に \"酒井忠存\" からのリンクがありませ ん (GET /keyword/酒井忠存)","keyword: \"船戸山\" に \"亀田町 (新潟県)\" からのリンクがありません (GET /keyword/亀田町 (新潟県))","keyword: \"藪田村\" に \"氷見市\" からのリンクがありません (GET /keyword/氷見市)","keyword: \"行政教区\" に \"キングスタウン\" からのリンクがありません (GET /keyword/キングスタウン)","keyword: \"観音橋\" に \"茨戸川\" からのリンクがありません (GET /keyword/茨戸川)","keyword: \"輪状甲状筋\" に \"人間の筋肉の一覧\" からのリンクがありません (GET /keyword/人間の筋肉の一覧)","keyword: \"鞍掛山\" に \"玖珂町\" からのリンクがありません (GET /keyword/玖珂町)","リクエス トがタイムアウトしました (POST /keyword)","朝夷奈切通 は既に表示されています (GET /)","錦糸町 は既に表示されています (GET /)","高橋雄一 は既に表示さ れています (GET /)"]}
```

というわけで時間切れです

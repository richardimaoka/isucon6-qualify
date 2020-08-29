2020/08/30 日曜日 一人Isucon6開始

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

benchとworkerはどちらもGoプロセスで、強調して動くのですが「workerがportalのwebサイトとやりとりし、workerを起動する」というしくのため、workerさえあれば手動で走らせてスコアを取得できます。

Azureの"deploy from template"でデプロイを試みたところprovisioningディレクトリ以下にあるAnsibleプレイブックが結構失敗している模様...(VMの/var/log/cloud-init-output.log)
各deploy.jsonのcustomdataをbase64でデコードして確認したところ、init.shを実行していただけなので、手動でinit.shと同様の作業をおこいました。

## 初回ベンチマーク実行

```
bench -target "http://10.0.1.4"
2020/08/30 03:29:28 start pre-checking
2020/08/30 03:29:58 pre-check finished and start main benchmarking
2020/08/30 03:30:28 benchmarking finished
{"pass":true,"score":0,"success":269,"fail":26,"messages":["Response code should be 200, got 500, data:  (GET /)","Response code should be 302, got 500, data:  (POST /login)","Response code should be 302, got 500, data: アイドルメーカー・エッジ (POST /keyword)","Response code should be 302, got 500, data: 北消防 署 (POST /keyword)","Response code should be 400, got 500, data: 2001年宇宙の旅 (POST /keyword)","Response code should be 400, got 500, data: 岩原裕二 (POST /keyword)","Response code should be 400, got 500, data: 高橋留美子 (POST /keyword)","Response code should be 403, got 500, data:  (POST /login)","リクエスト がタイムアウトしました (GET /)"]}
```

pre-checkingは成功、successも269あるので、isuda、 isutar、 isupamは正しく動いているようですが、スコアはゼロ点スタートですね。作業時間は、初回のベンチマークがクラウド上で走ってから最大8時間とのことなので同日午前11:30までですね。ここまでの作業でGoのbenchを見てしまったり、provisioningディレクトリの中身からVMとプロセスの構成を知るなど、若干カンニング気味ではあります…。isuda/isutarに関してはほぼ中身はみてないですが。
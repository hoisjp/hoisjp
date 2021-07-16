# Intro はじめに

Azure Application Insights は、監視サービスである Azure Monitor の機能のひとつで、インフラ層よりも上のアプリケーション層のパフォーマンスや状態を監視します。
サーバーサイドのアプリケーションログだけでなく、ウェブページのページビューやパフォーマンス解析も可能です。Google Analytics や Adobe Analytics の役割、といったほうが伝わりやすいかもしれません。
![alt](https://docs.microsoft.com/ja-jp/azure/azure-monitor/app/media/app-insights-overview/diagram.png)
出典： https://docs.microsoft.com/ja-jp/azure/azure-monitor/app/app-insights-overview

この記事では、この機能を使うときの料金を節約する方法を紹介します。

# Application Insights の料金

Application Insights の料金は、おおまかに以下の2つのパラメータで決まります。

- ログをボリュームに書きこむ量：データインジェスト（GB あたり〇〇円）
- ログを保持する期間：データ保持期間 Data Retention（ひと月、GBあたり〇〇円）

詳細は、公式ドキュメント [Azure Monitor の価格](https://azure.microsoft.com/ja-jp/pricing/details/monitor/) を参考に。さらに、 [料金計算ツール](https://azure.microsoft.com/ja-jp/pricing/calculator/?service=monitor)で見積もってみると想像つきやすいかもしれません。

上記2つのうち、後者のデータ保持期間は後から [ワークスペースの設定を変更することで調整できます](https://docs.microsoft.com/ja-jp/azure/azure-monitor/logs/manage-cost-storage#change-the-data-retention-period) が、**前者のデータインジェストはあらかじめログ収集時に調整する必要があります。**また、そもそも収集する量を小さくすることで、長期間保持したときの料金も抑えることができます。

# How to configure 設定方法
Web ページ解析は、ページに以下のような `<script>` タグのスニペットを埋め込むことでログデータが Azure へ送られます。

```
<script type="text/javascript">
!function(T,l,y){
...
cfg: { // Application Insights Configuration
    instrumentationKey: "YOUR_INSTRUMENTATION_KEY_GOES_HERE"
    /* ...Other Configuration Options... */
}});
</script>
```
※ スクリプトの多くを省略しています。正しいスニペットはドキュメントを見てください。
https://docs.microsoft.com/ja-jp/azure/azure-monitor/app/javascript

このスクリプト上の `cfg` に最低限必要な `instrumentationKey` が設定されていますが、この箇所に以降で説明する構成を行っていくことで、このスクリプトが送信するログの量を小さくすることができます。

構成フィールドの意味やすべての一覧は[ドキュメント](https://docs.microsoft.com/ja-jp/azure/azure-monitor/app/javascript#configuration)を参照ください。以下にデータインジェスト量に影響がありそうなフィールドを抜粋していきます。デフォルトで送信しないようになっているものもありますが、関連しそうなものは明示的に書きます。 **太字**になっている箇所が、デフォルト値と異なるものです。

| 名前        | Default     | 最小となる設定値 |
|:-----------|-------------|--------------|
| disableExceptionTracking | false        | **true**       |
| disableTelemetry | false        | **true**       |
| enableDebug | false        | false       |
| loggingLevelConsole | 0        | 0       |
| loggingLevelTelemetry | 1        | **0**       |
| samplingPercentage | 100        | 100 <br>※これを変更するとサンプリングされるためデフォルト値のまま |
| disableAjaxTracking | false     | **true**      |
| disableFetchTracking | true     | true      |
| disableCorrelationHeaders | false     | true      |
| correlationHeaderExcludedDomains | undefined     | **string[]** <br>※対象外ドメインを指定     |
| disableCookiesUsage | false     | **true** <br>※ただしcookieデータを扱わないだけでデータインジェスト量はそれほど変わらないはず |

## 設定サンプル
とりあえずこれを構成すれば限りなく最小に近づきます。ただし、本来必要としていたログがオフになってしまうと本末転倒ですので注意してください。

```
cfg: { // Application Insights Configuration
    instrumentationKey: "YOUR_INSTRUMENTATION_KEY_GOES_HERE",
    disableExceptionTracking: true,
    disableTelemetry: true,
    loggingLevelTelemetry: 0,
    disableAjaxTracking: true
}});
</script>
```
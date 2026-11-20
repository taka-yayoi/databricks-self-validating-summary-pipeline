# Databricks で自己検証する AI 要約パイプラインを動かす

Databricks Technical Blog「[Building a Self-Validating AI Summary Pipeline on Databricks](https://community.databricks.com/t5/technical-blog/building-a-self-validating-ai-summary-pipeline-on-databricks/ba-p/175369)」の構成を、小さなサンプルデータで動かすノートブックです。

ブログ記事には、作り直しのループの疑似コードとジャッジの定義が載っています。検証ルールの中身、データ、各関数の実装は載っていないので、このノートブックではその部分を独自に書いています。サンプルデータは架空のものです。

## 何をするノートブックか

A/B テストの結果を LLM で要約し、その要約を自動で検証します。

1. A/B テストの結果テーブルを作り、統計的に有意な行だけを読む
2. MLflow Prompt Registry に登録したプロンプトで要約を生成する
3. 要約を4段階で検証する
4. 検証に通らなければ、失敗の内容をプロンプトに足して作り直す (上限3回)
5. 最も良い試行を選び、PASS / SOFT FAIL / HARD FAIL を付ける
6. 一連の処理を MLflow トレースに残し、ジャッジ2種類の採点を同じトレースに書き込む

検証の4段階は次のとおりです。

| 段階 | 誰が判定するか | 見るもの |
| --- | --- | --- |
| deterministic | Python のコード | 感嘆符、実験ID、略語、文字数 |
| simple | 軽いモデル | 文体、構成 |
| complex | 推論の強いモデル | 矛盾、重複、誤解を招く一般化 |
| data | 生成とは別のモデル | 元データとの数値の一致、悪化した結果の書き漏らし |

プロンプトは2種類登録します。ルールを書いていないプロンプトと、ルールを書いたプロンプトです。ルールを書いていないプロンプトでも、検証と作り直しで要約が直る様子を確認できます。

## ファイル

| ファイル | 内容 |
| --- | --- |
| `self_validating_summary_pipeline.ipynb` | ノートブック本体 |

## 必要なもの

- Unity Catalog が有効な Databricks ワークスペース
- サーバーレスコンピュート
- テーブルとプロンプトを作成できるスキーマ。プロンプトの登録には、スキーマに対する `CREATE FUNCTION`、`EXECUTE`、`MANAGE` の権限が必要です
- MLflow Prompt Registry が使えること (ベータの機能です。ワークスペースのプレビュー設定で有効にします)
- 基盤モデル API のエンドポイント

## 使い方

1. `self_validating_summary_pipeline.ipynb` をワークスペースにインポートします
2. サーバーレスコンピュートに接続します
3. ノートブック上部のウィジェット `catalog` と `schema` に、使用するカタログとスキーマを指定します
4. 上から順にすべてのセルを実行します

実行には数分かかります。モデルを最大で70回ほど呼び出します。

### モデルについて

モデルは役割ごとに候補を並べてあり、ワークスペースで応答した最初のものを使います。候補の先頭はブログ記事の既定値です。使うモデルを固定したい場合は、「1. 設定」のセルにある `MODEL_CANDIDATES` を書き換えてください。

| 役割 | 用途 |
| --- | --- |
| generator | 要約の生成 |
| judge_simple | 文体と構成の検証 |
| judge_complex | 矛盾などの検証、ジャッジ |
| judge_data | 元データとの照合 |

### 作成されるもの

指定したスキーマに次のものが作成されます。

| 種類 | 名前 | 内容 |
| --- | --- | --- |
| テーブル | `svs_experiment_results` | サンプルの A/B テスト結果 (架空のデータ) |
| テーブル | `svs_summaries` | 要約と検証結果 |
| プロンプト | `svs_experiment_summary` | 要約用のプロンプト。実行するたびにバージョンが2つ増えます |

ノートブックのエクスペリメントには、要約1件につき1本のトレースが記録されます。

## 結果の見方

- 「9. 結果を見る」の表で、実験とプロンプトごとの最終結果と、試行ごとの失敗内容を確認できます
- エクスペリメントのトレース画面で `summarize_experiment` を開くと、`attempt_1`、`attempt_2` のように試行が並び、それぞれの中に生成と各段階の検証が入っています
- トレースの評価パネルに、`usefulness` (要約の有用性、1から3) と `check_diagnosis` (検証の誤判定の診断) が表示されます

LLM の出力は実行のたびに変わるので、試行回数や判定は毎回同じにはなりません。

## 後片付け

ノートブック末尾のセルのコメントを外して実行すると、テーブルが削除されます。プロンプトはカタログエクスプローラーから削除してください。

## 参考

- [LLM を呼ぶパイプラインも生成 AI アプリ: Databricks で検証と作り直しを組み込んだ要約パイプラインを試す (Qiita)](https://qiita.com/taka_yayoi/items/e21496d252a2aa3e533f)
- [Building a Self-Validating AI Summary Pipeline on Databricks](https://community.databricks.com/t5/technical-blog/building-a-self-validating-ai-summary-pipeline-on-databricks/ba-p/175369)
- [生成AI対応 MLflow 3](https://docs.databricks.com/aws/ja/mlflow3/genai/)
- [プロンプトの作成と編集](https://docs.databricks.com/aws/ja/mlflow3/genai/prompt-version-mgmt/prompt-registry/create-and-edit-prompts)
- [関数デコレータ](https://docs.databricks.com/aws/ja/mlflow3/genai/tracing/app-instrumentation/manual-tracing/function-decorator)
- [カスタムジャッジ](https://docs.databricks.com/aws/ja/mlflow3/genai/eval-monitor/concepts/judges/prompt-based-judge)

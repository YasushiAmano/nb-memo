# Anthropic Cloud Session

## Anthropicの CloudSession（クラウドセッション）とは？

CloudSession（クラウドセッション） は、AnthropicのCLI・AIエージェントツールである Claude Code を、手元のローカルPCではなく Anthropicが管理するクラウド上の仮想マシン（VM）で実行する機能 です。

- PCを閉じても動く:
    処理がクラウド側のVM（Ubuntu 24.04ベース：4 vCPU / 16GB RAM / 30GB Disk等）で独立して実行されるため、PCの電源を切ったりノートPCを閉じたりしてもバックグラウンドで作業が継続します。 
- マルチデバイス対応: 
    ブラウザ（claude.ai/code）、デスクトップアプリ、スマホアプリ（Codeタブ）、ターミナル（claude --cloud）のどこからでも進捗確認や追加指示が可能です。   
- 標準的な実行環境を完備: 
    Python, Node.js, Docker, PostgreSQL, Git などの主要ツールや開発言語があらかじめインストールされています。
- 定期実行（Routines）: 
    定期的にタスクをトリガーしてクラウドセッションを自走させることができます。  

## 現行のAWS（Bedrock）のコストを激減させる

- Claude 3.5 Haiku（または Claude 3 Haiku）を使う
- Bedrock Prompt Caching（プロンプトキャッシュ）の導入
- Bedrock Batch Inference（バッチ推論）の利用
- プロンプト構造（コンテキスト）の最適化

### リクエスト内の画像枚数制限（上限 20枚）

- 1回の InvokeModel（または Messages API）呼び出しの payload に載せられる image ブロックの最大数です。
    APIの並列呼び出し制限（レートリミット / Quota）
- Bedrockに対して「同時に何回リクエスト（API呼び出し）を投げられるか」という制限です。
    デフォルトでも毎秒数十〜数百リクエスト（TPS）まで許可されているため、「画像1枚（または5枚）を入れたAPI呼び出し」を同時に30回〜40回並列で投げることは全く問題ありません。

```
[1案件：画像 40枚]
       │
       ├─ (A) 5枚ずつ × 8グループに分割（または 1枚ずつ × 40分割）
       │
       ▼
【Step 1: Haiku による並列1次選別】
  ・Pythonの `asyncio` や `concurrent.futures` 等で
    8個（または40個）のリクエストを Bedrock（Haiku）へ一斉に並列送信（1〜2秒で全件完了）
  ・レスポンス: 各画像が条件を満たすか（True / False）の判定のみ回収
       │
       ▼
【Step 2: 採用画像のみ抽出】
  ・40枚の中から条件に合致した「採用画像（例: 3〜5枚）」のみが残る
       │
       ▼
【Step 3: Sonnet による文章生成】
  ・採用された 3〜5枚の画像（20枚の上限以下）だけを 1回のリクエストで Bedrock（Sonnet）へ送信
  ・最終的な高品質な文章を作成してDBへ保存
```


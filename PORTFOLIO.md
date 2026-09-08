# Organizing The Brain MEMO — 技術解説資料

## 1. プロジェクト概要

自由形式のメモを「書いて終わり」にせず、AIで構造化して次の行動へつなげる個人開発のiOSアプリです。入力は整った文章ではなく、箇条書き・感情・途中の思考が混在することを前提にしています。

### 担当範囲

- iOS UI・ローカルデータモデル
- リッチテキスト編集
- AI要約プロンプトと構造化出力
- Cloudflare Worker API
- 端末認証・レート制限・費用制御
- StoreKit 2とサブスクリプション検証
- AdMobリワード広告
- 本番デプロイと動作検証

## 2. システム構成

アプリ本体とAIバックエンドを分離しています。端末内で完結するメモ編集はSwiftDataへ保存し、秘密情報や全体費用に関わる処理だけをWorkerへ委譲します。

| レイヤー | 採用技術 | 役割 |
|---|---|---|
| Presentation | SwiftUI | メモ・要約・ネクストステップの3タブ |
| Rich Text | UIViewRepresentable / UITextView | 属性付き文字列の編集と選択範囲操作 |
| Persistence | SwiftData | Memo、チェック項目、要約、次の行動の保存 |
| Client Service | URLSession / async-await | 署名付きAPI通信 |
| Client Security | Keychain / CryptoKit | 端末シークレット保存とHMAC生成 |
| Edge Backend | Cloudflare Workers / TypeScript | 認証、回数制限、AI・課金連携 |
| State | Workers KV | 端末、利用回数、課金Tier、推定費用 |
| AI | Gemini Structured Output | KJ法要約と行動提案 |
| Monetization | StoreKit 2 / AdMob | Pro契約と広告ボーナス |

## 3. KJ法ベースのAI要約

### なぜKJ法か

Progressive Summarizationは、既に一定の構造を持つ文章を段階的に圧縮する用途と相性が良い一方、本アプリの入力は殴り書きです。そこで、断片を類似性でまとめて意味を見つけるKJ法の考え方を採用しました。

### 推論手順

1. 一つひとつの断片を意味単位として読む
2. 意味・感情・目的・因果関係が近い断片を集める
3. 各まとまりの本質を表すグループ名を付ける
4. グループ同士の関係から全体像と核心を文章化する
5. 具体的で実行可能なネクストステップを抽出する

### 出力の安定化

- JSON Schemaで`overview / groups / insights / nextSteps`を必須化
- temperatureを低めに設定
- 表示フォーマットを固定
- 「元文にない感情を創作しない」と明記
- 有益な行動がない場合は`nextSteps: []`を許可

要約と行動提案を別々に呼ばず、1リクエストで生成することで、文脈の一致とAPI原価の抑制を両立しています。

## 4. リッチテキスト実装

SwiftUIの標準TextEditorだけでは、選択範囲に対する太字・斜体・下線の操作が不足します。そのため`UITextView`を`UIViewRepresentable`でラップしました。

### 工夫した点

- 選択中は`NSAttributedString`の該当範囲へ属性を付与
- 未選択時は`typingAttributes`を更新
- Coordinator経由の更新をフラグで識別
- SwiftUI再描画時のカーソル末尾ジャンプを抑止
- `NSKeyedArchiver`で属性付き文字列をData化
- プレーンテキストも並行保持し、一覧表示・AI送信へ利用

## 5. 端末認証とAPI保護

### 初回プロビジョニング

1. アプリがWorkerの`/provision`を呼ぶ
2. WorkerがdeviceIdと256bitシークレットを発行
3. シークレットをKeychainへ保存
4. KVには`device:{id}`として保持

### 通常リクエスト

本文・端末ID・Unix時刻を連結し、HMAC-SHA256で署名します。Workerは時刻差、端末の存在、署名を確認します。APIキーをバイナリへ含めず、URLを知るだけでは上流AIを利用できない構成です。

### 現時点のトレードオフ

HMACシークレットはApp Attestほど強い「正規アプリ証明」ではありません。まず費用対効果を優先した構成で、悪用が観測された場合はApp AttestとDurable Objectsによる厳密なカウンターが拡張候補です。

## 6. 費用・不正利用対策

| 対策 | 目的 |
|---|---|
| 端末別日次上限 | 一般利用と連続実行を制御 |
| 発行元IP別のprovision上限 | 端末IDの大量発行を抑制 |
| 広告ボーナス上限 | 広告報酬の無制限加算を防止 |
| 月間費用サーキットブレーカー | 想定外請求時にAI呼び出しを停止 |
| 上流usageの記録 | トークン量から推定費用を加算 |

KVは結果整合性のため、厳密な課金台帳ではなく「ソフトリミット」として利用しています。厳密性が必要になればDurable ObjectsやSQLへ移行します。

## 7. サブスクリプション

- StoreKit 2で商品取得・購入・復元を実装
- `Transaction.currentEntitlements`と更新ストリームを監視
- クライアントからtransactionIdをWorkerへ送信
- WorkerがApp Store Server APIで取引を照合
- 有効期限付きでKVにPro Tierを保存

クライアントの自己申告だけでPro権限を付けない点が重要です。

## 8. リワード広告

無料枠到達時のみ広告視聴を選択肢として提示し、バナー広告は使いません。視聴完了コールバック後にWorkerへボーナスを要求し、自動的に要約を再試行します。

## 9. エラー設計

- 端末準備失敗
- 日次上限到達
- 月間費用上限到達
- サブスクリプション照合失敗
- 広告ボーナス上限
- ネットワーク失敗
- 上流API失敗・JSON解析失敗

それぞれをユーザー向け日本語メッセージへ変換し、再試行・広告・Pro登録など次の選択肢へ誘導します。再生成失敗時も、以前成功した要約を消さない方針です。

## 10. 品質保証と今後の改善

### 現在実施している検証

- `xcodebuild`によるシミュレータービルド
- `tsc --noEmit`によるBackend strict型検査
- Wranglerによる本番Workerデプロイ
- StoreKit Configurationを使ったローカル課金確認

### 今後強化する項目

- XCTestでモデル・状態遷移をテスト
- WorkerのVitest統合テスト
- GitHub ActionsでiOS Build / TypeScript Checkを自動化
- SwiftDataのVersionedSchema / SchemaMigrationPlan
- App Attestによる正規アプリ検証
- アクセシビリティ・Dynamic Type・長文負荷テスト

## 11. 面接で説明できるポイント

1. 「AIを呼んだ」だけでなく、品質・費用・秘密情報・悪用まで含めてプロダクト化したこと
2. KJ法をLLMプロンプトとJSON Schemaへ落とし込んだこと
3. SwiftUIで不足するリッチテキストをUIKit Bridgeで補ったこと
4. StoreKitの購入状態をサーバー側でも確認する境界設計
5. 完璧な防御を初期から作らず、リスクに応じた段階的な拡張経路を用意したこと

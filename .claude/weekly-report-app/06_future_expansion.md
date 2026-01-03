# 将来拡張の設計余地（Phase2以降）

## 概要
MVPリリース後、ユーザーフィードバックや運用実績をもとに拡張する機能の設計方針を整理します。

**拡張方針**:
- MVPで基盤を固め、Phase2以降で段階的に追加
- データモデル・API設計はPhase2を見越して拡張性を確保
- 既存機能への影響を最小化（後方互換性維持）

---

## 1. 音声入力機能

### 1.1 概要
週報作成時に音声で口頭報告し、自動で文字起こし→要約→各セクションに割当。

### 1.2 ユースケース
- 新入社員が金曜夕方、スマホで週報を口頭で報告（移動中など）
- 文字起こし結果を確認・修正後、生成実行

### 1.3 技術要件
| 要素 | 技術候補 |
|------|----------|
| **音声入力** | ブラウザ録音（WebRTC, MediaRecorder API） or ファイルアップロード |
| **文字起こしAPI** | OpenAI Whisper / Google Speech-to-Text / Azure Speech |
| **要約・分類** | LLM（音声→テキスト→セクション分類） |

### 1.4 画面追加
- 週報作成画面に「音声入力」タブ追加
  - マイクボタン（録音開始/停止）
  - またはファイルアップロード（.mp3, .wav, .m4a）
  - 文字起こし結果表示（編集可能）
  - 「セクション自動割当」ボタン→各セクションに振り分け

### 1.5 API設計
```
POST /api/reports/:id/transcribe
- body: audio_file (multipart/form-data)
- response: {
    "transcript": "文字起こしテキスト",
    "sections": {
      "tasks_done": "抽出されたタスク",
      "learnings": "抽出された学び",
      ...
    }
  }
```

### 1.6 データモデル拡張
- WeeklyReportテーブルに追加
  - `audio_file_url` (TEXT): 音声ファイルS3 URL
  - `transcript_text` (TEXT): 文字起こし元テキスト

### 1.7 運用考慮
- 音声ファイル保持期間（3ヶ月後削除など）
- 文字起こし精度（専門用語の誤変換）→ユーザーが修正可能なUI

### 1.8 コスト試算
- Whisper API: $0.006 / 分（5分で$0.03）
- 月50ユーザー×週1回×5分 = 約1000分/月 → $6/月

---

## 2. 週報テンプレート管理機能

### 2.1 概要
管理者が週報のセクション項目をカスタマイズ可能にする（現在は固定5項目）。

### 2.2 ユースケース
- 現場Aは「安全管理」項目を追加したい
- 現場Bは「学んだこと」を「技術的学び」「ビジネス的学び」に分割したい

### 2.3 画面追加
- 管理画面に「テンプレート管理」タブ
  - セクション一覧（名前、順序、必須/任意）
  - ドラッグ&ドロップで並び替え
  - 追加/削除/編集ボタン
- 現場管理画面で「テンプレート選択」（現場ごとに異なるテンプレート適用可能）

### 2.4 API設計
```
GET /api/admin/templates
POST /api/admin/templates
PUT /api/admin/templates/:id
DELETE /api/admin/templates/:id

GET /api/admin/sites/:id
PUT /api/admin/sites/:id (body: {template_id: ...})
```

### 2.5 データモデル拡張
```sql
CREATE TABLE templates (
  id UUID PRIMARY KEY,
  name VARCHAR(100),
  sections_json JSON, -- [{"key": "tasks_done", "label": "今週やったこと", "required": true, "order": 1}, ...]
  created_at TIMESTAMP,
  updated_at TIMESTAMP
);

ALTER TABLE sites ADD COLUMN template_id UUID REFERENCES templates(id);
```

### 2.6 プロンプト拡張
- プロンプト生成時にテンプレートのsections_jsonを読み込み
- セクション名・順序を動的に反映

### 2.7 移行戦略
- デフォルトテンプレート（現在の5項目）を作成
- 既存現場は全てデフォルトテンプレートに紐付け
- カスタムテンプレート作成は段階的に

---

## 3. 通知・連携機能

### 3.1 概要
週報提出時やレビュー完了時に、Slack/Teams/メールで通知。

### 3.2 ユースケース
| イベント | 通知対象 | 内容 |
|----------|----------|------|
| 週報提出 | OJT | 「◯◯さんが週報を提出しました」+リンク |
| レビュー完了 | 新入社員 | 「週報がレビューされました」+リンク |
| 期限リマインド | 新入社員 | 「週報未提出です（金曜18時）」 |
| 異常検知 | 管理者 | 「連続3週未提出」「評価が著しく低下」 |

### 3.3 技術要件
| 連携先 | 実装方法 |
|--------|----------|
| **Slack** | Incoming Webhook / Slack API（Bot投稿） |
| **Teams** | Incoming Webhook / Microsoft Graph API |
| **メール** | Nodemailer / SendGrid / AWS SES |

### 3.4 設定画面
- ユーザー設定で「通知設定」
  - 通知方法選択（Slack/Teams/メール/なし）
  - Slack: Webhook URL入力 or OAuth連携
  - メール: 送信先アドレス（デフォルトは登録メール）

### 3.5 API設計
```
POST /api/notifications/settings (ユーザーの通知設定保存)
POST /api/notifications/test (テスト通知送信)
```

### 3.6 データモデル拡張
```sql
CREATE TABLE notification_settings (
  id UUID PRIMARY KEY,
  user_id UUID REFERENCES users(id),
  method ENUM('slack', 'teams', 'email', 'none'),
  webhook_url VARCHAR(500), -- Slack/Teams用
  email_address VARCHAR(255),
  created_at TIMESTAMP,
  updated_at TIMESTAMP
);
```

### 3.7 実装方針
- 非同期ジョブキュー（Bull / Celery）で通知送信
- リトライ処理（送信失敗時は3回まで）
- 通知ログ（送信成功/失敗を記録）

---

## 4. 評価集計・ダッシュボード機能

### 4.1 概要
新入社員の成長トレンドを可視化し、人事・上長が全体を俯瞰できるダッシュボード。

### 4.2 表示内容
| グラフ/表 | 内容 |
|-----------|------|
| **評価推移** | 6項目×時系列グラフ（折れ線、週ごと） |
| **現場別平均** | 現場A/B/Cの評価平均比較（棒グラフ） |
| **項目別平均** | 全新入社員の業務理解/主体性などの項目別平均 |
| **課題キーワード** | 「困ったこと」から頻出ワードを抽出（ワードクラウド） |
| **提出率** | 週報提出率（期限内/遅延/未提出） |
| **成長ハイライト** | 評価が大きく向上した新入社員（表彰候補） |

### 4.3 画面追加
- 管理者ダッシュボードに「分析」タブ
  - 期間選択（月次/四半期/年次）
  - フィルタ（現場/新入社員）
  - グラフ・表の表示

### 4.4 API設計
```
GET /api/analytics/trends?user_id=&site_id=&period=
GET /api/analytics/site_comparison
GET /api/analytics/keyword_frequency
GET /api/analytics/submission_rate
```

### 4.5 技術要件
- グラフライブラリ: Chart.js / Recharts / D3.js
- 集計クエリ最適化（インデックス、キャッシュ）
- エクスポート機能（CSV/Excel）

### 4.6 データモデル拡張
- 集計用マテリアライズドビュー or 定期バッチで集計テーブル作成
```sql
CREATE TABLE analytics_summary (
  id UUID PRIMARY KEY,
  user_id UUID,
  site_id UUID,
  week_start DATE,
  avg_score DECIMAL(3,2),
  scores_json JSON, -- 項目別点数
  created_at TIMESTAMP
);
```

---

## 5. 多言語対応

### 5.1 概要
外国籍新入社員向けに英語・中国語などの週報生成・UI対応。

### 5.2 対応範囲
| 要素 | 対応内容 |
|------|----------|
| **UI** | 日本語/英語/中国語切替 |
| **週報生成** | 入力言語を検出し、同じ言語で生成 |
| **評価・アドバイス** | 多言語対応 |

### 5.3 技術要件
- i18n対応（react-i18next / vue-i18n）
- LLMプロンプトに言語指定追加
- 翻訳管理（JSON/YAML）

### 5.4 データモデル拡張
- Userテーブルに`preferred_language` (VARCHAR(10)) 追加

### 5.5 実装方針
- Phase2ではUI多言語のみ（週報生成は英語のみ追加）
- Phase3で中国語・その他言語対応

---

## 6. 週報比較・差分表示機能

### 6.1 概要
前週との比較、生成前後の差分を可視化。

### 6.2 ユースケース
- 新入社員が「今週は先週より成長したか」確認
- OJTが「評価が上がった項目」を視覚的に把握
- 生成前後の差分（手動編集箇所）を確認

### 6.3 画面追加
- 週報詳細画面に「前週と比較」ボタン
  - 評価点数の差分グラフ（+1, -2など）
  - 週報本文の差分（テキストdiff表示）
- 生成結果表示時に「編集箇所ハイライト」

### 6.4 技術要件
- テキスト差分ライブラリ（diff-match-patch）
- 評価差分計算ロジック

---

## 7. OJT指導計画テンプレート

### 7.1 概要
OJT向けに「◯ヶ月目はこのスキルを重点指導」というテンプレートを提供。

### 7.2 内容
- 1ヶ月目: 環境慣れ、基本操作
- 2ヶ月目: 業務理解、自立的タスク遂行
- 3ヶ月目: 品質意識、チーム連携

→ アドバイス生成時にテンプレートを参照し、「今月の重点項目」を強調

### 7.3 データモデル拡張
```sql
CREATE TABLE ojt_templates (
  id UUID PRIMARY KEY,
  month INT, -- 1, 2, 3...
  focus_areas_json JSON, -- [{"theme": "業務理解", "weight": 5}, ...]
  created_at TIMESTAMP
);
```

---

## 8. AIモデル選択・切替機能

### 8.1 概要
管理者が生成に使うLLMモデルを選択可能に（GPT-4o / Claude / Gemini など）。

### 8.2 ユースケース
- コスト削減のため軽量モデルに切替
- 品質重視でハイエンドモデルに切替
- A/Bテストで複数モデルを比較

### 8.3 設定画面
- 管理画面に「LLM設定」タブ
  - モデル選択（GPT-4o / GPT-4o-mini / Claude Sonnet / Claude Haiku）
  - トーン別に異なるモデル指定可能（社内向け=軽量、客先向け=高品質）

### 8.4 データモデル拡張
```sql
CREATE TABLE llm_configs (
  id UUID PRIMARY KEY,
  tone ENUM('internal', 'client'),
  model_name VARCHAR(50),
  temperature DECIMAL(2,1),
  max_tokens INT,
  created_at TIMESTAMP,
  updated_at TIMESTAMP
);
```

---

## 9. 週報の承認フロー強化

### 9.1 概要
現在は「新人→提出→OJT→レビュー」だが、さらに「OJT→上長→人事」のような多段階承認を追加。

### 9.2 ステータス拡張
- draft → submitted → ojt_reviewed → **manager_approved** → **hr_approved** → locked

### 9.3 画面追加
- 各承認者が「承認/差戻し」ボタン
- 差戻し時はコメント必須

### 9.4 データモデル拡張
- WeeklyReportテーブルにステータス追加
- Reviewテーブルに`approval_level` (ENUM: ojt, manager, hr) 追加

---

## 10. モバイルアプリ

### 10.1 概要
iOS/AndroidネイティブアプリまたはPWA（Progressive Web App）で週報作成。

### 10.2 メリット
- プッシュ通知（週報未提出リマインド）
- オフライン対応（下書き保存）
- スマホカメラで音声入力（既存Phase2機能と組み合わせ）

### 10.3 技術候補
- PWA（既存Webアプリをベースに拡張）
- React Native / Flutter（ネイティブアプリ）

---

## 実装優先順位（Phase2〜Phase4）

| Phase | 機能 | ビジネス価値 | 技術難易度 | 推奨順 |
|-------|------|--------------|-----------|--------|
| **Phase2** | 音声入力 | 高（工数削減） | 中 | 1 |
| **Phase2** | 通知連携（Slack/メール） | 高（運用効率） | 低 | 2 |
| **Phase2** | テンプレート管理 | 中（柔軟性） | 中 | 3 |
| **Phase3** | ダッシュボード・分析 | 高（可視化） | 中 | 4 |
| **Phase3** | 多言語対応 | 中（多様性） | 中 | 5 |
| **Phase3** | 週報比較・差分 | 中（利便性） | 低 | 6 |
| **Phase4** | OJT指導テンプレート | 中（品質） | 低 | 7 |
| **Phase4** | AIモデル選択 | 低（最適化） | 低 | 8 |
| **Phase4** | 承認フロー強化 | 低（運用次第） | 中 | 9 |
| **Phase4** | モバイルアプリ | 高（UX） | 高 | 10 |

---

## 設計での考慮事項（MVP時点で準備）

### データモデル
- テーブル設計は正規化し、拡張時にカラム追加のみで対応可能に
- JSON型を活用（sections_json, meta_jsonなど）で柔軟性確保

### API設計
- RESTful設計、バージョニング（/api/v1/...）
- 拡張時は新エンドポイント追加、既存APIは後方互換性維持

### プロンプト設計
- 設定ファイル化（JSON/YAML）でプロンプト変更が容易
- モデル切替時も同じインターフェース

### インフラ
- スケーラビリティ考慮（オートスケーリング、ロードバランサー）
- 音声ファイル保存先（S3など）は初期から想定

---

## まとめ

MVPは「週報作成→生成→レビュー→PDF」の最小機能に絞り、Phase2以降で段階的に拡張します。

**Phase2優先機能**:
1. 音声入力（工数削減の決定打）
2. 通知連携（運用効率向上）
3. テンプレート管理（現場ニーズ対応）

**Phase3以降**:
- ダッシュボード（経営層への可視化）
- 多言語対応（グローバル展開）
- 高度な分析・指導機能

データモデル・API設計はPhase2を見越して拡張性を確保し、スムーズな機能追加を実現します。

---

**以上**

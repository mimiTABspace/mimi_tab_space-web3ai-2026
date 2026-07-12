# バージョンアップ内容 - Self Care Quest v0.7.0 / Self Care Note v0.4.0

**バージョン**: SCQuest 0.6.0 → 0.7.0 / SCNote 0.3.0 → 0.4.0
**更新日**: 2026-07-13

---

## SCQuest v0.7.0 主要変更

### 背景テーマ機能

ショップのくじ・所持アイテムページから、アプリ全体の配色を変更できる機能を追加。

- テーマ一覧: 桜（ピンク系）・夜空（紺系）・青空（SCNote同系統の水色系、アクセントカラーも連動して変化）
- インベントリページ（`/shop/inventory`）のテーマアイテムに「装備する」ボタンを追加、装備中はダークモード設定より優先される
- 設定画面から `?filter=theme` 付きでインベントリへ遷移する導線を追加
- `--accent` / `--accent-hover` をCSS変数化し、テーマごとに配色を切り替え可能に変更

### 相棒セリフのAI生成（Gemini連携）

装備中のセリフパック（性格）に応じて、相棒のセリフをGeminiがその場で生成する機能を追加。

- 使用モデル: `gemini-3.1-flash-lite`（Google AI Studio無料枠）
- 既存の固定セリフ（xlsxで管理していたパック別セリフ集）をfew-shot例文としてGeminiに渡し、キャラのトーンを保ったまま新しいセリフを生成
- 対応シーン: ログイン/起動時・おやつ・レベルアップ・体調記録完了・服薬記録完了・通院完了・Daily Quest完了・種族解放時
- 1日10回までの呼び出し予算（`ai_daily_usage.serif_call_count`）、予算切れ・API失敗時は元の固定セリフにフォールバック
- セリフパック未装備の場合は今まで通り固定ロジックのまま（既存機能に影響なし）
- 各ページでの反応は優先度付きでキュー保存され、ペット画面に戻った際にまとめて確認できる

### 通院促し・服薬管理コメントのAI化

固定しきい値（3日連続で気分・疲労度が低い等）による通院促し判定を廃止し、Geminiによる総合的な判断に変更。

- 体調記録の保存時にEdge Function（`health-advice`）を呼び出し、直近の体調記録（気分・疲労度・睡眠・体温・体重の数値項目のみ）から受診を勧めるべきか判定
- **症状メモ（自由記述）はAIに送信しない**（無料枠APIはGoogle側の学習利用対象になり得るため、プライバシー配慮で数値項目のみに限定）
- 呼び出し予算: 通院促し1日3回・服薬管理コメント1日1回（別枠）。予算切れの場合は前回の判定結果を再利用
- 判定結果は `ai_daily_usage` テーブルにキャッシュし、ホーム画面はそのキャッシュを読むだけの設計に変更
- SCQuest・SCNoteは同じSupabaseプロジェクト・Edge Functionを共有し、呼び出し予算も両アプリ間で共有

### 表示文面の二重生成（アプリ間の一貫性対応）

同じ判定結果をSCQuest・SCNoteの両方で参照する設計上、「どちらのアプリが先に呼び出したか」で表示トーンが変わってしまう問題があった。これを解消するため、Edge Function側で装備中セリフパックの有無を自分で判定し、1回のGemini呼び出しで以下2種類の文面を同時生成する方式に変更。

- `messagePack` / `linePack`: SCQuestで表示（装備中セリフパックのキャラトーン、未装備時は固定トーンと同一）
- `messagePlain` / `linePlain`: SCNoteで表示（固めのシステム通知トーン）

### デバッグ用AIテストモード

開発者モードに「AIテストモード」画面を追加。実データ（health_logs等）を汚さず、AI呼び出し予算も消費せずに、体調・服薬の任意パターンを入力してAIの応答（両アプリ分の文面）を確認できる。

---

## SCNote v0.4.0 主要変更

SCQuestと同じSupabaseプロジェクト・Edge Function（`health-advice`）を共有して通院促し・服薬管理コメントのAI化を実装。

- 体調記録保存時に同じEdge Functionを呼び出し（追加のデプロイ・DB変更は不要）
- ホーム画面に🔔お知らせカードを新設し、通院促しメッセージを表示（固めのシステム通知トーン）
- 服薬管理コメントはトースト表示
- SCNoteには相棒キャラ・セリフパックの概念がないため、常に固定トーン（`messagePlain`/`linePlain`）を使用

---

## DBスキーマ変更（v0.7.0実装時に適用済み・SCQuest/SCNote共有プロジェクト）

```sql
-- user_settings テーブルに追加
ALTER TABLE user_settings ADD COLUMN IF NOT EXISTS equipped_theme text;
ALTER TABLE user_settings ADD COLUMN IF NOT EXISTS equipped_serif text;

-- ai_daily_usage テーブルを新設（1日単位のAI呼び出し予算・結果キャッシュ）
CREATE TABLE IF NOT EXISTS ai_daily_usage (
  user_id           uuid NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  date              date NOT NULL,
  serif_call_count  int NOT NULL DEFAULT 0,
  health_call_count int NOT NULL DEFAULT 0,
  medication_done   boolean NOT NULL DEFAULT false,
  last_health_result jsonb,
  last_medication_result jsonb,
  updated_at        timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (user_id, date)
);
```

---

## 技術メモ

- Edge Function: `generate-serif`（相棒セリフAI生成）・`health-advice`（通院促し・服薬コメントAI化）の2つを新設。いずれもDeno + Gemini REST API（`responseSchema`による構造化出力）を使用
- Gemini APIは当初 `gemini-2.5-flash-lite` を想定していたが、新規プロジェクトでは404となり使用不可だったため `gemini-3.1-flash-lite` に変更
- `health-advice` はEdge Function内で `user_settings.equipped_serif` を自ら参照し、クライアント側からパック情報を渡す必要がないよう設計（呼び出し元アプリを問わず一貫した挙動にするため）
- 症状メモ（自由記述）を一切AIに送らない方針のため、当初構想にあった「外部送信オプトイン同意トグル」の実装は不要になった
- アカウント削除・データリセット処理（両アプリ）に `ai_daily_usage` テーブルを削除対象として追加

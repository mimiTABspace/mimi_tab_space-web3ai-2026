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

## バグ修正

### Daily Quest完了判定が朝夕どちらか一方でも成立してしまう不具合を修正（SCQuest ホーム画面）

**バグ内容**
朝の体調記録だけを入力した時点で、Daily Quest完了ボーナス（XP+50・コイン+20）と、相棒セリフAI生成による「クエスト全クリー！！」といった全クリア祝いのセリフが発動してしまっていた。夕の記録がまだでも発動する状態。

**原因**
`app/page.tsx` の完了判定用フラグ `healthDoneVal` が `morningDoneVal || eveningDoneVal`（どちらか一方でtrue）になっており、実際に画面上部の「Daily Quest Completed!」バナーで使われている `morningDone && eveningDone`（両方必須）の判定条件と食い違っていた。

相棒セリフAI生成機能の追加によって、この不整合が「全クリしてないのに全クリ祝いのセリフが出る」という形で目に見えるようになったことで発覚。バグ自体はAI機能追加以前から存在していた。

**修正内容**
`healthDoneVal` を `morningDoneVal && eveningDoneVal` に変更し、朝・夕の両方の記録が完了した場合のみDaily Quest完了（ボーナス付与・AIセリフ生成）が発動するよう修正。

### 家族共有招待でログイン後にホーム画面へ戻ってしまい、共有が成立しない不具合を修正（SCQuest・2026-07-15）

**バグ内容**
未ログイン状態で招待リンク（`/invite/[token]`）を開き、「Googleでログインして承諾する」からログインすると、ログイン完了後に常にホーム画面（`/`）へ遷移してしまい、招待ページに戻れず「招待を承諾する」ボタンを押せないまま終わっていた。承諾操作（`family_shares`へのinsert）が実行されないため、共有が成立していなかった。

**原因**
`contexts/AuthContext.tsx` の `signInWithGoogle` がGoogle認証後のリダイレクト先を常に `/auth/callback` 固定にしており、`app/auth/callback/page.tsx` 側もログイン検知後に常に `router.replace("/")` していたため、ログイン前にいたページ（招待ページ）の情報が失われていた。

**修正内容**
ログイン開始時に現在のパスを `next` クエリパラメータとして `/auth/callback?next=...` に渡すように変更し、コールバックページ側もそのパラメータの遷移先へ `router.replace` するよう修正。これによりログイン後に招待ページへ戻り、そのまま承諾操作を続行できるようになった。

---

## 追加のUI改善（2026-07-15）

- 体調記録（`/health`）: 記録保存後に「過去の記録」セクションまで自動スクロールし、保存内容がすぐ確認できるように変更
- 服薬管理（`/medications`）: 「新しい薬を登録」フォームをアコーディオン化し、デフォルトで閉じた状態に変更（編集ボタン押下時は自動的に開く）
- 家族共有の招待メール入力欄に「招待する相手がログインで使うGoogleアカウントのメールアドレスを入力してね」という注意書きを追加

---

## プライバシーポリシーの更新（SCQuest v0.3.0 / SCNote v0.2.0）

AI機能でGoogleのGemini APIを利用するようになったことに伴い、両アプリのプライバシーポリシーを更新。

- 「3. 情報の利用目的」の「第三者への提供は一切行わない」という記述が実態と合わなくなったため見直し
- 新設した「4. AI機能について（Google Gemini APIの利用）」で以下を明記
  - AIで送信する情報は体調記録の数値項目（気分・疲労度・睡眠・体温・体重）と服薬の薬名・服薬日のみ
  - **症状メモ等の自由記述は送信しない**（プライバシー配慮のため意図的に除外）
  - Gemini APIは無料枠を利用しており、無料枠では送信内容がGoogle社のサービス改善に利用される場合がある旨を明記
  - SCQuestのみ、相棒セリフ生成についても追記（セリフパック設定・場面情報のみ使用、体調記録等の個人情報は含まない）

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

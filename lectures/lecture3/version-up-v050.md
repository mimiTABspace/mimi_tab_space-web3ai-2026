# バージョンアップ内容 - Self Care Quest v0.5.0 / Self Care Note v0.2.0

**バージョン**: SCQuest 0.4.0 → 0.5.0 / SCNote 0.1.0 → 0.2.0  
**更新日**: 2026-06-25

---

## SCQuest v0.5.0 主要変更

### 服薬タイミング管理システムの刷新（`/medications`）

従来の「1日の服薬回数（数値）」を廃止し、実際の服薬タイミングを直接設定する方式に変更。

**薬の登録フォーム**
- 食前・食後ラジオボタン（指定なし / 食前 / 食後）
- 服薬タイミングのチェックボックス複数選択（朝 / 昼 / 夕 / 寝る前 / 起きた後 / 間食）
- 選択中のタイミング一覧と「1日N回」の自動集計を表示

**服薬チェックUI**
- タイミングごとのボタンを並列表示（例: 朝 / 昼 / 夕）
- 服薬済みのタイミングは緑のチェックマーク付きボタンに変化
- 全タイミング完了で「完了」バッジを表示

**頓服の専用UI**
- 「必要なときだけ飲む」という性質に合わせてワンタップ記録に変更
- 登録フォームで錠剤数と1日の上限回数を横並びで設定（両方必須）
- 上限回数に達したボタンは「上限(N回)」表示でロック

**服薬メモ機能**
- 「▸ メモ（飲み合わせ・注意事項）」トグルで入力欄を展開
- 服薬チェック画面でメモが登録された薬に `📝 ○○` を小さく表示
- 例: 「ロキソニンと同時に飲まないこと」「必ず水と一緒に」

**後方互換**
- タイミング未設定の既存薬は `N/M回 ※タイミング未設定` の旧スタイルで表示

---

### 体調記録ページの時間自動切替（`/health`）

ページを開いた時刻に応じて朝/夕タブを自動選択。

| 時刻 | 自動選択 |
|------|---------|
| settings で設定した夕開始時刻（デフォルト16時）より前 | 朝の記録 |
| 夕開始時刻以降 | 夕の記録 |

設定は `/settings` の「時間帯設定」（`scq_evening_start_hour`）を参照。

---

### Daily Questの服薬判定改善（ホーム）

- 判定対象を「常備薬」→「処方薬（type: prescribed）」に変更
- 服用期間（start_date / end_date）でフィルタし、現在有効な薬のみを対象に
- 進捗バッジ表示: 「N/M回」形式で部分完了を可視化
- サブテキスト: 「あとN回飲もう」「今日の服薬完了！」に変化

---

### 家族共有の名前表示バグ修正（全ページ）

**バグ内容**  
AがBのデータを閲覧したとき、Bの名前欄にAのメールアドレスが表示されていた。

**原因**  
`family_shares` テーブルの `member_email` は「共有される側のメール」として保存されている。  
招待承諾時に `owner=B, shared_with=A, member_email=Aのメール` という行が作られるため、  
AがBのデータを見るときに `member_email` を参照するとAのメールになってしまっていた。

**修正方法**  
各ページで `family_shares where owner_id = 自分` の行（myShares）も追加取得。  
`myShares.find(s => s.shared_with_id === 相手のID)` でnickname / member_emailを引くように変更。  
これにより `nickname`（自分が相手につけた名前）または相手のメールを正しく表示できる。

**対象ページ**  
- `/health` — 体調ログの家族フィルタ
- `/appointments` — 通院予定の家族セクション
- `/routes` — Care Route の家族セクション（nicknameも表示するよう改善）
- `/medications` — 服薬チェックの家族セクション

---

## SCNote v0.2.0 主要変更

### 服薬タイミング管理システムの実装

SCQuest v0.5.0 の服薬タイミングUIをゲーム要素なしでポート。

- 登録フォーム: 食前・食後 ＋ タイミングチェックボックス（朝/昼/夕/寝る前/起きた後/間食）
- 服薬チェックUI: タイミングごとのボタン表示
- 頓服: ワンタップ記録 ＋ 錠剤数・上限回数の横並び設定
- 服薬メモ（アコーディオンUI）
- ホームの服薬チェックを処方薬・服用期間フィルタに変更

---

### ホームの体調記録を3段階表示

`healthDoneToday`（真偽値）を `morningDone` + `eveningDone` に分割。

| 状態 | サブテキスト | アイコン |
|------|------------|---------|
| 未記録 | 朝・夜の体調を入力 | ○（グレー） |
| 朝のみ記録済み | 朝済み・夕未記録 | ○（アンバー） |
| 夕のみ記録済み | 夕済み・朝未記録 | ○（アンバー） |
| 両方記録済み | 今日の記録済み | ✓（グリーン） |

---

### 体調記録ページの時間自動切替（`/health`）

12時以降にページを開くと自動で夕タブを選択。

---

### 家族共有の名前表示バグ修正（全ページ）

SCQuestと同じ原因・同じ修正。health / appointments / routes / medications 全ページ対応。

---

## バグ修正・技術的修正

| 種別 | 内容 |
|------|------|
| SCQuest バグ | ペットデバッグ時の `pet_profiles` selectに `affection_tabcat` が含まれておらず、tabcatのデバッグ処理が正常に動作しなかった問題を修正 |
| SCQuest 型エラー | `petImage()` 関数でペット種別の文字列を型アサーションなしにオブジェクトのインデックスキーとして使用していた問題を修正 |
| 両アプリ 型エラー | 体調ログの `myLogs` と `familyLogs` をマージ後のソートで `logged_at` プロパティが型推論上見えなかった問題を修正 |
| 両アプリ 型エラー | `next.config.ts` の `eslint` プロパティが `NextConfig` 型にないためTS警告が出ていた問題を型アサーションで抑制 |
| SCQuest 設定 | `tsconfig.json` の型チェック対象から `supabase/functions/`（Deno環境）を除外 |

---

## DBスキーマ変更（v0.5.0実装時に適用済み）

```sql
-- medications テーブルに追加
ALTER TABLE medications
  ADD COLUMN food_timing text,
  ADD COLUMN time_slots text[] DEFAULT '{}',
  ADD COLUMN notes text;

-- medication_logs テーブルに追加
ALTER TABLE medication_logs
  ADD COLUMN time_slot text;
```

---

## 技術メモ

- `time_slots: string[]` に設定タイミングを保存。服薬判定は `time_slots` の全スロットに対してログが存在するかを確認
- 頓服の上限回数は既存の `frequency_per_day` カラムを流用（処方薬・常備薬では使用しない）
- `isSlotTaken(medId, slot)`: `logsToday` から当日分のスロット別服薬済み判定
- `isMedDone(med)`: `time_slots` ありなら全スロット確認、なしなら `frequency_per_day` で従来通り
- 家族共有の名前参照: `myShares = family_shares where owner_id = 自分` を各ページの `Promise.all` に追加して取得。`owner_id = 自分, shared_with_id = 相手` の行に `member_email = 相手のメール` と `nickname = 自分が相手につけた名前` が格納されている
- SCNote の時間自動切替は固定値12時（SCQuestはlocalStorage設定値を参照）

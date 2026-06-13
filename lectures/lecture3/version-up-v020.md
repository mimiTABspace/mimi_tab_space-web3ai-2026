# バージョンアップ内容 - Self Care Quest v0.2.0


**バージョン**: 0.1.0 → 0.2.0  
**更新日**: 2026-06-09

---

## 主要新機能

### 家族共有機能

v0.2.0 の最大のアップデート。家族間でアプリのデータを共有できるようになった。

**仕組み:**
1. 設定ページから招待したい家族のメールアドレスを入力 → 招待リンクを生成
2. リンクをLINEなどで相手に送る
3. 相手がリンクを開いてGoogleログイン → 承認 → 共有スタート

**共有できるカテゴリ（個別ON/OFF）:**
- 体調記録
- 服薬記録
- 通院記録
- Care Route

**各ページの表示:**
- 「すべて / 自分のみ / 家族のみ」フィルターで表示切り替え
- 家族の記録は名前（メールアドレス）付きで表示
- 不要な共有記録は「×」で非表示にできる（共有元のデータは削除されない）

**追加ページ・機能:**
- `/invite/[token]` — 招待受け入れページ（未ログイン時はGoogleログイン案内）
- 設定ページに「家族共有」セクション追加
- 共有設定の解除・招待キャンセル機能

**Supabase テーブル追加:**
- `family_shares` — 共有関係と共有カテゴリの設定
- `family_invitations` — 招待トークン管理
- `dismissed_shared_items` — 非表示にした共有アイテム

---

## その他の追加機能

### Daily Quest Completed! バナー
- 体調入力と服薬（薬が登録されている場合）の両方を完了したら「Daily Quest Completed!」バナーをホームに表示

### 連続記録 (Streak) 機能
- `updateStreak` ヘルパーを追加
- 体調記録を保存するたびにストリークを更新
- 当日2回目以降の記録ではストリーク変更なし

---

## バグ修正・改善

| 種別 | 内容 |
|------|------|
| 型定義 | `FamilyShare` に `member_email` / `FamilyInvitation` に `inviter_email` を追加 |
| XP一元化 | `addXP` ヘルパーを `lib/supabase.ts` に追加。health / medications / appointments / pet の4ページで統一 |
| レベルアップ判定 | 体調記録・服薬・通院でのXP加算時にレベルアップ判定が行われていなかった問題を修正 |
| Care Route紐づけ | `care_route_id` に 0 が入るバグを修正（未選択時は `null` を送るよう変更） |
| シードデータ | `seedUserData` にエラーハンドリング追加。失敗しても認証フローがブロックされなくなった |
| 非表示機能 | 家族の共有アイテムを各ページで非表示（dismiss）できる機能を全4ページに追加 |

---

## インフラ・設定

### Vercel デプロイ
- `selfcare-quest.vercel.app` で本番公開開始
- `next.config.ts` に `typescript: { ignoreBuildErrors: true }` / `eslint: { ignoreDuringBuilds: true }` 追加（Vercelビルド対応）

### プライバシーポリシー更新
- 「5. 家族共有機能について」を追記
  - 共有の仕組み・設定変更・解除方法
  - 家族は閲覧のみで元データを変更できない旨

---

## 技術メモ

- 招待はトークン（UUID）をURLに含めるリンク方式。メール送信なし
- 家族共有のRLSポリシー: `owner_id = auth.uid() OR shared_with_id = auth.uid()` で両者がアクセス可能
- 非表示アイテムは `dismissed_shared_items` に記録し、表示時にフィルタリング
- `addXP(userId, xpGain, coinGain)` — XP加算 + レベルアップ判定 + coins更新を一括で処理

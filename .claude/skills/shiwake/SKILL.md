---
name: shiwake
description: マネーフォワードクラウド会計（MCP mfc_ca）の未仕訳明細を過去仕訳とルール表に基づいて振り分け、ユーザー承認後に仕訳登録する。Use when: 「振り分けをしろ」「未仕訳を処理して」「仕訳登録して」と言われたとき、毎月 5 日の月次処理。NOT for: 仕訳ルールの相談だけ（登録しない）、決算整理仕訳。
---

# 未仕訳の振り分け

正本ルール: `docs/shiwake-rules.md`。科目判定・API の注意点はすべてこのファイルに従う。最初に必ず読む。

## 手順

1. **再取得の依頼**: ユーザーに MF 画面「データ連携 → 登録済一覧 → 一括再取得」を押したか確認する。未実施なら依頼して待つ（MCP から再取得はできない）。
2. **疎通**: `mfc_ca_currentOffice` で会計期間を取得する。認証エラーなら `docs/shiwake-rules.md` と memory の `mf-cloud-mcp-setup` の対処をユーザーに案内して待つ。
3. **未仕訳の取得**: `mfc_ca_getTransactions` を `journalizing_statuses: ["none"]`、期間は会計期間の開始日〜今日、`per_page: 500` で取得する。0 件なら報告して終了。
4. **照合**: 各明細を `docs/shiwake-rules.md` の摘要パターン表に当てる。表にないものは `mfc_ca_getJournals`（会計期間、`per_page: 500`。結果が大きいときはツール結果ファイルを Python で読む）で類似の過去仕訳を探す。
5. **提案**: 「登録可」と「保留」に分けた表（日付・内容・金額・科目・根拠）を提示する。保留には理由を付ける。受取利息は総額と源泉分を併記する。**ここで必ず停止し、承認を待つ。**
6. **登録**（承認後のみ）:
   - `mfc_ca_postTransactionJournalize` に `transaction_id` と `account_id` を渡す。`tax_id` は指定しない。ID は API が返した URL エンコード済み文字列をそのまま使う。
   - 複数行が必要な仕訳（受取利息の源泉分、役員社宅の賃料など）は、戻り値の `journal.id` に `mfc_ca_putJournals` で branches を書き換える。
   - 明細に紐付かない仕訳（社宅使用料の未収入金など）は `mfc_ca_postJournals` で作成する。
   - エラーになった明細は原因を直して再実行し、重複登録がないことを戻り値の `number` で確認する。
7. **残件確認**: 再度 `mfc_ca_getTransactions` で未仕訳が保留分のみになったことを確認する。
8. **ルール更新**: 新しく確定した摘要パターンや残高（未収入金など）を `docs/shiwake-rules.md` に追記し、memory `mf-shiwake-rules` の要点も更新する。
9. **コミット**: `docs/shiwake-rules.md` などに変更があれば、このスキルの手順として `git add -A && git commit && git push` を実行する（コミットメッセージ例: `Update shiwake rules: 2026-10 月次処理`）。変更がなければスキップする。
10. **報告**: 登録した仕訳番号一覧、保留とその理由、税理士へ伝える事項、ユーザーへの依頼（代表の振込など）をまとめる。

## 禁止事項

- 承認前に `postTransactionJournalize` / `postJournals` / `putJournals` を呼ばない。
- 仕訳の削除はしない（必要ならユーザーに依頼する）。
- 確信のない明細を推測で登録しない。保留にする。

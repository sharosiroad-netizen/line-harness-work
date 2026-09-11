# L Harness セットアップ・トラブルシューティング記録

このリポジトリは、L Harness (self-hosted LINE CRM, https://github.com/Shudesu/line-harness-oss)
を `npx create-line-harness` で構築した際に遭遇した問題と対処法の記録です。
次回、別の環境で新規に構築する際に同じ問題で時間を溶かさないための備忘録。

## 1. LIFF ID が DB に保存されず `{{form_url:...}}` が壊れる

### 症状
- シナリオのボタン（`{{form_url:FORM_ID}}` を含む URI アクション）を含むメッセージが
  **一切配信されず、該当のシナリオ enrollment が `status = 'paused'` のまま止まる**。
- ログには「LINE API error: 400」系のエラーが cron 配信側 (`processStepDeliveries`) の
  catch でのみ記録され、即時配信パス (`pushImmediateFirstStep` 等) 側では
  静かに失敗する（呼び出し元には `false` が返るだけで理由が分からない）。
- CTA メニューや Tips 配信シーケンスなど、フォームリンクを含むほぼ全てのステップが
  影響を受ける（1ステップだけの問題ではない）。

### 根本原因
`line_accounts.liff_id` カラムが **NULL のまま** になっていた。
LINE Developers Console 側で LIFF アプリを作成・Publish し、Endpoint URL も設定した
つもりでも、その `liffId` の値が L Harness の DB (`line_accounts` テーブル) に
書き込まれるとは限らない（今回のケースでは書き込まれていなかった）。

`expandVariables()` (apps/worker/src/services/step-delivery.ts) の実装:
```ts
if (liffId) {
  result = result.replace(/\{\{form_url:([^}]+)\}\}/g, ...);
}
```
`liffId` が falsy (null) のときは **置換自体をスキップ** する仕様（意図的:
空文字にすると「リンクが消えたメッセージ」が届いてしまうため）。
結果、メッセージ本文に `{{form_url:xxx}}` という **文字列がそのまま** 残り、
LINE Messaging API がボタンの `uri` フィールドとして受理せず 400 エラーになる。

### 検出方法（次回はセットアップ直後にこれを実行する）
```sql
SELECT id, name, liff_id FROM line_accounts;
```
`liff_id` が NULL の行があれば要注意。また、影響範囲を洗い出すには:
```sql
SELECT s.name, ss.step_order FROM scenario_steps ss
JOIN scenarios s ON s.id = ss.scenario_id
WHERE ss.message_content LIKE '%form_url%';
```

### 対処法
LINE Developers Console の LIFF タブに表示されている LIFF ID
（例: `2011513813-v7GfyGKi` の形式）を直接 DB に書き込む:
```sql
UPDATE line_accounts SET liff_id = '<LIFF_ID>' WHERE id = '<line_account_id>';
```
書き込み後、既に `paused` になってしまった enrollment 行は
`DELETE FROM friend_scenarios WHERE id = '...'` して友だち側に再度トリガーを
引いてもらう（tag 付与や postback の再送信）か、管理画面から作り直す。

### 次回セットアップ時のチェックリスト
LIFF を設定した直後に必ず `SELECT liff_id FROM line_accounts` で NULL でないことを
確認する。管理画面 (admin dashboard) に liff_id 設定用の UI がない/機能しない場合は
上記の直接 UPDATE で対応する。

---

## 2. シナリオの「即時配信」は1ステップしか進まない（連続する delay=0 ステップが連鎖しない）

### 症状
- 友だち追加やボタンタップ直後、シナリオの **最初のメッセージだけ** は即座に届くが、
  同じシナリオ内で delay=0 に設定した **2通目以降** のメッセージは
  次の cron tick（設定次第で 1〜5分後）まで届かない。
- 「タップ後の次のシナリオ配信がすぐに届かない」というユーザー体感の直接原因。

### 根本原因
`pushImmediateFirstStep()` (apps/worker/src/services/immediate-first-step.ts) は
**「新規 enrollment のステップ1をすぐ送る」ことだけ** を目的に設計された関数で、
呼び出す度に `scenarioRow.steps` の **先頭から** 条件未設定の最初の delay=0
ステップを再計算して選び直す。すでに配信済みの enrollment に対して同じ関数を
ループで呼び出しても、`current_step_order` が「選び直されたステップの step_order」
以上であるため即座に `return false` するだけで、**2番目以降のステップには
絶対に進めない**（関数の設計スコープ外）。

同じ勘違いに基づく「ループで pushImmediateFirstStep を呼べば連鎖するはず」という
実装は複数箇所（friend_add / automations の start_scenario / フォーム送信後）に
同じ形で紛れ込みうるので注意。

### 正しい修正方法
cron が使っている **既存の1件配信ロジック** `processSingleDelivery()`
(apps/worker/src/services/step-delivery.ts) を、独立した関数
`pushNextDueStepIfImmediate()` として export し、それをカスケード呼び出しに使う。
この関数は enrollment の実際の `current_step_order` を見て「次に届くべきステップ」
を正しく特定し、かつ `next_delivery_at` が既に到来しているかを確認してから配信する
ため、cron と全く同じロジックをそのまま「今すぐ」実行できる。

修正箇所（3箇所、いずれも「ステップ1は pushImmediateFirstStep、2通目以降は
pushNextDueStepIfImmediate をループ」というパターン）:
- `apps/worker/src/routes/webhook.ts`（friend_add / follow イベント）
- `apps/worker/src/services/event-bus.ts`（automations の `start_scenario` アクション）
- `apps/worker/src/routes/forms.ts`（フォーム送信後の `on_submit_scenario_id`）

---

## 3. `wrangler deploy` が古い/壊れたビルドへ静かにリダイレクトされる

### 症状
- `wrangler.toml` を編集して `wrangler deploy` を実行しても、変更が反映されない
  （例: cron の実行間隔を変更しても古い設定のまま）。
- 一度 `vite build` を単体実行すると、以後の `wrangler deploy` が
  「Using redirected Wrangler configuration」と表示し、
  意図しないバンドルをデプロイしてしまう（動作が壊れることがある）。

### 根本原因
`@cloudflare/vite-plugin` は `vite build` 実行時に
`.wrangler/deploy/config.json`（`{"configPath": "../../dist/xxx/wrangler.json"}`）
という **リダイレクトポインタ** を書き込む。これが存在する限り、以後
`wrangler deploy`（設定ファイルを明示しない場合）は **ルートの `wrangler.toml`
ではなく、そのポインタが指す方** を使い続ける。

L Harness の公式デプロイ手順 (`packages/create-line-harness/src/steps/deploy-worker.ts`)
では本番アーティファクトは GitHub リリースから取得した **ビルド済みバイナリ**
(`dist/release/index.js`, `wrangler.toml` 側は `main = "dist/release/index.js"` +
`no_bundle = true`) をそのままデプロイする。ソースから手動で
`vite build && wrangler deploy` する場合は、**ビルド時点で `wrangler.toml` の
`main` を一時的に `"src/index.ts"` に、`no_bundle` を外した状態** にしてから
`vite build` → `wrangler deploy` する必要がある（`main` が既にビルド済み
バイナリを指したまま `vite build` すると、正しくビルドされない/壊れた
バンドルができる）。

### 対処法・チェックリスト
- 設定だけの変更（cron 間隔など）をデプロイしたいときは、
  `.wrangler/deploy/config.json` と `dist/<worker_name>/` ディレクトリが
  **残っていないこと** を確認してから `wrangler deploy --no-bundle` する
  （残っていれば削除してから実行）。
- ソースコードを修正してデプロイし直したいときは、必ず
  `wrangler.toml` の `main` を `"src/index.ts"` にして `no_bundle` を外した
  状態で `vite build` → `wrangler deploy` する（公式スクリプトと同じ手順）。
  この場合、公式リリースの `_version.ts` スタンプを失うため、以後の
  `create-line-harness update` の fork 検出に軽微な影響が出る点に注意。
- デプロイ後は `wrangler deployments list` で実際に新バージョンが
  100% トラフィックに乗っているか必ず確認する。

---

## 一般的な教訓

1. **本番相当のシナリオ機能（連続メッセージ、フォームリンク）は、設定変更直後に
   必ず実際にボタンをタップして end-to-end で検証する。** cron 配信に頼ると
   「数分後に届く／届かない」の判別がつきにくく、バグの発見が大幅に遅れる。
2. **`{{form_url:...}}` を含むシナリオを作る前に、必ず `line_accounts.liff_id`
   が設定済みであることを確認する。**
3. **`wrangler deploy` を叩く前に `.wrangler/deploy/config.json` の有無を確認する
   習慣をつける。** 存在する場合、それが指す設定が実際に使われる設定である。

# 新しいテーマで委員会を立ち上げる手順（第2世代）

テーマ固有の情報は `data/state.json` にしか無い。ルーティンのプロンプト（`ROUTINE_PROMPT.md`）もページ（`page.html`）も運用ルール（`data/rules.md`）もテーマ非依存なので、そのままコピーして使う。委員の人格は `.claude/agents/*.md` にあり、これがテーマごとに調整する主な対象になる。

## 1. リポジトリを作る

```
cp -r committee-template <theme>-committee-repo
cd <theme>-committee-repo
# data/state.json の ____ を埋める（下の「state.json の埋め方」参照）
git init && git add -A
git -c user.email=noreply@anthropic.com -c user.name=Claude commit -m "Set up <theme> committee"
~/.local/bin/gh repo create cc-routine-<theme>-committee --public --source=. --remote=origin --push
```

push はサンドボックスのネットワーク制限に当たるので `dangerouslyDisableSandbox: true` で実行する。

## 2. 委員を調整する

委員会は `.claude/agents/*.md` に定義された6人の独立サブエージェント（`tools_sync_team.py` の `ROSTER`: watanabe / suda / yoneda / ashiya / nishimura / takanashi）として動く。frontmatter（人格・口調）はそのまま使い回せるが、**「## あなたが自分で調べる領域」の本文はテーマに合わせて書き換えること**。

- 各エージェント定義の「あなたが自分で調べる領域」を、新しいテーマの対象に即した記述に書き換える。**担当領域を重複させないこと**（rules.md R21）。既定の6分担は「経営・提携／決算実数・開示の質／バリュエーション・資金配分・出口／英語圏の市場・競合／板・出来高／技術・材料の一次資料」。テーマがこれに合わない場合（投資以外のテーマ等）は担当そのものを組み替えてよいが、重複させないことだけは守る。
- 「## 申し送り（過去サイクルの実績）」は新規テーマでは空である。テンプレートのまま `（新規テーマ。実績なし。）` にしておき、サイクルが進んだら同期ループ側が実績を追記する（rules.md R20: 委員会自身は自分の `track` を書き換えない）。
- 委員を増減・入れ替える場合は、エージェント定義ファイルを追加・削除した上で `tools_sync_team.py` の `ROSTER` を編集すること。
- 調整が終わったら次を実行し、`data/state.json` の `team` を `.claude/agents/*.md` の frontmatter から再生成する（`team` を手で編集しても無視される。rules.md R11）:

```
python3 tools_sync_team.py
```

## 3. 比較検討モードなら拡張ルールを追記する

`data/state.json` の `mode` を `"comparison"` にした場合（複数候補の相対比較）は、`data/rules-comparison.md` の内容を `data/rules.md` の末尾に追記する。追記時、`data/rules-comparison.md` 内の年数定数（0.296年・1.295年など）やベンチマーク例は元テーマの実例なので、新しいテーマの基準日・判断期限に合わせて数値を置き直すこと。

`mode` を `"single"`（単独テーマの可否）にする場合は `data/rules-comparison.md` は使わない。`data/state.json` の `candidates` は空配列のままでよい（`tools_validate_cycle.py` は `candidates` が空ならランキング系の機械検査を自動でスキップする）。

## 4. ルーティンを作る

`RemoteTrigger` の `create` に以下を渡す。**`ROUTINE_PROMPT.md` の本文（`---` より下）をそのまま** `events[0].data.message.content` に入れる。テーマごとの書き換えは不要（委員の人数・名前は `.claude/agents/` を進行役が直接見て決めるため、プロンプト側は変更不要）。

```json
{"name": "<テーマ>委員会ラウンド進行",
 "cron_expression": "24 */4 * * *",
 "job_config": {"ccr": {
   "environment_id": "env_01VwsGefcy95FzegPjcNrSh1",
   "session_context": {
     "model": "claude-sonnet-5",
     "allowed_tools": ["Read","Write","Edit","Bash","WebSearch","WebFetch","Task","Agent"],
     "sources": [{"git_repository": {"url": "https://github.com/AGDRI/cc-routine-<theme>-committee"}}]},
   "events": [{"data": {"message": {"role": "user", "content": "<ROUTINE_PROMPT.md の本文>"}}}]}}}
```

注意:
- `environment_id` は必須。省略すると 400 になる。
- **`allowed_tools` に `Task` と `Agent` を必ず含めること。** 委員は独立したサブエージェントとして呼び出す設計（rules.md R9）なので、これが無いと進行役が委員を起動できない。
- `update` は `job_config` を**丸ごと置き換える**。部分更新するとプロンプトが消えるので、必ず `events` 全体を再送する。
- cron の「分」はサーバー側で実行時刻付近に書き換えられる。時刻の指定は「何時間おきか」だけが効くと考えてよい。`24 */4 * * *`（4時間ごと）が既定のcadenceに対応する。
- **手動 `run` を打つ前に `get` で `next_run_at` を確認すること。** 定期実行の直前直後に手動実行を重ねると2本が同じサイクル番号を書きに行く。実測では後発がリベースで吸収して事故にはならなかったが、無駄な1サイクルを消費する。

## 5. ページを公開する

```
cp committee-template/page.html <theme>-committee.html
# <title> の1行だけテーマ名に書き換える
```

`Artifact` で publish（`capabilities: {"db": {}}`、favicon は絵文字1〜2字）。file_path がURLを決めるので、テーマごとに別ファイル名にすること。page.html はテーマ非依存（`data/state.json` の内容を読んで描画する）なので、`<title>` 以外は変更不要。

## 6. 初回サイクルを試験実行して検証する

`RemoteTrigger` の `run` を1回打ち、5分ほど待って `get_run_log` で確認。生成された `cycle-1.json` を検証する:

```
python3 tools_validate_cycle.py 1
```

- `rounds[].statements` 配列がある（人物名フィールドではない）
- 各ラウンドの口火を切る人物が前ラウンドと異なり、全メンバーが各ラウンドで最低1回発言し、サイクル内に最低1ラウンド同一人物の二度発言がある
- `conclusion` に `probability` がなく `stance` / `outlook`（bull > base > bear）がある
- `conclusion.memberPositions` に全委員の立場があり、`adopted:false` の委員には `overrideReason` がある
- guest があれば sources が実在URL、statements の中間位置、直後に委員の反応がある
- FAIL が0件であること。WARN は内容を読んで妥当性を判断する

## 7. 同期ループを回す

**クラウド側から Artifact へは書き込めない**（承認待ちで固まる。検証済み）。**ページ側から GitHub も読めない**（CSP。jsDelivr は `/npm/` のみで `/gh/` は不可）。したがって同期はインタラクティブセッションからしかできない。

`ScheduleWakeup` で1時間ごとに自分を起こし、以下を行うループを組む:

1. リポジトリを `git pull`（`dangerouslyDisableSandbox: true`）
2. **フィードを更新する**（`data/feed/latest.json`）。株価テーマなら `data/state.json` の `priceFeed` もあわせて更新する（下の「株価の取り方」参照）。フィードには対象の終値・出来高・値幅、マクロ指標、直近のニュースと適時開示を書く。**株価取得は `WebFetch` を使うこと。`curl` は使えない**（下記参照）。
3. 未同期のサイクルがあれば、`tools_stamp_cycle.py N` で `committedAt`（gitのコミット時刻）を注入したコピーを作ってから `Artifact` の `write_db` に batch で書き込む:
   - `cycles/cycle-N` ← `tools_stamp_cycle.py` が出力した `sync-cycle-N.json` を `file_path` で指定
   - `state/meta` ← state.json の deskName/theme/mode/subject/candidates/premise/scope/entryPrice/decisionHorizon/stances/outlookLabels/targetRoundsPerCycle/targetCycles/currentCycle/cadence/updatedAt/team
   - `meta/audit` ← `data/meta/audit.json` を `{"entries": [...]}` に包んだファイルを `file_path` で指定
4. `tools_validate_cycle.py N` で pull 後のサイクルを検証する。FAIL があれば同期を止めて報告する。
5. 次の `ScheduleWakeup`（3600秒）を入れる。`currentCycle > targetCycles` かつ全同期済みなら `stop: true`

報告は静かに。stance が変わった／base が前サイクル比±5%以上動いた／委員会が rules.md に自己修正ルールを追記した／検証で問題が出た／エラーで止まった、のいずれかのときだけ数行で報告する。

### 株価の取り方（検証済み）

**curl は使えない。`WebFetch` ツールを使う。**

| 方法 | 結果 |
|---|---|
| `stooq.com/q/l/?s=...&e=csv` | **404**。`aapl.us` でも404なのでエンドポイント自体が死んでいる |
| `query1/query2.finance.yahoo.com` を curl | **429**。UA偽装・cookie+crumb フローを通しても429。共有IPのレート制限で回避不能 |
| `api.jquants.com` | 403（要登録） |
| Alpha Vantage / Twelve Data | 要APIキー。demo キーは拒否される |
| **`WebFetch` で `google.com/finance/quote/<code>:TYO`** | **成功**。株価＋JSTタイムスタンプが返る |
| **`WebFetch` で Yahoo の chart API URL** | **成功**。ただし小型モデルが要約するので数値がぶれることがある |

Google Finance を第一候補にし、設置者が値を伝えてきた場合はそちらを優先する（実測で両者の差は0.4%以内だった）。取得したら `data/state.json` の `priceFeed` を書き換えて push する:

```json
"priceFeed": {
  "asOf": "YYYY-MM-DDThh:mm+09:00",
  "source": "Google Finance (<code>:TYO) をインタラクティブセッションから取得",
  "updatedBy": "auto-sync",
  "prices": {"<key>": NNNN},
  "note": "委員会自身の調査結果より常に優先すること"
}
```

クラウド側（ルーティン）は egress proxy で株価サイトに届かないので、**クラウドに株価を取らせようとしないこと**。rules.md に「`priceFeed` を最優先で使う」ルール（R8）を置けば、委員会は自分で調べた古い値を捨ててこちらに従う（実測で検証済み）。あわせて「基準値の3%未満の変動は値動きであって新情報ではない」も入れてあるので、毎サイクル株価の上下を議論し始めることは防げる。

## state.json の埋め方

| フィールド | 中身 |
|---|---|
| `deskName` | ページ上部の小さなラベル。例「MEC Coverage Desk — 小規模投資会社 6人合議制」 |
| `theme` | 設問を1文で。**意思決定の形にする**。「Xは年内に◯円になるか」のような点予測にしないこと |
| `mode` | `"comparison"`（複数候補の相対比較）または `"single"`（単独テーマの可否）。`_modeNote` は削除してよい |
| `subject` | 対象（code / name / note）。単独テーマならここが主。比較検討でも代表表記として埋める |
| `candidates` | 比較検討モードの候補一覧（key / code / name / note）。単独テーマなら空配列 `[]` にする |
| `premise` | 設置者の仮説。「未検証の作業前提であり、妥当性自体も検証対象」と明記する |
| `scope.focus` | 議論すべき論点の列挙 |
| `scope.exclude` | **設置者が一般論の例として挙げただけの事柄**。ここに入れないと委員会が検証テーマに格上げしてしまう |
| `entryPrice` / `decisionHorizon` | ポジションの取得単価と判断期限。該当しないテーマなら null |
| `stances` | 結論の選択肢。投資以外なら「採用/保留/却下」等に差し替える |
| `outlookLabels` | 3シナリオの呼び名と単位 |
| `targetRoundsPerCycle` / `targetCycles` / `cadence` | 既定値は 6 / 15 / 4時間ごと。変更してもよいが rules.md R15 の前提（反論権で発言数が変動する）と整合させること |
| `team` | `.claude/agents/*.md` から `tools_sync_team.py` で再生成する。**直接編集しない** |
| `holdings` | 設置者の保有状況の説明。該当しなければ「該当なし」等でよい |
| `priceFeed` | 同期ループが書き込む。初期状態は値がすべて null／空でよい |

## 設計上の勘所

- **問いの形**: 点予測は当てられない。「何をすべきか」＋「3シナリオとその根拠」の形にすると、外れようのない予測ごっこにならず判断材料になる。
- **堂々巡りの防止**: 少人数固定だと必ず「一理ありますね」で収束する。rules.md の外部招聘（R1）と停滞判定（R2）がこれを壊す仕掛け。
- **自己修正の器**: 運営ルールをプロンプトではなくリポジトリの rules.md に置くことで、委員会自身が自己監査（R5）で問題を見つけてルールを追記できる。設置者が毎回指摘しなくても軌道修正が起きる。
- **変更してよい領域の線引き**: theme / premise / scope / subject / candidates / entryPrice / decisionHorizon / targetCycles / team / stances / priceFeed は設置者の領域。委員会は触らない。
- **異なる人格に同じ資料を読ませても echo chamber は壊れない。異なる資料を読ませることが本質である。** 委員を独立サブエージェント化し（rules.md R9）、各自が自分の担当領域を自分で調査する設計（担当を重複させない。R21）にして初めて、議論の質が単一セッション方式より明確に上がった。
- **報酬ではなく制約と情報が効く。** 「最も新事実を持ち込んだ委員を議長にする」という報酬設計は検討の上で却下した（rules.md R20）。委員はサイクルごとに新しく起動されるエージェントであり、継続的な動機を持たない。「多く出せば報われる」を指示に書くと、モデルは発見の申告を水増しし始める（Goodhartの法則）。効くのは制約（同意だけの発言禁止など）と情報（lockedFacts・priceFeed）である。
- **もっともらしい古い値は、明らかな異常値より危険である。** 検索結果が古いスナップショットを現在値の体裁で返してくることがあり、これは何度注意しても再発した。「人に注意させる」方式には限界があるため、進行役による機械検査（rules.md R14）が必要になった。
- **委員に rules.md を読ませない。** 全文（数十KB）を委員エージェントに読ませるとトークンを浪費し、レート制限に当たる（実際に1サイクル分の議論が丸ごと失われた事故があった）。委員が知る必要のある要点だけをプロンプトで渡すこと（詳細は `ROUTINE_PROMPT.md` 参照）。
- **エージェントを非同期起動しない。** バックグラウンドで委員や議長を起動して「完了を待ちます」とターンを終えると、セッションが待機状態のまま議論の記録が失われる事故が起きた。1回の起動の中で保存・commit・push まで必ず到達すること。

## 検証済みのプラットフォーム制約

- クラウド側（RemoteTrigger のルーティン実行）から Artifact へは書き込めない（承認待ちで固まる）。
- ページ（Artifact）側から GitHub リポジトリの内容は読めない（CSP。jsDelivr は `/npm/` のみで `/gh/` は不可）。
- クラウド側は多くの株価情報サイトに egress proxy で届かない（`curl` は 404/429 になる、AlphaVantage 等は要APIキー）。`WebFetch` で Google Finance / Yahoo chart API は成功する。
- したがって、GitHubリポジトリとArtifactページの同期、および株価取得は、いずれもインタラクティブセッション（同期ループ）からしか行えない。クラウド側のルーティンにこれらをやらせようとしないこと。

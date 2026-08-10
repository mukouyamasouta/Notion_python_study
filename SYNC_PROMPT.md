# Notion 同期ルーティン（毎週金曜 19:00 JST）

このリポジトリの `index.html` / `data.js` は、Notion の
**「🐍 Python知識ベース」データベース**（`https://app.notion.com/p/36fe3be8d5ac80dda919cd12d3f97555`）
の内容から生成された演習問題アプリです。このファイルは、毎週自動実行されるスケジュール済みタスクが
差分を検知して `data.js` を更新するための手順書です。人間が読んでも実行できるし、
Claude（cloud agent）が読んでそのまま実行することも想定しています。

## 実行タイミング

毎週 **金曜 19:00 (Asia/Tokyo)**。`schedule` の cron ルーティンとして登録済み。

## 目的

Notion 側で知識ベースのページが追加・編集されたら、それを検知して
`data.js` の `problems` / `tree` / `syncMeta` を更新し、GitHub にコミット & push する。

## 手順

1. **ソースを列挙する**
   - Notion MCP (`notion-fetch`) で `https://app.notion.com/p/36fe3be8d5ac80dda919cd12d3f97555`
     （データベース本体）を取得し、直下の行（ページ）一覧を得る。
   - `問題集`・`pythonデータ分析入門` のようにページの中にインラインデータベースが
     ネストされている場合は、その中のページも再帰的に列挙する
     （`notion-fetch` の結果に含まれる `<database ... data-source-url="collection://...">`
     をたどって `notion-query-data-sources` で子ページを取得する）。
   - 空ページ（`<blank-page>` と返るもの。例: 算術代入・標準入力・標準出力）は無視してよい。

2. **差分を検知する**
   - `data.js` の `DB.syncMeta.pages` に、これまで把握している全ページの
     `{url, title, path, lastSeenEditedAt, problemIds}` が入っている。
   - 列挙したページと `syncMeta.pages` を突き合わせ、
     - **新規ページ**（`syncMeta.pages` に存在しない url）
     - **更新されたページ**（Notion 側の最終更新日時が `lastSeenEditedAt`
       より新しい。`lastSeenEditedAt` が `null` の場合は `syncMeta.baselineSyncedAt`
       より新しければ更新とみなす）
     を洗い出す。
   - 最終更新日時は `notion-search` の検索結果に含まれる `timestamp` フィールド
     （該当ページをタイトルで検索して拾う）か、ページ本文の実質的な内容差分で判定する。

3. **変更があったページごとに問題を作り直す**
   - 該当ページを `notion-fetch` でフル取得し、内容を読む。
   - 既存の `data.js` 内の同ページ由来の問題（`syncMeta.pages[].problemIds` で特定）
     の書き方・難易度構成を参考に、**同じスタイル**で問題を作る:
     - 1つの分野（ページ）につき **原則2〜3段階の難易度**（★1〜★3）を用意する。
     - ★1: 「以下の手順に従って…」という番号付きの手順（`<ol class='steps'><li>…</li></ol>`）で、
       ゼロから書かせる形式。
     - ★2: 右のコードエリアに前提コード（`starter`）が用意されていて、
       それに続きを書き足す／一部を書き換える形式。
     - ★3: paiza のスキルチェックに近い、入出力仕様だけを示す実践形式
       （必要なら `stdin` / `stdinLabel` を設定し、標準入力を使わせる）。
     - 各問題は `title, status:"todo", difficulty, time, tags, path, notionUrl,
       statement(HTML文字列), expected, starter, hints[], solution, solutionNote`
       のフィールドを持つ（既存の `data.js` の他のエントリと同じ形）。
   - **`expected` は必ず実際に Python を実行して得た標準出力を使うこと。**
     手計算や記憶で埋めない。ローカルに `python3` があるので、
     `solution` のコードを `python3 -c "..."` や一時ファイルに書いて実行し、
     標準出力をそのまま `expected` に転記する（末尾の改行は取り除く）。
     `pandas` を使う問題は `pip install --break-system-packages pandas`
     （未インストールの場合）してから検証する。`pkgs: ["pandas"]` を問題に付与する。
   - `stdin` を使う問題は、実行時に `input=` でその文字列を渡して検証する。
   - 新規ページの場合は `data.js` の `tree` にも対応するフォルダ／リーフを追加する
     （既存の階層構造・命名パターンに合わせる）。

4. **`data.js` を更新する**
   - `problems` に新規・更新エントリを反映。
   - `tree` に新規フォルダ／問題を反映。
   - `syncMeta.pages` の該当エントリの `lastSeenEditedAt` を今回検知した最終更新日時に、
     `problemIds` を実際に紐づく問題IDの配列に更新。新規ページなら配列ごと追加。
   - `lastSync` を実行時刻（ISO8601, `+09:00`）に更新。
   - ファイル冒頭のコメント（自動生成である旨）は変更しない。
   - `data.js` は「`var DB = { ... };`」という1つの JS 文で構成されていること
     （実データを差し替えても、この形式を崩さないこと）。

5. **検証する**
   - 生成した `data.js` が構文的に正しい JS であることを確認する
     （例: `node -e "new Function(require('fs').readFileSync('data.js','utf8')+'; return DB;')()"`）。
   - 可能であれば `python3 -m http.server` などでローカルに配信し、
     ブラウザ（Claude Browser ツール）で対象の問題を開いて実際に「採点する」を押し、
     期待値と一致することを確認する。

6. **コミット & push する**
   - 変更が実際にある場合のみコミットする（差分ゼロなら何もしない）。
   - コミットメッセージ例: `Sync: Notion更新を反映（YYYY-MM-DD, N件更新）`
   - `main` ブランチへ直接 push する（このリポジトリは個人の学習用リポジトリのため、
     PR は不要）。
   - 末尾に `Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>` を付ける。

## 注意事項

- Notion 側の該当データベースへのアクセス権限（MCP 接続）が失われている場合は、
  無理に処理を続けず、何もコミットせずに終了してよい。
- 1回の実行で処理しきれないほど大量の変更があった場合は、無理に全部やろうとせず、
  明らかに新しい／大きく変わったページを優先して処理し、残りは次回に回してよい。
- 生成する問題文は、この演習アプリの既存のトーン（親しみやすい日本語、
  具体的な手順、実際に実行して検証済みの出力）を踏襲すること。

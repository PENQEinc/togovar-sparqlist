# AGENTS.md

このファイルは、この `togovar-sparqlist` リポジトリでAIエージェントが作業するときの指示です。人間向けのセットアップ、クエリ一覧、fork/upstream運用は `README.md` に書いてください。

## リポジトリの位置づけ（fork / upstream）

- このリポジトリ（`PENQEinc/togovar-sparqlist`）は [`togovar/sparqlist`](https://github.com/togovar/sparqlist) のforkです。開発はこのforkで行い、`upstream/develop` へPRを出す運用です。
- `README.md` / `AGENTS.md` / `CLAUDE.md` はfork固有のメタドキュメントです。特に指示がない限り、upstream向けPRのブランチにはこれらの変更を含めないでください（含めるかどうか迷う場合はユーザーに確認する）。
- これら3ファイルはSPARQListの`REPOSITORY_PATH`（このリポジトリ直下）に置かれているため、sparqlist自身がこれらも`.md`定義ファイルとして解析しようとする。見出し（`#`/`##`/`###`）の中でインラインコードや強調などをプレーンテキストと混在させると、sparqlistのMarkdownパーサー（fontoxpath）が`lower-case(text())`でエラーになり、`/-api/sparqlets`（一覧API）が`Promise.all`ごと失敗して**他の全エンドポイントの一覧取得も巻き添えで壊れる**。これら3ファイルの見出しはインライン要素を混在させないプレーンテキストにする。

### ブランチモデル(詳細はREADME.mdの「関連リポジトリとの開発フロー」を参照)

- `fork-base`: `upstream/develop` + フォーク限定メタドキュメントの1コミットを常に維持する土台ブランチ。`origin`にはpushするが、upstreamには反映しない。
- 機能ブランチは`fork-base`から分岐し、自由にcommitしてよい(試行錯誤の跡が残っても問題ない)。
- 機能が完成したら`git rebase --onto upstream/develop fork-base <feature-branch>`で`fork-base`由来のコミットを外し、`upstream/develop`起点・1コミットに変換してからPRを出す。作業中のブランチをそのままPRの対象にしない。
- upstreamへのマージ後は機能ブランチを削除し、`fork-base`を最新の`upstream/develop`にrebaseする。
- `fork-base`・機能ブランチとも履歴を書き換えるため、pushは`--force`ではなく`--force-with-lease`を使う。
- この運用は「`fork-base`から分岐している機能ブランチが同時に1つ(か少数)」であることを前提にする。複数の機能ブランチを並行して進める場合は、`fork-base`をrebaseするたびに他の機能ブランチとの整合を確認する。

## 役割分担

- `AGENTS.md`: AIが実装時に守る設計方針、作業ルール、判断基準、間違えやすい仕様を書く。
- `README.md`: リポジトリ概要、クエリ一覧、fork/upstream運用手順を書く。
- 各定義ファイル（`*.md`）自体がそのクエリの仕様書を兼ねる（`## Parameters` など）。stanza集のような個別READMEディレクトリは存在しない。

## 作業ルール

- 変更前に必ず対象の定義ファイル（`*.md`）全体を読む。他の定義ファイルの類似クエリがあれば、命名・構成・エラーハンドリングの慣習も確認する。`AGENTS.md` だけを根拠にしない。
- ユーザーの未コミット変更を勝手に戻さない。未追跡ファイルや別作業の差分は、今回の依頼に必要なものだけ触る。
- 既存の定義ファイルの構成（見出し順、パラメータ記法、ステップ命名）に合わせる。新しい書き方を持ち込まない。
- SPARQLクエリやJavaScriptステップで受け取るパラメータをそのまま別のクエリ文字列やURIに埋め込む変更をする場合は、injectionのリスクを必ず確認する（詳細は下記「実装方針」）。

## 技術スタック

| レイヤー         | 技術                                                    |
| ---------------- | ------------------------------------------------------- |
| 実行基盤         | [SPARQList](https://github.com/dbcls/sparqlist)（別リポジトリ、本体はここには含まれない） |
| 定義ファイル形式 | Markdown（`*.md`、1ファイル = 1 APIエンドポイント）      |
| テンプレート     | Handlebars（SPARQL/JS内の `{{param}}`, `{{#each}}` 等）  |
| クエリ本体       | SPARQL（一部Virtuoso拡張構文を使用、例: `DEFINE sql:select-option "order"`） |
| ステップ処理     | JavaScript（Node、`async`/`await` と `fetch` が中心）    |

- このリポジトリ自体には `package.json` やビルド設定はない。ビルド・バンドル・型チェックの概念がないフラットなMarkdown集。
- 実行にはSPARQList本体を別途用意し、このディレクトリを定義ディレクトリとして読み込ませる必要がある。

## ディレクトリ方針

```txt
.
  <name>.md   SPARQList定義ファイル（1ファイル = 1エンドポイント）
```

- 全定義ファイルはリポジトリ直下にフラットに置かれている。サブディレクトリへの分割は行わない（既存の慣習に合わせる）。
- ファイル名はスネークケース（例: `gene_summary.md`, `variant_clinvar.md`）。エンドポイントのパスはファイル名に対応する。
- 新規クエリを追加したら `README.md` の「クエリ一覧」も更新する。

## SPARQList定義ファイルの構成

各定義ファイルは概ね次の構成に従う。既存ファイル（`gene_summary.md`, `variant_summary.md` など）を確認してから合わせること。

1. `# タイトル`
2. `## Parameters`: 箇条書きでパラメータ名・説明。`example:` や `default:` のサブ項目を持つことが多い。
3. `## Endpoint`: 問い合わせ先。多くの場合 `{{SPARQLIST_TOGOVAR_SPARQL}}` のような環境変数風のHandlebars変数。
4. 1つ以上の名前付きステップ（`## \`ステップ名\`` の見出し + ` ```javascript ``` ` または ` ```sparql ``` ` コードブロック）。あるステップの戻り値は、同名のパラメータとして後続ステップに渡される。
5. 最終ステップは慣習的に `result` という名前になっていることが多い。

ステップの名前を変更する場合は、後続ステップの引数名も必ず合わせて変更する。

### 内部ヘルパーエンドポイント

- `resolve_variant.md` / `tgv2variant.md` / `variant2tgv.md` は、フロントエンドやstanzaから直接叩かれるのではなく、他の`.md`ファイルから`SPARQLIST_TOGOVAR_SPARQLIST`経由で自己参照的に呼び出される内部ヘルパーエンドポイント。
- 多くの`variant_*.md`（`variant_summary.md`, `variant_clinvar.md`, `variant_transcript.md`, `variant_gene.md`, `variant_gwas.md`, `variant_other_alternative_alleles.md` など）の`variant`ステップは共通で次の形をとり、`tgv_id`と`variant`（VCF表記）のどちらが渡されても`/api/resolve_variant`経由で正規化された変数IRIに解決してから後続のSPARQLへ渡す。

  ```javascript
  async ({SPARQLIST_TOGOVAR_SPARQLIST, variant, tgv_id}) => {
    let params;
    if (tgv_id.length > 0) {
      params = `tgv_id=${encodeURIComponent(tgv_id)}`;
    } else if (variant.length > 0) {
      params = `variant=${encodeURIComponent(variant)}`;
    } else {
      throw new Error("Either tgv_id or variant must be provided.");
    }

    const res = await fetch(SPARQLIST_TOGOVAR_SPARQLIST.concat(`/api/resolve_variant?${params}`));

    if (!res.ok) {
      throw new Error((await res.text()).replace(/^Error: /, ""));
    }

    return await res.text();
  }
  ```

  新しい`variant_*.md`を追加する場合はこのパターンをそのまま踏襲する。
- `resolve_variant.md`は`tgv_id`が渡されたときだけ内部で`/api/tgv2variant`（`tgv2variant.md`）を呼ぶ。この二段構成のヘルパーを介した呼び出し関係のため、`SPARQLIST_TOGOVAR_SPARQLIST`（このsparqlist自身のベースURL）が正しく設定されていないと`variant_*.md`系のエンドポイントは軒並み失敗する。

## 実装方針・間違えやすい仕様

- **SPARQLへのパラメータ埋め込みはHandlebarsによる文字列展開であり、自動エスケープされない。** `<http://identifiers.org/hgnc/{{hgnc_id}}>` のようにURIへ直接埋め込む場合や、`VALUES ?x { "{{param}}" }` のように埋め込む場合、パラメータが検証されていないとSPARQL injectionにつながる。既存の定義ファイル（`gene_genomic_context.md`, `disease_header.md`, `gene_publication.md` など）では、SPARQLに渡す前のJavaScriptステップで `hgnc_id.match(/^\d+$/)` や `medgen_cid.match(/^CN?\d+$/)` のように形式を正規表現で検証している。新しいパラメータをSPARQLへ渡す場合は同様の検証ステップを設ける。
- 外部REST APIを呼ぶ場合は `fetch` + `encodeURIComponent` でクエリ文字列を組み立てる既存パターンに合わせる。
- エラー時の戻り値の形は、そのステップの用途によって既存で2パターンが混在している。
  - テンプレート等での表示に使われる末端ステップ: `{error: "メッセージ"}` のようなオブジェクトを返す（例: `variant_mogplus.md`）。
  - 後続ステップへの入力になる中間ステップ: `try/catch` で `console.log(error)` した上で `return null` とし、後続で空扱いできるようにする（例: `gene_publication.md`, `variant_publication.md`）。
  - どちらに合わせるかは、編集対象のステップが「表示用の末端」か「後続へ渡す中間ステップ」かで判断し、既存の同種ステップの書き方を踏襲する。
- `SPARQLIST_TOGOVAR_APP`（TogoVar検索API/フロントエンドのベースURL）、`SPARQLIST_TOGOVAR_SPARQL`（SPARQLエンドポイントURL）、`SPARQLIST_TOGOVAR_SPARQLIST`（このsparqlist自身のベースURL、内部的に自分のAPIを叩く場合に使用）という環境変数風のHandlebars変数が使われている。URLをハードコードせず、対象ファイルが既に使っている変数に合わせる。
- 大量のID（PMIDなど）を扱うステップでは、既存の `variant_publication.md` / `gene_publication.md` のように一定件数ごとに分割してリクエストするバッチ処理パターンがある。件数上限を変える場合は理由を明確にする。
- SPARQLエンドポイントはRDFストア（Virtuoso）。バリアント関連の一部クエリがSPARQLではなくTogoVar検索API（Elasticsearchベース、`SPARQLIST_TOGOVAR_APP` 経由）を使っているのは、バリアントデータ量が多くRDFストアでは検索性能が出ないための例外的な歴史的経緯。この理由を理解せずに他のクエリもTogoVar APIへ統一する変更は提案・実施しない。
- 同じエンドポイントの同じ出力項目について、RDFストアとTogoVar検索APIの両方からデータを取得して補完する実装は行わない。特にVEP（JoGo, JSV1）などRDF由来であるべきアノテーションが欠けている場合、長いREF/ALTのバリアントなど一部データだけRDFが未ロードである可能性をまず疑い、検索APIで穴埋めしない。RDFに追加ロードされる見込みがある場合は実装を保留し、RDF側にデータが載った後にSPARQLだけで取得する形にする。
- SPARQListのJavaScriptステップは`vm.runInNewContext`で毎回独立したサンドボックス実行されるため、`require`/`import`もファイル間の共有もできない。`clinical_significance_key` / `review_status_stars` のような変換ロジックは `variant_clinvar.md` / `gene_clinvar.md` / `disease_clinvar.md` に重複して存在する。一方を更新したら、他方も同じ分類体系にすべきか確認する（自動では同期されない）。`variant_clinvar.md`は2026-07-10時点で最新化済みだが、`gene_clinvar.md` / `disease_clinvar.md`側は未確認のため、同じ用語ドリフトの影響を受けていないか確認が必要。
- ClinVarは分類用語を`assertion`→`classification`のように改称することがあり、改称前後どちらの表記も実データに混在する（例: `conflicting interpretations of pathogenicity` / `conflicting classifications of pathogenicity`、`no assertion for the individual variant` / `no classification provided`）。`interpretation`・`review_status`のどちらも対象になるため、文字列を`switch`で分岐する場合は新旧両方の表記を実データで確認してから書く（RDFストアに実際に存在する値を`DISTINCT`で確認するのが確実）。`review_status`→星の数の対応はClinVar公式（https://www.ncbi.nlm.nih.gov/clinvar/docs/review_status/）を正とする。
- ClinVarの`interpretation`（例: `"Uncertain significance"`）はVUSの中の細かい区分（TogoVarの`VH`/`VM`/`VL` = VUS-high/mid/low）を判定する材料を持たない。RDFストア側には`cvo:submission_count`（そのRCV分類への提出件数）という未使用の値があり、確信度の判定に使われている可能性はあるが、実際の判定ルールは未確認（2026-07-10時点）。判定ルールが分かるまで、VUSは`US`のまま扱い、`VH`/`VM`/`VL`を推測で実装しない。
- **CADD PHREDスコア（`cadd_phred`）は2026-07-24時点でデータソースが存在しない。** RDFストア（`variant/annotation/ensembl`グラフ）に`cadd`/`cadd_phred`系の述語は無く（`ASK`で全件確認済み）、本番のTogoVar検索API（`SPARQLIST_TOGOVAR_APP`、`https://grch38.togovar.org`）の`/api/search/variant`もレスポンス・クエリ条件のどちらにも`cadd_phred`を持たない（`400 Bad Request`になる）。一方、別リポジトリ`nbdc-forked-togovar`（`staging`ブランチ）の`scripts/GRCh38/openapi.yaml`には`CADDPhred`スキーマが、`app/frontend/src/types/api.d.ts`には`cadd_phred?: number`が定義されており、`togovar-stanza`のビルド済み`variant-transcript`スタンザにはCADD列の表示マークアップも残っているが、現行の`index.ts`にはCADDを取得・変換するロジックは実装されていない。つまりCADDはフロントエンド側で「表示予定だが未接続」の状態で、バックエンドにまだデータが無い。`variant_transcript.md`などにCADD取得を追加する依頼が来たら、まず本番のSPARQL/検索APIに実際にデータが載っているか確認し、無ければ実装を保留してユーザーに確認する（フィールド名だけ仮決めして空データのまま実装しない）。

## コメント規約

- 定義ファイル（`*.md`内のSPARQL/JavaScript）にはコメントを一切書かない。英語であってもupstream（`togovar/sparqlist`）へのPRでコメントは歓迎されないため。「なぜその形にしているか」を残したい場合はコミットメッセージやPR説明に書く。
- `README.md` / `AGENTS.md` / `CLAUDE.md` などfork限定のメタドキュメントは社内（PENQE）向けであり、この制約の対象外。従来通り日本語で、必要な理由の説明を書いてよい。

## 検証

このリポジトリには自動テスト・lint・ビルドの仕組みがない。変更後は以下の観点で手動確認する。

- SPARQLブロックの構文: 対象の `## Endpoint` に書かれたエンドポイントへ、変更したクエリをそのまま投げて結果を確認する（SPARQLエンドポイントのクエリUIなど）。
- JavaScriptステップの構文・ロジック: lintがないため、変更差分を注意深く読み、既存の類似ステップと書き方が矛盾していないか確認する。
- 可能であればSPARQList本体をローカルで起動し、このディレクトリを定義ディレクトリとして読み込ませて実際のAPIレスポンスを確認する（手順は `README.md` の「ローカルでの動作確認」を参照）。動作確認済みの例:

  ```bash
  # sparqlist本体のディレクトリで
  REPOSITORY_PATH=/path/to/togovar-sparqlist \
  PORT=3000 ADMIN_PASSWORD=changeme \
  SPARQLIST_TOGOVAR_SPARQL=https://grch38.togovar.org/sparql \
  npm start

  # 別ターミナルで
  curl "http://localhost:3000/api/gene_summary?hgnc_id=404"
  ```

- `## Endpoint` が `{{SPARQLIST_TOGOVAR_SPARQL}}` のような変数で、対応する環境変数を渡さずに叩くと `Failed to parse URL from` のようなエラーになる。エラーが出た場合はまず対象クエリが要求する環境変数（`SPARQLIST_TOGOVAR_APP` / `SPARQLIST_TOGOVAR_SPARQL` / `SPARQLIST_TOGOVAR_SPARQLIST`）が渡っているか確認する。
- ネットワークアクセスや外部APIへの疎通確認ができない環境の場合は、その旨を報告する。

## ドキュメント更新の目安

- 新しい定義ファイルを追加したら、`README.md` の「クエリ一覧」を更新する。
- パラメータや出力の形を変えた場合、TogoVarフロントエンドやtogovar-stanzaなど呼び出し側に影響する可能性があるため、変更内容をPR説明に明記する（呼び出し側リポジトリはこのリポジトリの範囲外なので直接は確認できない）。
- 環境変数（`SPARQLIST_TOGOVAR_*`）の使い方やエラー時の戻り値の形など、複数の定義ファイルにまたがる慣習を変える場合は、この `AGENTS.md` も更新する。

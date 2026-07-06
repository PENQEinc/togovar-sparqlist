# togovar-sparqlist

TogoVarで利用する [SPARQList](https://github.com/dbcls/sparqlist) 用クエリ定義集です。各 `.md` ファイルが1つのSPARQList APIエンドポイントに対応します。

公開環境:

- https://grch38.togovar.org/sparqlist

## このリポジトリについて

このリポジトリ（`PENQEinc/togovar-sparqlist`）は [`togovar/sparqlist`](https://github.com/togovar/sparqlist) のforkです。開発はこのforkで行い、upstream（`togovar/sparqlist`）の `develop` へPRを出す運用です。

このREADMEはforkでの開発を進めやすくするためのものです。

## セットアップ

このリポジトリ自体はSPARQList定義（Markdown）の集合であり、ビルドは不要です。ローカルで動作確認する場合は [SPARQList](https://github.com/dbcls/sparqlist) 本体を別途用意し、このリポジトリを定義ディレクトリとして読み込ませます。

### ローカルでの動作確認

1. このリポジトリとは別の場所に `dbcls/sparqlist` をcloneし、依存関係をインストール・ビルドする。

   ```bash
   git clone https://github.com/dbcls/sparqlist.git
   cd sparqlist
   npm install
   npm run build
   ```

2. このリポジトリ（`togovar-sparqlist`）を `REPOSITORY_PATH` に指定して起動する。TogoVarのSPARQLエンドポイントも環境変数で渡す。

   ```bash
   REPOSITORY_PATH=/path/to/togovar-sparqlist \
   PORT=3000 ADMIN_PASSWORD=changeme \
   SPARQLIST_TOGOVAR_SPARQL=https://grch38.togovar.org/sparql \
   npm start
   ```

3. 別ターミナルからエンドポイントを叩いて確認する（`<ファイル名>` は拡張子なし）。

   ```bash
   curl "http://localhost:3000/api/gene_summary?hgnc_id=404"
   ```

- `SPARQLIST_TOGOVAR_SPARQL` はTogoVarのSPARQLエンドポイント（本番: `https://grch38.togovar.org/sparql`）。SPARQLのみを使うクエリ（`gene_summary.md` など）はこれだけで動作確認できる。
- `SPARQLIST_TOGOVAR_APP`（TogoVar検索API/フロントエンドのベースURL）や `SPARQLIST_TOGOVAR_SPARQLIST`（このsparqlist自身のベースURL、自己参照的な呼び出しに使用）を使うクエリを試す場合は、対象環境（本番/staging）に応じた値を別途チームに確認する。ローカルでの自己参照確認だけなら `SPARQLIST_TOGOVAR_SPARQLIST=http://localhost:3000` で代用できる場合がある。

## ディレクトリ構成

```txt
*.md   SPARQList定義ファイル（1ファイル = 1エンドポイント）
```

各定義ファイルは概ね以下の構成です。

- タイトル（`# `見出し）
- `## Parameters`: 受け取るクエリパラメータ
- `## Endpoint`: 問い合わせ先のSPARQLエンドポイントなど
- コンパイル処理（JavaScript）や `## sparql` などのクエリ本体

## クエリ一覧

現在の定義ファイル:

- `colil.md`: Get citation count from Colil
- `disease_clinvar.md`: Disease report / Clinvar
- `disease_gwas.md`: Disease report / GWAS
- `disease_header.md`: Disease report / Header
- `disease_summary.md`: Disease report / Summary
- `gene_clinvar.md`: Gene report / Clinvar
- `gene_genomic_context.md`: Gene report / Genomic context
- `gene_gwas.md`: Gene report / GWAS
- `gene_header.md`: Gene report / Header
- `gene_mogplus.md`: Convert human variants to mouse genome positions per gene
- `gene_pdb_mapping.md`: Structure data for TogoVar gene page
- `gene_protein_browser.md`: Gene / Protein browser
- `gene_publication.md`: Gene report / Publication
- `gene_summary.md`: Gene report / Summary
- `gene_variant.md`: Gene report / Variant
- `jbrowse_gene_track.md`: jBrowse Gene Track
- `mouse_strain.md`: Mouse strain ID to mouse strain attributes
- `mouse2human.md`: Convert human variants to mouse genome positions from mouse genome range
- `resolve_variant.md`: Variant IRI from tgv ID or VCF notation
- `rs2tgvid.md`: Disease report / rs to TogoVarID for Disease-GWAS
- `tgv2rs.md`: TogoVar ID to dbSNP ID
- `tgv2variant.md`: TogoVar ID to Variant
- `variant_clinvar.md`: Variant report / ClinVar
- `variant_gene.md`: Variant report / Gene
- `variant_gwas.md`: Variant report / GWAS
- `variant_mogplus.md`: Search the counterpart variant between human and mouse
- `variant_other_alternative_alleles.md`: Variant report / Other alternative alleles
- `variant_publication.md`: TogoVar variant_publication stanza query
- `variant_summary.md`: Variant report / Summary
- `variant_transcript.md`: Variant report / Transcript
- `variant2tgv.md`: Convert VCF representation to TogoVar ID

## 関連リポジトリとの開発フロー

このリポジトリはTogoVarフロントエンド（[`togovar/togovar`](https://github.com/togovar/togovar)）やstanza集（[`togovar-stanza`](https://github.com/togovar/togovar-stanza)）から呼び出されます。それぞれ別リポジトリのため、変更が必要な場合は以下のフローに従います。

### ブランチモデル

前提として `upstream` に `togovar/sparqlist` を設定しておく（未設定の場合は `git remote add upstream https://github.com/togovar/sparqlist.git`）。

- **`fork-base`**: `upstream/develop` に、フォーク限定のメタドキュメント（`README.md` / `AGENTS.md` / `CLAUDE.md`）を足した1コミットだけを乗せた土台ブランチ。常に「`upstream/develop` + 1コミット」の状態を保つ。`origin` にはpushするが、**upstreamには絶対に反映しない**。
- **機能ブランチ**: `fork-base` から分岐して開発する。試行錯誤や細かいcommitを自由に積んでよい。

### 開発サイクル

1. `fork-base` から機能ブランチを作成し、開発する。

   ```bash
   git checkout -b feature/xxx fork-base
   ```

2. 機能が完成したら、`upstream/develop` を最新化し、機能ブランチを「`fork-base` 由来のコミットを除いて `upstream/develop` の上に1コミットとして乗せ直す」形にする。

   ```bash
   git fetch upstream
   git rebase --onto upstream/develop fork-base feature/xxx   # fork-base分の差分を落として develop 起点にする
   git reset --soft upstream/develop                          # 残ったコミットを1つにまとめる準備
   git commit -m "..."                                        # 1コミットとして作り直す
   ```

   `fork-base` が機能ブランチの直接の祖先でない場合（`fork-base` を後から更新した場合など）は、`fork-base` の代わりに `git merge-base fork-base feature/xxx` で得られる分岐点のコミットを指定する。

3. その1コミットの機能ブランチを `origin` にpushし、`PENQEinc/togovar-sparqlist` から `togovar/sparqlist` の `develop` へPRを出す（Assigneesにバックエンド担当を指定すると通知が届く）。
4. バックエンド担当がmergeとstagingへのdeployを行う。
5. マージ後、機能ブランチをローカル・`origin`双方から削除し、`fork-base` を最新の `upstream/develop` にrebaseして次のサイクルに備える。

   ```bash
   git checkout fork-base
   git fetch upstream
   git rebase upstream/develop
   git push --force-with-lease origin fork-base
   git branch -D feature/xxx
   git push origin --delete feature/xxx
   ```

upstream向けPRを1コミットにまとめることで、レビューしやすくし、フォーク側だけのcommit（README整備など試行錯誤の跡）が混ざらないようにします。`fork-base`・機能ブランチとも履歴を書き換えるため、pushには `--force`ではなく`--force-with-lease`を使う。この運用は「`fork-base`から分岐している機能ブランチが同時に1つ（か少数）」であることを前提にしている。複数の機能ブランチを並行して進める場合、`fork-base`をrebaseするたびに他の機能ブランチとの整合を確認すること。

## コントリビュータ向けメモ

- 定義ファイルの命名は既存のスネークケース（例: `gene_summary.md`, `variant_clinvar.md`）に合わせます。
- `SPARQLIST_TOGOVAR_APP` / `SPARQLIST_TOGOVAR_SPARQLIST` など環境変数経由の設定値は、既存の定義ファイルでの使い方に合わせます。
- 同じ出力項目をRDFストアとTogoVar検索APIの両方から取得して補完する実装は避けます。RDF由来のデータが欠けている場合は、検索APIで穴埋めせず、RDFの追加ロードやSPARQL側の取得方法を確認します。
- upstream（`togovar/sparqlist`）へ反映したくない変更（このREADMEなど、fork固有のドキュメント等）は、upstream向けPRのブランチに含めないようにします。

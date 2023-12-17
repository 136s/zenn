---
title: "もう NCBI GEO の多くの FASTQ を処理しなくてもいいかもしれない #souyakuAC2023"
emoji: "🧬"
type: "tech"
topics: ["rnaseq", "バイオインフォマティクス", "ncbi", "geo", "python"]
published: true
published_at: 2023-12-18 00:00
---


[創薬 (dry) Advent Calendar 2023](https://adventar.org/calendars/9092) Day 18 の記事です。 [#souyakuAC2023](https://twitter.com/hashtag/souyakuAC2023)
ケモインフォの記事がほとんどの中、バイオインフォ（遺伝子発現解析）周りの記事です。

本稿では [NCBI-generated RNA-seq count data](#ncbi-generated-rna-seq-count-data-とは) とそのダウンロード補助パッケージ [ncbi-counts](#ncbi-counts-の使い方) ([PyPI](https://pypi.org/project/ncbi-counts/)) を紹介します。

# GEO とは

[Gene Expression Omnibus (GEO)](https://www.ncbi.nlm.nih.gov/gds/) とは、[National Center for Biotechnology Information (NCBI)](https://www.ncbi.nlm.nih.gov/) が管理するゲノムデータリポジトリです。研究者が登録した遺伝子発現データなどを取得できます。本稿では NGS で計測したデータについて書きます。

例えば、SARS-CoV-2 の眼球への感染に関するある研究^[[Eriksen AZ, Møller R, Makovoz B, Uhl SA et al. SARS-CoV-2 infects human adult donor eyes and hESC-derived ocular epithelium. Cell Stem Cell 2021 Jul 1;28(7):1205-1220.e7.](https://doi.org/10.1016/j.stem.2021.04.028)] では、[眼オルガノイドの single-cell RNA-Seq データ (GSE165477)](https://www.ncbi.nlm.nih.gov/geo/query/acc.cgi?acc=GSE165477) と[角膜・角膜辺縁・強膜の bulk RNA-Seq データ (GSE164073)](https://www.ncbi.nlm.nih.gov/geo/query/acc.cgi?acc=GSE164073) が登録されています。
後者には以下の 18 サンプルが登録されていて、Title から想像するに、各組織で SARS-CoV-2 感染/非感染 3 サンプルずつの遺伝子発現を計測・登録していそうです。

![GSE164073 のサンプル一覧](/images/ncbi-gen-counts/GSE164073_samples.png)

GSM には、各サンプルの条件などが登録されています。

## 遺伝子発現量を取得する

各サンプルの遺伝子発現量を得るには、通常、2 つの方法があります。

1. データ登録者が計算したカウントデータを使う
2. [Sequence Read Archive (SRA)](https://www.ncbi.nlm.nih.gov/sra) の生のリードデータを自分で変換する

方法 1 として、ページ下部の **Supplementary file** にカウントデータが登録してある場合があります。
GSE164073 では [`GSE164073_Eye_count_matrix.csv.gz`](https://www.ncbi.nlm.nih.gov/geo/download/?acc=GSE164073&format=file&file=GSE164073%5FEye%5Fcount%5Fmatrix%2Ecsv%2Egz) が登録されているので、これらをダウンロードします。

![GSE164073 の Supplementary file](/images/ncbi-gen-counts/GSE164073_supp.png)

解凍して `GSE164073_Eye_count_matrix.csv` の中身を見ると、

![GSE164073_Eye_count_matrix.csv の抜粋](/images/ncbi-gen-counts/GSE164073_Eye_count_matrix.csv.png)

のように、column はサンプル名（18 列）、index は遺伝子シンボル（27,946 行）で、中身が対応する発現量になっています。

方法 2 として、**Supplementary file** の下の [SRA Run Selector](https://www.ncbi.nlm.nih.gov/Traces/study/?acc=PRJNA688734) リンクから SRA にアクセスします。
GSE164073 では 23.49 Gb 分ののリードデータ（FASTQ ファイル）が登録されているので、これをダウンロードします。

![PRJNA688734](/images/ncbi-gen-counts/PRJNA688734.png)

リードデータからカウントデータへの変換方法は多くの解説があるのでここでは説明しません。例えば↓

https://togotv.dbcls.jp/20231030.html

## カウントデータ利用のハードル

ただ、どちらにせよこれらの方法では、下流の解析にすぐ使えるとは限りません。例えば、このようなことが考えられます。

1. GEO への登録者がカウントデータをアップロードしていない場合がある。アップロードしてあっても、解析手法やリファレンスファイルやデータ形式が異なったりして、GSE を跨いでの比較可能性に注意が必要
2. リードデータからカウントデータへの変換が容易ではない（環境構築がめんどう、ファイルサイズが大きい、計算時間がかかる、など）

# NCBI-generated RNA-seq count data とは

これらのハードルを解消すべく、GEO に登録されているリードデータを、NCBI がカウントデータへ変換するという[取り組み](https://www.ncbi.nlm.nih.gov/geo/info/rnaseqcounts.html)が始まっていました。X.com でもあまり話題になってなさそうだった^[X 上での `https://www.ncbi.nlm.nih.gov/geo/info/rnaseqcounts.html` の[検索結果](https://x.com/search?q=https%3A%2F%2Fwww.ncbi.nlm.nih.gov%2Fgeo%2Finfo%2Frnaseqcounts.html)は [1 件](https://twitter.com/iNut/status/1649213014418296833)。] ことがこの記事を書く動機でした。

![NCBI-generated RNA-seq count data の目次](/images/ncbi-gen-counts/geo_rnaseqcounts_toc.png)

まだ BETA 版ですが、ヒトは計算が終わっていて^[ただ、いくつか見てみた感覚では、昔の GSE でも 20-30 % 程度は計算されておらず、何らかの要因で失敗しているものがあるようで、[Limitations and caveats](https://www.ncbi.nlm.nih.gov/geo/info/rnaseqcounts.html#limitations) の節で言及がされています。失敗した GSE は [How NCBI generates RNA-seq count data](https://www.ncbi.nlm.nih.gov/geo/info/rnaseqcounts.html#how) の節を読んで同じ条件で処理すれば比較可能になるかも？]、今年（2023 年）秋にはマウスについても完了すると記載があります^[ページ自体が "Last modified: April 25, 2023" とのことなので、遅延の可能性も…]。

> NCBI generates RNA-seq count data for only human and mouse RNA-seq runs submitted to GEO. The human RNA-seq count data are currently available and the mouse RNA-seq count data are expected to become available in Fall, 2023.

さて、このカウントデータを取得するには [GEO DataSets の検索結果](https://www.ncbi.nlm.nih.gov/gds?term=%22rnaseq%20counts%22%5BFilter%5D)から `Download data` をクリックするか、[`https://www.ncbi.nlm.nih.gov/geo/download/?acc={GSExxx}`](https://www.ncbi.nlm.nih.gov/geo/download/?acc=GSE164073) という URL をたたきます。（GSE ページからリンク貼ってくれればいいのに…）

![NCBI-generated RNA-seq count data の DL 方法](/images/ncbi-gen-counts/geo_rnaseqcounts_howto.png)

この Download data for GSExxx のページでは、**Submitter-supplied data** として GSE のページと同じファイルが、そして、**NCBI-generated data** としてカウントデータ (raw, FPKM, TPM) とアノテーションファイルがあります。

![Download data for GSE164073](/images/ncbi-gen-counts/GSE164073_download.png)

カウントデータのうち、正規化前の発現量の [`GSE164073_raw_counts_GRCh38.p13_NCBI.tsv`](https://www.ncbi.nlm.nih.gov/geo/download/?type=rnaseq_counts&acc=GSE164073&format=file&file=GSE164073_raw_counts_GRCh38.p13_NCBI.tsv.gz) は、column がサンプル名（18 列）、index が NCBI Gene ID（39,376 行）です。

![GSE164073_raw_counts_GRCh38.p13_NCBI.tsv](/images/ncbi-gen-counts/GSE164073_raw_counts_GRCh38.p13_NCBI.tsv.png)

アノテーションファイルの [`Human.GRCh38.p13.annot.tsv`](https://www.ncbi.nlm.nih.gov/geo/download/?format=file&type=rnaseq_counts&file=Human.GRCh38.p13.annot.tsv.gz) は、column がアノテーションの種類（18 列）、index が NCBI Gene ID（39,376 行）です。NCBI Gene ID は一意ですが、HGNC Gene Symbol は一意でなく（このバージョンでは TRNAV-CAC が 3 行存在）、EnsemblGeneID も欠測があります。

![Human.GRCh38.p13.annot.tsv](/images/ncbi-gen-counts/Human.GRCh38.p13.annot.tsv.png)

# ncbi-counts の使い方

ただ、筆者の使い方だとこんな感情が湧いてきました。

- URL の生成がめんどう
- GSE 内の一部のサンプルのみを抽出したい
  - カラム名が GSM 番号なので、GSE/GSM のページを見ないとサンプルの属性が分からない
- カウントと（一部の）アノテーションを同じテーブルに結合したい

そこで、これらを解決してカウントデータを便利にダウンロードできる Python パッケージを作ってみました。

https://pypi.org/project/ncbi-counts/

- yaml を用意して、コマンドラインで複数のカウントデータを一括ダウンロードする
- GSE 中の欲しい GSM の属性を正規表現で指定する
- アノテーションのうち、欲しいカラムを指定する

ことができます。

## 例: GSE164073 から各組織のカウントを取得する（コマンドライン）

前述したように、[GSE164073](https://www.ncbi.nlm.nih.gov/geo/query/acc.cgi?acc=GSE164073) には 3 組織それぞれで mock が 3サンプルと SARS-CoV-2 が 3 サンプルあります。例として、組織ごとのカウントデータを取得してみましょう。まず、このような yaml ファイルを作成して `sample_regex.yaml` として保存します。

```sample_regex.yaml
GSE164073:
- control:
    title: Cornea
    characteristics_ch1: mock
  treatment:
    title: Cornea
    characteristics_ch1: SARS-CoV-2
- control:
    title: Limbus
    characteristics_ch1: mock
  treatment:
    title: Limbus
    characteristics_ch1: SARS-CoV-2
- control:
    title: Sclera
    characteristics_ch1: mock
  treatment:
    title: Sclera
    characteristics_ch1: SARS-CoV-2
```

あとは ncbi-counts をインストールして、yaml ファイルを引数に渡せば実行できます。
`-k Symbol`  はアノテーションのうち、Symbol カラムを残すことを示し、`-c` は中間ファイルをすべて削除することを示します。他に用意してある引数は [PyPI ページ](https://pypi.org/project/ncbi-counts/)の Options を参照。

```sh
pip install ncbi-counts
python -m ncbi_counts sample_regex.yaml -k Symbol -c
```

すると、Cornea のカウントデータが GSE164073-1.tsv として以下のように出力されます。
GeneID 列がユニークな遺伝子 ID で、指定した Symbol カラムに遺伝子シンボルがあり、あとはサンプルごとの発現量です。yaml 中の群 (control, treatment) それぞれの正規表現にマッチしたサンプルについて、群名が GSM の prefix として付与されます。

![GSE164073-1.tsv](/images/ncbi-gen-counts/GSE164073-1.tsv.png)

Limbus と Sclera もそれぞれ GSE164073-2.tsv と GSE164073-3.tsv として出力されます。

## yaml ファイルの説明

GSE 番号の辞書→出力ファイル内の群名を key とする辞書のリスト→各群サンプルの属性の辞書 の 3 層構造としています。それぞれの辞書の要素は複数でも大丈夫です。

この例では、GSE 番号の辞書は key が `GSE164073`、出力ファイル内の群名は `control`, `treatment` でリストの要素は 3 つ、サンプルの属性の辞書は key が `title`, `characteristics_ch1` で value が対応する正規表現です。

サンプル属性の `title` は GSM の Title 属性（Cornea_mock_1 や Cornea_CoV2_1）に対応し、`characteristics_ch1` は Characteristics 属性（tissue, infection, time point の 3 行）に対応します。

![GSM4996084](/images/ncbi-gen-counts/GSM4996084.png)
![GSM4996087](/images/ncbi-gen-counts/GSM4996087.png)

属性の名前には Series の soft ファイルの !Sample_ 以降の文字列を使用できます（[GEOparse](https://geoparse.readthedocs.io/en/latest/index.html) の仕様）。

![GSE164073_family.soft](/images/ncbi-gen-counts/GSE164073_family.soft.png)

ただ、soft ファイルを参照せずとも、[SOFT submission instructions](https://www.ncbi.nlm.nih.gov/geo/info/soft.html) ページ内の [Sample Attributes](https://www.ncbi.nlm.nih.gov/geo/info/soft.html#sample_tab) や [SOFT download](https://www.ncbi.nlm.nih.gov/geo/info/soft.html#download) を見れば大体対応しそうな属性名は分かると思います。

## プログラム中での使用

また、プログラム中で `pandas.DataFrame` として取得するためには、上の yaml の代わりに同じ構造の辞書を用意することも可能です。

```python
from ncbi_counts import Series

series = Series(
    "GSE164073",
    [
        {
            "control": {"title": "Cornea", "characteristics_ch1": "mock"},
            "treatment": {"title": "Cornea", "characteristics_ch1": "SARS-CoV-2"},
        },
        {
            "control": {"title": "Limbus", "characteristics_ch1": "mock"},
            "treatment": {"title": "Limbus", "characteristics_ch1": "SARS-CoV-2"},
        },
        # GSM 番号で指定する場合は、例えばこう書けます
        {
            "control": {"geo_accession": "^GSM499609[6-8]$"},
            "treatment": {"geo_accession": "^GSM4996099$|^GSM4996100$|^GSM4996101$"},
        },
    ],
    keep_annot=["Symbol"],
    save_to=None,
)
series.generate_pair_matrix()
series.cleanup()  # コメントアウトすると中間ファイルが残ります
series.pair_count_list[0]  # Cornea のカウントデータ（GSE164073-1.tsv と同じ内容）
series.pair_count_list[1]  # Limbus のカウントデータ（GSE164073-2.tsv と同じ内容）
series.pair_count_list[2]  # Sclera のカウントデータ（GSE164073-3.tsv と同じ内容）
```

以上が ncbi-counts の大まかな使い方です。

NCBI-generated RNA-seq count data や GEO の構造等について完全に理解しているわけではないので、誤りの指摘・issue・pull request があればお待ちしてます。

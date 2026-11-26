# SBOM・SCAハンズオン（Syft／Grype・Trivy／Dependency-Track）

SBOMと脆弱性チェックを、無料のOSSツールで半日〜1日かけて体験するための練習用リポジトリです。

> ⚠️ `requirements.txt` には、脆弱性があることがわかっている古いバージョンを**わざと**書いています。インストールも実行もしないでください。

## このハンズオンの目的

Black Duckなどの商用SCAツールが内部でやっていることを、OSSツールで1つずつ分解して手で追います。そうすることで、商用ツールの出力を読み解き、正しいかどうかを自分で確かめられるようになります。

## 全体の流れ

```text
① 部品表を作る   ② 中身を確かめる   ③ 脆弱性を探す      ④ 影響なしを記録   ⑤ 見張り続ける          ⑥ 自動でチェック
   (Syft)     →               →   (Grype / Trivy) →   (VEX)        →   (Dependency-Track) →   (GitHub Actions)
```

| 言葉 | 意味 |
| --- | --- |
| SBOM | ソフトウェアの部品表。どのライブラリのどのバージョンを使っているかの一覧 |
| SCA | ソフトウェア構成分析。使っているOSSを洗い出し、脆弱性やライセンスを確かめること |
| 脆弱性の照合 | SBOMの部品を、脆弱性データベースと突き合わせること |
| VEX | 「この脆弱性は、うちの製品には影響しない」などの判断を、機械が読める形で残した文書 |
| purl | 部品を一意に表すID（例：`pkg:pypi/pyyaml@5.3`） |
| CycloneDX、SPDX | SBOMの代表的な2つのフォーマット |

### ツールの役割とBlack Duckとの対応

| 役割 | OSSツール | Black Duckでの相当機能 |
| --- | --- | --- |
| SBOM生成（部品の洗い出し） | Syft | Detectによるスキャン、BOM、SBOMレポート出力 |
| 脆弱性照合 | Grype、Trivy | 脆弱性の検出（Black Duck Security Advisoriesを含むナレッジベース） |
| 影響なしの記録 | OpenVEX（vexctl） | 脆弱性の修正状態（Remediation status）、VEXの出力 |
| 継続管理・新しい脆弱性の通知 | Dependency-Track | プロジェクトのバージョン管理、ポリシー、通知 |
| CI組み込み | GitHub Actions | Black Duck Security Scan（GitHub Action） |

## 結果のまとめ（2026-11-25〜26に実行）

| やったこと | 結果 |
| --- | --- |
| SBOMを作る | 5つの部品が見つかった。ただし、ライセンス・供給者・依存関係は空 |
| GrypeとTrivyで照合 | どちらも**28件**。表示するIDが違うだけで、中身は同じ |
| VEXを反映 | 1件を「影響なし」にして**27件** |
| PyYAMLを6.0.2に上げる | **26件**（PyYAMLの2件が消えた） |
| Dependency-Track | 設定を直すと、Grypeと同じ28件／26件 |
| GitHub Actions | 27件を検出し、High以上があるので**チェック失敗**（期待どおり） |

## ファイル

| ファイル | 中身 |
| --- | --- |
| `requirements.txt` | 古いバージョンの依存5つ（requests、urllib3、PyYAML、Jinja2、Flask） |
| `app.py` | PyYAMLを安全な `safe_load` だけで使うコード。Step 4で「影響なし」と言う根拠 |
| `sbom-lab.openvex.json` | PyYAMLの脆弱性 CVE-2020-1747 を「影響なし」とするVEX |
| `.github/workflows/sbom.yml` | PRごとに自動でSBOMを作り、脆弱性をチェックする設定 |

---

## 準備

個人PCでは、公開情報だけで作ったこの練習用プロジェクトだけを扱います。**顧客のコードや資料は一切入れないでください。**

必要なもの：

- ターミナル
- Docker（Step 5のDependency-Track用。メモリ8GB以上を推奨）
- jq（SBOMの中身を見るため）

### ツールのインストール

| ツール | macOS（Homebrew） | Windows | 確認コマンド |
| --- | --- | --- | --- |
| Syft | `brew install syft` | `winget install Anchore.Syft`、またはGitHub Releasesのzip | `syft version` |
| Grype | `brew install grype` | `winget install Anchore.Grype`、またはGitHub Releasesのzip | `grype version` |
| Trivy | `brew install trivy` | GitHub Releasesのzip（またはScoop、Chocolatey） | `trivy --version` |
| vexctl | `brew install vexctl` | GitHub Releasesのバイナリ、または `go install github.com/openvex/vexctl@latest` | `vexctl version` |

- wingetのパッケージIDは変わることがあります。見つからない場合は `winget search syft` で確認してください
- Windowsでは、WSL2（Ubuntu）を使うと、このREADMEのコマンドをそのまま実行できます
- 動作を確認したバージョン：Syft 1.57.2、Grype 0.122.1、Trivy 0.76.2、vexctl 0.4.4、Dependency-Track 5.2.1

### 練習用プロジェクト

このリポジトリをcloneすれば、そのまま使えます。自分で一から作る場合は、次のとおりです。

```bash
mkdir sbom-lab && cd sbom-lab

cat > requirements.txt <<'EOF'
requests==2.19.1
urllib3==1.24.1
PyYAML==5.3
Jinja2==2.10
Flask==1.0
EOF

# Step 4で「影響なし」の根拠に使うコード（PyYAMLをsafe_loadだけで使う）
cat > app.py <<'EOF'
import requests, yaml
print(yaml.safe_load("a: 1"))
EOF

git init && git add . && git commit -m "sbom lab"
```

古いバージョンには既知の脆弱性があるため、Step 3で確実に検出されます。

---

## Step 1：SBOMを作る（Syft）

1回の実行で、CycloneDXとSPDXの両方を出せます。

```bash
syft dir:. \
  --source-name sbom-lab --source-version 1.0.0 \
  -o cyclonedx-json=sbom.cdx.json \
  -o spdx-json=sbom.spdx.json \
  -o table
```

**ポイント**：`--source-name` と `--source-version` を付けないと、「何のSBOMか」がディレクトリ名から推測され、警告が出ます。製品名とバージョンは必ず指定します。

**実行結果**

```text
NAME      VERSION  TYPE
flask     1.0      python
jinja2    2.10     python
pyyaml    5.3      python
requests  2.19.1   python
urllib3   1.24.1   python
```

**試してみるとよいこと**

- `syft python:3.12-slim`：コンテナイメージをスキャンすると、OSパッケージ（debなど）まで含む数百件のSBOMになります
- 仮想環境に実際にインストールしてからスキャンする：推移的な依存（requestsが使うidnaやchardetなど）やライセンス情報が加わります。Step 2の「欠けている項目」がどう変わるか比べてみてください
- `syft convert sbom.cdx.json -o spdx-json`：フォーマットを変換できます

## Step 2：SBOMの中身を確かめる

`requirements.txt` から作ったSBOMには、直接依存の名前とバージョンしか入っていません。供給者・ライセンス・依存関係は空でした。**SBOMは作って終わりではなく、何が入っていて何が欠けているかを確かめる必要があります。**

**生成された内容**

- CycloneDX：`specVersion` は1.7。`metadata` に生成日時、ツール（syft）、対象（sbom-lab 1.0.0）が入る。各部品には `name`、`version`、`purl`（例：`pkg:pypi/flask@1.0`）、`cpe` が入る。`dependencies`（依存関係）は無し
- SPDX：`spdxVersion` は SPDX-2.3。`creationInfo` に作成者（Anchore, Incとsyft-1.57.2）と日時が入る。各パッケージの `supplier`、`downloadLocation`、`licenseConcluded`、`licenseDeclared` はすべて `NOASSERTION`（不明）。`relationships` は「requirements.txtから見つかった」という関係だけ

### 必須項目チェック

米国NTIAの最小要素をもとにした主な項目と、今回のSBOMの状態です。EUのCRA（サイバーレジリエンス法）向けには、ドイツBSIのTR-03183-2が、ライセンスやハッシュなど、さらに多くの項目を求めています。

| 項目 | CycloneDXの場所 | SPDXの場所 | 今回のSBOM |
| --- | --- | --- | --- |
| 部品名 | `components[].name` | `packages[].name` | あり |
| バージョン | `components[].version` | `packages[].versionInfo` | あり |
| 一意の識別子 | `purl`、`cpe` | `externalRefs`（purl、cpe23Type） | あり |
| 供給者 | `supplier` | `supplier` | 不明 |
| ライセンス | `licenses` | `licenseDeclared` | 不明 |
| 依存関係 | `dependencies` | `relationships`（DEPENDS_ON） | 無し |
| 作成者・作成日時 | `metadata.tools`、`metadata.timestamp` | `creationInfo` | あり |
| ハッシュ | `hashes` | `checksums` | 無し（`requirements.txt` ファイル自体にだけ付く） |

### 中身を確かめるコマンド

```bash
# 部品の名前・バージョン・purlを一覧にする
jq -r '.components[] | [.name, .version, .purl] | @tsv' sbom.cdx.json

# ライセンスが入っていない部品を数える
jq '[.components[] | select(.licenses == null)] | length' sbom.cdx.json

# SPDXで供給者が不明なパッケージ
jq -r '.packages[] | select(.supplier == "NOASSERTION") | .name' sbom.spdx.json
```

フォーマット自体が正しいかは、CycloneDXなら `cyclonedx-cli validate`、SPDXなら `pyspdxtools` で確かめられます。

## Step 3：脆弱性を探す（Grype・Trivy）

同じSBOMを2つのツールで照合し、結果の違いを見ます。ツールごとに使う脆弱性データベースや、バージョン範囲の判定方法が違うため、件数やIDが一致しないことがあります。この「ツールによって結果が違う」感覚は、商用ツールの結果を確かめるときにそのまま役立ちます。

**初回は時間がかかります**：各ツールが脆弱性データベース（数百MB）をダウンロードします。会社のプロキシ環境では失敗することがあります（Grypeは grype.anchore.io、Trivyは mirror.gcr.io または ghcr.io）。CIを作るときも、同じ通信の許可が必要です。

### Grype

```bash
# 表で表示
grype sbom:sbom.cdx.json

# JSONで保存（後で比較する用）
grype sbom:sbom.cdx.json -o json > grype.json

# High以上があれば終了コード2（CIのゲートで使う）
grype sbom:sbom.cdx.json --fail-on high; echo "exit=$?"

# 修正版があるものだけ表示
grype sbom:sbom.cdx.json --only-fixed
```

### Trivy

```bash
trivy sbom sbom.cdx.json
trivy sbom sbom.spdx.json          # SPDXも読める
trivy sbom --format json -o trivy.json sbom.cdx.json
trivy sbom --severity HIGH,CRITICAL --exit-code 1 sbom.cdx.json
```

### 見るところ

- requests、urllib3、PyYAML、Jinja2、Flaskのそれぞれに脆弱性が出ます
- 各行の「FIXED IN（修正済みバージョン）」が、そのまま「どこまで上げればよいか」になります
- IDは `CVE-...` と `GHSA-...`（GitHubの勧告ID）が混ざります

### 2つのツールの結果を比べる

```bash
jq -r '.matches[] | "\(.artifact.name)\t\(.vulnerability.id)"' grype.json | sort > g.txt
jq -r '.Results[].Vulnerabilities[]? | "\(.PkgName)\t\(.VulnerabilityID)"' trivy.json | sort > t.txt
diff g.txt t.txt
```

**実際の結果**：

- どちらも28件見つかりました
- ただし、Grypeは `GHSA-...`、Trivyは `CVE-...` で表示するので、上の `diff` では全部違うように見えます
- `grype sbom:sbom.cdx.json --by-cve -o json` で作り直して比べると、**Trivyと完全に一致**します
- 残った違いは修正版の書き方だけです。Grypeは一番近い修正版を、Trivyは系列ごとの修正版をすべて表示します（例：urllib3 の CVE-2023-43804 は、Grypeが「1.26.17」、Trivyが「2.0.6, 1.26.17」）

**PyYAMLのCritical 2件に注目**：CVE-2020-14343 は「CVE-2020-1747 の修正が不完全だった」という脆弱性です。`safe_load` しか使わないなら、こちらも「影響なし」と言えます。でも今回は、あえてStep 4のVEXに入れずに残しています。**VEXは脆弱性1つずつ判断が必要**だと体験するためです。

## Step 4：「影響なし」をVEXに記録する

VEXは、「この製品はこの脆弱性の影響を受けるか」を機械が読める形で残す文書です。状態は次の4つです。

| 状態 | 意味 |
| --- | --- |
| `not_affected` | 影響なし（理由が必須） |
| `affected` | 影響あり |
| `fixed` | 修正済み |
| `under_investigation` | 調査中 |

トリアージで「影響なし」と判断したものを、顧客や当局に示せる形で残す練習です。

**題材**：PyYAML 5.3 には、`full_load` やFullLoaderで信頼できないYAMLを読むと、任意のコードを実行されてしまう脆弱性（CVE-2020-1747）があります。`app.py` は安全な `safe_load` しか使っていないので、「脆弱なコードは実行されない」と言えます。

### 1. vexctlでVEXを作る

```bash
vexctl create \
  --author="your-name" \
  --product="pkg:pypi/pyyaml@5.3" \
  --vuln="CVE-2020-1747" \
  --aliases="GHSA-6757-jp84-gxfx" \
  --status="not_affected" \
  --justification="vulnerable_code_not_in_execute_path" \
  --impact-statement="app.py uses yaml.safe_load only" \
  > sbom-lab.openvex.json
```

- `--product` には、SBOMに入っているpurlをそのまま使います。本来は製品（例：コンテナイメージ）を `--product`、脆弱な部品を `--subcomponents` に分けて書きますが、ツールごとに照合の仕方が違うため、ここでは部品のpurlを `--product` にしています
- **`--aliases` が大事です**：Grypeはこの脆弱性を `GHSA-6757-jp84-gxfx` というIDで照合します。CVE IDだけのVEXだと、Trivyでは効きますが、Grypeでは**警告なしで無視**されます。`--aliases` でGHSAのIDも入れると、両方で効きます（または、Grypeを `--by-cve` で実行する方法もあります）

### 2. 照合結果に反映する

```bash
# Grype
grype sbom:sbom.cdx.json --vex sbom-lab.openvex.json --show-suppressed

# Trivy（VEXは実験的な機能）
trivy sbom --vex sbom-lab.openvex.json --show-suppressed sbom.cdx.json
```

CVE-2020-1747 が通常の一覧から消え、「suppressed（抑制）」として表示されれば成功です（28件 → 27件）。消えない場合は、purlと脆弱性IDが照合結果と完全に一致しているかを確かめてください。

### justification（理由）の選び方

| 値 | 意味 |
| --- | --- |
| `component_not_present` | その部品は製品に含まれていない（SBOMの誤検出など） |
| `vulnerable_code_not_present` | 部品はあるが、脆弱なコードは含まれていない |
| `vulnerable_code_not_in_execute_path` | 脆弱なコードはあるが、実行されない |
| `vulnerable_code_cannot_be_controlled_by_adversary` | 実行されるが、攻撃者が入力を制御できない |
| `inline_mitigations_already_exist` | 製品側に緩和策がある |

### 実務での注意

- `not_affected` は、「誰が、何を根拠に判断したか」まで残します。根拠の弱い判定は、修正しない言い訳になってしまいます
- CISA KEV（実際に悪用されている脆弱性の一覧）に載っているものは、「影響なし」とする前に特に慎重に確かめます
- 一番確実な対応は、修正版への更新です。`requirements.txt` を `PyYAML==6.0.2` に変えてSBOMを作り直すと、PyYAMLの検出が消えて26件になります

## Step 5：Dependency-Trackで見張り続ける

GrypeやTrivyは「今この瞬間」のチェックです。Dependency-TrackにSBOMを登録しておくと、後から公開された脆弱性も自動で拾い続けます。これは、CRAが求める「サポート期間中の脆弱性対応」や、定期的な再評価にあたる部分です。

### 1. 起動する

```bash
mkdir dtrack && cd dtrack
curl -LO https://dependencytrack.org/docker-compose.yml
docker compose up -d
```

| | URL |
| --- | --- |
| 画面 | http://localhost:8081 |
| API | http://localhost:8080 |

⚠️ **ポート番号に注意**：v5（ここでは5.2.1）では、APIが8080、画面が8081です。ネット上の記事に多い4.x系とは**逆**です。版によって変わるので、ダウンロードした `docker-compose.yml` で確かめてください。初期アカウントは admin / admin で、初回ログイン時にパスワードの変更を求められます。

起動直後は、脆弱性データの取り込みにしばらく（数十分のことも）かかります。その間に登録したSBOMは、脆弱性が0件に見えることがあります。

### 2. APIキーを用意する

画面の Administration → Access Management → Teams でチームを作り、`BOM_UPLOAD` と `PROJECT_CREATION_UPLOAD` の権限を付けてから、APIキーを発行します。

この2つの権限だけだと、APIでプロジェクト一覧を見ようとすると403になります。登録専用のキーとしては、これが正しい状態です。

### 3. SBOMを登録する

```bash
export DT_API_KEY="発行したキー"
curl -X POST "http://localhost:8080/api/v1/bom" \
  -H "X-Api-Key: ${DT_API_KEY}" \
  -F "autoCreate=true" \
  -F "projectName=sbom-lab" \
  -F "projectVersion=1.0.0" \
  -F "bom=@sbom.cdx.json"
```

Dependency-Trackが取り込めるのはCycloneDXだけです（SPDXは使えません）。フォーマットを選ぶときの判断材料になります。

### 4. 脆弱性ソースの設定（重要）

**初期設定のままだと、Flask、Jinja2、PyYAMLの脆弱性が見つかりません。** 有効な脆弱性ソースがNVDだけで、NVDはCPEで照合するため、PyPIの部品を見落とすからです。

| 設定 | 場所 | 変更前 → 変更後 |
| --- | --- | --- |
| OSVを有効にする | Administration → Vulnerability Sources → OSV → Enabled | 16件（requestsとurllib3だけ） → 72行 |
| 別名をまとめる | 同じ画面の Alias Synchronization | 72行（同じ脆弱性が GHSA、PYSEC、CVE の別IDで重複） → **28件** |

- OSVのデータ取り込みは1日1回（03:00 UTC）です。設定してすぐ反映したい場合は、取り込み後にSBOMを登録し直してください
- Alias Synchronizationを後から有効にした場合は、一度だけ Incremental Mirroring を無効にして全件を取り込み直し、終わったら元に戻します
- 設定後の結果：1.0.0は28件（Critical 2、High 10、Medium 16）、PyYAMLを6.0.2に上げた1.1.0は26件で、Grypeと一致しました
- Findings一覧はID単位なので72行のままです。タブのバッジ「28 / 72」の左側が、重複を除いた件数です

### 5. 画面で試すこと

- プロジェクトの Components タブと Audit Vulnerabilities タブで、部品と脆弱性を確かめる
- 脆弱性を開き、分析状態（Not Affectedなど）と理由を入力してトリアージする。プロジェクトのメニューから、その判定をCycloneDX VEXとして書き出せる
- `PyYAML==6.0.2` に上げたSBOMをバージョン1.1.0として登録し、バージョン間で脆弱性が減ったことを確かめる
- Policy Management で「SeverityがHigh以上なら違反」などのポリシーを作ってみる（Black Duckのポリシーに相当）

## Step 6：GitHub Actionsで自動チェック

PRごとにSBOMの生成と脆弱性のチェックを自動で行います。「PRでチェックし、リリースでSBOMを保管する」という運用を、OSSツールで小さく再現する練習です。

設定ファイルは [`.github/workflows/sbom.yml`](.github/workflows/sbom.yml) です。

```yaml
name: sbom
on:
  pull_request:
  push:
    tags: ['v*']
  workflow_dispatch:

jobs:
  sbom:
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@v7

      # SyftでSBOMを生成（Anchore公式のアクション）
      - uses: anchore/sbom-action@v0
        with:
          path: .
          format: cyclonedx-json
          output-file: sbom.cdx.json
          upload-artifact: false   # 保管はタグ時だけ下で行う

      # Grypeで照合。High以上で失敗させる
      - uses: anchore/scan-action@v7
        with:
          sbom: sbom.cdx.json
          vex: sbom-lab.openvex.json   # Step 4のVEXを反映
          by-cve: true                 # IDをCVEにそろえる
          severity-cutoff: high
          fail-build: true
          output-format: table

      # タグのときだけSBOMとVEXを保管
      - if: startsWith(github.ref, 'refs/tags/')
        uses: actions/upload-artifact@v7
        with:
          name: sbom-${{ github.ref_name }}
          path: |
            sbom.cdx.json
            sbom-lab.openvex.json
```

| いつ | 何をする |
| --- | --- |
| Pull Requestを作ったとき | SBOMを作る → 脆弱性をチェック（VEXも反映）→ High以上があれば失敗 |
| `v*` タグをpushしたとき | 上に加えて、SBOMとVEXを保管 |
| 手動 | Actionsタブの「Run workflow」 |

**設定のポイント**

- アクションは2026-11-25時点の最新メジャー（checkout、scan-action、upload-artifactはv7）を使っています。使う前に、各リポジトリのREADMEでバージョンと入力名を確かめてください
- `vex:` でStep 4のVEXを反映し、`by-cve: true` でIDをCVEにそろえています
- sbom-actionは、何も指定しないとSBOMを毎回Artifactとして保管します。タグのときだけ保管したいので、`upload-artifact: false` を付けています

### チェックが失敗するのは正しい動作です

このリポジトリは脆弱なバージョンをわざと使っているので、**チェックは必ず失敗します**。失敗することで、「危ない部品があったらマージを止める」仕組みが効いていることを確かめられます。

実際の結果（[PR #1](https://github.com/muffin-jp/sbom-sca-hands-on/pull/1)）：

- SBOMの生成：成功
- 脆弱性チェック：27件を検出（VEXで CVE-2020-1747 を抑制済み）
- High以上（例：PyYAMLの CVE-2020-14343、Critical）があるため失敗

チェックを通したい場合は、`requirements.txt` のバージョンを、結果の「FIXED IN」の列以上に上げてください。

### 注意

- SBOMファイル（`sbom*.cdx.json` など）はコミットしないでください。Syftがそれも読み込んでしまい、部品が二重に数えられます（`.gitignore` で除外済み）
- アクションが使うGrypeのバージョン（0.110.0）は、手元のバージョンと違うことがあります。結果がずれたら、まずバージョンを確かめてください
- Dependency-Trackへの自動登録は `DependencyTrack/gh-upload-sbom` でできますが、手元のDependency-TrackにはGitHubから届かないので、この練習では省略しています
- このワークフローは個人の学習用です。顧客環境に持ち込む場合は、成果物の扱いと外部コードの持ち込みルールを確かめてからにしてください

---

## 振り返り

終わったら、次のことを自分の言葉で説明できるか確かめてください。顧客との打ち合わせや、チームへの説明でそのまま使えます。

- [ ] マニフェスト（requirements.txt）と、インストール済み環境やイメージでは、SBOMの中身がどう違うか
- [ ] CycloneDXとSPDXで、同じ情報がどこに入るか（purl、供給者、ライセンス、依存関係）
- [ ] 必須項目のうち、ツールが自動で埋められないものはどれか
- [ ] 同じSBOMでも、GrypeとTrivyで検出結果が違う理由
- [ ] VEXの4つの状態と、`not_affected` に理由が必要な理由
- [ ] 「その場の照合」と「登録したSBOMの継続監視」の違いと、CRAとの関係

<details>
<summary>回答例（まず自分で考えてから開いてください）</summary>

**1. マニフェストとインストール済み環境・イメージの違い**

マニフェストから作ると、開発者が書いた直接依存の名前とバージョンだけが入ります（今回は5件で、ライセンス・供給者・ハッシュ・依存関係は空）。インストール済み環境をスキャンすると、推移的な依存、ライセンス、ハッシュが加わります。コンテナイメージでは、さらにOSパッケージまで入ります。マニフェストは「意図した構成」、インストール済み環境やイメージは「実際に出荷される構成」です。顧客への提出や脆弱性管理には、出荷されるものから作ったSBOMを使います。

**2. CycloneDXとSPDXで情報が入る場所**

| 情報 | CycloneDX | SPDX 2.3 |
| --- | --- | --- |
| purl | `components[].purl`（専用の項目） | `packages[].externalRefs[]`（`referenceType: purl`） |
| 供給者 | `components[].supplier` | `packages[].supplier`（不明なら `NOASSERTION`） |
| ライセンス | `components[].licenses[]` | `licenseDeclared`（宣言）と `licenseConcluded`（判断）の2つ |
| 依存関係 | 最上位の `dependencies[]` | `relationships[]`（`DEPENDS_ON` など） |

CycloneDXは脆弱性管理向けで、VEXも扱えます。SPDXはライセンス管理向けで、宣言と判断を分けて書けます。

**3. ツールが自動で埋められない項目**

- 供給者：パッケージの情報に無いことが多く、人が決める必要がある
- ライセンスの判断（`licenseConcluded`）：宣言されたライセンスは取れても、実際にどれが適用されるかは人の判断
- 依存関係：requirements.txtには親子関係の情報が無い
- 製品名とバージョン：指定しないとディレクトリ名から推測される
- 作成者としての組織：今回のSPDXでは、作成者がツールの作者（Anchore）になっていた

**4. GrypeとTrivyで結果が違う理由**

- 脆弱性データベースが違う（取り込み元、更新のタイミング、重大度の付け方）
- 代表として表示するIDが違う（今回は件数が同じでも、GHSAとCVEで全部ずれて見えた）
- バージョン範囲の解釈や、修正版の書き方が違う
- purlで照合するか、CPEも使うかが違う

比べる前に、IDをそろえることが大事です。

**5. VEXの4つの状態と、`not_affected` に理由が必要な理由**

状態は `not_affected`、`affected`、`fixed`、`under_investigation` の4つです。`not_affected` は、スキャナーの結果から脆弱性を消す判断なので、第三者（顧客、当局、監査）が後から確かめられる根拠が必要です。理由が無いと、「直したくないから影響なしにした」と区別できません。また、製品の使い方が変われば（例：`full_load` を使い始めたら）、判断も見直しが必要です。

**6. その場の照合と継続監視の違い、CRAとの関係**

その場の照合（Grype、Trivy）は、実行した時点のデータベースでチェックします。PRやリリースのゲートに向いていますが、出荷後に公開された脆弱性は、もう一度実行しないとわかりません。継続監視（Dependency-Track、Black Duck）は、登録したSBOMをデータベースの更新のたびにチェックし直し、新しい脆弱性を知らせます。CRAは、サポート期間中の脆弱性対応、SBOMの作成と保持、悪用されている脆弱性の報告を求めています。出荷後の脆弱性に気づくには、継続監視が必要です。

</details>

## Black Duckの出力を検証するときの観点

| 観点 | 確かめること |
| --- | --- |
| 部品の漏れ・誤検出 | 同じリポジトリをSyftでもスキャンし、件数と部品名を突き合わせる。C/C++のコピーされたソースなど、Black Duckのシグネチャスキャンでしか見つからないものもある |
| SBOMの必須項目 | Black Duckが出したSBOMをStep 2の表とjqで点検し、供給者・ライセンス・依存関係が埋まっているかを確かめる |
| 脆弱性の結果 | 同じSBOMをGrype・Trivyでも照合し、Black Duckにだけ出るもの・出ないものの理由を説明できるようにする。Black DuckはBDSA（独自のID）とCVEを併記するので、**CVEにそろえてから比べる** |
| フォーマットの互換性 | Black DuckのSBOMを、Dependency-TrackやTrivyが読めるか確かめる（顧客の顧客に渡すときに重要） |
| VEXの扱い | Black Duckで入力した修正状態が、どの形式のVEXとして出せるかを確かめる。**取り込む側のツールが使うIDがaliasに入っていないと、黙って無視される** |

この比較は、顧客の許可を得て、顧客の環境の中で行ってください。顧客のコードやSBOMを個人PCに持ち出さないようにします。

## このハンズオンでわかったこと

1. **IDをそろえてから比べる**：同じ脆弱性でも、ツールによって `CVE`、`GHSA`、`PYSEC` など違うIDで表示される
2. **VEXはIDが合わないと黙って無視される**：エラーにならないので気づきにくい
3. **ツールの設定で結果が変わる**：Dependency-Trackは、設定次第で16件にも72行にも28件にも見えた
4. **ツールのバージョンで手順が変わる**：ポート番号やアクションのバージョンなど。手順を書くときは、使ったバージョンも書いておく

## 参考資料

- [Syft](https://github.com/anchore/syft)、[Grype](https://github.com/anchore/grype)、[Trivy](https://github.com/aquasecurity/trivy)
- [vexctl（OpenVEX）](https://github.com/openvex/vexctl)
- [Chainguard：OpenVEXとvexctlの入門](https://edu.chainguard.dev/open-source/sbom/getting-started-openvex-vexctl/)
- [OpenSSF：OpenVEXとOSS脆弱性スキャナー](https://openssf.org/?p=4293)
- [Dependency-Track ダウンロード（Docker Compose）](https://dependencytrack.org/download)
- [Dependency-Track ドキュメント：Dockerでの導入](https://docs.dependencytrack.org/getting-started/deploy-docker/)
- [Dependency-Track ドキュメント：CI/CD連携（BOMの登録API）](https://docs.dependencytrack.org/usage/cicd/)
- [anchore/sbom-action](https://github.com/anchore/sbom-action)、[anchore/scan-action](https://github.com/anchore/scan-action)

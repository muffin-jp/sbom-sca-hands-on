# sbom-sca-hands-on

SBOMと脆弱性チェックを、無料のツールで体験するための練習用リポジトリです。

> ⚠️ `requirements.txt` には、脆弱性があることがわかっている古いバージョンを**わざと**書いています。インストールも実行もしないでください。

## このリポジトリでやること

```text
① 部品表を作る      ② 脆弱性を探す        ③ 影響なしを記録する    ④ 見張り続ける           ⑤ 自動でチェックする
   (Syft)        →    (Grype / Trivy)  →     (VEX)            →    (Dependency-Track)  →    (GitHub Actions)
```

| 言葉 | 意味 |
| --- | --- |
| SBOM | ソフトウェアの部品表。どのライブラリのどのバージョンを使っているかの一覧 |
| 脆弱性の照合 | SBOMの部品を、脆弱性データベースと突き合わせること |
| VEX | 「この脆弱性は、うちの製品には影響しない」などの判断を、機械が読める形で残した文書 |
| purl | 部品を一意に表すID（例：`pkg:pypi/pyyaml@5.3`） |

## 結果のまとめ

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
| `app.py` | PyYAMLを安全な `safe_load` だけで使うコード。「影響なし」と言える根拠 |
| `sbom-lab.openvex.json` | PyYAMLの脆弱性 CVE-2020-1747 を「影響なし」とするVEX |
| `.github/workflows/sbom.yml` | PRごとに自動でSBOMを作り、脆弱性をチェックする設定 |

## 準備

macOSの場合：

```bash
brew install syft grype trivy vexctl
```

Dependency-Trackを使う場合は、Dockerも必要です（メモリ8GB以上を推奨）。

確認したバージョン：Syft 1.57.2、Grype 0.122.1、Trivy 0.76.2、vexctl 0.4.4、Dependency-Track 5.2.1

## 手元で試す

### Step 1：SBOMを作る

```bash
syft dir:. \
  --source-name sbom-lab --source-version 1.0.0 \
  -o cyclonedx-json=sbom.cdx.json \
  -o spdx-json=sbom.spdx.json \
  -o table
```

**ポイント**：`--source-name` と `--source-version` で、何の製品のSBOMかを必ず書きます。

### Step 2：SBOMの中身を確かめる

```bash
jq -r '.components[] | [.name, .version, .purl] | @tsv' sbom.cdx.json
```

**わかること**：`requirements.txt` から作ったSBOMには、名前・バージョン・purlしか入っていません。ライセンス、供給者、依存関係は空です。SBOMは作って終わりではなく、何が欠けているかを確かめる必要があります。

### Step 3：脆弱性を探す

```bash
grype sbom:sbom.cdx.json
trivy sbom sbom.cdx.json
```

**わかること**：

- どちらも28件見つかります
- ただし、Grypeは `GHSA-...`、Trivyは `CVE-...` というIDで表示するので、そのまま比べると全部違うように見えます
- `grype sbom:sbom.cdx.json --by-cve` にすると、Trivyと完全に一致します

### Step 4：「影響なし」をVEXに記録する

PyYAML 5.3 には、`full_load` で危険なYAMLを読むとコードを実行されてしまう脆弱性（CVE-2020-1747）があります。でも `app.py` は安全な `safe_load` しか使っていないので、「影響なし」と言えます。

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

grype sbom:sbom.cdx.json --vex sbom-lab.openvex.json --show-suppressed
trivy sbom --vex sbom-lab.openvex.json --show-suppressed sbom.cdx.json
```

**わかること**：CVE-2020-1747 が「suppressed（抑制）」になり、28件 → 27件になります。

### Step 5：Dependency-Trackで見張り続ける

GrypeやTrivyは「今この瞬間」のチェックです。Dependency-TrackにSBOMを登録しておくと、後から新しい脆弱性が見つかったときにも気づけます。

```bash
mkdir dtrack && cd dtrack
curl -LO https://dependencytrack.org/docker-compose.yml
docker compose up -d
```

- 画面：http://localhost:8081
- API：http://localhost:8080

画面でチームを作り、`BOM_UPLOAD` と `PROJECT_CREATION_UPLOAD` の権限を付けてAPIキーを発行します。そのキーでSBOMを登録します。

```bash
curl -X POST "http://localhost:8080/api/v1/bom" \
  -H "X-Api-Key: ${DT_API_KEY}" \
  -F "autoCreate=true" \
  -F "projectName=sbom-lab" \
  -F "projectVersion=1.0.0" \
  -F "bom=@sbom.cdx.json"
```

**つまずきやすい点**：初期設定のままだと、Flask、Jinja2、PyYAMLの脆弱性が見つかりません。次の設定が必要です。

| 設定 | 場所 | 変更前 → 変更後 |
| --- | --- | --- |
| OSVを有効にする | Administration → Vulnerability Sources → OSV → Enabled | 16件（NVDだけ） → 72行 |
| 別名をまとめる | 同じ画面の Alias Synchronization | 72行（同じ脆弱性を別IDで重複） → **28件** |

OSVのデータは1日1回取り込まれます。すぐ反映したい場合は、時間を置いてからSBOMを登録し直してください。

## Step 6：GitHub Actionsで自動チェック

`.github/workflows/sbom.yml` は、次のことを自動で行います。

| いつ | 何をする |
| --- | --- |
| Pull Requestを作ったとき | SBOMを作る → 脆弱性をチェック（VEXも反映）→ High以上があれば失敗 |
| `v*` タグをpushしたとき | 上に加えて、SBOMとVEXを保管 |
| 手動 | Actionsタブの「Run workflow」 |

### チェックが失敗するのは正しい動作です

このリポジトリは脆弱なバージョンをわざと使っているので、**チェックは必ず失敗します**。失敗することで、「危ない部品があったらマージを止める」仕組みが効いていることを確かめられます。

実際の結果（[PR #1](https://github.com/muffin-jp/sbom-sca-hands-on/pull/1)）：

- SBOMの生成：成功
- 脆弱性チェック：27件を検出（VEXで CVE-2020-1747 を抑制済み）
- High以上（例：PyYAMLの CVE-2020-14343、Critical）があるため失敗

チェックを通したい場合は、`requirements.txt` のバージョンを、結果の「FIXED IN」の列以上に上げてください。

### 注意

- SBOMファイル（`sbom*.cdx.json` など）はコミットしないでください。Syftがそれも読み込んでしまい、部品が二重に数えられます
- アクションが使うGrypeのバージョン（0.110.0）は、手元のバージョンと違うことがあります。結果がずれたら、まずバージョンを確認してください

## 元の手順書から直したところ

元の手順書どおりに進めると、うまくいかない点がありました。

| Step | 手順書の問題 | 直し方 |
| --- | --- | --- |
| 3 | CVE-2020-14343（Critical）の扱いが書かれていない | これは「CVE-2020-1747 の修正が不完全だった」もの。`safe_load` だけなら、これも「影響なし」と言えるが、今回はVEXに入れずに残した。**VEXは脆弱性ごとに判断が必要**、という練習に使える |
| 4 | CVE IDだけでVEXを作ると、Grypeで効かない | Grypeは `GHSA-6757-jp84-gxfx` というIDで照合するため、CVE IDだけのVEXは**警告なしで無視**される。`--aliases` でGHSAのIDも入れる（または `grype --by-cve` を使う） |
| 5 | ポート番号が逆 | Dependency-Track v5 では **APIが8080、画面が8081**（手順書は4.x系の8081／8080）。手順書の `curl …:8081/api/v1/bom` は画面側に届いて失敗する |
| 6 | アクションのバージョンが古い | checkout、scan-action、upload-artifact を **v7** に更新。scan-actionの `vex:` 入力でVEXを反映。sbom-actionは何もしないとSBOMを毎回保管するので、タグのときだけ保管したいなら `upload-artifact: false` を付ける |

## わかったこと

1. **IDをそろえてから比べる**：同じ脆弱性でも、ツールによって `CVE`、`GHSA`、`PYSEC` など違うIDで表示される
2. **VEXはIDが合わないと黙って無視される**：エラーにならないので気づきにくい
3. **ツールの設定で結果が変わる**：Dependency-Trackは、設定次第で16件にも72行にも28件にも見えた
4. **ツールのバージョンで手順が変わる**：ポート番号やアクションのバージョンなど、手順書は使ったバージョンを書いておく

## 参考資料

- [Syft](https://github.com/anchore/syft)、[Grype](https://github.com/anchore/grype)、[Trivy](https://github.com/aquasecurity/trivy)
- [vexctl（OpenVEX）](https://github.com/openvex/vexctl)
- [Dependency-Track ドキュメント](https://docs.dependencytrack.org/)
- [anchore/sbom-action](https://github.com/anchore/sbom-action)、[anchore/scan-action](https://github.com/anchore/scan-action)

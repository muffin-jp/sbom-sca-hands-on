# sbom-sca-hands-on

SBOMとSCA（ソフトウェア構成分析）を、無料のOSSツールで一通り体験するための練習用リポジトリです。

「SBOMを作る → 脆弱性を照合する → 影響なしをVEXで記録する → 継続管理する → CIで自動化する」の流れを、手元とGitHub Actionsで再現します。

> **注意**：`requirements.txt` には、既知の脆弱性があるバージョンを**わざと**指定しています。実際にインストールしたり、実行したりしないでください。

## 中身

| ファイル | 役割 |
| --- | --- |
| `requirements.txt` | 古いバージョンを指定した5つの依存（requests、urllib3、PyYAML、Jinja2、Flask） |
| `app.py` | PyYAMLを `safe_load` だけで使うコード。VEXで「影響なし」とする根拠 |
| `sbom-lab.openvex.json` | PyYAMLの CVE-2020-1747 を `not_affected` とするOpenVEX |
| `.github/workflows/sbom.yml` | PRごとにSBOM生成とGrypeでの照合を行うワークフロー |

## 使うツール

| 役割 | ツール | 確認したバージョン |
| --- | --- | --- |
| SBOM生成 | Syft | 1.57.2 |
| 脆弱性照合 | Grype、Trivy | 0.122.1、0.76.2 |
| VEX作成 | vexctl（OpenVEX） | 0.4.4 |
| 継続管理 | Dependency-Track | 5.2.1 |
| CI | GitHub Actions | sbom-action v0、scan-action v7 |

macOSなら次のコマンドで入ります。

```bash
brew install syft grype trivy vexctl
```

## 手元で試す

### 1. SBOMを作る

```bash
syft dir:. \
  --source-name sbom-lab --source-version 1.0.0 \
  -o cyclonedx-json=sbom.cdx.json \
  -o spdx-json=sbom.spdx.json \
  -o table
```

`requirements.txt` から作ったSBOMには、直接依存の名前・バージョン・purlしか入りません。供給者、ライセンス、依存関係は空になります。

### 2. 脆弱性を照合する

```bash
grype sbom:sbom.cdx.json
trivy sbom sbom.cdx.json
```

どちらも28件を検出します。ただし、Grypeは主にGHSA、TrivyはCVEでIDを表示するため、そのまま比べると全部ずれて見えます。`grype --by-cve` を使うと、Trivyと完全に一致します。

### 3. VEXを反映する

```bash
grype sbom:sbom.cdx.json --vex sbom-lab.openvex.json --show-suppressed
trivy sbom --vex sbom-lab.openvex.json --show-suppressed sbom.cdx.json
```

どちらも CVE-2020-1747 が抑制され、27件になります。

VEXには、CVE IDに加えてGHSAのalias（`GHSA-6757-jp84-gxfx`）を入れています。CVE IDだけだと、Trivyでは効きますが、GHSAで照合するGrypeでは**警告なしで無視**されます。

### 4. Dependency-Trackで継続管理する

```bash
mkdir dtrack && cd dtrack
curl -LO https://dependencytrack.org/docker-compose.yml
docker compose up -d
```

v5系では、**APIが http://localhost:8080、画面が http://localhost:8081** です（4.x系とは逆）。SBOMの登録は次のとおりです。

```bash
curl -X POST "http://localhost:8080/api/v1/bom" \
  -H "X-Api-Key: ${DT_API_KEY}" \
  -F "autoCreate=true" \
  -F "projectName=sbom-lab" \
  -F "projectVersion=1.0.0" \
  -F "bom=@sbom.cdx.json"
```

APIキーは、`BOM_UPLOAD` と `PROJECT_CREATION_UPLOAD` の権限だけを付けたチームで発行します。

初期設定では、脆弱性ソースは**NVDだけが有効**です。NVDはCPEで照合するため、このリポジトリではFlask、Jinja2、PyYAMLの脆弱性が見つかりません。Administration → Vulnerability Sources → OSV を有効にし、さらに **Alias Synchronization** も有効にすると、Grypeと同じ28件（PyYAML 6.0.2に上げた版では26件）になります。

## GitHub Actions

`.github/workflows/sbom.yml` は、次のタイミングで動きます。

| タイミング | 内容 |
| --- | --- |
| Pull Request | SBOM生成 → Grypeで照合（VEXを反映）→ High以上があれば失敗 |
| `v*` タグの push | 上に加えて、SBOMとVEXをArtifactとして保管 |
| 手動実行 | Actionsタブの「Run workflow」 |

このリポジトリは脆弱なバージョンをわざと使っているので、**`sbom` チェックは失敗するのが正しい動作**です。ゲートが効いていることの確認になります。チェックを通したい場合は、`requirements.txt` のバージョンを、照合結果の「FIXED IN」以上に上げてください。

SBOMファイル（`sbom*.cdx.json` など）はコミットしないでください。Syftがリポジトリ内のSBOMファイルも読み込むため、部品が二重に数えられます。

## わかったこと

- **IDをそろえてから比べる**：同じ脆弱性でも、GHSA、CVE、PYSEC（OSV）など、ツールやソースによって表示されるIDが違う
- **VEXはIDが一致しないと黙って無視される**：取り込む側のツールが使うIDをaliasに入れる
- **ツールの設定で結果が変わる**：Dependency-Trackは、脆弱性ソースとalias同期の設定で、16件 → 72行 → 28件と見え方が変わった
- **ツールのバージョンで手順が変わる**：Dependency-Trackのポートや、GitHub Actionsのメジャーバージョンなど

## 参考資料

- [Syft](https://github.com/anchore/syft)、[Grype](https://github.com/anchore/grype)、[Trivy](https://github.com/aquasecurity/trivy)
- [vexctl（OpenVEX）](https://github.com/openvex/vexctl)
- [Dependency-Track ドキュメント](https://docs.dependencytrack.org/)
- [anchore/sbom-action](https://github.com/anchore/sbom-action)、[anchore/scan-action](https://github.com/anchore/scan-action)

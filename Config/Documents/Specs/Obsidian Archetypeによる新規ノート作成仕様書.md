# Obsidian Archetypeによる新規ノート作成仕様書
## 1. システム概要と前提

### 1.1 目的
新規ファイル作成時、ファイル名に基づき最適なArchetypeを自動選択し、多段継承、ルール評価、およびFront Matterの`settings`キー下にネストされた設定定義に従ってFront Matterと本文を完全に再構築・挿入する。

### 1.2 処理原則
* **設定のネスト**: システム設定値はすべて、定義ファイル（`Settings.md`、`Archetypes.md`など）のFront Matter内の**`settings`キー**下にネストされて格納される。
* **呼び出し起点**: すべての新規ファイル作成時、最初に**`Archetypes.md`ファイルが呼び出され**、システムロジックの中央ハブとして動作する。
* **Front Matter**: Archetypeファイルの既存Front Matterは**参照しない**。新規ファイルの初期値は**適用された `rules[].value` または `rules[].script` の実行結果のみ**をソースとする。
* **本文**: `type: Archetype` ファイルのFront Matterに続く部分（本文）は、新規作成されるファイルの本文テンプレートとして使用される。本文の動的な値は、**TemplaterJSのプレースホルダー記法**（例: `<% tp.file.title %>`, `<% tp.date.now() %>`）で定義される。
* **本文の継承制御**: `Archetype`ファイルに定義された`includeParentBody`プロパティに基づき、親の本文テンプレートのインクルードを制御する。
* **継承パス**: プロパティ収集ロジックは、常に**祖先（`Archetypes`）から子孫（新規ファイル）の順**に処理を実行する。
* **競合解決**: 同一キーを持つファイルの定義は、後から読み込まれた定義が**上書き**する（後勝ちルール）。
* **Settingsの役割**: `Settings`ファイルは**究極のカスタマイズ層**であり、値がなければ継承ツリーを通じて`Archetypes`へフォールバックする。

### 1.3 継承階層の全体像

すべてのファイルタイプは、究極的に`Archetypes`に収束する。

| ルート | 階層 | Type名 | Extend (親) |
| :--- | :--- | :--- | :--- |
| **システム定義** | L3 $\rightarrow$ L2 $\rightarrow$ L1 | `Settings`, `Property`, `Archetype` | `Config` |
| | L2 $\rightarrow$ L1 | `Config` | `Archetypes` |
| **コンテンツ** | L4+ $\rightarrow$ L3 $\rightarrow$ L2 $\rightarrow$ L1 | `ブログ記事`, `DailyNote` など | `Entry`, `Periodic` など |
| | L3 $\rightarrow$ L2 $\rightarrow$ L1 | `Entry`, `Periodic` | `Post` |
| | L2 $\rightarrow$ L1 | `Post` | `Archetypes` |
| **頂点** | L1 | **`Archetypes`** | (なし) |

### 1.4 想定されるディレクトリツリー構造

| パス | ファイル名 (例) | Type | 役割 |
| :--- | :--- | :--- | :--- |
| `Config/Archetypes/` | **`Archetypes.md`** | (頂点) | システム全体の究極の抽象基底。 |
| `Config/Settings/` | `Settings.md` | `settings` | システム設定のカスタマイズ層。 |
| `Config/Properties/` | `Property.md` | `Property` | プロパティ定義。 |
| **`Config/Cache/`** | **`Cache.json`** | **(キャッシュ)** | **システム設定と統合データの高速読み込み用。** |

---

## 2. データ構造（Front Matter スキーマ）

### 2.1 Property定義ファイル (`type: Property`)

| プロパティ | データ型 | 説明 | 備考 |
| :--- | :--- | :--- | :--- |
| `parent` | string | 親となるプロパティのkey。**システム設定プロパティの場合は**`settings`**を設定する。** | |
| `rules[].value` | 任意 | **初期値**。`script`が未定義の場合に採用。 | |
| **`rules[].script`** | **string** | **初期値を決定するために実行されるTemplaterJSスクリプト**。`value`よりも優先して評価される。 | |
| `rules[].weight` | number | 0 以上の数値。作成されたファイル内でのプロパティの**並び順**を制御する（低いほど優先）。 | |

### 2.2 Archetype定義ファイル (`type: Archetype`)

| プロパティ | データ型 | 説明 |
| :--- | :--- | :--- |
| `key` | string | Archetypeの一意の識別子。 |
| `extend` | string | 継承元となる親 Archetype のキー名。 |
| `format` | string | この Archetype に合致する命名規則（Moment.jsトークン、Templater変数を含む）。 |
| `weight` | number | 0 以上の数値。テンプレート自動選択時の優先度（低いほど優先）。 |
| **`includeParentBody`** | **boolean** | **親 Archetype の本文を自動的にインクルードするか**を制御する。デフォルトは`true`。 |

---

## 3. 処理フロー（TemplaterJSスクリプト）

### Step 0: 初期設定とデータ定義の取得（キャッシュの最大活用）

このステップの目的は、`AllDefinitions`（全ての統合済み定義情報）を高速かつ正確に確定することである。

1.  **キャッシュファイルのパス定義**:
    * **キャッシュファイル**: `Config/Cache/Cache.json`（形式は**JSON**）とする。

2.  **キャッシュの有無と整合性チェック**:
    * `Cache.json`を読み込み、格納されている**`hash`**値と、**すべての定義ファイル**（`Settings.md`、`Archetypes.md`、全`Property.md`など）の現在の**複合ハッシュ値**を比較する。
    * **有効な場合（ハッシュが一致）**: キャッシュされた`SystemConfig`（統合済み設定値）と`AllDefinitions`（統合済みArchetype/Property定義）を採用する。
    * **無効な場合（ハッシュが不一致またはファイルなし）**:
        * すべての定義ファイルをフル走査し、`Settings` $\rightarrow$ `Config` $\rightarrow$ `Archetypes`の順に競合解決を適用して`SystemConfig`を確定する。
        * 確定した設定に基づき、すべての定義ファイルを収集・統合し、`AllDefinitions`を構築する。
        * **統合データと新しい複合ハッシュ値**を`Config/Cache/Cache.json`にJSON形式で永続化する。

3.  **フロー制御（修正点）**: このステップの完了後、処理フローはキャッシュの有効性に関わらず、**次の動的処理である Step 1 へ進む**。

### Step 1: テンプレートの自動選択

1.  **メタデータの利用**: `AllDefinitions`から、全 Archetype の`format`、`weight`、日付フラグを読み込む。
2.  **自動選択（日付系優先）**:
    * **日付系 Archetype**: 新規ファイル名が`format`パターンに**完全に一致**する場合、そのArchetypeを**自動的に確定**する（最優先）。
    * **一般系 Archetype**: 日付系で一致がない場合、残りの候補の中から、`format`との**完全一致**をもって自動的に確定する。
3.  **ユーザー選択**: 上記のいずれにも一致しなかった場合、残りの**一般系 Archetype**を`weight`などに基づきソートし、ユーザー選択により確定する。

### Step 2: 継承ツリーの構築とプロパティの初期値決定

1.  **継承ツリー構築**: 確定 Archetype の `extend` を再帰的に辿り、**`Archetypes`を頂点とする完全な継承ツリー**を構築する。
2.  **プロパティの複合収集とルール評価**:
    * 構築された継承ツリーを**祖先から子孫の順**にループする。
    * **ルール伝播**: 各階層で、全 Property 定義の `rules` を評価し、`rules[].inherit: true` の定義を子孫に伝播させる。
    * **初期値決定**: `rules[].script`があればそれを実行し、次に`rules[].value`を採用する。

### Step 3: データ型検証と Front Matter の構築

1.  **検証・制御**: 収集したプロパティに対し、`Property.dataType`検証と`displayPolicy: "conditional"`による空値の挿入制御を行う。
2.  **Front Matterのネスト構築**: `Property.parent`情報に基づき、ネストされたYAMLオブジェクトとして構造を構築する。
3.  **並び順**: 構築されたFront Matterのキーは、`Property.rules[].weight`の低い順にソートする。

### Step 4: 本文テンプレートの挿入と解決

1.  **本文の抽出とインクルード**: 確定 Archetype から親 Archetype に遡り、`includeParentBody`制御に基づいて本文を結合する。
2.  **プレースホルダーの解決**: 抽出・結合された本文全体に対し、TemplaterJSのAPIを使用してすべての**プレースホルダー**（`<%...%>`）を解決し、動的な値に置き換える。
3.  **ファイル出力**: **Step 3で構築されたFront Matter**と**解決後の本文**を結合し、最終的なファイルとして新規作成する。
# Bottled AI を fork 先から動かす手順

## 1. この資料の対象

Bottled AI は、**Slay the Spire** を外部 Python プロセスから自動操作する bot です。[1] 本体だけを起動してもゲームとは通信できないため、ゲーム本体、ModTheSpire、BaseMod、StSLib、Communication Mod を同じ環境に用意します。[1] [2] [3] [4] [5]

この資料では、fork したリポジトリを Slay the Spire のインストール先へ配置し、Communication Mod から `main.py` を起動するまでを説明します。対象 OS は Windows と macOS です。

> **重要:** リポジトリの配置場所は、任意の作業ディレクトリではなく、Slay the Spire のゲームインストール先にある `bottled_ai` フォルダです。Communication Mod の設定で相対パスを使うため、この場所を前提にしています。

## 2. 必要なもの

| 項目 | 必要条件・用途 |
| --- | --- |
| Slay the Spire | Steam 版のゲーム本体。bot の操作対象です。 |
| Python | **3.11.8 以上**。Windows では PATH に追加し、macOS では Xcode 経由の Python 3.11 以上を使用します。[1] |
| pip | Python パッケージマネージャー。リポジトリの `requirements.txt` は現在空ですが、Python の実行環境確認に使用できます。 |
| BaseMod | Slay the Spire の mod 基盤です。[2] |
| StSLib | Bottled AI が利用するライブラリ mod です。[3] |
| ModTheSpire | mod を有効化してゲームを起動するローダーです。[4] |
| Communication Mod | ゲームと外部 Python プロセスを接続します。[5] |

## 3. Python を確認する

ターミナルまたは PowerShell で次のコマンドを実行します。

```bash
python --version
pip --version
```

`python` が見つからない場合は、環境に応じて次を試します。

```bash
# macOS / Linux 系
python3 --version
python3 -m pip --version
```

Python のバージョンが 3.11.8 未満の場合は、先に Python を更新してください。Windows では Python のインストーラーで **Add Python to PATH** を有効にするか、インストール後に PATH を設定します。

## 4. fork をゲームフォルダへ配置する

### Windows

Steam で Slay the Spire のプロパティを開き、**インストール済みファイル → 参照**を選択してゲームのインストールフォルダを開きます。通常は次のような場所です。

```text
E:\Steam\steamapps\common\SlayTheSpire
```

そのフォルダの中で、fork を `bottled_ai` という名前で clone します。`YOUR_GITHUB_NAME` は fork を作成した GitHub アカウント名に置き換えてください。

```powershell
cd "E:\Steam\steamapps\common\SlayTheSpire"
git clone https://github.com/YOUR_GITHUB_NAME/bottled_ai.git bottled_ai
cd bottled_ai
```

### macOS

Steam のライブラリで Slay the Spire を右クリックし、**ローカルファイルを閲覧**または **パッケージの内容を表示**を選択します。README で案内されている `Resources` 配下をゲームの実行環境として使用し、その中に `bottled_ai` を配置します。[1]

```bash
cd "/path/to/SlayTheSpire/Contents/Resources"
git clone https://github.com/YOUR_GITHUB_NAME/bottled_ai.git bottled_ai
cd bottled_ai
```

clone 後、最低限次のファイルが存在することを確認します。

```text
bottled_ai/
├── main.py
├── requirements.txt
├── run_controller.txt
├── presentation_config.py
├── rs/
├── tests/
└── docs/
```

## 5. Steam Workshop の mod を導入する

Steam Workshop で次の 4 つを購読します。各 mod の導入元は本家 README に記載されたリンクです。[1]

| Mod | Workshop |
| --- | --- |
| BaseMod | [Steam Workshop の BaseMod](https://steamcommunity.com/sharedfiles/filedetails/?id=1605833019) [2] |
| StSLib | [Steam Workshop の StSLib](https://steamcommunity.com/sharedfiles/filedetails/?id=1609158507) [3] |
| ModTheSpire | [Steam Workshop の ModTheSpire](https://steamcommunity.com/sharedfiles/filedetails/?id=1605060445) [4] |
| Communication Mod | [Steam Workshop の Communication Mod](https://steamcommunity.com/sharedfiles/filedetails/?id=2131373661) [5] |

すべてを購読した後、ModTheSpire からゲームを一度起動し、上記 mod を有効にします。この初回起動で、Communication Mod の設定ファイルが生成されます。設定ファイルが作られる前に `config.properties` を編集しようとすると、対象ファイルが見つからないことがあります。

## 6. Communication Mod の設定を変更する

ゲームを mod 有効状態で一度起動した後、Communication Mod の設定フォルダに移動します。

| OS | 設定フォルダ |
| --- | --- |
| Windows | `%LOCALAPPDATA%\ModTheSpire\CommunicationMod\` |
| macOS | `~/Library/Preferences/ModTheSpire/CommunicationMod/` |

このフォルダ内の `config.properties` をテキストエディタで開き、`command` の設定を次のようにします。既存の `command=` 行がある場合は、同じキーを重複させず置き換えてください。ModTheSpire 直下の `config.properties` ではなく、`CommunicationMod` サブディレクトリ内のファイルを編集します。

### Windows

```properties
command=python ./bottled_ai/main.py
```

### macOS

```properties
command=python3 ./bottled_ai/main.py
```

設定値の相対パスは、README の手順どおりゲームのインストール先に `bottled_ai` を配置した場合に対応します。[1] Python の実体が `python3` ではなく別の場所にある場合は、Communication Mod から起動できる Python の実行ファイル名または絶対パスへ変更してください。

## 7. bot を起動する

1. ModTheSpire で Slay the Spire を起動します。
2. ゲームのメインメニューから **Mods** を選択します。
3. **Communication Mod** を選択します。
4. **Config**（Return の隣）を開きます。
5. **Start external process** を選択します。[1]

起動すると `main.py` が `PEACEFUL_PUMMELING` 戦略を使用し、`run_amount = 1` の設定でランダム seed のゲームを 1 回開始します。これは現在のリポジトリにある `main.py` の設定に基づく挙動です。特定の seed を使う場合や、実行回数・戦略を変える場合は、`main.py` の `run_seeds`、`run_amount`、`strategy` を編集してください。

## 8. 起動できない場合の確認順

Communication Mod の外部プロセスには 10 秒のタイムアウトがあります。10 秒程度待っても何も起きない場合は、次の順に確認します。

| 確認箇所 | 確認内容 |
| --- | --- |
| フォルダ配置 | `main.py` がゲームインストール先の `bottled_ai` 直下にあるか確認します。 |
| Python | ターミナルで `python --version` または `python3 --version` を実行し、3.11.8 以上か確認します。 |
| mod | BaseMod、StSLib、ModTheSpire、Communication Mod が有効か確認します。 |
| 設定ファイル | `config.properties` に正しい OS 用の `command=` が 1 行だけ存在するか確認します。 |
| 起動方法 | 通常起動ではなく、Mods → Communication Mod → Config → Start external process の順で起動します。 |
| ログ | ModTheSpire のコンソールと、Slay the Spire フォルダ内の `communication_mod_errors.log` を確認します。[1] |

設定変更後は、ゲームと外部プロセスをいったん終了してから再起動してください。特に `config.properties` のパス区切り文字、`python` / `python3` の名前、clone 先のフォルダ名 `bottled_ai` は、起動失敗の原因になりやすい箇所です。

## 9. コード変更前のローカル確認

ゲームを起動せずに、Python ファイルの構文だけを確認するには次を実行します。

```bash
python -m compileall -q main.py rs tests
```

リポジトリには `tests/` 以下に pytest 用のテストが含まれています。環境に pytest がない場合は、次のように開発用仮想環境へ導入して実行できます。

```bash
# Windows PowerShell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install -U pip pytest
python -m pytest

# macOS
python3 -m venv .venv
source .venv/bin/activate
python3 -m pip install -U pip pytest
python3 -m pytest
```

なお、ゲームとの通信を伴う `main.py` の実行は、Slay the Spire と必要な mod が起動している状態で行ってください。ゲームなしで直接 `python main.py` を実行しても、Communication Mod との通信接続がないため、通常の bot ランとしては成立しません。

## 10. 参考資料

[1]: https://github.com/tonbiattack/bottled_ai/blob/main/README.md "Bottled AI README"
[2]: https://steamcommunity.com/sharedfiles/filedetails/?id=1605833019 "BaseMod - Steam Workshop"
[3]: https://steamcommunity.com/sharedfiles/filedetails/?id=1609158507 "StSLib - Steam Workshop"
[4]: https://steamcommunity.com/sharedfiles/filedetails/?id=1605060445 "ModTheSpire - Steam Workshop"
[5]: https://steamcommunity.com/sharedfiles/filedetails/?id=2131373661 "Communication Mod - Steam Workshop"

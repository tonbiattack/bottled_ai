# Windows ローカル起動の作業記録

## 対象と結果

2026-08-25 に Windows 環境で Bottled AI を Slay the Spire と接続するためのローカル設定を行った記録です。ゲームは ModTheSpire 経由で起動し、BaseMod、StSLib、Communication Mod がすべて読み込まれることをログで確認しました。

この記録は特定の PC の実施結果です。ゲームのインストール先や Python コマンド名が異なる場合は、値を自分の環境に合わせて読み替えてください。

## 実施した設定

### 1. リポジトリの配置

次のゲームインストール先にリポジトリを clone しました。

```text
C:\Program Files (x86)\Steam\steamapps\common\SlayTheSpire\bottled_ai
```

`bottled_ai` はゲームフォルダ直下に置く必要があります。Communication Mod の起動コマンドが相対パス `./bottled_ai/main.py` を使用するためです。

### 2. Python の確認

ローカルの `python --version` は 3.13.14 でした。README の最低要件（3.11.8 以上）を満たしています。

テスト実行用にユーザー領域へ pytest を導入しました。

```powershell
python -m pip install --user pytest
```

アプリケーション自体の `requirements.txt` は空です。pytest はテスト専用です。

### 3. Steam Workshop mod の確認と配置

次の Workshop ID のファイルがローカルに存在することを確認しました。

| Mod | Workshop ID |
| --- | --- |
| ModTheSpire | `1605060445` |
| BaseMod | `1605833019` |
| StSLib | `1609158507` |
| Communication Mod | `2131373661` |

ModTheSpire は Steam と接続できず Workshop の自動列挙に失敗しました。そのため、ゲームフォルダ内の `mods` ディレクトリへ、既にダウンロード済みの次の JAR を配置しました。

```text
mods/
├── BaseMod.jar
├── StSLib.jar
└── CommunicationMod.jar
```

`%LOCALAPPDATA%\ModTheSpire\mod_lists.json` の既定リストにも上記 3 ファイルを登録しました。ModTheSpire 自体は起動用の JAR であり、mod の選択対象には含めません。

### 4. Communication Mod の外部プロセス設定

`%LOCALAPPDATA%\ModTheSpire\config.properties` に次の 1 行を設定しました。

```properties
command=python .\\bottled_ai\\main.py
```

この設定により、ゲームのメニューから Communication Mod の外部プロセスを開始すると、ゲームフォルダを基準に `bottled_ai/main.py` が起動します。

## 検証結果

### 構文と単体テスト

全 Python ファイルのコンパイルに成功しました。

```powershell
python -m compileall -q .
```

テストは、テスト用パッケージを優先して解決する必要があります。この環境では次のコマンドで実行しました。

```powershell
$repo = 'C:\Program Files (x86)\Steam\steamapps\common\SlayTheSpire\bottled_ai'
$env:PYTHONPATH = "$repo\tests;$repo"
python -m pytest tests -q
```

結果は **1157 passed, 1 skipped** でした。

> `python -m pytest` だけで実行すると、`tests/` と `rs/` のどちらの `ai` パッケージを先に解決するかが曖昧になり、テスト用フィクスチャの import に失敗します。上記の `PYTHONPATH` を使ってください。

### Mod を読み込んだゲーム起動

次のコマンドで ModTheSpire をランチャー画面なしで起動しました。

```powershell
$game = 'C:\Program Files (x86)\Steam\steamapps\common\SlayTheSpire'
$java = Join-Path $game 'jre\bin\java.exe'
$mts = 'C:\Program Files (x86)\Steam\steamapps\workshop\content\646570\1605060445\ModTheSpire.jar'
& $java -jar $mts --skip-launcher
```

起動ログでは次を確認しました。

- Mod list: `basemod (5.56.0)`、`stslib (2.12.0)`、`CommunicationMod (1.2.1)`
- `Initializing mods...` が完了
- `Starting game...` の後、`registerModBadge : Communication Mod` を出力

ログはゲームフォルダの `sendToDevs\logs\SlayTheSpire.log` にあります。

## bot を実際に開始する操作

ゲームがメインメニューまで表示された後、次の順で操作します。

1. **Mods** を選ぶ。
2. **Communication Mod** を選ぶ。
3. **Config** を開く。
4. **Start external process** を選ぶ。

この操作はゲーム UI 上で行う必要があります。外部プロセスはゲームとの標準入出力接続を介して通信するため、ゲームなしで `python main.py` だけを直接実行しても bot のランは開始できません。

起動に失敗した場合は、Communication Mod の 10 秒タイムアウト後に、ゲームフォルダの `communication_mod_errors.log` と ModTheSpire のコンソール出力を確認してください。

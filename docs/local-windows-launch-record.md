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

Communication Mod が読む設定ファイルは、ModTheSpire 直下ではなく、次のサブディレクトリ内です。

```text
%LOCALAPPDATA%\ModTheSpire\CommunicationMod\config.properties
```

このファイルに次の 1 行を設定します。

```properties
command=python ./bottled_ai/main.py
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

### 自動で開始する設定

ゲームの開始と同時に bot も開始する場合は、`%LOCALAPPDATA%\ModTheSpire\CommunicationMod\config.properties` を次の内容にします。

```properties
command=python ./bottled_ai/main.py
runAtGameStart=true
```

`runAtGameStart=false` に戻すと、前節の手動開始へ戻ります。変更後はゲームを再起動します。

### 自動開始の実行確認（2026-08-25）

上記の自動開始設定で ModTheSpire を再起動し、次を確認しました。

- Communication Mod のログに `Received message from external process: ready` が出力された。
- 子プロセスとして `python ./bottled_ai/main.py` が継続実行された。
- `bottled_ai/logs/default.log` に `Starting up`、`start Watcher ...`、ゲーム状態の応答が記録された。

この3点がそろえば、外部プロセスの起動だけでなく、bot とゲームの標準入出力による接続も成立しています。

### 日本語表示での文字化けと終了

ゲーム表示が日本語（`LANGUAGE: JPN`）の環境では、Communication Mod から Python へ渡されるカード名が文字化けすることがあります。文字化けした名前を `choose` コマンドへ渡すと、Communication Mod は無効な引数としてエラー応答を返します。

この環境では `preferences/STSGameplaySettings` とそのバックアップの `LANGUAGE` を `ENG` にしてゲームを再起動し、英語のカード名で通信するようにしました。表示言語の変更は、この bot の名前ベースの判断・コマンド送信を安定させるための回避策です。

また、bot は Communication Mod が返すエラー応答を通常のゲーム状態として扱わず、エラーメッセージを含む例外として記録するようにしています。これにより、`in_game` キーがないエラー応答で `KeyError` になるのを防ぎ、停止理由をログから確認できます。

## 戦略を試すときのメモ

### Ironclad: `REQUESTED_STRIKE`

2026-08-25 の試行では、Ironclad 用の `REQUESTED_STRIKE` は防御を優先するように見え、通常のプレイでは弱いという所感になりました。この評価は単発の試行に基づくもので、勝率を測定して確定した結論ではありません。

実装を確認すると、この戦略はキャラクターとして `IRONCLAD` を指定し、デッキから取り除く優先順位の先頭に `defend` と `defend+` を置いています。戦闘の選択は共通の戦闘ハンドラとカード・イベント・ショップなどの戦略固有ハンドラを組み合わせて行うため、「防御優先」の原因をこの設定だけに帰すことはできません。

したがって、Ironclad が弱いという仮説を検証するには、同一バージョン・同一条件で複数 seed を実行し、勝率、到達 Act、被ダメージ、ターン数を記録して比較します。試用目的では、別戦略へ切り替えて挙動を比較するのが安全です。

### Silent: `SHIVS_AND_GIGGLES`

Ironclad の試行後、`main.py` の `strategy` を `SHIVS_AND_GIGGLES` に変更して Silent を試すことにしました。戦略を変えた場合は、外部 Python プロセスを再起動する必要があります。`runAtGameStart=true` を使っている場合は、ModTheSpire 経由でゲームを再起動します。

## 起動失敗の調査記録（2026-08-25）

### 症状

ゲーム内の **Start external process** を複数回実行しても bot は開始せず、ゲームログには次のエラーだけが記録されました。

```text
ERROR communicationmod.CommunicationMod> Could not start external process.
```

`communication_mod_errors.log` は 0 バイトで、`bottled_ai/logs/` にも bot の実行ログは作られていませんでした。

### 確認した事実

1. ModTheSpire の起動ログでは BaseMod、StSLib、Communication Mod の読み込み完了を確認しました。したがって mod 未読み込みは原因ではありません。
2. Python の構文チェックとテストは成功しています。bot が起動してから例外になった証拠もありません。
3. Communication Mod の JAR を確認すると、`SpireConfig("CommunicationMod", "config", ...)` で設定を開き、取得した `command` を空白で分割して `ProcessBuilder` に渡しています。
4. 実際に生成されていた `%LOCALAPPDATA%\ModTheSpire\CommunicationMod\config.properties` の内容は次のとおりでした。

   ```properties
   command=
   runAtGameStart=false
   ```

5. 一方で、起動コマンドは誤って `%LOCALAPPDATA%\ModTheSpire\config.properties` に書かれていました。このファイルは Communication Mod の設定としては読まれません。

### 原因

**Communication Mod が参照する `command` が空文字のままでした。** そのため、Python や `main.py` を起動する前に `ProcessBuilder` の生成に失敗し、ゲームログの `Could not start external process.` が出ています。

空のエラーログはこの結論と整合します。Communication Mod はプロセス生成に成功してから子プロセスの標準エラーを `communication_mod_errors.log` へリダイレクトするため、生成前の失敗ではそのファイルに Python の例外は記録されません。

### 対処と再確認手順

1. `%LOCALAPPDATA%\ModTheSpire\CommunicationMod\config.properties` の `command=` を、上記の `command=python ./bottled_ai/main.py` に置き換えます。
2. ゲームを終了してから ModTheSpire 経由で再起動します。設定は Mod の初期化時に読み込まれます。
3. **Mods → Communication Mod → Config → Start external process** を実行します。
4. 成功時は、ゲームログに `Received message from external process: ready` が記録され、`bottled_ai/logs/default.log` が生成されます。
5. それでも失敗する場合は、`communication_mod_errors.log` の新しい内容を確認します。この段階では Python のエラー出力が残るため、次の切り分けに利用できます。

この調査では設定ファイルを変更していません。上記は、記録時点のログと実装を根拠にした修正手順です。

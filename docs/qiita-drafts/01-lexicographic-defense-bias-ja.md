---
title: "Slay the Spire 自動操作AIが防御しすぎた理由――辞書式評価と1ターン先読みの罠"
tags:
  - Python
  - AI
  - ゲームAI
  - SlayTheSpire
  - アルゴリズム
private: false
updated_at: ""
id: ""
organization_url_name: ""
slide: false
ignorePublish: false
---

## はじめに

Slay the Spire を自動操作するボットを動かすと、「敵が攻撃してくるターンにブロックを積みすぎる」「攻撃すれば敵の強化前に倒せるのに、防御を選んで戦闘を伸ばす」という挙動に遭遇した。局所的には数点のHPを守れているのに、戦闘全体では敵の攻撃回数を増やし、最後にはより大きなHP損失になる。これはゲームAIで起きやすい、**短期の安全性を過大評価して長期の戦闘時間コストを見落とす**問題である。

本記事では Bottled AI の実装を読み、なぜこの傾向が Ironclad に限らず複数キャラクターへ出うるのかを分析する。結論から言うと、原因は「防御カードを多く取る設定」ではない。**共通の比較器が即時被ダメージを敵撃破進捗より先に辞書式で評価し、シミュレータが一般には次ターン以後の敵強化を展開しない**ことにある。[1] [2]

> 本記事はコミット `2f1152d` 時点のコード分析である。Ironclad の試行は単発だったため、キャラクター別の勝率差や改善幅を断定しない。ここで扱うのは、観測された挙動を生みうる設計上の因果関係である。

## まず誤解を解く: 「Defendを先に削除する」ことは防御偏重の説明にならない

Ironclad の `REQUESTED_STRIKE` には、カード削除の優先リストで `defend`、`strike`、`defend+`、`strike+` を並べる設定がある。[3] しかしこれは、ショップやイベント等で**デッキから何を取り除くか**の方針である。戦闘中に「この手札では Defend と Strike のどちらをプレイするか」を選ぶコードではない。

戦闘手の選択は、戦略固有の設定より下の共通層が担う。`CommonBattleHandler` は現在の敵・階層・ターンを見て比較器を選び、ほとんどの通常戦では `CommonGeneralComparator` を使う。[1] Ironclad、Silent、Watcher、Claw 系 Defect は基本的にこの共有ハンドラを使うため、そこで生じた評価の偏りはキャラクター横断で現れる。

| 層 | 役割 | 今回の論点 |
| --- | --- | --- |
| 戦略設定 | キャラクター、カード報酬、削除、ショップの方針 | デッキの形には影響するが、手札内のプレイ順を直接決めない |
| 共通戦闘ハンドラ | 比較器選択と探索起動 | 多くのキャラクターで共有される |
| 共通比較器 | 候補手の優劣を決める | 被ダメージを早く比較する |
| 戦闘シミュレータ | カード効果と敵ターンを反映する | 将来の敵行動を一般には展開しない |

## 原因1: 評価関数は加点式ではなく、辞書式比較だった

一般的なゲームAIの説明では、「ダメージに -1 点、敵HP削減に +0.5 点」といった重み付きスコアを想像しがちである。しかし、このボットの共通比較器はその方式ではない。

`does_challenger_defeat_the_best()` は比較関数のリストを先頭から順に実行し、**最初に差が出た指標だけで勝敗を決める**。[4] 後ろにある指標は、先行する指標が同率のときしか参照されない。

既定の優先順位には、戦闘敗北の回避・戦闘勝利・復活手段の保持・ドロー・無形化が続き、その直後に `least_incoming_damage_over_1` がある。一方で、敵撃破数、最低敵HP、総敵HPはその後に位置する。[5]

```python
# 概念図。実コードは比較関数のリストを順に評価する。
for criterion in ordered_criteria:
    result = criterion(best, candidate)
    if result is not None:
        return result
```

`least_incoming_damage_over_1` は、候補間で 2 点以上の被ダメージ差があれば、被ダメージの少ない候補を勝たせる。[6] したがって、敵を倒し切れない局面では、次のような比較になりやすい。

| 候補 | 今ターンの被ダメージ | 敵HPへの進捗 | 共通比較器の結論 |
| --- | ---: | ---: | --- |
| Defend | 0 | 0 | 被ダメージ軸で有利 |
| Strike | 6 | 12 | 敵HP比較に到達する前に不利 |

ここで重要なのは、Strike の 12 ダメージが悪いのではないことだ。**先に「今ターンの 6 ダメージを受ける」という差が出ると、12 ダメージの進捗を比較する順番が来ない。**

これは係数が少し大きいという問題より強い。加点式なら、防御 6 と攻撃 12 の交換条件を係数調整で滑らかに変えられる。辞書式比較では優先順位の前後を入れ替えるだけで意思決定が不連続に変わる。防御偏重を直すなら、「ブロックの点数を下げる」のではなく、**どの局面で被ダメージ比較より撃破進捗を先に置くか**を設計する必要がある。

## 原因2: 先読みは「現在の敵行動を一度受ける」地点で止まる

候補手の探索は手札から可能なプレイ列を作り、その各状態に対して `end_turn()` を一度呼んで比較する。[2] これは現在ターンのコンボや、現在見えている敵攻撃をブロックでどこまで抑えられるかを評価するには有効である。

ただし、ライブのゲーム状態から取り込まれる敵情報は、HP、ブロック、パワー、**現在の** `move_base_damage` と `move_hits` が中心である。[7] `end_turn()` も、その時点の damage・hits・Strength・Weak などを使って敵攻撃を適用する。[8] 次のターンにどの意図へ移るか、敵の通常の強化・回復・行動パターンがどう変わるかを共通の遷移モデルとしては展開していない。

```text
現在の手札
  ├─ 防御を打つ
  └─ 攻撃を打つ
       ↓
現在表示中の敵行動を1回適用
       ↓
「ターン終了時の状態」を比較

一般には見ないもの:
  次ターン以後の敵意図 / 強化 / 回復 / 実際に引くカード / 累積被ダメージ
```

この設計では、敵がこのターンに 10 ダメージを出すならブロック 8 の価値は正確に近く計算できる。しかし、「今回 8 を受け入れて敵を早く倒せば、次の 15 ダメージ攻撃や強化を一回減らせる」という価値は、一般には比較器に渡らない。

山札からのドローも、実カードを展開するのではなく `CARD_FROM_DRAW` というプレースホルダーで近似している。[9] そのため、攻撃で戦闘を短縮したときの将来の手札品質、または防御カードを残したときの次ターンの防御再現性も、完全には比較できない。

## なぜ一部の敵には強く見えるのか

実装には例外比較器がある。Gremlin Nob 用比較器は、スキルを使うと敵が強くなるコストを補正し、共通の被ダメージ優先を外す。[10] Sentry 用比較器には、3体が生存している間は端の敵を積極的に倒すための優先順位がある。[11]

これは良い工夫であると同時に、共通ロジックの限界を示している。**長期戦が危険な理由を汎用的には予測できないため、既知の敵ごとに「ここでは攻める」という例外を追加している**のである。実際、既存の制約文書には Writhing Mass の意図変化や Time Eater の回復について、まだ認識できない旨が残されている。[12]

## 11,000経路の探索上限も複雑な局面を悪化させる

既定の探索上限は 11,000 経路である。[1] プロジェクト自身の制約文書も、手札が多く、プレイ可能なカードが多く、敵が3体以上いる場合は打ち切られ、右端のカードのような不自然な手が選ばれうると記している。[12]

これは防御偏重そのものの第一原因ではない。しかし多敵戦や大量ドローでは、理想的な攻撃列を十分に列挙できないまま探索が終わる可能性がある。比較器を改善しても、比較対象に有効な攻撃列が存在しなければ意味がない。評価設計と探索予算はセットで観測すべきである。

## 改善案: 防御を弱くするのではなく、時間の価値を入れる

最初の改善として、すべてを数ターン完全探索する必要はない。次の情報を加え、`least_incoming_damage_over_1` の前に置く局面を限定するだけでも、意図はかなり明確になる。

| 追加する概念 | 具体例 | 期待する効果 |
| --- | --- | --- |
| 危険域 | ターン後HPが安全下限を下回る、次の既知攻撃で致死圏 | 本当に必要な防御は維持する |
| 撃破ターン短縮 | 今の攻撃で敵行動を1回減らせる | 攻撃の将来防御価値を可視化する |
| 敵スケーリング圧 | Strength増加、回復、意図変化、時間経過ダメージ | 放置コストを比較に入れる |
| 防御の持越し | Barricade、Blur、Calipers、弱体化 | 次ターンにも価値が残る防御を正当に評価する |

中期的には、現在ターンを完全展開した後に次の 1～2 ターンだけ限定ロールアウトする方法が現実的である。敵については全敵の完全再現を目指す前に、強化・回復・意図変化を持つ敵だけをデータ駆動の遷移表にする。これなら「今回の 6 点を守る」対「敵の行動回数を1回減らす」を、少なくとも短い地平で比較できる。

検証も、単発のプレイ感だけでは足りない。同一 seed 群で勝率、到達 Act、戦闘ターン数、戦闘別の累積被ダメージ、探索打ち切り回数を記録し、特にスケーリング敵に対するターン数と被ダメージを回帰指標にするべきである。現行テストは個別局面の期待コマンドを多数確認しており、Gremlin Nob や Sentry の特例も保護している。[13] その土台の上に、**「数点を受けても一ターン早く強化敵を止めるべき局面」**を新たな固定テストとして足すとよい。

## おわりに

今回の「守りすぎ」は、AIがHPを大切にしすぎたというより、**見える一敵ターンの被ダメージを、見えない将来の戦闘時間より先に判定するよう実装されていた**ことの帰結だった。

ゲームAIでは、「今のダメージを減らす」ことと「敵に次の行動をさせない」ことの両方が防御である。後者を評価器に表現できるようにすると、防御カードを不当に嫌わず、かつ敵を強化前に倒す攻撃性も取り戻せる。

## 参照

[1]: https://github.com/tonbiattack/bottled_ai/blob/2f1152d2e2b33721d0fef3ce02feed12b6e03da8/rs/common/handlers/common_battle_handler.py "共通戦闘ハンドラ"
[2]: https://github.com/tonbiattack/bottled_ai/blob/2f1152d2e2b33721d0fef3ce02feed12b6e03da8/rs/calculator/executor.py#L12-L45 "戦闘探索と評価"
[3]: https://github.com/tonbiattack/bottled_ai/blob/2f1152d2e2b33721d0fef3ce02feed12b6e03da8/rs/ai/requested_strike/config.py#L1-L31 "Ironclad 戦略設定"
[4]: https://github.com/tonbiattack/bottled_ai/blob/2f1152d2e2b33721d0fef3ce02feed12b6e03da8/rs/common/comparators/common_general_comparator.py#L142-L164 "辞書式比較の実装"
[5]: https://github.com/tonbiattack/bottled_ai/blob/2f1152d2e2b33721d0fef3ce02feed12b6e03da8/rs/common/comparators/common_general_comparator.py#L13-L56 "既定比較器の優先順位"
[6]: https://github.com/tonbiattack/bottled_ai/blob/2f1152d2e2b33721d0fef3ce02feed12b6e03da8/rs/common/comparators/core/comparisons.py#L69-L72 "閾値付き被ダメージ比較"
[7]: https://github.com/tonbiattack/bottled_ai/blob/2f1152d2e2b33721d0fef3ce02feed12b6e03da8/rs/calculator/game_state_converter.py#L113-L127 "敵状態の変換"
[8]: https://github.com/tonbiattack/bottled_ai/blob/2f1152d2e2b33721d0fef3ce02feed12b6e03da8/rs/calculator/battle_state.py#L651-L703 "敵ターン処理"
[9]: https://github.com/tonbiattack/bottled_ai/blob/2f1152d2e2b33721d0fef3ce02feed12b6e03da8/rs/calculator/battle_state.py#L749-L786 "ドローの近似"
[10]: https://github.com/tonbiattack/bottled_ai/blob/2f1152d2e2b33721d0fef3ce02feed12b6e03da8/rs/common/comparators/gremlin_nob_comparator.py "Gremlin Nob 固有比較器"
[11]: https://github.com/tonbiattack/bottled_ai/blob/2f1152d2e2b33721d0fef3ce02feed12b6e03da8/rs/common/comparators/three_sentry_comparator.py "Sentry 固有比較器"
[12]: https://github.com/tonbiattack/bottled_ai/blob/2f1152d2e2b33721d0fef3ce02feed12b6e03da8/docs/capabilities.md "既知の制約"
[13]: https://github.com/tonbiattack/bottled_ai/blob/2f1152d2e2b33721d0fef3ce02feed12b6e03da8/tests/ai/common/handlers/test_battle_handler.py "戦闘ハンドラのテスト"

---
title: "【馬券戦略編③】ポートフォリオ戦略と総合評価：リスクを分散し、最強の布陣を組む"
emoji: "🛡️"
type: "tech"
topics: ["python", "機械学習", "ポートフォリオ理論", "リスク管理", "数理最適化"]
published: false
---

# 【馬券戦略編③】ポートフォリオ戦略と総合評価：リスクを分散し、最強の布陣を組む

## はじめに：単一戦略の限界

前回までで、単勝や複勝など、各券種ごとに「勝てるルール」を見つけ出しました。
しかし、一つの戦略だけで運用することにはリスクがあります。

- **単勝戦略**: 回収率は高いが、的中率が低く、不的中が続くと精神的にきつい（ドローダウンが大きい）。
- **複勝戦略**: 的中率は高いが、利益率が低く、大きく資産を増やすのが難しい。

そこで導入するのが、複数の戦略を組み合わせる**「ポートフォリオ戦略」**です。
今回は、異なる性格の戦略を組み合わせ、リスクを抑えながら安定した収益を目指す仕組みについて解説します。

## 1. ポートフォリオの効果

`notebooks/39_betting_strategy_simulation_monthly.ipynb` で、以下の2つの戦略を組み合わせたシミュレーションを行いました。

1.  **Win Aggressive**: 期待値重視の単勝戦略（的中率低、爆発力あり）
2.  **Place Stable**: 的中率重視の複勝戦略（的中率高、安定感あり）

### シミュレーション結果

単体では月ごとの収支のブレが激しい「単勝戦略」も、「複勝戦略」と組み合わせることで、資産曲線が滑らかになることが確認できました。
複勝がコツコツと稼いでドローダウン（資産の減少）を埋め合わせ、その間に単勝がホームランを打つ、という相互補完の関係が成立しています。

## 2. 戦略マネージャーの実装

このポートフォリオ運用をシステム化するために、`BettingStrategyManager` を実装しました。

**ファイル**: `src/keiba/services/analysis/domain/logic/betting_strategy_manager.py`

このクラスは、設定ファイル（CSV）から複数の戦略を読み込み、レースごとにそれらを適用して、最終的な「買い目リスト」を生成します。

```python
class BettingStrategyManager:
    def generate_bets(self, prediction_df, odds_list):
        bets = []
        for strategy in self.strategies:
            # 各戦略（単勝、複勝、三連単など）ごとに買い目を評価
            if strategy.bet_type == 'TANSHO':
                candidates = self._evaluate_single_horse_strategy(...)
            elif strategy.bet_type == 'SANRENTAN':
                candidates = self._evaluate_sanrentan_strategy(...)
            
            bets.extend(candidates)
        return bets
```

これにより、「攻めの単勝」と「守りの複勝」、さらには「一発逆転の三連単」を、一つのシステム内で同時に運用することが可能になります。

## 3. 総合評価指標：`betting_score`

最終的にどの戦略をポートフォリオに組み込むか？その判断基準として、前回の記事でも触れた**「シャープレシオ（Sharpe Ratio）」**を中核的な評価指標として採用しました。

$$ シャープレシオ = \frac{平均リターン}{リターンの標準偏差（ブレ幅）} $$

単に利益が大きいだけでなく、**「毎レース安定して利益が出る」**戦略を高く評価する考え方です。
このアプローチにより、「一発屋」の不安定な戦略が排除され、実運用に耐えうる堅実なルールを選定しています。

## 4. 最強の布陣（設定ファイル）

シャープレシオによる最適化の結果、選ばれた戦略は `src/keiba/config/betting_strategies.csv` に保存され、本番システムで読み込まれます。

```csv
strategy_id,bet_type,ev_threshold,prob_threshold,max_bets,is_active
tansho_opt_2026_v1,TANSHO,1.50,0.098,,true
sanrentan_opt_2026_v1,SANRENTAN,3.20,0.0005,1,true
```

- **tansho_opt**: 期待値1.5倍以上、勝率9.8%以上の馬を狙う主力戦略。
- **sanrentan_opt**: 超高配当を狙う、1点買いの宝くじ戦略（アクセント）。

## まとめ

- 複数の戦略を組み合わせることで、収支の安定化を図る。
- `BettingStrategyManager` で、異なる券種の戦略を一元管理する。
- **シャープレシオ**を評価指標として用いることで、リスクとリターンのバランスが良い戦略を選定する。

これで、データ収集からモデル予測、そして馬券購入戦略まで、競馬予想AIの全貌を解説しました。
このシステムは、毎週のレースで自動的にデータを更新し、学習し、進化し続けています。
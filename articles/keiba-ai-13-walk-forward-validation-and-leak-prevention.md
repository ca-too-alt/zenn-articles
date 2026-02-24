---
title: "【検証編①】未来を予測する：Walk-Forward法とリークの完全排除"
emoji: "🧪"
type: "tech"
topics: ["python", "機械学習", "時系列データ", "モデル検証", "データリーク"]
published: false
---

# 【検証編①】未来を予測する：Walk-Forward法とリークの完全排除

## はじめに：K-Fold交差検証の罠

機械学習の教科書では、モデルの評価手法として「K-Fold交差検証（データをランダムに分割して検証）」がよく紹介されます。
しかし、競馬予測のような**時系列データ**において、これをそのまま適用するのは危険です。

なぜなら、ランダム分割では**「未来のレースデータを使って、過去のレースを予測する」**という状況が発生してしまうからです。
これでは、本番環境（未来は未知）での性能を正しく測ることができません。

そこで採用したのが、**Walk-Forward Validation（Expanding Window方式）**です。

## 1. Walk-Forward Validation (Expanding Window)

この手法は、時間の流れに沿って検証を行う、最も現実に即した方法です。

1.  **初期学習**: 過去の一定期間（例：2019年〜2021年）のデータでモデルを学習。
2.  **予測**: その直後の期間（例：2022年1月）をテストデータとして予測。
3.  **期間拡大**: テストデータを学習データに加え（2019年〜2022年1月）、モデルを再学習。
4.  **次月予測**: 次の期間（2022年2月）を予測。

これを繰り返すことで、常に「その時点での過去データ」のみを使って未来を予測するシミュレーションが可能になります。

## 2. シミュレーションの実装

`notebooks/35_rank_learning_simulation.ipynb` で、このプロセスを実装しています。

```python
# シミュレーションの疑似コード
current_date = start_date
while current_date <= end_date:
    # 1. 予測対象期間の設定（1ヶ月分など）
    target_period = get_next_period(current_date)
    
    # 2. その時点のモデルで予測を実行
    # （まだこの期間の正解は知らない前提）
    predictions = model.predict(target_period)
    
    # 3. 予測結果を保存
    save_results(predictions)
    
    # 4. 時間を進め、特徴量を更新（正解データを取り込む）
    update_features(target_period)
    
    # 5. 定期的にモデルを再学習（Expanding Window）
    if is_retrain_timing(current_date):
        model.train(all_data_until_now)
        
    current_date = next_date
```

このノートブックを実行することで、過去数年分にわたる「擬似的な本番運用」を行い、月ごとの回収率や精度の推移をグラフ化して検証しています。

## 3. データリークの完全排除

時系列検証を行っても、**「特徴量の中に未来の情報が含まれている（データリーク）」**と、全てが台無しになります。
このプロジェクトでは、以下の2段階でリークを徹底的に防いでいます。

### ① 特徴量生成での防御 (`feature_generator.py`)

「騎手の過去勝率」などを計算する際、単純に集計すると「今回のレース結果」が含まれてしまいます。
そこで、`shift(1)` を使って明示的に1行ずらすことで、**「前走までのデータ」**のみを参照するようにしています。

```python
# src/keiba/services/analysis/infrastructure/feature_generator.py

def add_expanding_feature(df, group_cols, target_col, ...):
    # ...
    # shift(1) で現在のレースを除外してから集計
    race_grouped[f'{new_name}_val'] = race_grouped.groupby(cols)[temp_agg_col].shift(1)
    
    # expanding().mean() で過去の累積平均を計算
    race_grouped[new_name] = race_grouped.groupby(cols)[f'{new_name}_val'].expanding().mean()
```

### ② 学習時の防御 (`columns.py`)

モデルに渡すカラムリストを厳格に管理し、**「レース後にしか分からない情報」**を物理的に遮断しています。

```python
# src/keiba/config/columns.py

# 学習時に必ず除外するカラム（禁止リスト）
DROP_COLS_M4 = [
    'rank',           # 着順（答え）
    'finish_time',    # 走破タイム（答え）
    'odds_win',       # 確定オッズ（レース直前まで変動するため、学習には使わない）
    'popularity',     # 人気順
    'last_3f_time',   # 上がり3F（未来）
    # ...
]
```

特にオッズ関連は、学習時には「人気馬＝強い」というバイアスがかかりすぎるのを防ぐため、あえて特徴量から除外し、**「純粋な能力予測」**を行った上で、最後の馬券戦略でのみオッズを使用するという方針をとっています。

## まとめ

- 時系列データでは、ランダム分割ではなく **Walk-Forward法** で検証する。
- `shift(1)` を使って、特徴量生成時のリークを防ぐ。
- 設定ファイルで禁止カラムを定義し、学習データへの未来情報の混入を物理的に防ぐ。

この厳格な検証プロセスを経ることで、シミュレーション上の回収率と、本番での回収率の乖離を最小限に抑えることができます。

次回は、この検証プロセスを効率化し、継続的に運用するためのシステム基盤について解説します。
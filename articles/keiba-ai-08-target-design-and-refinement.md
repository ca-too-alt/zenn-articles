---
title: "【モデル編③】サブモデルの進化と工夫：予測は「何を」させるか"
emoji: "🛠️"
type: "tech"
topics: ["python", "機械学習", "回帰分析", "特徴量設計", "データ分析"]
published: false
---

# 【モデル編③】サブモデルの進化と工夫：予測は「何を」させるか

## はじめに：予測モデルの「解像度」を上げる

3モデル連携アーキテクチャという「骨格」はできましたが、当初、各サブモデルの予測精度は芳しくありませんでした。

- **位置取り予測**: 「逃げ」か「先行」かの大雑把な分類しかできず、微妙な位置取りを表現できない。
- **タイム予測**: 馬場状態やペースに引きずられ、予測タイムが安定しない。

これらの課題を解決するために行ったのは、アルゴリズムの変更ではなく、**「モデルに何を予測させるか（目的変数の再設計）」**と**「予測結果をどう解釈するか（後処理の工夫）」**という、地道な改善サイクルでした。

## 1. 位置取り予測モデルの進化

### 課題①：分類問題の限界

最初は、各馬の脚質を「逃げ」「先行」「差し」「追込」の4択で当てる**分類問題**として位置取りを予測していました。
しかし、これでは「逃げ馬の中でもハナを主張する馬」と「2番手の馬」が同じ「逃げ」カテゴリになってしまい、展開の機微を捉えきれませんでした。

### 改善①：目的変数を「位置取りスコア」へ (回帰問題化)

そこで、`戦法予測ロジック改修案.txt` に基づき、目的変数を**「レース中盤の位置取りを0.0〜1.0で表すスコア」**に変更し、回帰問題として解くことにしました。

```python
# c:\keiba_app\src\keiba\services\analysis\infrastructure\feature_generator.py

def add_target_variables(df: pd.DataFrame) -> pd.DataFrame:
    # ...
    # 展開が落ち着く第2〜第4コーナーの平均順位を計算
    target_corners = ['corner_2', 'corner_3', 'corner_4']
    avg_rank = df[target_corners].mean(axis=1)
    
    # (順位 - 1) / (頭数 - 1) で正規化し、0.0(先頭)〜1.0(最後方)のスコアに変換
    df['position_score'] = (avg_rank - 1) / (df['num_horses'] - 1)
    # ...
    return df
```

これにより、モデルはより連続的で詳細な位置取りを学習できるようになりました。

### 課題②：予測値が中央に寄る「Mode Collapse」

回帰問題にしたことで新たな問題が発生しました。モデルがリスクを避けるようになり、**予測値が平均値（0.5前後）に集中してしまう**現象です。
これでは「全頭が中団」という、面白みのない予測しかできません。

### 改善②：予測値のレース内スケーリング

この問題を解決するため、モデルの生の予測値をそのまま使わず、レース内で再評価する後処理 `apply_position_scaling` を導入しました。

```python
# c:\keiba_app\src\keiba\services\analysis\domain\logic\scaling.py

def apply_position_scaling(df: pd.DataFrame, ...) -> pd.DataFrame:
    # ...
    def calculate_scaled_values(x): # x はレースごとの予測スコア(Series)
        n = len(x)
        # 1. 予測スコアのレース内順位から、均等なスコアを計算
        ranks = x.rank(method='average')
        score_uniform = (ranks - 1) / (n - 1)
        
        # 2. 予測スコアの分布を維持したまま正規化
        score_dist = (x - x.min()) / (x.max() - x.min())
            
        # 3. 上記2つを重み付けしてブレンドする
        return (1 - distribution_weight) * score_uniform + distribution_weight * score_dist

    df['pred_position_score'] = df.groupby('race_id')['pred_position_score'].transform(calculate_scaled_values)
    return df
```

この処理は、`training_service.py` の `apply_model_1_prediction` 内で呼び出されます。
これにより、予測の順序関係は保ちつつ、分布をレース全体に広げることで、「この馬は予測1位だからスコア0.1、この馬は予測18位だからスコア0.9」のように、メリハリのある展開予測が可能になりました。

## 2. タイム予測モデルの進化

### 課題：外的要因に左右される絶対タイム

最終モデルであるタイム予測モデル（Model 4）は、当初、各馬の「走破タイム」そのものを予測していました。
しかし、走破タイムは馬場状態（高速馬場か、時計がかかる馬場か）やレースペースに大きく左右されるため、予測が非常に不安定でした。

### 改善：目的変数を「タイムの偏差」へ

そこで、予測する対象を「絶対タイム」から**「レースの平均タイムからどれだけ速い/遅いか」という偏差**に変更しました。

```python
# c:\keiba_app\src\keiba\services\analysis\application\training_service.py

def train_model_4(df: pd.DataFrame) -> lgb.LGBMRegressor:
    # ...
    # 目的変数を「レース平均タイムとの差」に設定
    race_mean_time = df_model.groupby('race_id')['finish_time'].transform('mean')
    y = df_model['finish_time'] - race_mean_time
    # ...
    model.fit(X_train, y_train, ...)
    return model
```

これにより、モデルは馬場状態といった外的要因の揺らぎを直接予測する必要がなくなり、**「与えられた条件下で、他の馬より相対的にどれだけ速く走れるか」**という、より本質的な能力の予測に集中できるようになりました。

## まとめ：地道な改善が生む、確かな精度

優れたアーキテクチャも、それを構成する個々の部品（サブモデル）が貧弱では機能しません。

- **目的変数の再設計**: モデルに「何を」解かせるかを研ぎ澄ます。
- **後処理の工夫**: モデルの出力を「どう」解釈し、現実に近づけるか。

この2つの地道な改善サイクルを回し続けることで、3モデル連携システムは初めてその真価を発揮し、予測精度を一段階引き上げることができました。

次回は、こうして得られた予測結果を、最終的にどのように馬券購入に結びつけるのか、「馬券戦略と最適化」について解説します。
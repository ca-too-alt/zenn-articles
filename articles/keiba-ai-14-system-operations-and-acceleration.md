---
title: "【検証編②】システム運用と高速化：ParquetとWeb UIの活用"
emoji: "⚙️"
type: "tech"
topics: ["python", "機械学習", "FastAPI", "データ基盤", "MLOps"]
published: false
---

# 【検証編②】システム運用と高速化：ParquetとWeb UIの活用

## はじめに：待ち時間をゼロにする

AIモデルの開発において、最も無駄な時間は「データの読み込み待ち」です。
数年分のレースデータを毎回データベースからSQLで取得し、Pythonで加工していると、学習を始めるまでに数分〜数十分かかってしまいます。

これでは、パラメータを少し変えて試すだけの実験も億劫になります。
そこで導入したのが、**Parquet（パーケット）形式によるデータスナップショット**です。

## 1. Parquetスナップショットの威力

Parquetは、列指向のデータ保存フォーマットで、Pandas（Pythonのデータ分析ライブラリ）との相性が抜群に良く、読み書きが非常に高速です。

### スナップショット作成スクリプト

`scripts/create_analysis_snapshot.py` を使って、データベースの中身をまるごと「分析用データマート」としてファイルに保存します。

```python
# scripts/create_analysis_snapshot.py

def main():
    # ...
    # 1. DBから全データをロード
    all_raw_data = load_and_merge_data_from_db(start_date=start_date, end_date=end_date)
    
    # 2. 特徴量生成パイプラインを実行
    features_df = create_features_for_prediction(raw_df=all_raw_data, ...)
    
    # 3. Parquetファイルとして保存
    features_df.to_parquet(output_path)
```

このスクリプトを週に1回（レース結果確定後など）実行しておけば、普段の実験ではこのParquetファイルを読み込むだけで済みます。
SQLで数分かかっていた処理が、**数秒**で完了するようになります。

## 2. ハイブリッドなデータローダー

分析コード側では、「Parquetファイルがあればそれを使い、なければDBから取得する」という柔軟な仕組みを実装しています。

**ファイル**: `src/keiba/services/analysis/infrastructure/data_loader.py`

```python
def load_features_from_parquet(file_path: Path = None) -> pd.DataFrame:
    """
    作成済みの特徴量ファイル(Parquet)をロードします。
    デフォルトでは output/features/features_latest.parquet を読み込みます。
    """
    if file_path is None:
        file_path = Path(env.OUTPUT_DIR) / 'features' / 'features_latest.parquet'
    
    if not file_path.exists():
        raise FileNotFoundError("Feature file not found...")
        
    return pd.read_parquet(file_path)
```

これにより、開発者はデータの場所を意識することなく、常に最速の方法でデータを取得できます。

## 3. Web UIによる運用管理

開発段階ではコマンドライン（黒い画面）でスクリプトを叩いていましたが、実運用では不便です。
そこで、**FastAPI** を使ってWeb管理画面を作成しました。

### バックエンド (`main.py`)

`src/keiba/services/web_api/main.py` がWebサーバーのエントリーポイントです。
ここでは、データ収集や分析の機能をAPIとして公開しています。

```python
def create_app() -> FastAPI:
    app = FastAPI(title="競馬予想アプリ", ...)

    # 各機能のルーターを登録
    app.include_router(collection_router, prefix="/collection", ...)
    app.include_router(analysis_router, prefix="/analysis", ...)
    app.include_router(prediction_router, prefix="/prediction", ...)
    
    return app
```

### フロントエンド (`index.html`)

`src/keiba/services/web_api/templates/index.html` は、シンプルなダッシュボードです。
「データ更新」「学習開始」「今週の予想」といったボタンが配置されており、クリックするだけで裏側のPythonスクリプトが実行されます。

これにより、エンジニアでない人でも（あるいはスマホからでも）、競馬予想AIを操作できるようになります。

## まとめ

- **Parquet** を使うことで、データ読み込み時間を劇的に短縮し、試行錯誤のサイクルを高速化した。
- **FastAPI** でWeb UIを構築し、コマンド操作不要の快適な運用環境を実現した。

「私がAIに作ってもらった競馬予想AIシステム」についての解説は以上になります。
今後は気が向けば、各種シミュレーション結果などをナレッジとしてあげようかと思います。
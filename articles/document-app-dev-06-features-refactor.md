---
title: "AI×Amplify Gen2で月40万を1万に！非エンジニアの開発記 #6：Featuresリファクタリング"
emoji: "🏗️"
type: "tech"
topics: ["amplify", "react", "architecture", "refactoring", "typescript"]
published: false
---

## 概要
第5話までで、認証、ユーザー管理、アクセス制限といった主要な基盤機能が揃いました。しかし、機能が追加されるたびに `src/components` フォルダ内にファイルが増え続け、どのファイルがどの機能に関連しているのか、どこを修正するとどこに影響が出るのかが把握しづらい「スパゲッティ状態」になりつつありました。

特に非エンジニアとして開発を進める中で、AIにコードを渡す際も「関連するファイルが多すぎて一度に渡せない」「依存関係が複雑でAIが誤った修正を提案する」といった課題が顕著になってきました。今回は、将来の拡張に耐えうる「Featuresアーキテクチャ」への大規模リファクタリングの記録をお届けします。

## 実装内容

### 1. Featuresアーキテクチャの採用
単なる「コンポーネント」「フック」といった技術的な役割でフォルダを分けるのではなく、「ユーザー管理」「ドキュメント管理」といった「機能（Feature）」単位でコードをカプセル化する設計を採用しました。

新しく構築したディレクトリ構造は以下の通りです。

```text
src/
  features/
    auth/                  # 認証関連（ログイン、パスワードリセット）
    user-management/       # ユーザー・ロール管理
      components/          # 機能固有のUI
      hooks/               # 機能固有のロジック
      routes/              # 機能固有のページコンポーネント
      services/            # API呼び出しなどの外部連携
      types/               # 型定義
    document-management/   # ドキュメント管理（フォルダ、ファイル）
  components/              # 共通UI（ボタン、ダイアログ等）
  hooks/                   # 共通フック（通知管理等）
  services/                # 共通サービス（Amplifyクライアント等）
```

### 2. ビジネスロジックの「Service層」への分離
これまでコンポーネント内に直接記述していた `generateClient` を用いたデータ操作ロジックを、`services/` 配下の独立した関数として切り出しました。

```typescript
// src/features/user-management/services/userService.ts
import { generateClient } from 'aws-amplify/data';
import type { Schema } from '../../../../amplify/data/resource';

const client = generateClient<Schema>();

export const userService = {
  // ユーザー一覧を取得するロジックを分離
  async listUsers() {
    const { data, errors } = await client.models.User.list();
    if (errors) throw new Error('ユーザー情報の取得に失敗しました');
    return data;
  },
  
  // ロール（グループ）を割り当てるAPI呼び出し
  async assignRole(userId: string, groupName: string) {
    const response = await fetch(`${import.meta.env.VITE_API_ENDPOINT}/users/${userId}/roles`, {
      method: 'POST',
      body: JSON.stringify({ groupName }),
    });
    return response.json();
  }
};
```

### 3. エントリポイントの整理
各機能のページ定義を `routes/` に集約し、`App.tsx` からはそれらを呼び出すだけという構成に変更しました。

## 遭遇した問題

### 1. インポートパスの「相対パス地獄」
ファイルを移動したことにより、既存の `import { ... } from '../../components/Button'` といった記述がすべて壊れました。特に深い階層に移動したファイルでは `../../../../` のように遡る階層が増え、手動での修正は不可能に近い状態でした。

### 2. 循環参照の発生
「ユーザー管理機能」の中で「ドキュメント管理機能」の一部のコンポーネントを使いたいといった状況が発生し、機能間でお互いがお互いをインポートし合う「循環参照」が発生。ビルドエラーや実行時の予期せぬ挙動に繋がりました。

### 3. AIのコンテキスト消失
ファイルを細かく分割した結果、AIに修正を依頼する際に「どのファイルとどのファイルが関係しているか」の全体像を伝えきれず、断片的なアドバイスしか得られなくなる局面がありました。

## 解決アプローチ

### 1. AI（Gemini Code Assist）を活用した一括変換
インポートパスの修正は、AIの得意分野です。VS Code上で「現在のフォルダ構造を元に、このファイルのインポートパスをすべて修正して」と指示を出すことで、機械的な作業を大幅に短縮しました。また、TypeScriptのエイリアス機能（`@/` 形式）の導入も検討しましたが、Amplify Sandbox環境との相性を考慮し、今回は確実な相対パス修正を選択しました。

### 2. 機能間の「依存ルール」の策定
循環参照を防ぐため、以下の明確なルールを設けました。
- 機能 A は 機能 B の内部コンポーネントを直接インポートしてはいけない。
- 共通で使う部品は `src/components`（Shared）へ移動させる。
- 機能間のデータ連携は、`src/services` の共通サービス層を介して行う。

### 3. フォルダ構成の「地図」の作成
AIにプロジェクト構造を正しく理解させるため、`README.md` や `docs/` 内に最新のディレクトリ構成と各フォルダの役割を記述したテキストファイルを用意しました。これをプロンプトに含めることで、AIの提案精度を維持しました。

## 最終的な解決策

### リファクタリング後のクリーンな `App.tsx`
機能単位でルートを分離したことにより、メインのルーティング定義が非常に見通しの良いものになりました。

```tsx
// src/App.tsx (抜粋)
import { BrowserRouter as Router, Routes, Route } from 'react-router-dom';
import { DocumentRoutes } from './features/document-management/routes';
import { UserManagementRoutes } from './features/user-management/routes';
import { AuthRoutes } from './features/auth/routes';

const App = () => {
  return (
    <Router>
      <Routes>
        {/* 認証関連のルート */}
        <Route path="/auth/*" element={<AuthRoutes />} />
        
        {/* 認証済みレイアウト内での機能別ルート */}
        <Route element={<AuthenticatedLayout />}>
          <Route path="/documents/*" element={<DocumentRoutes />} />
          <Route path="/admin/users/*" element={<UserManagementRoutes />} />
          <Route path="/" element={<Navigate to="/documents" />} />
        </Route>
      </Routes>
    </Router>
  );
};
```

### 共通UIコンポーネントの抽出
ボタンやテーブルなどの基本部品を `src/components` に集約し、各機能からはそれらを再利用する形に統一しました。

```tsx
// src/components/ui/DataTable.tsx
import { Table, TableBody, TableCell, TableHead, TableRow } from '@mui/material';

interface DataTableProps<T> {
  columns: { key: keyof T; label: string }[];
  data: T[];
}

export function DataTable<T>({ columns, data }: DataTableProps<T>) {
  return (
    <Table>
      <TableHead>
        <TableRow>
          {columns.map((col) => <TableCell key={String(col.key)}>{col.label}</TableCell>)}
        </TableRow>
      </TableHead>
      <TableBody>
        {data.map((row, i) => (
          <TableRow key={i}>
            {columns.map((col) => <TableCell key={String(col.key)}>{String(row[col.key])}</TableCell>)}
          </TableRow>
        ))}
      </TableBody>
    </Table>
  );
}
```

## 学んだこと

### フォルダ構成は「思考の地図」
適切なディレクトリ設計を行うことは、単にファイルを整理するだけでなく、開発者の「思考」を整理することに他なりません。「この修正は document-management フォルダ内だけで完結する」と確信を持てる構造は、開発の心理的負担を劇的に下げてくれました。

### 非エンジニアこそ「設計」を重視すべき理由
知識が限られているからこそ、AIに頼る場面が多くなります。しかし、AIは「きれいな設計」を自発的に維持してくれるわけではありません。人間が「地図（アーキテクチャ）」を定義し、AIをその地図の上で働かせることが、長期間の開発を成功させる鍵であることを学びました。

### 疎結合のメリット
Service層を分離したことで、将来的にAmplify Gen2以外のAPIに差し替えることになっても、UI（コンポーネント）側の修正は最小限で済むようになりました。この「関心の分離」という概念は、プログラムの堅牢性を高める上で非常に強力な武器になります。

## 次回予告
コードの整理整頓が完了し、いよいよドキュメント管理の核心部分に迫ります。ファイル名が変更されても、これまでの履歴を完璧に追跡できるようにする「堅牢なバージョン管理システム」と、その要となる `versionGroupId` の設計について詳しく紹介します。
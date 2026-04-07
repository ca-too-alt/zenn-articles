---
title: "AI×Amplify Gen2で月40万を1万に！非エンジニアの開発記 #1：認証基盤構築"
emoji: "🚀"
type: "tech"
topics: ["amplify", "aws", "react", "typescript", "authentication"]
published: true
---

## 概要
本連載では、非エンジニアである私がAIの助けを借りて、AWS Amplify Gen2を用いた実用的なドキュメント管理アプリを開発した記録を共有します。第1回となる今回は、プロジェクトの立ち上げから、Amplify Gen2の新しいリソース定義方法を用いた認証基盤の構築、そしてフロントエンドのアーキテクチャ設計について解説します。

## 実装内容
### 1. Amplify Gen2の初期セットアップと認証機能の定義
Amplify Gen2では、TypeScriptを使用してクラウドインフラをコードとして定義します。まず、ユーザーがメールアドレスでログイン・登録できる基本的な認証機能を`amplify/auth/resource.ts`で設定しました。

```typescript
// amplify/auth/resource.ts
import { defineAuth } from '@aws-amplify/backend';

export const auth = defineAuth({
  loginWith: {
    email: true, // メールアドレスでのログインを有効化
  },
  userAttributes: {
    'custom:employeeId': { // 後々必要になる社員番号用のカスタム属性を定義
      dataType: 'String',
      mutable: true,
    },
  },
  // アプリケーションに必要な初期グループを定義します。
  // これにより、サンドボックス起動時にこれらのグループの存在が保証されます。
  groups: ['Admins', 'Users', '設計', '品証', '営業', '物流'],
});
```

この設定により、AmplifyがAWS Cognito User Poolを自動的にプロビジョニングし、ユーザー認証のバックエンドが構築されます。

### 2. フロントエンドでのAmplify設定とアプリケーションの起動
次に、フロントエンドのReactアプリケーションからAmplifyのバックエンドリソースに接続するための設定を行います。`src/main.tsx`でAmplifyライブラリを初期化し、自動生成される`amplify_outputs.json`を読み込ませます。

```tsx
// src/main.tsx
import ReactDOM from 'react-dom/client'
import './index.css'

async function main() {
  // Amplifyライブラリと設定ファイルを動的にインポート
  const { Amplify } = await import('aws-amplify');
  const outputs = (await import('../amplify_outputs.json')).default;

  // Amplifyの設定
  Amplify.configure(outputs);

  // 設定完了後にAppを動的にインポート（generateClientなどが設定前に呼ばれるのを防ぐため）
  const { default: App } = await import('./App');

  // Root要素の確認
  const rootElement = document.getElementById('root');
  if (!rootElement) {
    throw new Error('Root element not found');
  }

  // React アプリケーションの起動
  ReactDOM.createRoot(rootElement).render(<App />);
}

main();
```

この`main.tsx`の工夫により、Amplifyの設定が完了してからReactアプリケーションがレンダリングされるため、バックエンドリソースへのアクセスが安定します。

### 3. `react-router-dom`によるルーティング設計
アプリケーションの画面遷移を管理するために`react-router-dom`を導入しました。`src/App.tsx`でルーティングを設定し、認証状態に応じて表示するページを切り替えるようにしました。

```tsx
// src/App.tsx (抜粋 - ルーティング設定部分)
import { BrowserRouter as Router, Routes, Route, Navigate, Outlet } from 'react-router-dom';
import DocumentManagementPage from './features/document-management/routes/DocumentManagementPage';
import UserManagementPage from './features/user-management/routes/UserManagementPage'; 
// ... その他のインポート
import LoginPage from './features/auth/routes/LoginPage';
import { Header } from './components/layout/Header';

// 認証済みユーザー向けの共通レイアウト
const AuthenticatedLayout = ({ isAdmin, user, onSignOut }: { isAdmin: boolean; user: AppUser | null; onSignOut: () => Promise<void> }) => (
  <View>
    <Header 
      user={{ ...user, isAdmin, name: user?.name } as AuthUser & { isAdmin: boolean; name?: string }} 
      onSignOut={onSignOut} 
    />
    <Outlet context={{ user, isAdmin }} />
  </View>
);

const App = () => {
  // ... ユーザー認証状態の管理 (useState, useEffect)

  return (
    <Router>
      <Routes>
        {!user ? ( // ユーザーが認証されていない場合
          <>
            <Route path="/login" element={<LoginPage />} />
            {/* 開発環境でのみ、未認証でもユーザー作成ページにアクセス可能にする */}
            {process.env.NODE_ENV === 'development' && (
              <Route path="/user-management/create" element={<CreateUserPage />} />
            )}
            {/* 未認証時はログインページにリダイレクト */}
            <Route path="*" element={<Navigate to="/login" />} />
          </>
        ) : ( // ユーザーが認証済みの場合
          <>
            {/* 認証済みユーザーは共通レイアウトを適用 */}
            <Route element={<AuthenticatedLayout isAdmin={isAdmin} user={user} onSignOut={handleSignOut} />}>
              <Route path="/" element={<DocumentManagementPage />} />
              <Route path="/login" element={<Navigate to="/" />} />
              {/* 管理者専用ルート (isAdminフラグで制御) */}
              {isAdmin ? (
                <>
                  <Route path="/admin" element={<AdminDashboardPage />} />
                  <Route path="/user-management" element={<UserManagementPage />} />
                  {/* ... その他の管理者ルート */}
                </>
              ) : (
                <>
                  {/* 一般ユーザーは管理者ルートにアクセス不可 */}
                  <Route path="/admin" element={<Navigate to="/" />} />
                  {/* ... その他の管理者ルートへのリダイレクト */}
                </>
              )}
            </Route>
            {/* 認証済みで、どのルートにもマッチしない場合はルートにリダイレクト */}
            <Route path="*" element={<Navigate to="/" />} />
          </>
        )}
      </Routes>
    </Router>
  );
};
export default App;
```

## 遭遇した問題
開発の初期段階では、ログイン後の画面表示を単一のコンポーネント内で状態変数を使って切り替えることを検討していました。しかし、このアプローチではブラウザの「戻る」ボタンが機能しない、URLによる直接アクセスができない、機能が増えるにつれてコードが複雑化し拡張性が低いといった問題が予想されました。

## 解決アプローチ
これらの問題を解決するため、早い段階で`react-router-dom`を用いたURLベースのページ遷移アーキテクチャへ移行することを決定しました。これにより、各機能を独立したページコンポーネントとして分離し、URLと紐付けることで、ウェブサイトのような直感的で拡張性の高いナビゲーションを実現します。

## 最終的な解決策
`react-router-dom`を導入し、`src/App.tsx`でアプリケーション全体のルーティングを管理する構成を採用しました。認証状態に応じて`LoginPage`と`AuthenticatedLayout`を切り替え、`AuthenticatedLayout`内でさらに管理者専用ルートを設けることで、認証ガードとページコンポーネントの分離を実現しました。これにより、機能ごとの関心を分離し、将来的な機能追加やメンテナンスが容易な基盤を構築できました。

## 学んだこと
アプリケーション開発の初期段階で、しっかりとしたページルーティングの設計を確立しておくことの重要性を再認識しました。特に、認証機能を持つアプリケーションでは、認証状態に応じた適切な画面遷移とアクセス制御の基盤を早期に構築することが、その後の機能追加（特に管理者専用画面の追加など）をスムーズに進めるための鍵となります。

## 次回予告
次回は、Amplify Gen2での開発中に直面した、一見すると複雑で解決が難しい「循環参照エラー」と、開発環境の思わぬ落とし穴であるNode.jsのバージョン問題について、その原因と解決までの道のりを紹介します。

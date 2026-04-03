---
title: "AI×Amplify Gen2で月40万を1万に！非エンジニアの開発記 #4：ユーザー管理システム"
emoji: "👥"
type: "tech"
topics: ["amplify", "aws", "cognito", "dynamodb", "appsync"]
published: false
---

## 概要
第3話では、社員番号によるカスタム認証フローを構築しました。ユーザーがログインできるようになった次に取り組むべきは、管理者が利用ユーザーを登録・停止・管理するための「ユーザー管理システム」です。

AWS Amplify Gen2環境において、認証基盤であるAmazon Cognitoと、アプリケーションデータを保持するAmazon DynamoDBの間でユーザー情報をどのように同期させ、管理画面を構築したのか。その設計思想と、実装中に直面した「型の不整合」という大きな壁の乗り越え方を解説します。

## 実装内容

### 1. DynamoDBでのUserモデル設計
Amplify Gen2では、`amplify/data/resource.ts`でデータモデルを定義します。Cognito上の認証情報（IDやメールアドレス）と、アプリ固有の属性（部署、役職、ステータス）を紐付けるため、`User`モデルを以下のように設計しました。

```typescript
// amplify/data/resource.ts
const schema = a.schema({
  User: a.model({
    // Cognitoのsub(UUID)を主キーとして使用
    userId: a.string().required(),
    name: a.string(),
    email: a.string().required(),
    employeeId: a.string(),
    department: a.string(),
    position: a.string(),
    groups: a.string().array(), // 所属ロール（Cognitoグループ名）の配列
    status: a.string(), // 'enabled' or 'disabled'
  })
  .identifier(['userId'])
  .secondaryIndexes((index) => [
    index('employeeId').name('byEmployeeId'),
  ])
  .authorization((allow) => [
    // 管理者のみが全操作可能
    allow.group('Admins').to(['create', 'read', 'update', 'delete']),
  ]),
});
```

### 2. 管理者用カスタムクエリ（Lambda連携）の定義
ユーザー一覧を表示する際、DynamoDBのデータだけでなく「現在のCognitoのステータス」も同時に取得したいケースがあります。これに対応するため、Cognito Admin APIを呼び出すLambda関数をバックエンドとして定義し、GraphQLから呼び出せるようにしました。

```typescript
// amplify/data/resource.ts (追加部分)
const schema = a.schema({
  // ... 前述のUserモデル

  // カスタムクエリの型定義
  CognitoUser: a.customType({
    Username: a.string(),
    UserStatus: a.string(),
    Enabled: a.boolean(),
    // ... 必要な属性
  }),

  listUsers: a.query()
    .returns(a.ref('CognitoUser').array())
    .handler(a.handler.function(listUsersFunction))
    .authorization((allow) => allow.group('Admins')),
});
```

### 3. フロントエンド管理画面のUI
React（Vite）側では、`@aws-amplify/ui-react`やMaterial UI（MUI）を使い、取得したユーザー情報をテーブル表示します。

```tsx
// src/features/user-management/routes/UserManagementPage.tsx (抜粋)
import { generateClient } from 'aws-amplify/data';
import type { Schema } from '../../../../amplify/data/resource';

const client = generateClient<Schema>();

export default function UserManagementPage() {
  const [users, setUsers] = useState<Schema['User']['type'][]>([]);

  useEffect(() => {
    const fetchUsers = async () => {
      const { data: userList } = await client.models.User.list();
      setUsers(userList);
    };
    fetchUsers();
  }, []);

  return (
    <TableContainer>
      <Table>
        <TableHead>
          <TableRow>
            <TableCell>氏名</TableRow>
            <TableCell>社員番号</TableRow>
            <TableCell>メールアドレス</TableRow>
            <TableCell>ステータス</TableRow>
          </TableRow>
        </TableHead>
        <TableBody>
          {users.map((user) => (
            <TableRow key={user.userId}>
              <TableCell>{user.name}</TableCell>
              <TableCell>{user.employeeId}</TableCell>
              <TableCell>{user.email}</TableCell>
              <TableCell>{user.status}</TableCell>
            </TableRow>
          ))}
        </TableBody>
      </Table>
    </TableContainer>
  );
}
```

## 遭遇した問題

### 1. `a.json()` 型のキャッシュによる不整合
当初、Lambda関数からの複雑なレスポンスを柔軟に受け取るため、戻り値の型を `a.json()` として定義していました。しかし、フロントエンドで以下のようなエラーが頻発しました。

> `TypeError: users.map is not a function`

調査の結果、AmplifyのGraphQLクライアントが内部的に持つキャッシュ機構により、初回のAPI取得時は「JSON文字列」、2回目以降は「パース済みのオブジェクト」が返されるという挙動の差異が発生していることが判明しました。

### 2. Lambda関数へのIAM権限付与漏れ
管理画面からユーザーを作成する際、Lambda関数内で `AdminCreateUser` などのCognito APIを呼び出しますが、適切なIAMポリシーが設定されていないため、`AccessDeniedException` が発生しました。

## 解決アプローチ

### 1. 厳密な型定義による「脱 a.json()」
動的なJSON型に頼るのではなく、GraphQLスキーマ内でレスポンス構造を `a.customType()` や `a.ref()` を使って厳密に定義する方針に転換しました。これにより、Amplifyクライアントが自動的に型安全なコードを生成し、フロントエンドでの型変換処理（`JSON.parse` 等）が不要になります。

### 2. `backend.ts` による中央集権的な権限管理
Amplify Gen2のベストプラクティスに基づき、`amplify/backend.ts` でバックエンド全体の司令塔として、各Lambda関数に必要なIAMポリシーを直接アタッチする構成を採用しました。

## 最終的な解決策

### 厳格なスキーマ定義
不具合の温床となっていた `a.json()` を廃止し、構造化されたカスタム型を定義しました。

```typescript
// amplify/data/resource.ts (最終形)
const schema = a.schema({
  ListUsersResponse: a.customType({
    users: a.ref('CognitoUser').array(),
    nextToken: a.string(),
  }),
  // ...
});
```

### 権限付与の正規化
`amplify/backend.ts` で、ユーザー管理を担うLambda関数に対し、Cognitoユーザープールへの操作権限を明示的に付与しました。

```typescript
// amplify/backend.ts
const userPoolArn = backend.auth.resources.userPool.userPoolArn;

backend.inviteUserFunction.resources.lambda.addToRolePolicy(
  new PolicyStatement({
    effect: Effect.ALLOW,
    actions: [
      'cognito-idp:AdminCreateUser',
      'cognito-idp:AdminEnableUser',
      'cognito-idp:AdminDisableUser'
    ],
    resources: [userPoolArn],
  })
);
```

### Lambda内でのAppSyncクライアント初期化
LambdaからDynamoDB（AppSync経由）を操作する場合、`authMode: 'iam'` を指定して初期化することで、関数の実行ロール（IAM）権限で安全にデータを操作できるようにしました。

```typescript
// Lambdaハンドラー内
const client = generateClient<Schema>({
  authMode: 'iam',
});
```

## 学んだこと

### 「スキーマが第一」の原則
Amplify Gen2開発において、`a.json()` による「なんとなく通る」設計は、後のフロントエンド開発で大きな負債になることを痛感しました。面倒でも最初にしっかりと型（Schema）を定義することが、結果的に開発スピードを最大化させる最短ルートです。

### 認証とDBの同期戦略
「Cognitoが持つべき情報（認証）」と「DynamoDBが持つべき情報（アプリ属性）」を明確に分けつつ、Cognitoの `sub` をキーとして同期させる設計は、将来的なスケールや属性追加に対して非常に柔軟であることを学びました。

### IAM権限の可視化
`backend.ts` を通じてインフラ構成（どの関数が何にアクセスできるか）をコードで記述することで、AWSコンソールを直接触らなくてもセキュリティ設定を管理・レビューできる IaC (Infrastructure as Code) の利点を再認識しました。

## 次回予告
ユーザー管理ができるようになると、次に必要になるのは「誰がどのフォルダを閲覧・編集できるか」という権限制御です。次回は、CognitoグループとDynamoDBを組み合わせた「役割ベースのアクセス制御（RBAC）」の実装について詳しく紹介します。
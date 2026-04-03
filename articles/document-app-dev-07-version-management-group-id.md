---
title: "AI×Amplify Gen2で月40万を1万に！非エンジニアの開発記 #7：堅牢なバージョン管理"
emoji: "📜"
type: "tech"
topics: ["amplify", "aws", "s3", "dynamodb", "database"]
published: false
---

## 概要
ドキュメント管理システムにおいて、最も重要な機能の一つが「バージョン管理（履歴管理）」です。いつ、誰が、どのファイルを更新したのかを正確に把握できなければ、業務システムとしての信頼性は保てません。

開発初期、私は「ファイル名」をキーにして履歴を紐付けていました。しかし、これでは「ファイル名が変更された瞬間に履歴が途切れる」という致命的な欠陥がありました。今回は、ファイル名という不確かな情報に依存せず、不変のID（`versionGroupId`）を用いることで実現した、堅牢なバージョン管理システムの構築記録を紹介します。

## 実装内容

### 1. データモデルへの `versionGroupId` の導入
まず、DynamoDBのスキーマ定義（`amplify/data/resource.ts`）を修正しました。個々のファイル（`FileMetadata`）がどの「ドキュメントの系列」に属するかを示す共通のIDとして、`versionGroupId` を追加しました。

```typescript
// amplify/data/resource.ts
const schema = a.schema({
  FileMetadata: a.model({
    titleId: a.string().required(),
    fileName: a.string().required(),
    filePath: a.string().required(), // S3上のパス
    version: a.string().required(),  // "1.0", "2.0" などの文字列
    isLatest: a.boolean().required(),
    fileSize: a.integer(),
    uploadedBy: a.string(),
    // バージョンをグループ化するための不変ID
    versionGroupId: a.string(),
  })
  .secondaryIndexes((index) => [
    index('titleId').name('byTitleId'),
    // このGSIにより、同一グループの履歴を高速に取得可能
    index('versionGroupId').name('byVersionGroupId'),
  ])
  .authorization((allow) => [
    allow.owner().to(['create', 'read', 'update', 'delete']),
    allow.group('Admins').to(['create', 'read', 'update', 'delete']),
  ]),
});
```

### 2. アップロード時のID生成・引き継ぎロジック
ファイルをアップロードする際、それが「新規ファイル」なのか「既存ファイルの更新」なのかによって、`versionGroupId` の扱いを切り替えるロジックを実装しました。

```typescript
// src/features/document-management/hooks/useFileUpload.ts (抜粋)
export const useFileUpload = () => {
  const uploadFile = useCallback(
    async (file: File, titleId: string, versionGroupId?: string) => {
      try {
        // versionGroupIdが渡されていなければ新規(uuid生成)、あればそれを引き継ぐ(更新)
        const currentVersionGroupId = versionGroupId || uuidv4();

        // バージョン番号の自動採番（既存がある場合は最新+1.0など）
        const nextVersion = await generateNextVersion(currentVersionGroupId);

        // S3へアップロード
        const filePath = `public/${titleId}/${Date.now()}_${file.name}`;
        await uploadData({ path: filePath, data: file }).result;

        // DynamoDBにメタデータを保存（versionGroupIdを紐付ける）
        await saveFileMetadata({
          filePath,
          titleId,
          fileName: file.name,
          version: nextVersion,
          versionGroupId: currentVersionGroupId,
          isLatest: true
        });

        return { success: true };
      } catch (error) {
        console.error('Upload Error:', error);
        return { success: false, error };
      }
    },
    [saveFileMetadata]
  );
  // ...
};
```

## 遭遇した問題

### 1. ファイル名変更による「履歴の断絶」
当初の実装では、`fileName` と `titleId` が一致するものを同一ドキュメントの履歴として表示していました。しかし、ユーザーが「重要書類_修正版.pdf」を「重要書類_最終版.pdf」として上書きアップロード（バージョンアップ）した場合、システム上は別ドキュメントとして扱われ、古い履歴が表示されなくなる問題が発生しました。

### 2. Amplify Sandbox のスキーマ同期不全
`amplify/data/resource.ts` にフィールドを追加しても、`npx amplify sandbox` のホットリロードだけでは、DynamoDBのGSI（グローバルセカンダリインデックス）が正しく構築されないケースがありました。これにより、新しく追加した `versionGroupId` で検索をかけると API エラー（`ValidationException`）が返ってくる状態に陥りました。

### 3. バージョン番号の型制約
バージョンを `number` 型（数値）で管理していましたが、「1.10」と「1.1」が数値として同一視されてしまったり、ソート順が意図通りにならないといった問題に直面しました。

## 解決アプローチ

### 1. 「変わる値」をキーにしない設計
AI（Gemini）との議論を通じて、「ファイル名はメタデータ（属性）に過ぎない」という結論に達しました。ドキュメントのアイデンティティを保つために、ファイル名とは一切関係のない `uuid` を `versionGroupId` として割り振るアーキテクチャに転換しました。

### 2. サンドボックス環境の完全再起動
スキーマ同期の問題については、一度 `npx amplify sandbox` を停止し、再起動することで AWS 側のインフラ定義（CloudFormation）を強制的に更新させました。それでも反映されない場合は、一度 `amplify sandbox delete` で環境をリセットすることで解決を図りました。

### 3. バージョン番号の文字列化とセマンティック管理
バージョン管理を数値ではなく `string` 型に変更しました。これにより、「1.0.1」のような柔軟な表記が可能になり、フロントエンドでのソートロジックを文字列ベース（またはドット区切りの数値変換）で制御することで正確な順序を担保しました。

## 最終的な解決策

### 不変IDによる履歴の紐付け
`versionGroupId` を主軸とした設計により、ファイル名が「旧名.pdf」から「新名.pdf」に変わっても、同じ ID を持つレコードとして履歴画面に一貫して表示されるようになりました。

```mermaid
graph LR
    subgraph DynamoDB (FileMetadata)
        A[Ver 1.0 / file_A.pdf / versionGroupId: uuid-123]
        B[Ver 2.0 / file_B.pdf / versionGroupId: uuid-123]
        C[Ver 3.0 / file_C.pdf / versionGroupId: uuid-123]
    end

    A --- B
    B --- C
    Note over A,C: ファイル名が変わってもIDが共通なので履歴として繋がる
```

### バージョン履歴表示のUI（React）
`versionGroupId` を使って特定のドキュメント系列を全取得し、新しい順に並べることで、ユーザーが過去のどの時点のファイルもダウンロードできる画面を実現しました。

```tsx
// src/features/document-management/components/views/VersionsView.tsx (抜粋)
export const VersionsView = ({ versionGroupId }: { versionGroupId: string }) => {
  const [history, setHistory] = useState<FileItem[]>([]);

  useEffect(() => {
    const loadHistory = async () => {
      // versionGroupIdでGSIクエリを実行
      const { data } = await client.models.FileMetadata.listByVersionGroupId({
        versionGroupId: versionGroupId
      });
      // 日付順にソートしてセット
      setHistory(data.sort((a, b) => b.createdAt.localeCompare(a.createdAt)));
    };
    loadHistory();
  }, [versionGroupId]);

  return (
    <List>
      {history.map((file) => (
        <ListItem key={file.id}>
          <ListItemText 
            primary={`Ver ${file.version}: ${file.fileName}`} 
            secondary={`更新日: ${new Date(file.createdAt).toLocaleString()}`} 
          />
          <Button onClick={() => handleDownload(file.path)}>ダウンロード</Button>
        </ListItem>
      ))}
    </List>
  );
};
```

## 学んだこと

### データベース設計の鉄則：「変わる値」をキーにしない
非エンジニアとして最も学びが大きかったのは、データベース設計における「一貫性」の考え方です。ファイル名のようにユーザーがいつでも変更できる値を、データの紐付け（リレーション）に使用してはいけないという、プログラミングの基本原則を身をもって体験しました。

### AIへの「データ構造」の相談
AIに「コード」を書いてもらう前に、「データ構造（スキーマ）」をどうすべきか相談することが、手戻りを防ぐ最大の近道です。「ファイル名が変わった時どうする？」という問いかけに対し、AIが `versionGroupId` のような抽象的な ID の導入を提案してくれたことが、今回の成功の鍵でした。

### インフラ変更の「重さ」
Amplify Gen2は非常に便利ですが、背後にある DynamoDB のインデックス（GSI）作成など、AWS 側の変更には時間がかかったり、同期が不安定になったりすることがあります。エラーが出た際に、コードを疑う前に「一度環境をリセットして待つ」という、クラウド開発特有の作法を学びました。

## 次回予告
バージョン管理という「裏側のロジック」が固まりました。次は、ユーザーが実際に触れる部分の利便性、すなわち UI/UX の向上です。大量のファイルを一括で ZIP ダウンロードしたり、ブラウザ上で PDF や画像を直接プレビューしたりといった、「かゆいところに手が届く」機能の実装について紹介します。
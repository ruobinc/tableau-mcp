# Tableau MCP 機能解説ガイド

## 概要

Tableau MCPは、AIアプリケーション（Claude、Cursor、VS Codeなど）とTableauを統合するためのMCP（Model Context Protocol）サーバーです。このサーバーを使うことで、AIアシスタントがTableauのデータソースにクエリを実行したり、ワークブックやビューの情報を取得したりできるようになります。

## アーキテクチャ

```
MCPクライアント（Claude Desktop等）
    ↓
Tableau MCPサーバー
    ↓
Tableau REST API / VizQL Data Service / Metadata API
    ↓
Tableau Server / Tableau Cloud
```

## 認証の仕組み

### 1. PAT認証（Personal Access Token）
- 環境変数で`PAT_NAME`と`PAT_VALUE`を設定
- 初回リクエスト時にTableau ServerにPATを使って認証
- セッショントークンを取得

### 2. JWT認証（JSON Web Token）
- 各ツールは必要なスコープ（権限）を定義
- 例：`tableau:content:read`、`tableau:viz_data_service:read`
- JWTトークンを生成して、細かい権限管理を実現

**コード例**（src/restApiInstance.ts:79-119）:
```typescript
// useRestApi関数が認証を管理
// 1. PAT認証でサインイン
const signInResult = await api.authenticationMethods.signIn(credentials);

// 2. 必要に応じてJWTトークンを取得
if (jwtScopes) {
  const jwt = await api.authenticationMethods.getJwt(jwtScopes);
}
```

## 主要機能の詳細解説

### 1. データソース機能

#### 1.1 list-datasources（データソース一覧取得）
**目的**: Tableauサイト上の公開データソースの一覧を取得

**ロジック**:
1. REST APIの`/sites/{siteId}/datasources`エンドポイントを呼び出し
2. オプションでフィルタ条件を適用（名前、プロジェクト名、作成日など）
3. ページネーション処理でデータを取得
4. 結果を返す

**使用例**:
```javascript
// 「Finance」プロジェクト内のデータソースを検索
filter: "projectName:eq:Finance"

// 2023年以降に作成されたデータソース
filter: "createdAt:gt:2023-01-01T00:00:00Z"
```

**コードロジック**（src/tools/listDatasources/listDatasources.ts:86-114）:
```typescript
const datasources = await useRestApi({
  callback: async (restApi) => {
    // ページネーション処理
    const datasources = await paginate({
      getDataFn: async (pageConfig) => {
        // REST APIでデータソース一覧を取得
        const { pagination, datasources: data } =
          await restApi.datasourcesMethods.listDatasources({
            siteId: restApi.siteId,
            filter: validatedFilter,
            pageSize: pageConfig.pageSize,
            pageNumber: pageConfig.pageNumber,
          });
        return { pagination, data };
      },
    });
    return datasources;
  },
});
```

#### 1.2 get-datasource-metadata（メタデータ取得）
**目的**: データソースのフィールド情報（カラム、データ型、説明など）を取得

**ロジック**:
1. **VizQL Data Service API**を呼び出してベースメタデータを取得
2. **Metadata API**（GraphQL）で追加情報を取得
   - フィールドの説明
   - データカテゴリ
   - セマンティックロール
   - 計算フィールドの数式など
3. 2つのAPIの結果を統合して返す

**コードロジック**（src/tools/getDatasourceMetadata/getDatasourceMetadata.ts:115-150）:
```typescript
// ステップ1: VizQL Data Serviceからベースメタデータ取得
const readMetadataResult = await restApi.vizqlDataServiceMethods.readMetadata({
  datasource: { datasourceLuid },
});

// ステップ2: Metadata API（GraphQL）から追加情報取得
const listFieldsResult = await restApi.metadataMethods.graphql(query);

// ステップ3: 2つの結果を統合
return Ok(combineFields(readMetadataResult.value, listFieldsResult));
```

**GraphQLクエリ例**:
```graphql
query datasourceFieldInfo {
  publishedDatasources(filter: { luid: "datasource-id" }) {
    name
    description
    fields {
      name
      dataType
      role
      aggregation
      formula  # 計算フィールドの場合
    }
  }
}
```

#### 1.3 query-datasource（データクエリ実行）
**目的**: データソースに対してクエリを実行してデータを取得

**ロジック**:
1. クエリパラメータのバリデーション
   - フィールド名の検証
   - フィルタ値の検証
2. VizQL Data Service APIの`/queryDatasource`エンドポイントを呼び出し
3. クエリ結果を返す

**クエリ構造**:
```typescript
{
  datasource: {
    datasourceLuid: "データソースID"
  },
  query: {
    fields: [
      { fieldName: "State", function: "NONE" },
      { fieldName: "Sales", function: "SUM" }
    ],
    filters: [
      {
        fieldName: "Order Date",
        filterType: "RANGE",
        values: { min: "2025-01-01", max: "2025-12-31" }
      }
    ],
    orderBy: [
      { fieldName: "Sales", direction: "DESC" }
    ],
    limit: 5
  }
}
```

**バリデーション処理**（src/tools/queryDatasource/queryDatasource.ts:85-102）:
```typescript
// フィルタ値のバリデーション
const filterValidationResult = await validateFilterValues(
  server,
  query,
  restApi.vizqlDataServiceMethods,
  datasource,
);

if (filterValidationResult.isErr()) {
  // エラーメッセージを返す
  return new Err({ type: 'filter-validation', message: errorMessage });
}
```

### 2. コンテンツ探索機能

#### 2.1 search-content（コンテンツ検索）
**目的**: ワークブック、ビュー、データソースなど、すべてのコンテンツタイプを横断的に検索

**ロジック**:
1. 検索条件を構築（検索語、フィルタ、並び順）
2. REST APIの`/searchContent`エンドポイントを呼び出し
3. 関連度スコアまたは指定した条件で結果をソート
4. 簡略化した結果を返す

**検索パラメータ**:
- `terms`: 検索キーワード
- `filter`: コンテンツタイプ、所有者、更新日でフィルタ
- `orderBy`: 並び順（閲覧数、最終更新日など）
- `limit`: 取得件数（最大2000）

**並び順オプション**:
- `hitsTotal`: 総閲覧数
- `hitsSmallSpanTotal`: 直近1ヶ月の閲覧数
- `hitsMediumSpanTotal`: 直近3ヶ月の閲覧数
- `hitsLargeSpanTotal`: 直近1年の閲覧数

**コードロジック**（src/tools/contentExploration/searchContent.ts:61-88）:
```typescript
const orderByString = orderBy ? buildOrderByString(orderBy) : undefined;
const filterString = filter ? buildFilterString(filter) : undefined;

const response = await restApi.contentExplorationMethods.searchContent({
  terms,
  page: 0,
  limit: limit ?? 100,
  orderBy: orderByString,
  filter: filterString,
});

return reduceSearchContentResponse(response);
```

### 3. ビュー機能

#### 3.1 get-view-image（ビュー画像取得）
**目的**: Tableauビューの画像（PNG）を取得

**ロジック**:
1. REST APIの`/views/{viewId}/image`エンドポイントを呼び出し
2. 幅、高さ、解像度を指定可能
3. PNGデータをBase64エンコードしてツール結果として返す

**コードロジック**（src/tools/views/getViewImage.ts:28-54）:
```typescript
const pngData = await restApi.viewsMethods.queryViewImage({
  viewId,
  siteId: restApi.siteId,
  width: width ?? 800,
  height: height ?? 800,
  resolution: 'high',
});

// PNGデータをBase64エンコード
return convertPngDataToToolResult(pngData);
```

#### 3.2 get-view-data（ビューデータ取得）
**目的**: ビューの基盤となるデータを取得

**ロジック**:
1. REST APIの`/views/{viewId}/data`エンドポイントを呼び出し
2. CSV形式でデータを取得
3. 結果を返す

#### 3.3 list-views（ビュー一覧取得）
**目的**: サイト上のビューの一覧を取得

**ロジック**:
- データソース一覧と同様のフィルタリング・ページネーション処理
- ビュー名、プロジェクト名、更新日などでフィルタ可能

### 4. ワークブック機能

#### 4.1 get-workbook（ワークブック情報取得）
**目的**: ワークブックの詳細情報を取得

**ロジック**:
1. REST APIの`/workbooks/{workbookId}`エンドポイントを呼び出し
2. ワークブック情報を返す

#### 4.2 list-workbooks（ワークブック一覧取得）
**目的**: サイト上のワークブックの一覧を取得

**ロジック**:
- フィルタリング・ページネーション処理
- ワークブック名、プロジェクト名、所有者などでフィルタ可能

### 5. Pulse機能（メトリクス管理）

Pulse機能は、Tableauのメトリクス（KPI）管理システムとの統合を提供します。

#### 5.1 list-all-pulse-metric-definitions（メトリクス定義一覧）
**目的**: サイト上のすべてのメトリクス定義を取得

#### 5.2 list-pulse-metrics-from-metric-definition-id（メトリクス値取得）
**目的**: 特定のメトリクス定義から実際のメトリクス値を取得

#### 5.3 generate-pulse-metric-value-insight-bundle（インサイト生成）
**目的**: メトリクス値に対するインサイト（変化の説明など）を生成

## エラーハンドリング

Tableau MCPは包括的なエラーハンドリングを実装しています：

### 1. バリデーションエラー
```typescript
// フィールド名のバリデーション
if (!metadata.fields.some(f => f.fieldName === field.fieldName)) {
  throw new Error(`フィールド '${field.fieldName}' が見つかりません`);
}
```

### 2. API エラー
```typescript
if (result.isErr()) {
  return new Err({
    type: 'tableau-error',
    error: result.error,
  });
}
```

### 3. 機能無効エラー
```typescript
if (!config.enableVizqlDataService) {
  return getVizqlDataServiceDisabledError();
}
```

## ページネーション処理

大量のデータを効率的に取得するためのページネーション実装：

**コードロジック**（src/utils/paginate.ts）:
```typescript
const paginate = async ({ pageConfig, getDataFn }) => {
  let allData = [];
  let currentPage = 1;
  let hasMore = true;

  while (hasMore) {
    const { data, pagination } = await getDataFn({
      pageNumber: currentPage,
      pageSize: pageConfig.pageSize,
    });

    allData = [...allData, ...data];

    // 制限に達したか、これ以上データがない場合は終了
    hasMore = allData.length < pageConfig.limit &&
              pagination.hasMore;
    currentPage++;
  }

  return allData;
};
```

## セキュリティ機能

### 1. シークレットマスキング
ログに機密情報が出力されないようマスキング処理を実装：

**コードロジック**（src/logging/secretMask.ts）:
```typescript
const maskSecrets = (text: string, secrets: string[]): string => {
  let masked = text;
  for (const secret of secrets) {
    if (secret) {
      masked = masked.replaceAll(secret, '***');
    }
  }
  return masked;
};
```

### 2. JWT スコープベース認可
各ツールは最小限必要なスコープのみを要求：
- `tableau:content:read` - コンテンツの読み取り
- `tableau:viz_data_service:read` - データクエリ実行
- `tableau:views:download` - ビュー画像のダウンロード

## 設定オプション

環境変数で動作をカスタマイズ可能：

```bash
# 必須設定
SERVER=https://tableau-server.com
SITE_NAME=my_site
PAT_NAME=my_pat
PAT_VALUE=pat_secret_value

# オプション設定
INCLUDE_TOOLS=query-datasource,list-datasources  # 有効にするツール
EXCLUDE_TOOLS=pulse-*  # 無効にするツール
MAX_RESULT_LIMIT=1000  # 最大取得件数
DISABLE_QUERY_DATASOURCE_FILTER_VALIDATION=false  # フィルタ検証の無効化
DISABLE_METADATA_API_REQUESTS=false  # Metadata API無効化
```

## パフォーマンス最適化

### 1. レスポンスの簡略化
APIから取得した詳細なレスポンスを、必要な情報のみに絞り込み：

```typescript
const reduceSearchContentResponse = (response) => {
  return response.results.map(item => ({
    id: item.id,
    name: item.name,
    type: item.contentType,
    viewCount: item.hitsTotal,
    // 不要な情報は除外
  }));
};
```

### 2. 条件付きAPI呼び出し
必要な場合のみ追加APIを呼び出し：

```typescript
if (config.disableMetadataApiRequests) {
  // Metadata APIをスキップ
  return simplifyReadMetadataResult(readMetadataResult.value);
}
```

## まとめ

Tableau MCPの主要な設計原則：

1. **モジュール性**: 各ツールは独立して動作
2. **型安全性**: Zodスキーマによる厳密な型定義
3. **エラーハンドリング**: 包括的なエラー処理とバリデーション
4. **セキュリティ**: スコープベース認可とシークレットマスキング
5. **パフォーマンス**: ページネーションとレスポンス最適化
6. **拡張性**: 新しいツールを簡単に追加可能

これらの機能により、AIアプリケーションはTableauと安全かつ効率的に統合できます。

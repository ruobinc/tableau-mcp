# Tableau MCP 実践例と処理フロー

## 目次
1. [実践例：データ分析クエリの実行](#実践例データ分析クエリの実行)
2. [処理フロー図](#処理フロー図)
3. [コード詳細解説](#コード詳細解説)
4. [トラブルシューティング](#トラブルシューティング)

---

## 実践例：データ分析クエリの実行

### シナリオ
「Superstoreデータソースから、2025年の売上トップ5の州を取得したい」

### ステップバイステップ

#### Step 1: データソースを検索
```javascript
// ツール: list-datasources
{
  "filter": "name:eq:Superstore"
}
```

**レスポンス:**
```json
[
  {
    "id": "abc123-datasource-luid",
    "name": "Superstore",
    "projectName": "Sales Analytics",
    "createdAt": "2024-01-15T10:30:00Z"
  }
]
```

#### Step 2: データソースのメタデータを取得
```javascript
// ツール: get-datasource-metadata
{
  "datasourceLuid": "abc123-datasource-luid"
}
```

**レスポンス:**
```json
{
  "datasourceName": "Superstore",
  "fields": [
    {
      "fieldName": "State",
      "dataType": "STRING",
      "role": "DIMENSION"
    },
    {
      "fieldName": "Sales",
      "dataType": "REAL",
      "role": "MEASURE",
      "defaultAggregation": "SUM"
    },
    {
      "fieldName": "Order Date",
      "dataType": "DATE",
      "role": "DIMENSION"
    }
  ]
}
```

#### Step 3: クエリを実行
```javascript
// ツール: query-datasource
{
  "datasourceLuid": "abc123-datasource-luid",
  "query": {
    "fields": [
      {
        "fieldName": "State",
        "function": "NONE"
      },
      {
        "fieldName": "Sales",
        "function": "SUM"
      }
    ],
    "filters": [
      {
        "fieldName": "Order Date",
        "filterType": "RANGE",
        "values": {
          "min": "2025-01-01",
          "max": "2025-12-31"
        }
      }
    ],
    "orderBy": [
      {
        "fieldName": "Sales",
        "direction": "DESC"
      }
    ],
    "limit": 5
  }
}
```

**レスポンス:**
```json
{
  "data": [
    { "State": "California", "Sales": 146388.34 },
    { "State": "New York", "Sales": 93922.99 },
    { "State": "Washington", "Sales": 65539.90 },
    { "State": "Texas", "Sales": 43421.76 },
    { "State": "Pennsylvania", "Sales": 42688.31 }
  ]
}
```

---

## 処理フロー図

### 1. 認証フロー

```
┌─────────────────┐
│ MCPクライアント  │
└────────┬────────┘
         │ 1. ツール呼び出し
         ▼
┌─────────────────┐
│ Tableau MCP     │
│ Server          │
└────────┬────────┘
         │ 2. useRestApi()呼び出し
         ▼
┌─────────────────┐
│ 認証チェック     │◄─── トークンキャッシュ確認
└────────┬────────┘
         │ 3. トークンなし or 期限切れ
         ▼
┌─────────────────┐
│ PAT認証実行      │
│ POST /auth/     │
│ signin          │
└────────┬────────┘
         │ 4. セッショントークン取得
         ▼
┌─────────────────┐
│ JWT取得          │
│ (必要なスコープ) │
└────────┬────────┘
         │ 5. JWTトークン取得
         ▼
┌─────────────────┐
│ API呼び出し実行  │
└────────┬────────┘
         │ 6. レスポンス
         ▼
┌─────────────────┐
│ MCPクライアント  │
└─────────────────┘
```

### 2. query-datasource の詳細フロー

```
┌─────────────────────────────────────────────┐
│ 1. クエリリクエスト受信                        │
│    - datasourceLuid                         │
│    - query (fields, filters, orderBy, etc)  │
└──────────────────┬──────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────┐
│ 2. データソースLUIDバリデーション               │
│    validateDatasourceLuid()                 │
└──────────────────┬──────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────┐
│ 3. 認証処理                                  │
│    - PAT認証                                │
│    - JWTトークン取得                         │
│      スコープ: tableau:viz_data_service:read│
└──────────────────┬──────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────┐
│ 4. フィルタバリデーション（オプション）          │
│    validateFilterValues()                   │
│    - SET/MATCHフィルタの値を検証              │
│    - 存在しない値の場合エラー                 │
└──────────────────┬──────────────────────────┘
                   │ バリデーションOK
                   ▼
┌─────────────────────────────────────────────┐
│ 5. VizQL Data Service API呼び出し           │
│    POST /vizqlDataService/queryDatasource   │
│    {                                        │
│      datasource: { datasourceLuid, ... },   │
│      query: { fields, filters, ... },       │
│      options: { returnFormat: "OBJECTS" }   │
│    }                                        │
└──────────────────┬──────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────┐
│ 6. レスポンス処理                            │
│    - 成功: データを返す                      │
│    - エラー: handleQueryDatasourceError()   │
└──────────────────┬──────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────┐
│ 7. ログ記録                                  │
│    - リクエストID                           │
│    - 実行時間                               │
│    - 成功/失敗                              │
└──────────────────┬──────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────┐
│ 8. クライアントにレスポンス返却               │
└─────────────────────────────────────────────┘
```

### 3. get-datasource-metadata の二段階取得フロー

```
┌────────────────────────────────────────┐
│ メタデータ取得リクエスト                 │
└───────────────┬────────────────────────┘
                │
                ▼
┌────────────────────────────────────────┐
│ 【第1段階】VizQL Data Service API      │
│ POST /vizqlDataService/readMetadata    │
│                                        │
│ 取得情報:                               │
│ - フィールド名                          │
│ - データ型                              │
│ - デフォルト集計関数                     │
│ - カラムクラス (COLUMN/CALCULATION等)   │
└───────────────┬────────────────────────┘
                │
                ▼
┌────────────────────────────────────────┐
│ Metadata API無効チェック                │
│ config.disableMetadataApiRequests?     │
└───────┬───────────┬────────────────────┘
        │ YES       │ NO
        │           ▼
        │    ┌────────────────────────────────────┐
        │    │ 【第2段階】Metadata API (GraphQL)  │
        │    │ POST /api/metadata/graphql         │
        │    │                                    │
        │    │ GraphQLクエリ:                     │
        │    │ - フィールド説明                    │
        │    │ - データカテゴリ                    │
        │    │ - セマンティックロール               │
        │    │ - 計算式 (計算フィールドの場合)      │
        │    │ - その他の拡張メタデータ             │
        │    └────────────┬───────────────────────┘
        │                 │
        │                 ▼
        │    ┌────────────────────────────────────┐
        │    │ 結果の統合                          │
        │    │ combineFields()                    │
        │    │ - VizQL結果をベースに               │
        │    │ - Metadata APIの情報を追加          │
        │    └────────────┬───────────────────────┘
        │                 │
        └────────┬────────┘
                 │
                 ▼
┌────────────────────────────────────────┐
│ 統合されたメタデータを返却               │
│                                        │
│ 例:                                    │
│ {                                      │
│   fieldName: "Sales",                  │
│   dataType: "REAL",                    │
│   description: "Total sales amount",   │
│   role: "MEASURE",                     │
│   aggregation: "SUM"                   │
│ }                                      │
└────────────────────────────────────────┘
```

---

## コード詳細解説

### 1. ツールの登録プロセス

**server.ts の registerTools メソッド:**
```typescript
registerTools = (): void => {
  // 1. 設定から有効なツールをフィルタリング
  const toolsToRegister = this._getToolsToRegister();

  // 2. 各ツールを登録
  for (const { name, description, paramsSchema, annotations, callback } of toolsToRegister) {
    this.tool(name, description, paramsSchema, annotations, callback);
  }
};

private _getToolsToRegister = (): Array<Tool<any>> => {
  const { includeTools, excludeTools } = getConfig();

  // 3. すべてのツールファクトリーからツールインスタンスを作成
  const tools = toolFactories.map((factory) => factory(this));

  // 4. includeTools/excludeToolsに基づいてフィルタリング
  const toolsToRegister = tools.filter((tool) => {
    if (includeTools.length > 0) {
      return includeTools.includes(tool.name);
    }
    if (excludeTools.length > 0) {
      return !excludeTools.includes(tool.name);
    }
    return true;
  });

  return toolsToRegister;
};
```

### 2. REST API呼び出しのラッパー（useRestApi）

**restApiInstance.ts:**
```typescript
export const useRestApi = async <T>({
  config,
  requestId,
  server,
  jwtScopes,
  callback,
}: UseRestApiParams<T>): Promise<T> => {
  // 1. 認証情報の準備
  const credentials = {
    server: config.server,
    siteName: config.siteName,
    personalAccessTokenName: config.personalAccessTokenName,
    personalAccessTokenSecret: config.personalAccessTokenSecret,
  };

  try {
    // 2. REST APIインスタンス作成（キャッシュから取得または新規作成）
    const api = await getRestApiInstance(credentials);

    // 3. サインイン処理（必要な場合）
    if (!api.isSignedIn) {
      const signInResult = await api.authenticationMethods.signIn(credentials);
      api.updateSessionAfterSignIn(signInResult);
    }

    // 4. JWTトークン取得（スコープが指定されている場合）
    if (jwtScopes && jwtScopes.length > 0) {
      const jwt = await api.authenticationMethods.getJwt(jwtScopes);
      api.setJwt(jwt);
    }

    // 5. コールバック実行（実際のAPI呼び出し）
    const result = await callback(api);

    return result;
  } catch (error) {
    // 6. エラーハンドリング
    server.sendLoggingMessage({
      level: 'error',
      data: `API call failed: ${error.message}`,
    });
    throw error;
  }
};
```

### 3. バリデーション処理の詳細

**validateFilterValues.ts:**
```typescript
export const validateFilterValues = async (
  server: Server,
  query: Query,
  vizqlMethods: VizqlDataServiceMethods,
  datasource: Datasource,
): Promise<Result<void, ValidationError[]>> => {
  const errors: ValidationError[] = [];

  // 1. SET型とMATCH型フィルタを抽出
  const setAndMatchFilters = query.filters?.filter(
    (f) => f.filterType === 'SET' || f.filterType === 'MATCH'
  ) ?? [];

  // 2. 各フィルタの値を検証
  for (const filter of setAndMatchFilters) {
    // 3. フィールドの有効な値を取得
    const validValuesResult = await vizqlMethods.getDomainValues({
      datasource,
      fieldName: filter.fieldName,
    });

    if (validValuesResult.isErr()) {
      errors.push({
        fieldName: filter.fieldName,
        message: `Failed to get valid values for field`,
      });
      continue;
    }

    const validValues = validValuesResult.value;
    const filterValues = Array.isArray(filter.values) ? filter.values : [filter.values];

    // 4. フィルタ値が有効な値リストに含まれるかチェック
    for (const value of filterValues) {
      if (!validValues.includes(value)) {
        errors.push({
          fieldName: filter.fieldName,
          message: `Invalid value '${value}'. Valid values: ${validValues.join(', ')}`,
        });
      }
    }
  }

  // 5. エラーがあればErr、なければOkを返す
  return errors.length > 0 ? new Err(errors) : new Ok(undefined);
};
```

### 4. ページネーション処理

**paginate.ts:**
```typescript
export const paginate = async <T>({
  pageConfig,
  getDataFn,
}: PaginateParams<T>): Promise<T[]> => {
  let allData: T[] = [];
  let currentPage = 1;
  let hasMore = true;

  while (hasMore) {
    // 1. 現在のページのデータを取得
    const { data, pagination } = await getDataFn({
      pageNumber: currentPage,
      pageSize: pageConfig.pageSize ?? 100,
    });

    // 2. データを蓄積
    allData = [...allData, ...data];

    // 3. 終了条件チェック
    const reachedLimit = pageConfig.limit && allData.length >= pageConfig.limit;
    const noMoreData = !pagination.hasMore;

    hasMore = !reachedLimit && !noMoreData;

    // 4. 次のページへ
    currentPage++;
  }

  // 5. limit以下に切り詰めて返す
  return pageConfig.limit ? allData.slice(0, pageConfig.limit) : allData;
};
```

---

## トラブルシューティング

### 問題 1: "VizQL Data Service is disabled"

**原因:**
- Tableau ServerでVizQL Data Serviceが有効になっていない
- または`DISABLE_VIZQL_DATA_SERVICE=true`が設定されている

**解決策:**
```bash
# 環境変数を確認
echo $DISABLE_VIZQL_DATA_SERVICE

# Tableau Server管理者に連絡してサービスを有効化してもらう
# または、設定ファイルで有効化
```

### 問題 2: "Invalid filter value"

**原因:**
- フィルタに存在しない値を指定している
- データ型が一致していない

**解決策:**
```javascript
// 1. まずメタデータを取得
await getDatasourceMetadata({ datasourceLuid });

// 2. getDomainValues で有効な値を確認（内部的にバリデーションで使用）
// 3. 正しい値を使用してクエリを再実行
```

### 問題 3: "Authentication failed"

**原因:**
- PATの名前または値が間違っている
- PATの権限が不足している
- PATの有効期限が切れている

**解決策:**
```bash
# 1. PAT情報を確認
echo $PAT_NAME
echo $PAT_VALUE  # 注意: 本番環境では表示しない

# 2. Tableau Serverで新しいPATを作成
# 必要な権限:
# - サイトの表示
# - コンテンツの表示
# - データソースのクエリ実行

# 3. 環境変数を更新
export PAT_NAME="new_pat_name"
export PAT_VALUE="new_pat_value"
```

### 問題 4: "Too many results"

**原因:**
- 結果件数が`MAX_RESULT_LIMIT`を超えている

**解決策:**
```bash
# 1. limitパラメータを指定
{
  "filter": "...",
  "limit": 100
}

# 2. または、MAX_RESULT_LIMITを増やす
export MAX_RESULT_LIMIT=5000
```

### 問題 5: パフォーマンスが遅い

**原因:**
- 大量のデータを取得している
- Metadata APIの呼び出しに時間がかかっている

**解決策:**
```bash
# 1. Metadata APIを無効化（メタデータ取得が不要な場合）
export DISABLE_METADATA_API_REQUESTS=true

# 2. フィルタバリデーションを無効化（パフォーマンス優先の場合）
export DISABLE_QUERY_DATASOURCE_FILTER_VALIDATION=true

# 3. limit、pageSize を適切に設定
{
  "pageSize": 50,
  "limit": 200
}
```

---

## ベストプラクティス

### 1. エラーハンドリング
```typescript
try {
  const result = await queryDatasource({ datasourceLuid, query });
  // 成功処理
} catch (error) {
  if (error.type === 'validation') {
    // バリデーションエラーの処理
  } else if (error.type === 'tableau-error') {
    // Tableau APIエラーの処理
  } else {
    // その他のエラー
  }
}
```

### 2. 段階的なデータ取得
```typescript
// 1. まず少量のデータで試す
const testResult = await queryDatasource({
  datasourceLuid,
  query: { ...query, limit: 10 }
});

// 2. 結果を確認してから全量取得
if (testResult.success) {
  const fullResult = await queryDatasource({
    datasourceLuid,
    query
  });
}
```

### 3. キャッシングの活用
```typescript
// メタデータは頻繁に変わらないのでキャッシュ推奨
const metadataCache = new Map();

const getMetadata = async (datasourceLuid) => {
  if (!metadataCache.has(datasourceLuid)) {
    const metadata = await getDatasourceMetadata({ datasourceLuid });
    metadataCache.set(datasourceLuid, metadata);
  }
  return metadataCache.get(datasourceLuid);
};
```

### 4. 適切なフィルタリング
```typescript
// 悪い例: 全データ取得後にフィルタリング
const allData = await queryDatasource({ datasourceLuid, query: {} });
const filtered = allData.filter(row => row.State === 'California');

// 良い例: サーバー側でフィルタリング
const data = await queryDatasource({
  datasourceLuid,
  query: {
    filters: [
      { fieldName: 'State', filterType: 'SET', values: ['California'] }
    ]
  }
});
```

---

以上が、Tableau MCPの実践的な使用例と処理フローの詳細解説です。

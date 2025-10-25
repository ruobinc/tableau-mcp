# Tableau MCP - メタデータ取得の完全解説

## 目次
1. [概要](#概要)
2. [なぜ2つのAPIを使うのか](#なぜ2つのapiを使うのか)
3. [処理フローの詳細](#処理フローの詳細)
4. [VizQL Data Service API（第1段階）](#vizql-data-service-api第1段階)
5. [Metadata API（第2段階）](#metadata-api第2段階)
6. [データ統合ロジック](#データ統合ロジック)
7. [パラメータ処理](#パラメータ処理)
8. [最適化とトークン削減](#最適化とトークン削減)
9. [エラーハンドリング](#エラーハンドリング)
10. [実践例](#実践例)

---

## 概要

`get-datasource-metadata`ツールは、Tableauデータソースのフィールド情報を取得します。このツールの特徴は、**2つの異なるAPI**を呼び出して、結果を統合することです。

```
┌──────────────────────────────────────────┐
│  get-datasource-metadata                 │
│                                          │
│  入力: datasourceLuid                    │
│  出力: 統合されたフィールドメタデータ       │
└──────────────────┬───────────────────────┘
                   │
         ┌─────────┴─────────┐
         ▼                   ▼
┌─────────────────┐  ┌─────────────────┐
│ VizQL Data      │  │ Metadata API    │
│ Service API     │  │ (GraphQL)       │
│                 │  │                 │
│ - 基本メタデータ │  │ - 拡張メタデータ │
│ - クエリ可能     │  │ - 説明文        │
│   フィールド     │  │ - セマンティック │
└─────────────────┘  └─────────────────┘
         │                   │
         └─────────┬─────────┘
                   ▼
         ┌─────────────────┐
         │ combineFields() │
         │ 統合処理        │
         └─────────────────┘
```

---

## なぜ2つのAPIを使うのか

### VizQL Data Service API の役割
- **クエリ実行可能なフィールド**のみを返す
- データ型、デフォルト集計関数など、**クエリに必須の情報**を提供
- レスポンスが速い
- **実際にクエリで使用できるフィールド名**を正確に取得

### Metadata API（GraphQL）の役割
- **ビジネスコンテキスト**を提供（フィールドの説明文など）
- セマンティックロール、データカテゴリなど、**AIがフィールドを理解するための情報**
- より詳細なメタデータ
- VizQL Data Serviceでは取得できない情報を補完

### なぜ両方必要？

**VizQL Data Serviceだけの場合:**
```json
{
  "fieldName": "Sales",
  "dataType": "REAL",
  "defaultAggregation": "SUM"
}
```
→ フィールドが何を表すのか分からない

**Metadata APIだけの場合:**
```json
{
  "name": "Sales",
  "description": "Total sales amount in USD",
  "role": "MEASURE"
}
```
→ クエリで使えるフィールド名が分からない、クエリに使えないフィールドも含まれる可能性

**両方を統合すると:**
```json
{
  "name": "Sales",
  "dataType": "REAL",
  "defaultAggregation": "SUM",
  "description": "Total sales amount in USD",
  "role": "MEASURE",
  "semanticRole": "CURRENCY"
}
```
→ **クエリ実行可能 + ビジネスコンテキスト** の両方を持つ完全な情報

---

## 処理フローの詳細

### メインの処理フロー

**src/tools/getDatasourceMetadata/getDatasourceMetadata.ts:105-160**

```typescript
callback: async ({ datasourceLuid }, { requestId }): Promise<CallToolResult> => {
  const config = getConfig();
  const query = getGraphqlQuery(datasourceLuid);

  return await getDatasourceMetadataTool.logAndExecute({
    requestId,
    args: { datasourceLuid },
    callback: async () => {
      return await useRestApi({
        config,
        requestId,
        server,
        jwtScopes: ['tableau:content:read', 'tableau:viz_data_service:read'],
        callback: async (restApi) => {
          // ========================================
          // 【第1段階】VizQL Data Service API呼び出し
          // ========================================
          const readMetadataResult = await restApi.vizqlDataServiceMethods.readMetadata({
            datasource: { datasourceLuid },
          });

          // エラーチェック
          if (readMetadataResult.isErr()) {
            return Err({ type: 'feature-disabled' });
          }

          // Metadata APIが無効の場合は早期リターン
          if (config.disableMetadataApiRequests) {
            return Ok(simplifyReadMetadataResult(readMetadataResult.value));
          }

          // ========================================
          // 【第2段階】Metadata API（GraphQL）呼び出し
          // ========================================
          let listFieldsResult: GraphQLResponse;

          try {
            listFieldsResult = await restApi.metadataMethods.graphql(query);
          } catch {
            // Metadata APIが利用できない場合は、VizQLの結果のみを返す
            return Ok(simplifyReadMetadataResult(readMetadataResult.value));
          }

          // ========================================
          // 【第3段階】2つの結果を統合
          // ========================================
          return Ok(combineFields(readMetadataResult.value, listFieldsResult));
        },
      });
    },
  });
}
```

### 処理フローの各段階

```
┌────────────────────────────────────────────────┐
│ 1. リクエスト受信                               │
│    datasourceLuid: "abc-123-def-456"          │
└────────────────┬───────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────┐
│ 2. JWTトークン取得                              │
│    スコープ:                                    │
│    - tableau:content:read                     │
│    - tableau:viz_data_service:read            │
└────────────────┬───────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────┐
│ 3. VizQL Data Service API呼び出し              │
│    POST /vizqlDataService/readMetadata        │
│    {                                          │
│      datasource: {                            │
│        datasourceLuid: "abc-123-def-456"      │
│      }                                        │
│    }                                          │
└────────────────┬───────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────┐
│ 4. VizQLレスポンス受信                          │
│    {                                          │
│      data: [                                  │
│        {                                      │
│          fieldCaption: "Sales",               │
│          dataType: "REAL",                    │
│          defaultAggregation: "SUM",           │
│          columnClass: "COLUMN"                │
│        },                                     │
│        ...                                    │
│      ],                                       │
│      extraData: {                             │
│        parameters: [...]                      │
│      }                                        │
│    }                                          │
└────────────────┬───────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────┐
│ 5. Metadata API無効チェック                     │
│    config.disableMetadataApiRequests?         │
└────┬───────────────────────────────┬──────────┘
     │ YES                           │ NO
     │                               │
     ▼                               ▼
┌─────────────────┐    ┌──────────────────────────┐
│ simplify して   │    │ 6. GraphQLクエリ生成      │
│ 早期リターン     │    │    query = `             │
│                 │    │      query {             │
│                 │    │        publishedDatasources│
│                 │    │        ...               │
│                 │    │      }                   │
│                 │    │    `                     │
└─────────────────┘    └──────────┬───────────────┘
                                  │
                                  ▼
                       ┌──────────────────────────┐
                       │ 7. Metadata API呼び出し  │
                       │    POST /api/metadata/   │
                       │         graphql          │
                       └──────────┬───────────────┘
                                  │
                                  ▼
                       ┌──────────────────────────┐
                       │ 8. GraphQLレスポンス受信 │
                       │    {                     │
                       │      data: {              │
                       │        publishedDatasources│
                       │        fields: [          │
                       │          {                │
                       │            name: "Sales", │
                       │            description:   │
                       │              "Total...",  │
                       │            role: "MEASURE"│
                       │          }                │
                       │        ]                  │
                       │      }                    │
                       │    }                      │
                       └──────────┬───────────────┘
                                  │
                                  ▼
                       ┌──────────────────────────┐
                       │ 9. combineFields()       │
                       │    統合処理               │
                       └──────────┬───────────────┘
                                  │
                 ┌────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────┐
│ 10. 統合結果を返却                              │
│     {                                         │
│       fields: [                               │
│         {                                     │
│           name: "Sales",                      │
│           dataType: "REAL",                   │
│           defaultAggregation: "SUM",          │
│           columnClass: "COLUMN",              │
│           description: "Total sales...",      │
│           role: "MEASURE",                    │
│           semanticRole: "CURRENCY"            │
│         }                                     │
│       ],                                      │
│       parameters: [...]                       │
│     }                                         │
└────────────────────────────────────────────────┘
```

---

## VizQL Data Service API（第1段階）

### readMetadata メソッドの実装

**src/sdks/tableau/methods/vizqlDataServiceMethods.ts:69-84**

```typescript
readMetadata = async (
  readMetadataRequest: z.infer<typeof ReadMetadataRequest>,
): Promise<Result<MetadataResponse, 'feature-disabled'>> => {
  try {
    // POST /vizqlDataService/readMetadata
    return Ok(await this._apiClient.readMetadata(readMetadataRequest, { ...this.authHeader }));
  } catch (error) {
    // 404エラー = VizQL Data Serviceが無効
    if (
      isErrorFromAlias(this._apiClient.api, 'readMetadata', error) &&
      error.response.status === 404
    ) {
      return Err('feature-disabled');
    }
    throw error;
  }
};
```

### リクエストの構造

**src/sdks/tableau/apis/vizqlDataServiceApi.ts:25-30**

```typescript
export const ReadMetadataRequest = z
  .object({
    datasource: Datasource,  // { datasourceLuid: string }
    options: QueryOptions.optional(),  // { returnFormat?, debug? }
  })
  .passthrough();
```

### レスポンスの構造

**src/sdks/tableau/apis/vizqlDataServiceApi.ts:116-124**

```typescript
export const MetadataOutput = z
  .object({
    data: z.array(FieldMetadata),  // フィールドの配列
    extraData: z.object({
      parameters: z.array(Parameter),  // パラメータの配列
    }),
  })
  .partial()
  .passthrough();
```

### FieldMetadata の詳細

**src/sdks/tableau/apis/vizqlDataServiceApi.ts:71-82**

```typescript
const FieldMetadata = z
  .object({
    fieldName: z.string(),              // 内部フィールド名
    fieldCaption: z.string(),           // 表示名（クエリで使用）
    dataType: DataType,                 // INTEGER, REAL, STRING, DATETIME, etc.
    defaultAggregation: Function,       // SUM, AVG, COUNT, etc.
    columnClass: z.enum([               // フィールドの種類
      'COLUMN',                         // 通常のカラム
      'BIN',                            // ビンフィールド
      'GROUP',                          // グループフィールド
      'CALCULATION',                    // 計算フィールド
      'TABLE_CALCULATION'               // テーブル計算
    ]),
    formula: z.string(),                // 計算式（計算フィールドの場合）
    logicalTableId: z.string(),         // 論理テーブルID
  })
  .partial()
  .passthrough();
```

### VizQL API レスポンス例

```json
{
  "data": [
    {
      "fieldName": "[Federated.1p9mqm80vqeqgk15h0vap1fhaxr9].[sum:Sales:qk]",
      "fieldCaption": "Sales",
      "dataType": "REAL",
      "defaultAggregation": "SUM",
      "columnClass": "COLUMN",
      "logicalTableId": "Federated.1p9mqm80vqeqgk15h0vap1fhaxr9"
    },
    {
      "fieldName": "[Federated.1p9mqm80vqeqgk15h0vap1fhaxr9].[none:State:nk]",
      "fieldCaption": "State",
      "dataType": "STRING",
      "defaultAggregation": "NONE",
      "columnClass": "COLUMN",
      "logicalTableId": "Federated.1p9mqm80vqeqgk15h0vap1fhaxr9"
    },
    {
      "fieldName": "[Calculation_123456789]",
      "fieldCaption": "Profit Ratio",
      "dataType": "REAL",
      "defaultAggregation": "SUM",
      "columnClass": "CALCULATION",
      "formula": "SUM([Profit]) / SUM([Sales])"
    }
  ],
  "extraData": {
    "parameters": [
      {
        "parameterCaption": "Year",
        "parameterType": "LIST",
        "dataType": "INTEGER",
        "value": 2024,
        "members": [2020, 2021, 2022, 2023, 2024, 2025]
      }
    ]
  }
}
```

---

## Metadata API（第2段階）

### GraphQLクエリの生成

**src/tools/getDatasourceMetadata/getDatasourceMetadata.ts:18-79**

```typescript
export const getGraphqlQuery = (datasourceLuid: string): string => `
  query datasourceFieldInfo {
    publishedDatasources(filter: { luid: "${datasourceLuid}" }) {
      name
      description
      owner {
        name
      }
      fields {
        name
        isHidden
        description
        descriptionInherited {
          attribute
          value
        }
        fullyQualifiedName
        __typename

        # AnalyticsField（すべてのフィールドタイプの基底）
        ... on AnalyticsField {
          __typename
        }

        # ColumnField（通常のカラム）
        ... on ColumnField {
          dataCategory      # QUANTITATIVE, ORDINAL, NOMINAL, etc.
          role              # DIMENSION, MEASURE
          dataType          # INTEGER, REAL, STRING, etc.
          defaultFormat     # 表示フォーマット
          semanticRole      # GEOGRAPHIC, TEMPORAL, etc.
          aggregation       # SUM, AVG, COUNT, etc.
          aggregationParam  # 集計パラメータ
        }

        # CalculatedField（計算フィールド）
        ... on CalculatedField {
          dataCategory
          role
          dataType
          defaultFormat
          semanticRole
          aggregation
          aggregationParam
          formula           # 計算式
          isAutoGenerated   # 自動生成フラグ
          hasUserReference  # ユーザー参照フラグ
        }

        # BinField（ビンフィールド）
        ... on BinField {
          dataCategory
          role
          dataType
          formula
          binSize           # ビンのサイズ
        }

        # GroupField（グループフィールド）
        ... on GroupField {
          dataCategory
          role
          dataType
          hasOther          # "その他"グループの有無
        }

        # CombinedSetField（組み合わせセットフィールド）
        ... on CombinedSetField {
          delimiter         # 区切り文字
          combinationType   # 組み合わせタイプ
        }
      }
    }
  }`;
```

### GraphQL APIの実装

**src/sdks/tableau/methods/metadataMethods.ts:28-30**

```typescript
graphql = async (query: string): Promise<GraphQLResponse> => {
  // POST /api/metadata/graphql
  return await this._apiClient.graphql({ query }, { ...this.authHeader });
};
```

### GraphQLレスポンスの型定義

**src/sdks/tableau/apis/metadataApi.ts:4-54**

```typescript
export const graphqlResponse = z.object({
  data: z.object({
    publishedDatasources: z.array(
      z.object({
        name: z.string().nullable(),
        description: z.string().nullable(),
        owner: z.object({
          name: z.string().nullable(),
        }),
        fields: z.array(
          z.object({
            name: z.string(),
            isHidden: z.boolean().nullable(),
            description: z.string().nullable(),
            descriptionInherited: z
              .array(
                z.object({
                  attribute: z.string(),
                  value: z.string().nullable(),
                }).nullable(),
              )
              .nullable(),
            fullyQualifiedName: z.string(),
            __typename: z.string(),
            // フィールドタイプごとのプロパティ
            dataCategory: z.string().nullish(),
            role: z.string().nullish(),
            dataType: z.string().nullish(),
            defaultFormat: z.string().nullish(),
            semanticRole: z.string().nullish(),
            aggregation: z.string().nullish(),
            aggregationParam: z.string().nullish(),
            formula: z.string().nullish(),
            isAutoGenerated: z.boolean().nullish(),
            hasUserReference: z.boolean().nullish(),
            binSize: z.number().nullish(),
            hasOther: z.boolean().nullish(),
            delimiter: z.string().nullish(),
            combinationType: z.string().nullish(),
          }),
        ),
      }),
    ),
  }),
});
```

### Metadata API レスポンス例

```json
{
  "data": {
    "publishedDatasources": [
      {
        "name": "Superstore",
        "description": "Sample Superstore dataset",
        "owner": {
          "name": "John Doe"
        },
        "fields": [
          {
            "name": "Sales",
            "isHidden": false,
            "description": "Total sales amount in USD",
            "descriptionInherited": null,
            "fullyQualifiedName": "[Superstore].[Sales]",
            "__typename": "ColumnField",
            "dataCategory": "QUANTITATIVE",
            "role": "MEASURE",
            "dataType": "REAL",
            "defaultFormat": "$#,##0.00",
            "semanticRole": "CURRENCY",
            "aggregation": "SUM",
            "aggregationParam": null
          },
          {
            "name": "State",
            "isHidden": false,
            "description": "US State name",
            "descriptionInherited": null,
            "fullyQualifiedName": "[Superstore].[State]",
            "__typename": "ColumnField",
            "dataCategory": "NOMINAL",
            "role": "DIMENSION",
            "dataType": "STRING",
            "defaultFormat": null,
            "semanticRole": "GEOGRAPHIC",
            "aggregation": null,
            "aggregationParam": null
          },
          {
            "name": "Profit Ratio",
            "isHidden": false,
            "description": "Calculated profit margin",
            "descriptionInherited": null,
            "fullyQualifiedName": "[Calculations].[Profit Ratio]",
            "__typename": "CalculatedField",
            "dataCategory": "QUANTITATIVE",
            "role": "MEASURE",
            "dataType": "REAL",
            "defaultFormat": "0.00%",
            "semanticRole": null,
            "aggregation": "SUM",
            "aggregationParam": null,
            "formula": "SUM([Profit]) / SUM([Sales])",
            "isAutoGenerated": false,
            "hasUserReference": true
          }
        ]
      }
    ]
  }
}
```

---

## データ統合ロジック

### combineFields 関数の詳細

**src/tools/getDatasourceMetadata/datasourceMetadataUtils.ts:115-208**

```typescript
export function combineFields(
  readMetadataResult: MetadataResponse,  // VizQL結果
  listFieldsResult: GraphQLResponse,     // Metadata API結果
): FieldsResult {
  const combinedFields: FieldsResult = {
    fields: [],
    parameters: [],
  };

  // ========================================
  // ステップ1: VizQLのデータがない場合の処理
  // ========================================
  if (!readMetadataResult.data) {
    // フォールバック: Metadata APIの結果のみを使用
    if (listFieldsResult.data.publishedDatasources[0]?.fields.length) {
      for (const field of listFieldsResult.data.publishedDatasources[0].fields) {
        const toPush: Field = { name: field.name };
        if (field.dataType) {
          toPush.dataType = field.dataType;
        }
        if (field.aggregation) {
          toPush.defaultAggregation = field.aggregation;
        }
        populateFieldWithAdditionalProperties(field, toPush);
        combinedFields.fields.push(toPush);
      }
    }
    return combinedFields;
  }

  // ========================================
  // ステップ2: VizQLの結果をベースにフィールドを作成
  // ========================================
  // 重要: VizQLの結果のみがクエリで使用可能なフィールド
  for (const field of readMetadataResult.data) {
    const toPush: Field = {
      name: field.fieldCaption,           // 表示名を使用
      dataType: field.dataType,
      columnClass: field.columnClass,
    };

    if (field.defaultAggregation) {
      toPush.defaultAggregation = field.defaultAggregation;
    }

    if (field.formula) {
      toPush.formula = field.formula;
    }

    combinedFields.fields.push(toPush);
  }

  // ========================================
  // ステップ3: パラメータの追加
  // ========================================
  if (readMetadataResult.extraData?.parameters) {
    for (const parameter of readMetadataResult.extraData.parameters) {
      const toPush: Parameter = {
        name: parameter.parameterCaption,
        parameterType: parameter.parameterType,
        dataType: parameter.dataType,
        value: parameter.value,
      };

      // パラメータタイプごとの追加プロパティ
      if (parameter.parameterType === 'LIST' && parameter.members) {
        toPush.members = parameter.members;
      } else if (parameter.parameterType === 'QUANTITATIVE_DATE') {
        toPush.minDate = parameter.minDate;
        toPush.maxDate = parameter.maxDate;
        toPush.periodValue = parameter.periodValue;
        toPush.periodType = parameter.periodType;
      } else if (parameter.parameterType === 'QUANTITATIVE_RANGE') {
        toPush.min = parameter.min;
        toPush.max = parameter.max;
        toPush.step = parameter.step;
      }

      combinedFields.parameters.push(toPush);
    }
  }

  // Metadata APIの結果がない場合は早期リターン
  if (!listFieldsResult.data.publishedDatasources[0]?.fields.length) {
    return combinedFields;
  }

  // ========================================
  // ステップ4: Metadata APIの情報で拡張
  // ========================================
  // combinedFieldsの各フィールドに対して、対応するMetadata APIの情報を追加
  for (const field of combinedFields.fields) {
    // 名前でマッチング
    const matchingListField = listFieldsResult.data.publishedDatasources[0].fields.find(
      (f) => f.name === field.name,
    );

    if (matchingListField) {
      // 追加プロパティを設定
      populateFieldWithAdditionalProperties(matchingListField, field);
    }
  }

  return combinedFields;
}
```

### populateFieldWithAdditionalProperties 関数

**src/tools/getDatasourceMetadata/datasourceMetadataUtils.ts:210-241**

```typescript
function populateFieldWithAdditionalProperties(
  sourceField: Field,   // Metadata APIのフィールド
  targetField: Field    // VizQLベースのフィールド
): void {
  // 説明文
  if (sourceField.description) {
    targetField.description = sourceField.description;
  }

  // 継承された説明文
  if (sourceField.descriptionInherited?.length) {
    targetField.descriptionInherited = sourceField.descriptionInherited;
  }

  // データカテゴリ（QUANTITATIVE, ORDINAL, NOMINAL, etc.）
  if (sourceField.dataCategory) {
    targetField.dataCategory = sourceField.dataCategory;
  }

  // ロール（DIMENSION, MEASURE）
  if (sourceField.role) {
    targetField.role = sourceField.role;
  }

  // デフォルトフォーマット
  if (sourceField.defaultFormat) {
    targetField.defaultFormat = sourceField.defaultFormat;
  }

  // 計算式（計算フィールドの場合）
  if (sourceField.formula) {
    targetField.formula = sourceField.formula;

    // 自動生成フラグ
    if (sourceField.isAutoGenerated != undefined) {
      targetField.isAutoGenerated = sourceField.isAutoGenerated;
    }

    // ユーザー参照フラグ
    if (sourceField.hasUserReference != undefined) {
      targetField.hasUserReference = sourceField.hasUserReference;
    }
  }

  // ビンサイズ（ビンフィールドの場合）
  if (sourceField.binSize != undefined) {
    targetField.binSize = sourceField.binSize;
  }
}
```

### 統合処理の詳細フロー

```
VizQL結果:                    Metadata API結果:
┌──────────────────┐         ┌──────────────────┐
│ fieldCaption:    │         │ name:            │
│   "Sales"        │◄────┐   │   "Sales"        │
│ dataType: "REAL" │     │   │ description:     │
│ defaultAgg: "SUM"│     └───│   "Total sales..."
└──────────────────┘  名前で  │ role: "MEASURE"  │
                      マッチング│ semanticRole:    │
                              │   "CURRENCY"     │
                              └──────────────────┘
         │                            │
         └────────┬───────────────────┘
                  │ combineFields()
                  ▼
         ┌────────────────────┐
         │ 統合結果:           │
         │ name: "Sales"      │
         │ dataType: "REAL"   │
         │ defaultAgg: "SUM"  │
         │ description:       │
         │   "Total sales..." │
         │ role: "MEASURE"    │
         │ semanticRole:      │
         │   "CURRENCY"       │
         └────────────────────┘
```

---

## パラメータ処理

### パラメータの種類

Tableauのパラメータは3つのタイプがあります：

#### 1. ANY_VALUE（任意の値）
```typescript
{
  name: "Country",
  parameterType: "ANY_VALUE",
  dataType: "STRING",
  value: "USA"
}
```

#### 2. LIST（リスト）
```typescript
{
  name: "Year",
  parameterType: "LIST",
  dataType: "INTEGER",
  value: 2024,
  members: [2020, 2021, 2022, 2023, 2024, 2025]
}
```

#### 3. QUANTITATIVE_RANGE（数値範囲）
```typescript
{
  name: "Price Range",
  parameterType: "QUANTITATIVE_RANGE",
  dataType: "REAL",
  value: 100.0,
  min: 0,
  max: 1000,
  step: 10
}
```

#### 4. QUANTITATIVE_DATE（日付範囲）
```typescript
{
  name: "Date Range",
  parameterType: "QUANTITATIVE_DATE",
  dataType: "DATE",
  value: "2024-01-01",
  minDate: "2020-01-01",
  maxDate: "2025-12-31",
  periodValue: 1,
  periodType: "MONTHS"
}
```

### パラメータ処理のコード

**src/tools/getDatasourceMetadata/datasourceMetadataUtils.ts:167-191**

```typescript
// VizQLのextraDataからパラメータを取得
if (readMetadataResult.extraData?.parameters) {
  for (const parameter of readMetadataResult.extraData.parameters) {
    const toPush: Parameter = {
      name: parameter.parameterCaption,
      parameterType: parameter.parameterType,
      dataType: parameter.dataType,
      value: parameter.value,
    };

    // パラメータタイプごとに追加プロパティを設定
    if (parameter.parameterType === 'LIST' && parameter.members) {
      // リスト型: 選択可能な値のリスト
      toPush.members = parameter.members;
    } else if (parameter.parameterType === 'QUANTITATIVE_DATE') {
      // 日付範囲型: 最小日付、最大日付、期間
      toPush.minDate = parameter.minDate;
      toPush.maxDate = parameter.maxDate;
      toPush.periodValue = parameter.periodValue;
      toPush.periodType = parameter.periodType;
    } else if (parameter.parameterType === 'QUANTITATIVE_RANGE') {
      // 数値範囲型: 最小値、最大値、ステップ
      toPush.min = parameter.min;
      toPush.max = parameter.max;
      toPush.step = parameter.step;
    }

    combinedFields.parameters.push(toPush);
  }
}
```

---

## 最適化とトークン削減

### なぜ最適化が必要か

Metadata APIの生のレスポンスは非常に大きく、AIモデルのコンテキストウィンドウを圧迫します。そのため、必要な情報のみを抽出して返します。

### simplifyReadMetadataResult 関数

**src/tools/getDatasourceMetadata/datasourceMetadataUtils.ts:55-113**

```typescript
export function simplifyReadMetadataResult(readMetadataResult: MetadataResponse): FieldsResult {
  const simplifiedResponse: FieldsResult = {
    fields: [],
    parameters: [],
  };

  if (!readMetadataResult.data) {
    return simplifiedResponse;
  }

  // ========================================
  // 最小限のフィールド情報のみを含める
  // ========================================
  for (const field of readMetadataResult.data) {
    const toPush: Field = {
      name: field.fieldCaption,           // 必須: フィールド名
      dataType: field.dataType,           // 必須: データ型
      columnClass: field.columnClass,     // 必須: カラムクラス
    };

    // オプション: デフォルト集計関数（あれば追加）
    if (field.defaultAggregation) {
      toPush.defaultAggregation = field.defaultAggregation;
    }

    // オプション: 計算式（計算フィールドの場合）
    if (field.formula) {
      toPush.formula = field.formula;
    }

    simplifiedResponse.fields.push(toPush);
  }

  // パラメータの追加
  if (readMetadataResult.extraData?.parameters) {
    for (const parameter of readMetadataResult.extraData.parameters) {
      // ... パラメータ処理（上記参照）
    }
  }

  return simplifiedResponse;
}
```

### トークン削減の効果

**最適化前（VizQL生レスポンス）:**
```json
{
  "data": [
    {
      "fieldName": "[Federated.1p9mqm80vqeqgk15h0vap1fhaxr9].[sum:Sales:qk]",
      "fieldCaption": "Sales",
      "dataType": "REAL",
      "defaultAggregation": "SUM",
      "columnClass": "COLUMN",
      "logicalTableId": "Federated.1p9mqm80vqeqgk15h0vap1fhaxr9",
      "aggregationParam": null,
      "formula": null,
      // ... その他多数のプロパティ
    }
  ]
}
```
→ **約150トークン/フィールド**

**最適化後（simplify後）:**
```json
{
  "fields": [
    {
      "name": "Sales",
      "dataType": "REAL",
      "columnClass": "COLUMN",
      "defaultAggregation": "SUM"
    }
  ]
}
```
→ **約30トークン/フィールド（80%削減）**

---

## エラーハンドリング

### VizQL Data Service無効エラー

**src/tools/getDatasourceMetadata/getDatasourceMetadata.ts:129-131**

```typescript
if (readMetadataResult.isErr()) {
  return Err({ type: 'feature-disabled' });
}
```

**エラーメッセージ:**
```
VizQL Data Service is not enabled on this Tableau Server.
Please contact your administrator to enable it.
```

### Metadata API利用不可エラー

**src/tools/getDatasourceMetadata/getDatasourceMetadata.ts:140-146**

```typescript
try {
  listFieldsResult = await restApi.metadataMethods.graphql(query);
} catch {
  // Metadata APIが利用できない場合は、VizQLの結果のみを返す
  return Ok(simplifyReadMetadataResult(readMetadataResult.value));
}
```

**挙動:**
- エラーを投げずに、VizQLの結果のみを返す
- ユーザーには通知しない（透過的なフォールバック）

### データソースLUIDバリデーション

**src/tools/getDatasourceMetadata/getDatasourceMetadata.ts:104**

```typescript
argsValidator: validateDatasourceLuid,
```

**validateDatasourceLuid 関数（src/tools/validateDatasourceLuid.ts）:**
```typescript
export const validateDatasourceLuid = async (
  args: { datasourceLuid: string },
  server: Server,
): Promise<void> => {
  const { datasourceLuid } = args;

  // データソースが存在するかチェック
  const datasources = await listDatasources({ filter: `luid:eq:${datasourceLuid}` });

  if (datasources.length === 0) {
    throw new Error(`Datasource with LUID '${datasourceLuid}' not found`);
  }
};
```

---

## 実践例

### 例1: 基本的なメタデータ取得

**入力:**
```javascript
{
  "datasourceLuid": "abc-123-def-456"
}
```

**VizQL API レスポンス:**
```json
{
  "data": [
    {
      "fieldCaption": "Order Date",
      "dataType": "DATE",
      "defaultAggregation": "YEAR",
      "columnClass": "COLUMN"
    },
    {
      "fieldCaption": "Sales",
      "dataType": "REAL",
      "defaultAggregation": "SUM",
      "columnClass": "COLUMN"
    },
    {
      "fieldCaption": "Profit Margin",
      "dataType": "REAL",
      "defaultAggregation": "SUM",
      "columnClass": "CALCULATION",
      "formula": "SUM([Profit]) / SUM([Sales])"
    }
  ]
}
```

**Metadata API レスポンス:**
```json
{
  "data": {
    "publishedDatasources": [{
      "fields": [
        {
          "name": "Order Date",
          "description": "Date when order was placed",
          "role": "DIMENSION",
          "semanticRole": "TEMPORAL"
        },
        {
          "name": "Sales",
          "description": "Total sales amount in USD",
          "role": "MEASURE",
          "semanticRole": "CURRENCY",
          "defaultFormat": "$#,##0.00"
        },
        {
          "name": "Profit Margin",
          "description": "Calculated profit margin percentage",
          "role": "MEASURE",
          "formula": "SUM([Profit]) / SUM([Sales])",
          "isAutoGenerated": false
        }
      ]
    }]
  }
}
```

**統合結果（最終出力）:**
```json
{
  "fields": [
    {
      "name": "Order Date",
      "dataType": "DATE",
      "columnClass": "COLUMN",
      "defaultAggregation": "YEAR",
      "description": "Date when order was placed",
      "role": "DIMENSION",
      "semanticRole": "TEMPORAL"
    },
    {
      "name": "Sales",
      "dataType": "REAL",
      "columnClass": "COLUMN",
      "defaultAggregation": "SUM",
      "description": "Total sales amount in USD",
      "role": "MEASURE",
      "semanticRole": "CURRENCY",
      "defaultFormat": "$#,##0.00"
    },
    {
      "name": "Profit Margin",
      "dataType": "REAL",
      "columnClass": "CALCULATION",
      "defaultAggregation": "SUM",
      "formula": "SUM([Profit]) / SUM([Sales])",
      "description": "Calculated profit margin percentage",
      "role": "MEASURE",
      "isAutoGenerated": false
    }
  ],
  "parameters": []
}
```

### 例2: パラメータを含むメタデータ取得

**VizQL API レスポンス（extraData）:**
```json
{
  "extraData": {
    "parameters": [
      {
        "parameterCaption": "Region",
        "parameterType": "LIST",
        "dataType": "STRING",
        "value": "West",
        "members": ["East", "West", "Central", "South"]
      },
      {
        "parameterCaption": "Sales Threshold",
        "parameterType": "QUANTITATIVE_RANGE",
        "dataType": "REAL",
        "value": 10000,
        "min": 0,
        "max": 100000,
        "step": 1000
      }
    ]
  }
}
```

**統合結果:**
```json
{
  "fields": [...],
  "parameters": [
    {
      "name": "Region",
      "parameterType": "LIST",
      "dataType": "STRING",
      "value": "West",
      "members": ["East", "West", "Central", "South"]
    },
    {
      "name": "Sales Threshold",
      "parameterType": "QUANTITATIVE_RANGE",
      "dataType": "REAL",
      "value": 10000,
      "min": 0,
      "max": 100000,
      "step": 1000
    }
  ]
}
```

### 例3: Metadata API無効の場合

**config.disableMetadataApiRequests = true の場合:**

**最終出力（simplify版）:**
```json
{
  "fields": [
    {
      "name": "Sales",
      "dataType": "REAL",
      "columnClass": "COLUMN",
      "defaultAggregation": "SUM"
    },
    {
      "name": "State",
      "dataType": "STRING",
      "columnClass": "COLUMN"
    }
  ],
  "parameters": []
}
```

→ **description、role、semanticRoleなどの拡張メタデータは含まれない**

---

## まとめ

### メタデータ取得の重要ポイント

1. **二段階取得**
   - VizQL Data Service: クエリ可能なフィールドの正確な情報
   - Metadata API: ビジネスコンテキストと詳細情報

2. **統合ロジック**
   - VizQLの結果をベースに（クエリ可能性を保証）
   - Metadata APIの情報で拡張（AIの理解を向上）
   - 名前でマッチングして統合

3. **最適化**
   - 不要な情報を削除してトークン削減
   - フォールバック機能で可用性を確保
   - 設定で柔軟に制御可能

4. **エラーハンドリング**
   - 透過的なフォールバック
   - 明確なエラーメッセージ
   - バリデーションによる事前チェック

5. **パラメータサポート**
   - 3つのパラメータタイプに対応
   - タイプごとの適切なプロパティ設定
   - クエリでパラメータを使用可能

この二段階取得と統合のアプローチにより、Tableau MCPはAIアプリケーションに対して、**正確かつ意味のある**メタデータを提供できます。

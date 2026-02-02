# sqlutil 包详细说明

## 概述

`sqlutil` 包是 Grafana Plugin SDK Go 中用于处理 SQL 数据源的核心工具包。它提供了一套完整的功能来简化从 SQL 数据库查询数据并将其转换为 Grafana 数据帧(DataFrame)的过程。

## 主要功能

### 1. SQL 查询宏系统
- 提供动态查询替换机制
- 支持时间范围、时间间隔等常用宏
- 可扩展的自定义宏支持

### 2. 类型转换系统
- 自动将 SQL 类型转换为 DataFrame 支持的类型
- 支持可空类型(Nullable Types)
- 提供自定义转换器机制

### 3. 数据帧生成
- 从 SQL 查询结果自动生成数据帧
- 支持动态类型推断
- 支持多结果集处理

### 4. 数据重采样
- 时间序列数据重采样
- 填充缺失数据点
- 支持多种填充模式

## 核心组件

### Query (查询模型)

**文件**: `query.go`

`Query` 结构体表示用户从面板/查询编辑器提交的查询。

```go
type Query struct {
    RawSQL         string            // 原始 SQL 查询语句
    Format         FormatQueryOption // 数据格式化选项
    ConnectionArgs json.RawMessage   // 连接参数
    
    RefID         string            // 查询引用 ID
    Interval      time.Duration     // 查询时间间隔
    TimeRange     backend.TimeRange // 查询时间范围
    MaxDataPoints int64             // 最大数据点数
    FillMissing   *data.FillMissing // 填充缺失值的配置
    
    // 宏相关字段
    Schema string // 数据库模式名
    Table  string // 表名
    Column string // 列名
}
```

#### 格式选项

- `FormatOptionTimeSeries` - 格式化为时间序列
- `FormatOptionTable` - 格式化为表格
- `FormatOptionLogs` - 日志格式
- `FormatOptionTrace` - 追踪格式
- `FormatOptionMulti` - 多序列格式

#### 主要方法

- `GetQuery(query backend.DataQuery)` - 从 backend.DataQuery 解析查询对象
- `WithSQL(query string)` - 创建带有新 SQL 语句的查询副本
- `ErrorFrameFromQuery(query *Query)` - 创建错误数据帧

---

### Macros (宏系统)

**文件**: `macros.go`

宏系统允许在 SQL 查询中使用动态占位符,在执行前自动替换为实际值。

#### 内置宏

| 宏名称 | 说明 | 示例 |
|-------|------|------|
| `$__interval` | 查询时间间隔 | `10m` |
| `$__interval_ms` | 时间间隔(毫秒) | `600000` |
| `$__timeFilter(column)` | 时间范围过滤器 | `time >= '2023-01-01' AND time <= '2023-01-02'` |
| `$__timeFrom(column)` | 开始时间过滤 | `time >= '2023-01-01'` |
| `$__timeTo(column)` | 结束时间过滤 | `time <= '2023-01-02'` |
| `$__timeGroup(column, period)` | 时间分组 | `datepart(year, time), datepart(month, time)` |
| `$__table` | 表名 | `my_table` |
| `$__column` | 列名 | `my_column` |

#### 使用示例

```go
query := &Query{
    RawSQL: "SELECT * FROM $__table WHERE $__timeFilter(timestamp)",
    Table: "metrics",
    TimeRange: backend.TimeRange{
        From: time.Now().Add(-1 * time.Hour),
        To:   time.Now(),
    },
}

interpolatedSQL, err := Interpolate(query, DefaultMacros)
// 结果: "SELECT * FROM metrics WHERE timestamp >= '...' AND timestamp <= '...'"
```

#### 自定义宏

```go
customMacros := Macros{
    "database": func(query *Query, args []string) (string, error) {
        return "production_db", nil
    },
}

sql, err := Interpolate(query, customMacros)
```

---

### Converter (类型转换器)

**文件**: `converter.go`

转换器系统负责将 SQL 扫描的数据类型转换为 DataFrame 支持的类型。

#### Converter 结构

```go
type Converter struct {
    Name           string           // 转换器名称
    InputScanType  reflect.Type     // SQL 扫描类型
    InputTypeName  string           // SQL 类型名称
    InputTypeRegex *regexp.Regexp   // 类型名称正则匹配
    InputColumnName string          // 列名匹配
    FrameConverter FrameConverter   // 帧转换器
    Dynamic        bool             // 是否动态推断类型
}
```

#### FrameConverter

```go
type FrameConverter struct {
    FieldType     data.FieldType  // DataFrame 字段类型
    ConverterFunc func(interface{}) (interface{}, error) // 转换函数
    ConvertWithColumn func(interface{}, sql.ColumnType) (interface{}, error) // 带列类型的转换
}
```

#### 内置转换器

- `NullStringConverter` - 可空字符串
- `NullDecimalConverter` - 可空浮点数
- `NullInt64Converter` - 可空 64 位整数
- `NullInt32Converter` - 可空 32 位整数
- `NullInt16Converter` - 可空 16 位整数
- `NullTimeConverter` - 可空时间
- `NullBoolConverter` - 可空布尔值
- `NullByteConverter` - 可空字节

#### 创建自定义转换器

```go
customConverter := Converter{
    Name:          "UUID Converter",
    InputScanType: reflect.TypeOf(sql.NullString{}),
    InputTypeName: "UUID",
    FrameConverter: FrameConverter{
        FieldType: data.FieldTypeNullableString,
        ConverterFunc: func(in interface{}) (interface{}, error) {
            v := in.(*sql.NullString)
            if !v.Valid {
                return (*string)(nil), nil
            }
            // 自定义 UUID 处理逻辑
            formatted := formatUUID(v.String)
            return &formatted, nil
        },
    },
}
```

---

### FrameFromRows (数据帧生成)

**文件**: `sql.go`

从 SQL 查询结果生成 Grafana 数据帧。

#### 函数签名

```go
func FrameFromRows(
    rows *sql.Rows, 
    rowLimit int64, 
    converters ...Converter
) (*data.Frame, error)
```

#### 参数说明

- `rows` - SQL 查询结果行
- `rowLimit` - 行数限制(-1 表示无限制)
- `converters` - 可选的自定义转换器列表

#### 特性

1. **自动类型推断**: 如果未提供转换器,自动根据 SQL 类型选择合适的 DataFrame 类型
2. **可空类型处理**: 自动处理 NULL 值
3. **行数限制**: 达到限制时添加警告通知
4. **多结果集支持**: 自动处理多个结果集
5. **动态类型推断**: 支持运行时类型检测

#### 使用示例

```go
// 基本使用
rows, err := db.Query("SELECT id, name, created_at FROM users")
if err != nil {
    return err
}
defer rows.Close()

frame, err := sqlutil.FrameFromRows(rows, 1000)
if err != nil {
    return err
}

// 使用自定义转换器
converters := []sqlutil.Converter{
    sqlutil.Converter{
        InputColumnName: "metadata",
        InputScanType:   reflect.TypeOf(sql.NullString{}),
        FrameConverter: sqlutil.FrameConverter{
            FieldType: data.FieldTypeNullableJSON,
            ConverterFunc: func(in interface{}) (interface{}, error) {
                // 自定义 JSON 解析逻辑
            },
        },
    },
}

frame, err := sqlutil.FrameFromRows(rows, 1000, converters...)
```

---

### 动态类型推断

**文件**: `dynamic_frame.go`

当 SQL 驱动不提供足够的类型信息时,动态类型推断系统会通过检查实际数据来确定字段类型。

#### 工作原理

1. **第一遍扫描**: 读取数据并检测每列的实际类型
2. **类型映射**: 将检测到的 Go 类型映射到 DataFrame 字段类型
3. **转换器覆盖**: 允许通过列名指定自定义转换器
4. **数据填充**: 使用确定的类型创建数据帧并填充数据

#### 支持的类型检测

- `time.Time`, `*time.Time` → 时间类型
- 数值类型 (`float64`, `int`, `int64` 等) → Float64
- `string` → 字符串类型
- `[]uint8` → 字符串(字节数组)
- 其他类型 → 默认转为字符串

#### 启用动态推断

```go
dynamicConverter := sqlutil.Converter{
    Dynamic: true,
}

frame, err := sqlutil.FrameFromRows(rows, 1000, dynamicConverter)
```

---

### ScanRow (扫描行处理)

**文件**: `scanrow.go`

`ScanRow` 管理 SQL 行的元数据和扫描过程。

#### 结构定义

```go
type ScanRow struct {
    Columns []string       // 列名
    Types   []reflect.Type // 列类型
}
```

#### RowConverter

```go
type RowConverter struct {
    Row        *ScanRow
    Converters []Converter
}
```

#### 主要功能

1. **列类型映射**: 将 SQL 列类型映射到扫描类型
2. **转换器匹配**: 根据列名、类型名或正则表达式匹配转换器
3. **默认转换器**: 为未匹配的列自动选择默认转换器
4. **可扫描行生成**: 创建可用于 `sql.Rows.Scan()` 的指针切片

#### MakeScanRow 函数

```go
func MakeScanRow(
    colTypes []*sql.ColumnType,
    colNames []string,
    converters ...Converter
) (*RowConverter, error)
```

功能:
- 验证列名唯一性
- 匹配自定义转换器
- 为未匹配列选择默认转换器
- 处理可空类型

---

### 数据重采样

**文件**: `resample.go`

时间序列数据的重采样功能,用于调整数据点间隔。

#### 函数签名

```go
func ResampleWideFrame(
    f *data.Frame,
    fillMissing *data.FillMissing,
    timeRange backend.TimeRange,
    interval time.Duration
) (*data.Frame, error)
```

#### 重采样策略

1. **时间对齐**: 将数据点对齐到指定的时间间隔
2. **缺失值填充**: 根据 `FillMissing` 配置填充缺失数据
3. **中间点处理**: 处理落在同一间隔内的多个数据点

#### 填充模式

- `Previous` - 使用前一个值
- `Null` - 填充 NULL
- `Value` - 使用指定值

#### 使用示例

```go
fillMode := &data.FillMissing{
    Mode: data.FillModePrevious,
}

resampledFrame, err := sqlutil.ResampleWideFrame(
    originalFrame,
    fillMode,
    timeRange,
    10 * time.Minute,
)
```

**注意**: 此功能已标记为废弃,建议新项目使用基于 dataplane 的解决方案。

---

## 文件结构

```
sqlutil/
├── doc.go              # 包文档
├── converter.go        # 类型转换器定义和实现
├── converter_test.go   # 转换器测试
├── frame.go            # 数据帧基础操作
├── sql.go              # SQL 行转数据帧
├── sql_test.go         # SQL 功能测试
├── query.go            # 查询模型定义
├── query_test.go       # 查询模型测试
├── macros.go           # 宏系统实现
├── macros_test.go      # 宏系统测试
├── scanrow.go          # 扫描行处理
├── dynamic_frame.go    # 动态类型推断
├── dynamic_frame_test.go # 动态推断测试
├── resample.go         # 数据重采样
└── resample_test.go    # 重采样测试
```

## 完整使用示例

### 示例 1: 基本 SQL 查询转数据帧

```go
package main

import (
    "database/sql"
    "fmt"
    
    "github.com/grafana/grafana-plugin-sdk-go/data/sqlutil"
    _ "github.com/lib/pq" // PostgreSQL 驱动
)

func QueryToFrame(db *sql.DB) error {
    // 执行查询
    rows, err := db.Query(`
        SELECT 
            timestamp,
            metric_name,
            value
        FROM metrics
        WHERE timestamp >= $1 AND timestamp <= $2
    `, startTime, endTime)
    if err != nil {
        return err
    }
    defer rows.Close()
    
    // 转换为数据帧
    frame, err := sqlutil.FrameFromRows(rows, 10000)
    if err != nil {
        return err
    }
    
    fmt.Printf("Generated frame with %d rows and %d columns\n", 
        frame.Rows(), len(frame.Fields))
    
    return nil
}
```

### 示例 2: 使用宏进行查询插值

```go
func InterpolateQuery(query *sqlutil.Query) (string, error) {
    // 定义自定义宏
    customMacros := sqlutil.Macros{
        "database": func(q *sqlutil.Query, args []string) (string, error) {
            return "production", nil
        },
        "shard": func(q *sqlutil.Query, args []string) (string, error) {
            // 根据时间范围计算分片
            return calculateShard(q.TimeRange), nil
        },
    }
    
    // 执行插值
    interpolated, err := sqlutil.Interpolate(query, customMacros)
    if err != nil {
        return "", err
    }
    
    return interpolated, nil
}
```

### 示例 3: 自定义类型转换

```go
func QueryWithCustomConverter(db *sql.DB) (*data.Frame, error) {
    // 定义 JSON 列转换器
    jsonConverter := sqlutil.Converter{
        Name:            "JSON Converter",
        InputColumnName: "properties",
        InputScanType:   reflect.TypeOf(sql.NullString{}),
        FrameConverter: sqlutil.FrameConverter{
            FieldType: data.FieldTypeNullableJSON,
            ConverterFunc: func(in interface{}) (interface{}, error) {
                v := in.(*sql.NullString)
                if !v.Valid {
                    return (*json.RawMessage)(nil), nil
                }
                
                raw := json.RawMessage(v.String)
                return &raw, nil
            },
        },
    }
    
    // 定义地理坐标转换器
    geoConverter := sqlutil.Converter{
        Name:          "Geography Converter",
        InputTypeName: "geography",
        InputScanType: reflect.TypeOf(sql.NullString{}),
        FrameConverter: sqlutil.FrameConverter{
            FieldType: data.FieldTypeNullableString,
            ConverterFunc: func(in interface{}) (interface{}, error) {
                v := in.(*sql.NullString)
                if !v.Valid {
                    return (*string)(nil), nil
                }
                // 格式化地理数据
                formatted := formatGeoData(v.String)
                return &formatted, nil
            },
        },
    }
    
    rows, err := db.Query("SELECT id, name, properties, location FROM entities")
    if err != nil {
        return nil, err
    }
    defer rows.Close()
    
    return sqlutil.FrameFromRows(rows, 5000, jsonConverter, geoConverter)
}
```

### 示例 4: 完整的查询数据源实现

```go
func (ds *MyDataSource) QueryData(
    ctx context.Context,
    req *backend.QueryDataRequest,
) (*backend.QueryDataResponse, error) {
    response := backend.NewQueryDataResponse()
    
    for _, q := range req.Queries {
        // 解析查询
        query, err := sqlutil.GetQuery(q)
        if err != nil {
            response.Responses[q.RefID] = backend.ErrDataResponse(
                backend.StatusBadRequest, err.Error())
            continue
        }
        
        // 插值宏
        interpolatedSQL, err := sqlutil.Interpolate(query, sqlutil.DefaultMacros)
        if err != nil {
            response.Responses[q.RefID] = backend.ErrDataResponse(
                backend.StatusBadRequest, err.Error())
            continue
        }
        
        // 执行查询
        rows, err := ds.db.QueryContext(ctx, interpolatedSQL)
        if err != nil {
            response.Responses[q.RefID] = backend.ErrDataResponse(
                backend.StatusInternal, err.Error())
            continue
        }
        
        // 转换为数据帧
        frame, err := sqlutil.FrameFromRows(rows, query.MaxDataPoints)
        rows.Close()
        
        if err != nil {
            response.Responses[q.RefID] = backend.ErrDataResponse(
                backend.StatusInternal, err.Error())
            continue
        }
        
        // 设置元数据
        frame.RefID = query.RefID
        frame.Meta = &data.FrameMeta{
            ExecutedQueryString: interpolatedSQL,
        }
        
        // 根据格式选项处理数据帧
        switch query.Format {
        case sqlutil.FormatOptionTimeSeries:
            frame, err = data.LongToWide(frame, query.FillMissing)
        case sqlutil.FormatOptionTable:
            // 保持宽格式
        }
        
        if err != nil {
            response.Responses[q.RefID] = backend.ErrDataResponse(
                backend.StatusInternal, err.Error())
            continue
        }
        
        response.Responses[query.RefID] = backend.DataResponse{
            Frames: data.Frames{frame},
        }
    }
    
    return response, nil
}
```

## 最佳实践

### 1. 转换器设计

- **优先使用列名匹配**: 列名匹配比类型名匹配更精确
- **使用正则表达式**: 对于变体类型使用 `InputTypeRegex`
- **处理 NULL 值**: 始终正确处理可空类型

### 2. 宏使用

- **合并默认宏**: 使用 `maps.Copy()` 合并自定义宏和默认宏
- **参数验证**: 在宏函数中验证参数数量和类型
- **错误处理**: 返回清晰的错误信息

### 3. 性能优化

- **设置行限制**: 避免加载过多数据到内存
- **使用流式处理**: 大数据集考虑分批处理
- **预编译转换器**: 重用转换器实例

### 4. 错误处理

- **使用 backend.DownstreamError**: 包装下游错误
- **提供上下文**: 错误信息包含列名、类型等上下文
- **验证输入**: 在处理前验证查询参数

### 5. 类型安全

- **检查类型断言**: 使用双值断言避免 panic
- **验证字段类型**: 确保转换后的类型匹配 FieldType
- **测试边界情况**: 测试 NULL、空字符串等边界情况

## 常见问题

### Q1: 如何处理不支持的 SQL 类型?

使用自定义转换器将其转换为字符串或其他支持的类型:

```go
converter := sqlutil.Converter{
    InputTypeName: "UNSUPPORTED_TYPE",
    InputScanType: reflect.TypeOf(sql.NullString{}),
    FrameConverter: sqlutil.FrameConverter{
        FieldType: data.FieldTypeNullableString,
        ConverterFunc: func(in interface{}) (interface{}, error) {
            // 转换逻辑
        },
    },
}
```

### Q2: 如何处理重复的列名?

`MakeScanRow` 会自动检测并返回错误。在 SQL 查询中使用别名:

```sql
SELECT 
    a.id as a_id,
    b.id as b_id
FROM table_a a
JOIN table_b b ON a.key = b.key
```

### Q3: 动态类型推断什么时候使用?

当 SQL 驱动的 `ColumnType.ScanType()` 返回 `nil` 或不准确时使用动态推断:

```go
dynamicConverter := sqlutil.Converter{Dynamic: true}
frame, err := sqlutil.FrameFromRows(rows, -1, dynamicConverter)
```

### Q4: 如何调试转换器匹配?

添加日志查看转换器选择过程:

```go
for i, colType := range colTypes {
    fmt.Printf("Column %d: Name=%s, Type=%s\n", 
        i, colType.Name(), colType.DatabaseTypeName())
}
```

## 相关资源

- [Grafana Plugin SDK Documentation](https://grafana.com/docs/grafana/latest/developers/plugins/)
- [Data Frames Guide](https://grafana.com/docs/grafana/latest/developers/plugins/data-frames/)
- [Backend Plugin Development](https://grafana.com/docs/grafana/latest/developers/plugins/backend/)

## 贡献

欢迎提交 Issue 和 Pull Request 到 [grafana-plugin-sdk-go](https://github.com/grafana/grafana-plugin-sdk-go) 仓库。

## 许可证

本包是 Grafana Plugin SDK Go 的一部分,使用 Apache 2.0 许可证。

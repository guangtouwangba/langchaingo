# Embedding Model Configuration Fix

## 问题描述

在修复之前，当使用 `vertex.New()` 创建客户端时，存在以下问题：

1. **隐式强依赖**: 即使只想使用 Gemini 模型进行文本生成，`vertex.New()` 也会强制依赖 `textembedding-gecko` 模型的权限
2. **配置被忽略**: `WithDefaultEmbeddingModel` 选项虽然存在，但在实际使用中被完全忽略
3. **硬编码模型**: `palmclient.CreateEmbedding()` 使用硬编码的模型名称

## 解决方案

### 1. 扩展 PaLMClient 结构体

```go
// PaLMClient represents a Vertex AI based PaLM API client.
type PaLMClient struct {
	client         *aiplatform.PredictionClient
	projectID      string
	embeddingModel string // The embedding model to use (e.g., "textembedding-gecko")
}
```

### 2. 添加新的构造函数

```go
// New returns a new Vertex AI based PaLM API client.
func New(ctx context.Context, projectID, location string, opts ...option.ClientOption) (*PaLMClient, error) {
	return NewWithEmbeddingModel(ctx, projectID, location, embeddingModelName, opts...)
}

// NewWithEmbeddingModel returns a new Vertex AI based PaLM API client with custom embedding model.
func NewWithEmbeddingModel(ctx context.Context, projectID, location, embeddingModel string, opts ...option.ClientOption) (*PaLMClient, error) {
	// ... implementation
}
```

### 3. 修改 CreateEmbedding 方法

```go
// CreateEmbedding creates embeddings.
func (c *PaLMClient) CreateEmbedding(ctx context.Context, r *EmbeddingRequest) ([][]float32, error) {
	params := map[string]interface{}{}
	responses, err := c.batchPredict(ctx, c.embeddingModel, r.Input, params) // 使用实例变量而不是硬编码
	// ...
}
```

### 4. 更新 vertex.New() 函数

```go
palmClient, err := palmclient.NewWithEmbeddingModel(
	ctx,
	clientOptions.CloudProject,
	clientOptions.CloudLocation,
	clientOptions.DefaultEmbeddingModel, // 使用配置的模型
	clientOptions.ClientOptions...)
```

## 使用示例

### 修复前
```go
// 即使设置了自定义embedding模型，实际还是使用 "textembedding-gecko"
vertex, err := vertex.New(ctx, 
	googleai.WithDefaultEmbeddingModel("custom-embedding-model"),
	// 其他选项...
)
```

### 修复后
```go
// WithDefaultEmbeddingModel 现在可以正常工作
vertex, err := vertex.New(ctx, 
	googleai.WithDefaultEmbeddingModel("custom-embedding-model"),
	// 其他选项...
)
// vertex.CreateEmbedding() 现在会使用 "custom-embedding-model"
```

## 向后兼容性

- 保持了完全的向后兼容性
- 现有的 `palmclient.New()` 调用继续使用默认的 `textembedding-gecko` 模型
- 默认配置不变，确保现有代码无需修改

## 修复效果

✅ `WithDefaultEmbeddingModel` 选项现在可以正常工作  
✅ 用户可以自定义 embedding 模型，避免对 `textembedding-gecko` 的强依赖  
✅ 保持完全向后兼容性  
✅ 代码更加清晰和可配置  
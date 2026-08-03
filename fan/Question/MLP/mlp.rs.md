# `candle-binding/src/ffi/mlp.rs` 用途分析

## `mlp.rs` 的作用

这个文件是 **Rust → Go 的 FFI 桥梁**，将基于 Candle 框架的 **GPU 加速 MLP 神经网络推理**能力暴露给 Go 语言的 Semantic Router 使用。它实现了 FusionFactory (arXiv:2507.10540) 论文中 **query-level 模型路由**的推理部分。

**调用链路**: Python 训练 → JSON 模型文件 → Rust/Candle 加载推理 → C FFI → Go CGO → Semantic Router

---

## 具体使用场景

### 场景 1：加载预训练模型并在 GPU 上做 LoRA 路由推理

用户请求到达 Semantic Router，需要从多个 LoRA adapter 中选择最优的一个：

```
用户查询 "翻译这段法语" 
  → 文本编码为 1024 维 embedding (Go 层)
  → candle_mlp_from_json_with_device(json, 2)   // 加载到 Metal GPU
  → candle_mlp_select(handle, embedding, 1024)   // GPU 推理
  → 返回 "translation-lora"                      // 选中翻译专用 LoRA
```

对应代码路径：`ml_adapter.go:296` `CreateMLPSelector()` → `modelselection.MLPSelector.Load()` → FFI `candle_mlp_from_json`

### 场景 2：混合精度推理降低显存和延迟

在高并发场景下，用 F16 半精度加速推理：

```
candle_mlp_new_with_device_and_dtype(1, 1)  // CUDA + F16
// 模型权重自动转为半精度，推理速度提升约 2x，显存减半
```

这让路由器在 GPU 上同时服务更多并发请求，适合部署在 NVIDIA GPU 的推理集群。

### 场景 3：工厂模式批量注册 MLP 选择器

`factory.go:267-273` 中，系统启动时自动创建 MLP selector 并注册到全局 Registry：

```go
// factory.go:267 - 启动时自动创建
mlpAdapter, err := CreateMLPSelector(mlCfg, f.embeddingFunc)
registry.Register(MethodMLP, mlpAdapter)
```

之后所有请求通过 `GetSelector(MethodMLP)` 获取选择器。这意味着 MLP 路由器作为 KNN/KMeans/SVM 之外的**第四种 ML 选择算法**被统一管理。

### 场景 4：模型序列化 — 导出/导入训练好的路由器

```
Python 训练 MLP → 导出 JSON → Go 服务加载
                                ↓
candle_mlp_from_json(json_ptr)  // 从 JSON 反序列化权重
candle_mlp_to_json(handle)      // 运行时导出，用于模型版本管理或热更新
```

支持不停机模型热替换：新模型训练完成后，导出 JSON，Go 服务重新 `from_json` 加载即可。

### 场景 5：跨平台设备自适应

同一份代码在不同硬件上自动选择最优后端：

| device_type | 场景 |
|---|---|
| `0` (CPU) | 开发调试 / 无 GPU 的轻量级部署 |
| `1` (CUDA) | NVIDIA GPU 生产环境 (Linux 服务器) |
| `2` (Metal) | Apple Silicon 本地开发 (M1/M2 Mac) |

代码通过 `#[cfg(feature = "cuda")]` / `#[cfg(feature = "metal")]` 编译时选择后端，运行时优雅降级到 CPU。

---

## 总结

这个 FFI 层的核心价值是让 Go 编写的 Semantic Router 能利用 **Rust + Candle 的 GPU 加速能力**做高性能 MLP 神经网络推理，用于从多个候选 LLM/LoRA 中智能选择最适合当前查询的模型。

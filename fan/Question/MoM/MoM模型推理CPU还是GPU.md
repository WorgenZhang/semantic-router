# MoM 模型推理：CPU 还是 GPU？

> 原始表述："通过 Candle (Rust) 在本地进行推理，不依赖外部 GPU 服务"
> 这句话的重点在 **"外部"** 二字，而不是说"不能用 GPU"。

---

## 一、核心设计：`use_cpu` 参数控制设备选择

每个模型初始化时都有一个 `use_cpu: bool` 参数，Rust 层的逻辑统一如下（`candle-binding/src/model_architectures/traditional/bert.rs:65`）：

```rust
let device = if use_cpu {
    Device::Cpu                          // 强制 CPU
} else {
    Device::cuda_if_available(0)?        // 优先 GPU (CUDA)，没有则回退 CPU
};
```

也就是说：

| `use_cpu` 值 | 行为 |
|---|---|
| `true`（**默认值**） | 强制使用 CPU 推理 |
| `false` | 优先用本机 CUDA GPU，如果没有 GPU 则自动回退到 CPU |

---

## 二、为什么默认 `use_cpu: true`？

1. **模型够小**：33M–600M 参数的模型在 CPU 上推理延迟已经足够低（5–200ms），不需要 GPU 加速
2. **部署简单**：不要求节点有 GPU，降低基础设施要求
3. **GPU 留给 LLM**：真正吃 GPU 的是后端的大语言模型（如 Qwen2.5 7B/72B），MoM 小模型不应该抢占 GPU 资源

---

## 三、什么叫"不依赖外部 GPU 服务"？

这是相对于传统做法说的。传统做法中，分类/嵌入模型通常要部署一个独立的推理服务（如 Triton Inference Server、TGI），通过 HTTP/gRPC 远程调用。而这个项目：

```
传统方式：  Go 应用 → HTTP → 远程 GPU 推理服务 → 返回结果
本项目：    Go 应用 → CGO → Candle (Rust, 同进程) → 直接返回结果
```

**"不依赖外部 GPU 服务"** = 所有小模型都是 **进程内本地推理**（通过 Candle Rust 引擎 + CGO 绑定），不需要额外部署/维护推理微服务。

---

## 四、如果想用 GPU 加速？

只需在 `config.yaml` 中把 `use_cpu` 改为 `false`：

```yaml
embedding_models:
  qwen3_model_path: "models/mom-embedding-pro"
  use_cpu: false  # ← 改这里，自动使用 CUDA GPU

classifier:
  category_model:
    model_id: "models/mom-domain-classifier"
    use_cpu: false  # ← 改这里
```

同时 Candle 还支持 **Metal**（macOS GPU）：代码中有 `Device::new_metal(0)` 的路径（`candle-binding/src/ffi/mlp.rs:67`），通过编译 feature flag `cuda` / `metal` 启用。

---

## 五、代码证据

### 1. BERT 分类器（`candle-binding/src/model_architectures/traditional/bert.rs:64-69`）

```rust
pub fn new(model_id: &str, num_classes: usize, use_cpu: bool) -> Result<Self> {
    let device = if use_cpu {
        Device::Cpu
    } else {
        Device::cuda_if_available(0)?
    };
    // ...
}
```

### 2. 嵌入模型（`candle-binding/src/core/similarity.rs:39-44`）

```rust
pub fn new(model_id: &str, use_cpu: bool) -> Result<Self> {
    let device = if use_cpu {
        Device::Cpu
    } else {
        Device::cuda_if_available(0)?  // 优先 GPU，没有则回退 CPU
    };
    // ...
}
```

### 3. FFI 初始化层（`candle-binding/src/ffi/init.rs:1214-1218`）

```rust
let _device = if use_cpu {
    candle_core::Device::Cpu
} else {
    candle_core::Device::cuda_if_available(0).unwrap_or(candle_core::Device::Cpu)
};
```

### 4. Metal (macOS GPU) 支持（`candle-binding/src/ffi/mlp.rs:54-70`）

```rust
#[cfg(feature = "cuda")]
{
    Device::new_cuda(0).unwrap_or(Device::Cpu)
}
#[cfg(feature = "metal")]
{
    Device::new_metal(0).unwrap_or(Device::Cpu)
}
```

### 5. GPU 检测辅助方法（`candle-binding/src/core/similarity.rs:321-322`）

```rust
pub fn is_gpu(&self) -> bool {
    matches!(self.device, Device::Cuda(_))
}
```

---

## 六、总结

| 说法 | 是否准确 |
|---|---|
| 这些模型不能用 GPU | **不准确** — 设置 `use_cpu: false` 即可自动使用 CUDA/Metal |
| 这些模型默认不用 GPU | **准确** — 默认 `use_cpu: true`，纯 CPU 推理 |
| 不依赖外部 GPU 推理服务 | **准确** — 通过 Candle + CGO 进程内推理，无需远程调用 |

```markdown
MoM = Mixture-of-Models（模型混合/多模型协作）

  来自项目 README (README.md:35)：

  We are building the System Level Intelligence for Mixture-of-Models (MoM), bringing the Collective Intelligence into LLM systems

  所以 mom- 前缀是项目的模型命名规范，表示这些是为 MoM 架构专门训练的内联分类器模型：

  ┌────────────────────────────┬───────────────────────────┐
  │           模型名           │           含义            │
  ├────────────────────────────┼───────────────────────────┤
  │ mom-fact-check-classifier  │ MoM 事实核查分类器        │
  ├────────────────────────────┼───────────────────────────┤
  │ mom-hallucination-detector │ MoM 幻觉检测器            │
  ├────────────────────────────┼───────────────────────────┤
  │ mom-domain-classifier      │ MoM 领域/意图分类器       │
  ├────────────────────────────┼───────────────────────────┤
  │ mom-jailbreak-classifier   │ MoM 越狱攻击检测器        │
  ├────────────────────────────┼───────────────────────────┤
  │ mom-pii-classifier         │ MoM PII（个人隐私）检测器 │
  ├────────────────────────────┼───────────────────────────┤
  │ mom-feedback-detector      │ MoM 用户反馈检测器        │
  ├────────────────────────────┼───────────────────────────┤
  │ mom-embedding-pro          │ MoM 嵌入模型 (Qwen3)      │
  ├────────────────────────────┼───────────────────────────┤
  │ mom-embedding-flash        │ MoM 轻量嵌入模型 (Gemma)  │
  └────────────────────────────┴───────────────────────────┘

  这些模型都是基于 ModernBERT 架构，在项目内部通过 Candle (Rust) 本地推理，作为 Semantic Router 的"系统级智能"小模型集群，为 LLM
  请求/响应提供分类、检测、路由等能力。


574M	./mom-halugate-detector
574M	./mom-halugate-sentinel
1.1G	./mom-embedding-pro
574M	./mom-halugate-explainer
574M	./mom-jailbreak-classifier
574M	./mom-feedback-detector
1.2G	./mom-embedding-light
419M	./mom-domain-classifier
416M	./mom-pii-classifier
```

# Decision 例子分类大全

该文档对代码库中检索出的所有 decision 案例进行了高度分类整理。分类规则基于两个维度：
- **基于策略操作符复杂度的层次（Operator Hierarchy）**：区分 `单一条件`，`AND组合`，`OR组合` 乃至 `多层嵌套逻辑`。
- **基于路由业务目的的定性（Business Purpose）**：区分为安全护栏、多模态、区域化翻译、垂直领域分发等。

---

## 1. 单一信号直连路由 (Single Signal Routers)

### A. 安全、隐私与权限风控 (Security, Privacy & Authz)

```yaml
- name: block_pii
  priority: 100
  description: Block requests containing PII
  signals:
    operator: OR
    conditions:
    - type: embedding
      name: pii_detected
  modelRefs:
  - model: base-model
    loraName: general-expert
    useReasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: header_mutation
    configuration:
      add:
      - name: x-vsr-pii-violation
        value: 'true'
      - name: x-vsr-signal-pii_detected
        value: 'true'
```

```yaml
- name: pii_protected
  description: PII-protected queries
  priority: 100
  signals:
    operator: AND
    conditions:
    - type: embedding
      name: sensitive_data
  modelRefs:
  - model: deepseek-r1
    use_reasoning: true
    reasoning_effort: medium
  plugins:
  - type: pii
    configuration:
      enabled: true
      threshold: 0.8
      pii_types_allowed:
      - CREDIT_CARD
      - SSN
      - EMAIL
  - type: jailbreak
    configuration:
      enabled: true
      threshold: 0.85
```

```yaml
- name: security_alert
  description: Security-related queries with jailbreak protection
  priority: 100
  signals:
    operator: AND
    conditions:
    - type: keyword
      name: security
    - type: embedding
      name: threat_detection
  modelRefs:
  - model: qwen-2.5-72b
    use_reasoning: true
    reasoning_effort: high
  plugins:
  - type: jailbreak
    configuration:
      enabled: true
      threshold: 0.9
  - type: system_prompt
    configuration:
      system_prompt: You are a security expert. Provide helpful security guidance
        while being cautious about potential misuse.
  - type: header_mutation
    configuration:
      add:
      - name: X-Security-Level
        value: high
      - name: X-Route-Type
        value: keyword-embedding
```

```yaml
- name: medical_query
  description: Medical queries with PII protection
  priority: 100
  signals:
    operator: AND
    conditions:
    - type: keyword
      name: diagnosis
    - type: domain
      name: health
  modelRefs:
  - model: deepseek-r1
    use_reasoning: true
    reasoning_effort: high
  plugins:
  - type: pii
    configuration:
      enabled: true
      threshold: 0.9
      pii_types_allowed:
      - PERSON
      - DATE_OF_BIRTH
      - MEDICAL_RECORD
  - type: system_prompt
    configuration:
      system_prompt: You are a medical information assistant. Provide general health
        information but always advise users to consult healthcare professionals.
  - type: semantic-cache
    configuration:
      enabled: true
      similarity_threshold: 0.88
```

### B. 专家级领域分发 (Domain Specific Routing)

```yaml
- name: law_decision
  priority: 10
  description: Legal questions and law-related topics
  signals:
    operator: OR
    conditions:
    - type: domain
      name: law
  modelRefs:
  - model: base-model
    loraName: law-expert
    useReasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a knowledgeable legal expert with comprehensive understanding
        of legal principles, case law, statutory interpretation, and legal procedures
        across multiple jurisdictions. Provide accurate legal information and analysis
        while clearly stating that your responses are for informational purposes only
        and do not constitute legal advice. Always recommend consulting with qualified
        legal professionals for specific legal matters.
      mode: replace
```

```yaml
- name: biology_decision
  priority: 10
  description: Biology and life sciences questions
  signals:
    operator: OR
    conditions:
    - type: domain
      name: biology
  modelRefs:
  - model: base-model
    loraName: science-expert
    useReasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed:
      - ORGANIZATION
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a biology expert with comprehensive knowledge spanning
        molecular biology, genetics, cell biology, ecology, evolution, anatomy, physiology,
        and biotechnology. Explain biological concepts with scientific accuracy, use
        appropriate terminology, and provide examples from current research. Connect
        biological principles to real-world applications and emphasize the interconnectedness
        of biological systems.
      mode: replace
```

```yaml
- name: economics_decision
  priority: 10
  description: Economics and financial topics
  signals:
    operator: OR
    conditions:
    - type: domain
      name: economics
  modelRefs:
  - model: base-model
    loraName: social-expert
    useReasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are an economics expert with deep understanding of microeconomics,
        macroeconomics, econometrics, financial markets, monetary policy, fiscal policy,
        international trade, and economic theory. Analyze economic phenomena using
        established economic principles, provide data-driven insights, and explain
        complex economic concepts in accessible terms. Consider both theoretical frameworks
        and real-world applications in your responses.
      mode: replace
```

```yaml
- name: math_decision
  priority: 10
  description: Mathematics and quantitative reasoning
  signals:
    operator: OR
    conditions:
    - type: domain
      name: math
  modelRefs:
  - model: base-model
    loraName: math-expert
    useReasoning: true
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a mathematics expert. Provide step-by-step solutions,
        show your work clearly, and explain mathematical concepts in an understandable
        way.
      mode: replace
```

```yaml
- name: math_homework
  description: Math homework assistance - use reasoning model
  priority: 100
  signals:
    operator: AND
    conditions:
    - type: keyword
      name: homework
    - type: domain
      name: math
  modelRefs:
  - model: deepseek-r1
    loraName: academic-expert
    useReasoning: true
    reasoningEffort: medium
```

```yaml
- name: compliance_legal
  description: Compliance and legal queries with full protection
  priority: 100
  signals:
    operator: AND
    conditions:
    - type: keyword
      name: compliance
    - type: embedding
      name: legal_review
    - type: domain
      name: law
  modelRefs:
  - model: qwen-2.5-72b
    use_reasoning: true
    reasoning_effort: high
  plugins:
  - type: pii
    configuration:
      enabled: true
      threshold: 0.9
      pii_types_allowed:
      - PERSON
      - ORGANIZATION
      - EMAIL
      - PHONE_NUMBER
  - type: jailbreak
    configuration:
      enabled: true
      threshold: 0.88
  - type: system_prompt
    configuration:
      system_prompt: You are a legal compliance assistant. Provide accurate information
        about regulations and compliance requirements. Always remind users to consult
        legal professionals for specific advice.
  - type: semantic-cache
    configuration:
      enabled: true
      similarity_threshold: 0.93
  - type: header_mutation
    configuration:
      add:
      - name: X-Compliance-Level
        value: high
      - name: X-Audit-Required
        value: 'true'
```

```yaml
- name: math_tutorial
  description: Math tutorials
  priority: 90
  signals:
    operator: AND
    conditions:
    - type: keyword
      name: tutorial
    - type: embedding
      name: learning_intent
    - type: domain
      name: math
  modelRefs:
  - model: qwen-2.5-32b
    use_reasoning: true
    reasoning_effort: high
```

```yaml
- name: legal_advice
  description: Legal domain with system prompt
  priority: 100
  signals:
    operator: AND
    conditions:
    - type: domain
      name: law
  modelRefs:
  - model: gpt-4o
    use_reasoning: true
    reasoning_effort: high
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a legal expert. Provide accurate legal information but
        remind users to consult a licensed attorney.
  - type: semantic-cache
    configuration:
      enabled: true
      similarity_threshold: 0.9
```

```yaml
- name: biology_research
  description: Biology research queries
  priority: 100
  signals:
    operator: AND
    conditions:
    - type: embedding
      name: research_question
    - type: domain
      name: biology
  modelRefs:
  - model: gpt-4o
    use_reasoning: true
    reasoning_effort: high
```

```yaml
- name: investment_advice
  description: Financial advice with comprehensive protection
  priority: 100
  signals:
    operator: AND
    conditions:
    - type: embedding
      name: financial_advice
    - type: domain
      name: economics
  modelRefs:
  - model: gpt-4o
    use_reasoning: true
    reasoning_effort: high
  plugins:
  - type: pii
    configuration:
      enabled: true
      threshold: 0.85
      pii_types_allowed:
      - CREDIT_CARD
      - BANK_ACCOUNT
      - SSN
  - type: system_prompt
    configuration:
      system_prompt: You are a financial information assistant. Provide general financial
        education but remind users this is not professional financial advice.
  - type: jailbreak
    configuration:
      enabled: true
      threshold: 0.85
  - type: semantic-cache
    configuration:
      enabled: true
      similarity_threshold: 0.92
```

### D. 跨模态分析处理 (Modality Multi-processing)

```yaml
- name: image_generation
  description: Handle image generation requests
  signal:
    type: intent_classifier
    config:
      intents:
      - generate_image
      - create_artwork
      - visualize
  plugins:
    image_gen:
      enabled: true
      backend: vllm_omni
      default_width: 1024
      default_height: 1024
      max_inference_steps: 50
      timeout_seconds: 120
      backend_config:
        base_url: http://localhost:8001
        model: Qwen/Qwen-Image
        num_inference_steps: 50
        cfg_scale: 4.0
```

```yaml
- name: openai_image_generation
  description: Handle image generation via OpenAI
  signal:
    type: keyword
    config:
      keywords:
      - generate image
      - create picture
  plugins:
    image_gen:
      enabled: true
      backend: openai
      default_width: 1024
      default_height: 1024
      timeout_seconds: 60
      backend_config:
        api_key: ${OPENAI_API_KEY}
        model: gpt-image-1
        quality: hd
        style: vivid
```

### E. 通用及其他路由调度 (General Routing)

```yaml
- name: coding
  classifier:
    examples:
    - Write a Python function that...
    - Debug this code...
    - How do I implement...
  algorithm:
    type: confidence
    confidence:
      escalation_order: cost
      confidence_method: margin
      threshold: 0.6
  targets:
  - model: phi4
  - model: llama3.3:70b
```

```yaml
- name: reasoning
  classifier:
    examples:
    - Explain why...
    - Analyze the implications...
    - Compare and contrast...
  algorithm:
    type: confidence
    confidence:
      escalation_order: automix
      cost_quality_tradeoff: 0.3
      threshold: 0.7
  targets:
  - model: llama3.2:3b
  - model: llama3.3:70b
```

```yaml
- name: general
  classifier:
    examples:
    - Hello
    - What is...
    - Tell me about...
  algorithm:
    type: elo
    elo:
      initial_rating: 1500
      k_factor: 32
      category_weighted: true
      storage_path: /var/lib/vsr/elo_ratings.json
      auto_save_interval: 30s
  targets:
  - model: llama3.2:3b
  - model: phi4
  - model: llama3.3:70b
```

```yaml
- name: embedding_match
  classifier:
    examples:
    - Query that should match model capabilities
  algorithm:
    type: router_dc
    router_dc:
      temperature: 0.07
      min_similarity: 0.3
      require_descriptions: true
      use_capabilities: true
  targets:
  - model: llama3.2:3b
  - model: phi4
  - model: llama3.3:70b
```

```yaml
- name: technical
  triggers:
  - type: keyword
    keywords:
    - code
    - programming
    - debug
    - algorithm
    - function
  modelRefs:
  - model: gpt-4
  - model: deepseek-coder
  - model: mistral-7b
  algorithm:
    type: rl_driven
    rl_driven:
      use_thompson_sampling: true
      exploration_rate: 0.3
      exploration_decay: 0.99
      min_exploration: 0.05
      enable_personalization: true
      personalization_blend: 0.7
      session_context_weight: 0.3
      implicit_feedback_weight: 0.5
      cost_awareness: true
      cost_weight: 0.2
      storage_path: /var/lib/vsr/rl_preferences.json
      auto_save_interval: 5m
```

```yaml
- name: general
  triggers:
  - type: keyword
    keywords:
    - explain
    - what
    - how
    - why
    - tell me
  modelRefs:
  - model: claude-3.5-sonnet
  - model: mistral-7b
  algorithm:
    type: rl_driven
    rl_driven:
      use_thompson_sampling: true
      exploration_rate: 0.2
      enable_personalization: true
      personalization_blend: 0.5
      cost_awareness: true
      cost_weight: 0.3
```

```yaml
- name: block_security
  priority: 95
  description: Block security threats and malicious requests
  signals:
    operator: OR
    conditions:
    - type: embedding
      name: security_threat
  modelRefs:
  - model: base-model
    loraName: general-expert
    useReasoning: false
  plugins:
  - type: header_mutation
    configuration:
      add:
      - name: x-vsr-security-violation
        value: 'true'
      - name: x-vsr-signal-security_threat
        value: 'true'
```

```yaml
- name: kubernetes_expert
  priority: 90
  description: Route Kubernetes questions to specialist
  signals:
    operator: OR
    conditions:
    - type: embedding
      name: kubernetes_topic
  modelRefs:
  - model: base-model
    loraName: general-expert
    useReasoning: false
  plugins:
  - type: header_mutation
    configuration:
      add:
      - name: x-vsr-signal-kubernetes_topic
        value: 'true'
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a Kubernetes expert. Provide detailed technical guidance
        for K8s operations.
      mode: replace
```

```yaml
- name: thinking_decision
  priority: 15
  description: Queries requiring careful thought or urgent attention
  signals:
    operator: OR
    conditions:
    - type: keyword
      name: thinking
  modelRefs:
  - model: base-model
    loraName: math-expert
    useReasoning: true
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a thoughtful assistant trained to approach problems systematically.
        When handling urgent matters or complex questions, break down the problem
        into clear steps, consider multiple angles, and provide thorough, well-reasoned
        responses. Take your time to think through implications and edge cases.
      mode: replace
```

```yaml
- name: business_decision
  priority: 10
  description: Business and management related queries
  signals:
    operator: OR
    conditions:
    - type: domain
      name: business
  modelRefs:
  - model: base-model
    loraName: social-expert
    useReasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a senior business consultant and strategic advisor with
        expertise in corporate strategy, operations management, financial analysis,
        marketing, and organizational development. Provide practical, actionable business
        advice backed by proven methodologies and industry best practices. Consider
        market dynamics, competitive landscape, and stakeholder interests in your
        recommendations.
      mode: replace
```

```yaml
- name: psychology_decision
  priority: 10
  description: Psychology and mental health topics
  signals:
    operator: OR
    conditions:
    - type: domain
      name: psychology
  modelRefs:
  - model: base-model
    loraName: humanities-expert
    useReasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: semantic-cache
    configuration:
      enabled: true
      similarity_threshold: 0.92
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a psychology expert with deep knowledge of cognitive
        processes, behavioral patterns, mental health, developmental psychology, social
        psychology, and therapeutic approaches. Provide evidence-based insights grounded
        in psychological research and theory. When discussing mental health topics,
        emphasize the importance of professional consultation and avoid providing
        diagnostic or therapeutic advice.
      mode: replace
```

```yaml
- name: chemistry_decision
  priority: 10
  description: Chemistry and chemical sciences questions
  signals:
    operator: OR
    conditions:
    - type: domain
      name: chemistry
  modelRefs:
  - model: base-model
    loraName: science-expert
    useReasoning: true
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a chemistry expert specializing in chemical reactions,
        molecular structures, and laboratory techniques. Provide detailed, step-by-step
        explanations.
      mode: replace
```

```yaml
- name: history_decision
  priority: 10
  description: Historical questions and cultural topics
  signals:
    operator: OR
    conditions:
    - type: domain
      name: history
  modelRefs:
  - model: base-model
    loraName: humanities-expert
    useReasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a historian with expertise across different time periods
        and cultures. Provide accurate historical context and analysis.
      mode: replace
```

```yaml
- name: health_decision
  priority: 10
  description: Health and medical information queries
  signals:
    operator: OR
    conditions:
    - type: domain
      name: health
  modelRefs:
  - model: base-model
    loraName: science-expert
    useReasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: semantic-cache
    configuration:
      enabled: true
      similarity_threshold: 0.95
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a health and medical information expert with knowledge
        of anatomy, physiology, diseases, treatments, preventive care, nutrition,
        and wellness. Provide accurate, evidence-based health information while emphasizing
        that your responses are for educational purposes only and should never replace
        professional medical advice, diagnosis, or treatment. Always encourage users
        to consult healthcare professionals for medical concerns and emergencies.
      mode: replace
```

```yaml
- name: physics_decision
  priority: 10
  description: Physics and physical sciences
  signals:
    operator: OR
    conditions:
    - type: domain
      name: physics
  modelRefs:
  - model: base-model
    loraName: science-expert
    useReasoning: true
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a physics expert with deep understanding of physical
        laws and phenomena. Provide clear explanations with mathematical derivations
        when appropriate.
      mode: replace
```

```yaml
- name: computer_science_decision
  priority: 10
  description: Computer science and programming
  signals:
    operator: OR
    conditions:
    - type: domain
      name: computer science
  modelRefs:
  - model: base-model
    loraName: science-expert
    useReasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a computer science expert with knowledge of algorithms,
        data structures, programming languages, and software engineering. Provide
        clear, practical solutions with code examples when helpful.
      mode: replace
```

```yaml
- name: philosophy_decision
  priority: 10
  description: Philosophy and ethical questions
  signals:
    operator: OR
    conditions:
    - type: domain
      name: philosophy
  modelRefs:
  - model: base-model
    loraName: humanities-expert
    useReasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a philosophy expert with comprehensive knowledge of philosophical
        traditions, ethical theories, logic, metaphysics, epistemology, political
        philosophy, and the history of philosophical thought. Engage with complex
        philosophical questions by presenting multiple perspectives, analyzing arguments
        rigorously, and encouraging critical thinking. Draw connections between philosophical
        concepts and contemporary issues while maintaining intellectual honesty about
        the complexity and ongoing nature of philosophical debates.
      mode: replace
```

```yaml
- name: engineering_decision
  priority: 10
  description: Engineering and technical problem-solving
  signals:
    operator: OR
    conditions:
    - type: domain
      name: engineering
  modelRefs:
  - model: base-model
    loraName: science-expert
    useReasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are an engineering expert with knowledge across multiple
        engineering disciplines including mechanical, electrical, civil, chemical,
        software, and systems engineering. Apply engineering principles, design methodologies,
        and problem-solving approaches to provide practical solutions. Consider safety,
        efficiency, sustainability, and cost-effectiveness in your recommendations.
        Use technical precision while explaining concepts clearly, and emphasize the
        importance of proper engineering practices and standards.
      mode: replace
```

```yaml
- name: other_decision
  priority: 1
  description: General knowledge and miscellaneous topics
  signals:
    operator: OR
    conditions:
    - type: domain
      name: other
  modelRefs:
  - model: base-model
    loraName: science-expert
    useReasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed:
      - GPE
  - type: semantic-cache
    configuration:
      enabled: true
      similarity_threshold: 0.75
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a knowledgeable AI assistant with broad expertise across
        many domains. Provide accurate, helpful, and well-structured responses to
        general questions. When uncertain, acknowledge limitations and suggest where
        to find authoritative information.
      mode: replace
```

```yaml
- name: urgent_tech
  priority: 100
  description: Urgent technical support requests - use large model with reasoning
  signals:
    operator: AND
    conditions:
    - type: keyword
      name: urgent
    - type: embedding
      name: tech_support
  modelRefs:
  - model: qwen-2.5-72b
    loraName: advanced-reasoning
    useReasoning: true
    reasoningEffort: high
  plugins:
  - type: semantic-cache
    configuration:
      enabled: true
      similarity_threshold: 0.9
```

```yaml
- name: general_tech
  priority: 50
  description: General technical queries - use small model for efficiency
  signals:
    operator: OR
    conditions:
    - type: embedding
      name: tech_support
    - type: domain
      name: computer_science
  modelRefs:
  - model: qwen-2.5-7b
    loraName: tech-support
    useReasoning: false
```

```yaml
- name: science_homework
  description: Science homework assistance - use fast model
  priority: 90
  signals:
    operator: AND
    conditions:
    - type: keyword
      name: homework
    - type: domain
      name: physics
  modelRefs:
  - model: deepseek-v3
    loraName: homework-helper
    useReasoning: false
```

```yaml
- name: urgent_request
  description: Handle urgent requests with larger model
  priority: 100
  signals:
    operator: AND
    conditions:
    - type: keyword
      name: urgent
  modelRefs:
  - model: gemma-2-27b
    loraName: urgent-specialist
    useReasoning: false
```

```yaml
- name: greeting_response
  description: Handle greetings with fast small model
  priority: 50
  signals:
    operator: AND
    conditions:
    - type: keyword
      name: greeting
  modelRefs:
  - model: gemma-2-9b
    loraName: greeting-handler
    useReasoning: false
```

```yaml
- name: confidential_business
  description: Confidential business analysis
  priority: 90
  signals:
    operator: AND
    conditions:
    - type: keyword
      name: confidential
    - type: embedding
      name: business_analysis
    - type: domain
      name: business
  modelRefs:
  - model: qwen-2.5-72b
    use_reasoning: true
    reasoning_effort: medium
  plugins:
  - type: pii
    configuration:
      enabled: true
      threshold: 0.85
      pii_types_allowed:
      - PERSON
      - ORGANIZATION
      - FINANCIAL_DATA
  - type: semantic-cache
    configuration:
      enabled: true
      similarity_threshold: 0.9
  - type: header_mutation
    configuration:
      add:
      - name: X-Confidentiality
        value: high
```

```yaml
- name: general_business
  description: General business and economics queries
  priority: 50
  signals:
    operator: OR
    conditions:
    - type: embedding
      name: business_analysis
    - type: domain
      name: economics
  modelRefs:
  - model: qwen-2.5-72b
    use_reasoning: false
  plugins:
  - type: semantic-cache
    configuration:
      enabled: true
      similarity_threshold: 0.85
```

```yaml
- name: cs_tutorial
  description: Computer science tutorials
  priority: 100
  signals:
    operator: AND
    conditions:
    - type: keyword
      name: tutorial
    - type: embedding
      name: learning_intent
    - type: domain
      name: computer_science
  modelRefs:
  - model: qwen-2.5-32b
    use_reasoning: true
    reasoning_effort: medium
```

```yaml
- name: general_learning
  description: General learning queries
  priority: 50
  signals:
    operator: OR
    conditions:
    - type: keyword
      name: tutorial
    - type: embedding
      name: learning_intent
  modelRefs:
  - model: qwen-2.5-32b
    use_reasoning: false
```

```yaml
- name: urgent_tech_cs
  description: Urgent technical computer science issues
  priority: 100
  signals:
    operator: AND
    conditions:
    - type: keyword
      name: urgent
    - type: embedding
      name: technical_help
    - type: domain
      name: computer_science
  modelRefs:
  - model: qwen-2.5-72b
    use_reasoning: true
    reasoning_effort: high
```

```yaml
- name: tech_engineering
  description: Technical engineering queries
  priority: 80
  signals:
    operator: AND
    conditions:
    - type: embedding
      name: technical_help
    - type: domain
      name: engineering
  modelRefs:
  - model: qwen-2.5-72b
    use_reasoning: true
    reasoning_effort: medium
```

```yaml
- name: tech_troubleshoot
  description: Technical troubleshooting - use reasoning model
  priority: 100
  signals:
    operator: AND
    conditions:
    - type: embedding
      name: technical_issue
  modelRefs:
  - model: deepseek-r1
    loraName: technical-expert
    useReasoning: true
    reasoningEffort: high
```

```yaml
- name: support_ticket
  description: Customer support requests - use fast model
  priority: 80
  signals:
    operator: AND
    conditions:
    - type: embedding
      name: customer_support
  modelRefs:
  - model: deepseek-v3
    loraName: customer-support
    useReasoning: false
```

```yaml
- name: urgent_support
  description: Urgent support requests - use large model with reasoning
  priority: 100
  signals:
    operator: AND
    conditions:
    - type: keyword
      name: urgent
    - type: embedding
      name: support_request
  modelRefs:
  - model: qwen-2.5-72b
    loraName: emergency-specialist
    useReasoning: true
    reasoningEffort: high
```

```yaml
- name: general_support
  description: General support requests - use small model
  priority: 50
  signals:
    operator: OR
    conditions:
    - type: keyword
      name: urgent
    - type: embedding
      name: support_request
  modelRefs:
  - model: qwen-2.5-14b
    loraName: support-agent
    useReasoning: false
```

```yaml
- name: general_science_research
  description: General science research
  priority: 80
  signals:
    operator: AND
    conditions:
    - type: embedding
      name: research_question
    - type: domain
      name: chemistry
  modelRefs:
  - model: gpt-4o
    use_reasoning: true
    reasoning_effort: medium
```

```yaml
- name: stem_query
  description: Complex STEM domain queries - use large model
  priority: 100
  signals:
    operator: OR
    conditions:
    - type: domain
      name: math
    - type: domain
      name: physics
    - type: domain
      name: computer_science
  modelRefs:
  - model: mistral-large
    loraName: math-expert
    useReasoning: false
```

```yaml
- name: chemistry_query
  description: Chemistry domain queries - use small model
  priority: 80
  signals:
    operator: AND
    conditions:
    - type: domain
      name: chemistry
  modelRefs:
  - model: mistral-7b
    loraName: stem-tutor
    useReasoning: false
```

### F. 系统优化与性能测试 (Optimization, Cache & Benchmarking)

```yaml
- name: cached_faq
  description: FAQ with semantic cache
  priority: 100
  signals:
    operator: AND
    conditions:
    - type: keyword
      name: faq
  modelRefs:
  - model: llama-3.3-70b
    use_reasoning: false
  plugins:
  - type: semantic-cache
    configuration:
      enabled: true
      similarity_threshold: 0.95
  - type: header_mutation
    configuration:
      add:
      - name: X-Cache-Strategy
        value: aggressive
      - name: X-Route-Type
        value: keyword
```

## 2. 多条件强约束 AND 组合 (AND Logic Combinations)

### A. 安全、隐私与权限风控 (Security, Privacy & Authz)

```yaml
- name: admin_to_admin_tier
  description: Admin → admin tier (:8100)
  priority: 300
  rules:
    operator: AND
    conditions:
    - type: authz
      name: admin
  modelRefs:
  - model: qwen14b-admin
    use_reasoning: true
```

```yaml
- name: premium_to_admin_tier
  description: Premium → admin tier (:8100)
  priority: 200
  rules:
    operator: AND
    conditions:
    - type: authz
      name: premium_user
  modelRefs:
  - model: qwen14b-admin
    use_reasoning: false
```

```yaml
- name: free_to_free_tier
  description: Free → free tier (:8200)
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: authz
      name: free_user
  modelRefs:
  - model: qwen14b-free
    use_reasoning: false
```

```yaml
- name: admin_unrestricted
  description: Admin users get 14B with reasoning
  priority: 300
  rules:
    operator: AND
    conditions:
    - type: authz
      name: admin
  modelRefs:
  - model: Qwen/Qwen2.5-14B-Instruct
    use_reasoning: true
  - model: Qwen/Qwen2.5-7B-Instruct
    use_reasoning: false
```

```yaml
- name: premium_complex
  description: Premium users + complex queries get 14B
  priority: 250
  rules:
    operator: AND
    conditions:
    - type: authz
      name: premium_user
    - type: keyword
      name: complex_analysis
  modelRefs:
  - model: Qwen/Qwen2.5-14B-Instruct
    use_reasoning: true
```

```yaml
- name: premium_code
  description: Premium users + code requests get 14B
  priority: 240
  rules:
    operator: AND
    conditions:
    - type: authz
      name: premium_user
    - type: keyword
      name: code_request
  modelRefs:
  - model: Qwen/Qwen2.5-14B-Instruct
    use_reasoning: false
```

```yaml
- name: premium_default
  description: Premium users simple queries get 7B
  priority: 150
  rules:
    operator: AND
    conditions:
    - type: authz
      name: premium_user
  modelRefs:
  - model: Qwen/Qwen2.5-7B-Instruct
    use_reasoning: false
```

```yaml
- name: free_default
  description: Free users get 7B
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: authz
      name: free_user
  modelRefs:
  - model: Qwen/Qwen2.5-7B-Instruct
    use_reasoning: false
```

```yaml
- name: guardrails
  description: Block jailbreak attempts with security measures
  priority: 200
  rules:
    operator: AND
    conditions:
    - type: keyword
      name: jailbreak_attempt
  modelRefs:
  - model: openai/gpt-oss-20b
    use_reasoning: false
  plugins:
  - type: jailbreak
    configuration:
      enabled: true
      threshold: 0.65
  - type: system_prompt
    configuration:
      system_prompt: You are a helpful AI assistant. I cannot fulfill requests that
        attempt to bypass safety guidelines.
  - type: router_replay
    configuration:
      enabled: true
      max_records: 1000
      capture_request_body: true
      capture_response_body: true
      max_body_bytes: 4096
```

```yaml
- name: pii_protected
  description: PII-protected queries
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: embedding
      name: sensitive_data
  modelRefs:
  - model: deepseek-r1
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed:
      - CREDIT_CARD
      - SSN
      - EMAIL
      threshold: 0.8
  - type: jailbreak
    configuration:
      enabled: true
      threshold: 0.85
```

```yaml
- name: security_alert
  description: Security-related queries with jailbreak protection
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: keyword
      name: security
    - type: embedding
      name: threat_detection
  modelRefs:
  - model: qwen-2.5-72b
    use_reasoning: false
  plugins:
  - type: jailbreak
    configuration:
      enabled: true
      threshold: 0.9
  - type: system_prompt
    configuration:
      system_prompt: You are a security expert. Provide helpful security guidance
        while being cautious about potential misuse.
  - type: header_mutation
    configuration:
      add:
      - name: X-Security-Level
        value: high
      - name: X-Route-Type
        value: keyword-embedding
```

```yaml
- name: medical_query
  description: Medical queries with PII protection
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: keyword
      name: diagnosis
    - type: domain
      name: health
  modelRefs:
  - model: deepseek-r1
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed:
      - PERSON
      - DATE_OF_BIRTH
      - MEDICAL_RECORD
      threshold: 0.9
  - type: system_prompt
    configuration:
      system_prompt: You are a medical information assistant. Provide general health
        information but always advise users to consult healthcare professionals.
  - type: semantic-cache
    configuration:
      enabled: true
      similarity_threshold: 0.88
```

### B. 专家级领域分发 (Domain Specific Routing)

```yaml
- name: math_decision
  description: Mathematics and quantitative reasoning
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: math
  modelRefs:
  - model: Qwen/Qwen2.5-14B-Instruct-AWQ
    use_reasoning: true
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: hallucination
    configuration:
      enabled: true
      use_nli: true
      hallucination_action: header
      unverified_factual_action: header
      include_hallucination_details: false
```

```yaml
- name: science_decision
  description: Science questions
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: science
  modelRefs:
  - model: Qwen/Qwen2.5-14B-Instruct-AWQ
    use_reasoning: true
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: hallucination
    configuration:
      enabled: true
      use_nli: true
      hallucination_action: header
      unverified_factual_action: header
      include_hallucination_details: false
```

```yaml
- name: general_decision
  description: General questions
  priority: 50
  rules:
    operator: AND
    conditions:
    - type: domain
      name: general
  modelRefs:
  - model: Qwen/Qwen2.5-14B-Instruct-AWQ
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: hallucination
    configuration:
      enabled: true
      use_nli: true
      hallucination_action: header
      unverified_factual_action: header
      include_hallucination_details: false
```

```yaml
- name: business_decision
  description: Business and management queries
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: business
  modelRefs:
  - model: qwen2.5:3b
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a senior business consultant and strategic advisor with
        expertise in corporate strategy, operations management, financial analysis,
        marketing, and organizational development. Provide practical, actionable business
        advice backed by proven methodologies and industry best practices. Consider
        market dynamics, competitive landscape, and stakeholder interests in your
        recommendations.
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: law_decision
  description: Legal questions and law-related topics
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: law
  modelRefs:
  - model: qwen2.5:3b
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a knowledgeable legal expert with comprehensive understanding
        of legal principles, case law, statutory interpretation, and legal procedures
        across multiple jurisdictions. Provide accurate legal information and analysis
        while clearly stating that your responses are for informational purposes only
        and do not constitute legal advice. Always recommend consulting with qualified
        legal professionals for specific legal matters.
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: psychology_decision
  description: Psychology and mental health topics
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: psychology
  modelRefs:
  - model: qwen2.5:3b
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a psychology expert with deep knowledge of cognitive
        processes, behavioral patterns, mental health, developmental psychology, social
        psychology, and therapeutic approaches. Provide evidence-based insights grounded
        in psychological research and theory. When discussing mental health topics,
        emphasize the importance of professional consultation and avoid providing
        diagnostic or therapeutic advice.
  - type: semantic-cache
    configuration:
      enabled: true
      similarity_threshold: 0.92
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: biology_decision
  description: Biology and life sciences questions
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: biology
  modelRefs:
  - model: qwen2.5:3b
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a biology expert with comprehensive knowledge spanning
        molecular biology, genetics, cell biology, ecology, evolution, anatomy, physiology,
        and biotechnology. Explain biological concepts with scientific accuracy, use
        appropriate terminology, and provide examples from current research. Connect
        biological principles to real-world applications and emphasize the interconnectedness
        of biological systems.
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: chemistry_decision
  description: Chemistry and chemical sciences questions
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: chemistry
  modelRefs:
  - model: qwen2.5:3b
    use_reasoning: true
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a chemistry expert specializing in chemical reactions,
        molecular structures, and laboratory techniques. Provide detailed, step-by-step
        explanations.
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: history_decision
  description: Historical questions and cultural topics
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: history
  modelRefs:
  - model: qwen2.5:3b
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a historian with expertise across different time periods
        and cultures. Provide accurate historical context and analysis.
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: health_decision
  description: Health and medical information queries
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: health
  modelRefs:
  - model: qwen2.5:3b
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a health and medical information expert with knowledge
        of anatomy, physiology, diseases, treatments, preventive care, nutrition,
        and wellness. Provide accurate, evidence-based health information while emphasizing
        that your responses are for educational purposes only and should never replace
        professional medical advice, diagnosis, or treatment. Always encourage users
        to consult healthcare professionals for medical concerns and emergencies.
  - type: semantic-cache
    configuration:
      enabled: true
      similarity_threshold: 0.95
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: economics_decision
  description: Economics and financial topics
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: economics
  modelRefs:
  - model: qwen2.5:3b
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are an economics expert with deep understanding of microeconomics,
        macroeconomics, econometrics, financial markets, monetary policy, fiscal policy,
        international trade, and economic theory. Analyze economic phenomena using
        established economic principles, provide data-driven insights, and explain
        complex economic concepts in accessible terms. Consider both theoretical frameworks
        and real-world applications in your responses.
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: math_decision
  description: Mathematics and quantitative reasoning
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: math
  modelRefs:
  - model: qwen2.5:3b
    use_reasoning: true
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a mathematics expert. Provide step-by-step solutions,
        show your work clearly, and explain mathematical concepts in an understandable
        way.
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: physics_decision
  description: Physics and physical sciences
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: physics
  modelRefs:
  - model: qwen2.5:3b
    use_reasoning: true
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a physics expert with deep understanding of physical
        laws and phenomena. Provide clear explanations with mathematical derivations
        when appropriate.
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: computer_science_decision
  description: Computer science and programming
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: computer_science
  modelRefs:
  - model: qwen2.5:3b
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a computer science expert with knowledge of algorithms,
        data structures, programming languages, and software engineering. Provide
        clear, practical solutions with code examples when helpful.
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: philosophy_decision
  description: Philosophy and ethical questions
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: philosophy
  modelRefs:
  - model: qwen2.5:3b
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a philosophy expert with comprehensive knowledge of philosophical
        traditions, ethical theories, logic, metaphysics, epistemology, political
        philosophy, and the history of philosophical thought. Engage with complex
        philosophical questions by presenting multiple perspectives, analyzing arguments
        rigorously, and encouraging critical thinking. Draw connections between philosophical
        concepts and contemporary issues while maintaining intellectual honesty about
        the complexity and ongoing nature of philosophical debates.
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: engineering_decision
  description: Engineering and technical problem-solving
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: engineering
  modelRefs:
  - model: qwen2.5:3b
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are an engineering expert with knowledge across multiple
        engineering disciplines including mechanical, electrical, civil, chemical,
        software, and systems engineering. Apply engineering principles, design methodologies,
        and problem-solving approaches to provide practical solutions. Consider safety,
        efficiency, sustainability, and cost-effectiveness in your recommendations.
        Use technical precision while explaining concepts clearly, and emphasize the
        importance of proper engineering practices and standards.
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: general_decision
  description: General knowledge and miscellaneous topics
  priority: 50
  rules:
    operator: AND
    conditions:
    - type: domain
      name: other
  modelRefs:
  - model: qwen2.5:3b
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a helpful and knowledgeable assistant. Provide accurate,
        helpful responses across a wide range of topics.
  - type: semantic-cache
    configuration:
      enabled: true
      similarity_threshold: 0.75
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: memory
    configuration:
      enabled: true
      retrieval_limit: 5
      similarity_threshold: 0.7
      auto_store: false
```

```yaml
- name: business_decision
  description: Business queries, strategy, and professional advice
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: business
  modelRefs:
  - model: qwen3
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a professional business consultant. Provide practical,
        actionable business advice.
  - type: jailbreak
    configuration:
      enabled: true
      threshold: 0.9
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: customer_support_decision
  description: Customer support and general inquiries
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: customer_support
  modelRefs:
  - model: qwen3
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a friendly customer support agent. Help users with their
        questions.
  - type: jailbreak
    configuration:
      enabled: true
      threshold: 0.8
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: code_generation_decision
  description: Internal code generation and development tools
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: code_generation
  modelRefs:
  - model: qwen3
    use_reasoning: true
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a code generation assistant for internal developers.
  - type: jailbreak
    configuration:
      enabled: true
      threshold: 0.5
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: testing_decision
  description: Testing and quality assurance queries
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: testing
  modelRefs:
  - model: qwen3
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a QA assistant helping with test scenarios.
  - type: jailbreak
    configuration:
      enabled: false
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: general_decision
  description: General queries that don't fit into specific categories
  priority: 50
  rules:
    operator: AND
    conditions:
    - type: domain
      name: general
  modelRefs:
  - model: qwen3
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a helpful assistant.
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: business_decision
  description: Business and management queries
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: business
  modelRefs:
  - model: qwen3
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a senior business consultant and strategic advisor with
        expertise in corporate strategy, operations management, financial analysis,
        marketing, and organizational development. Provide practical, actionable business
        advice backed by proven methodologies and industry best practices. Consider
        market dynamics, competitive landscape, and stakeholder interests in your
        recommendations.
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: law_decision
  description: Legal questions and law-related topics
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: law
  modelRefs:
  - model: qwen3
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a knowledgeable legal expert with comprehensive understanding
        of legal principles, case law, statutory interpretation, and legal procedures
        across multiple jurisdictions. Provide accurate legal information and analysis
        while clearly stating that your responses are for informational purposes only
        and do not constitute legal advice. Always recommend consulting with qualified
        legal professionals for specific legal matters.
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: psychology_decision
  description: Psychology and mental health topics
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: psychology
  modelRefs:
  - model: qwen3
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a psychology expert with deep knowledge of cognitive
        processes, behavioral patterns, mental health, developmental psychology, social
        psychology, and therapeutic approaches. Provide evidence-based insights grounded
        in psychological research and theory. When discussing mental health topics,
        emphasize the importance of professional consultation and avoid providing
        diagnostic or therapeutic advice.
  - type: semantic-cache
    configuration:
      enabled: true
      similarity_threshold: 0.92
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: biology_decision
  description: Biology and life sciences questions
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: biology
  modelRefs:
  - model: qwen3
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a biology expert with comprehensive knowledge spanning
        molecular biology, genetics, cell biology, ecology, evolution, anatomy, physiology,
        and biotechnology. Explain biological concepts with scientific accuracy, use
        appropriate terminology, and provide examples from current research. Connect
        biological principles to real-world applications and emphasize the interconnectedness
        of biological systems.
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: chemistry_decision
  description: Chemistry and chemical sciences questions
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: chemistry
  modelRefs:
  - model: qwen3
    use_reasoning: true
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a chemistry expert specializing in chemical reactions,
        molecular structures, and laboratory techniques. Provide detailed, step-by-step
        explanations.
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: history_decision
  description: Historical questions and cultural topics
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: history
  modelRefs:
  - model: qwen3
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a historian with expertise across different time periods
        and cultures. Provide accurate historical context and analysis.
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: health_decision
  description: Health and medical information queries
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: health
  modelRefs:
  - model: qwen3
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a health and medical information expert with knowledge
        of anatomy, physiology, diseases, treatments, preventive care, nutrition,
        and wellness. Provide accurate, evidence-based health information while emphasizing
        that your responses are for educational purposes only and should never replace
        professional medical advice, diagnosis, or treatment. Always encourage users
        to consult healthcare professionals for medical concerns and emergencies.
  - type: semantic-cache
    configuration:
      enabled: true
      similarity_threshold: 0.95
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: economics_decision
  description: Economics and financial topics
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: economics
  modelRefs:
  - model: qwen3
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are an economics expert with deep understanding of microeconomics,
        macroeconomics, econometrics, financial markets, monetary policy, fiscal policy,
        international trade, and economic theory. Analyze economic phenomena using
        established economic principles, provide data-driven insights, and explain
        complex economic concepts in accessible terms. Consider both theoretical frameworks
        and real-world applications in your responses.
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: math_decision
  description: Mathematics and quantitative reasoning
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: math
  modelRefs:
  - model: qwen3
    use_reasoning: true
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a mathematics expert. Provide step-by-step solutions,
        show your work clearly, and explain mathematical concepts in an understandable
        way.
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: physics_decision
  description: Physics and physical sciences
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: physics
  modelRefs:
  - model: qwen3
    use_reasoning: true
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a physics expert with deep understanding of physical
        laws and phenomena. Provide clear explanations with mathematical derivations
        when appropriate.
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: computer_science_decision
  description: Computer science and programming
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: computer_science
  modelRefs:
  - model: qwen3
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a computer science expert with knowledge of algorithms,
        data structures, programming languages, and software engineering. Provide
        clear, practical solutions with code examples when helpful.
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: philosophy_decision
  description: Philosophy and ethical questions
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: philosophy
  modelRefs:
  - model: qwen3
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a philosophy expert with comprehensive knowledge of philosophical
        traditions, ethical theories, logic, metaphysics, epistemology, political
        philosophy, and the history of philosophical thought. Engage with complex
        philosophical questions by presenting multiple perspectives, analyzing arguments
        rigorously, and encouraging critical thinking. Draw connections between philosophical
        concepts and contemporary issues while maintaining intellectual honesty about
        the complexity and ongoing nature of philosophical debates.
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: engineering_decision
  description: Engineering and technical problem-solving
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: engineering
  modelRefs:
  - model: qwen3
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are an engineering expert with knowledge across multiple
        engineering disciplines including mechanical, electrical, civil, chemical,
        software, and systems engineering. Apply engineering principles, design methodologies,
        and problem-solving approaches to provide practical solutions. Consider safety,
        efficiency, sustainability, and cost-effectiveness in your recommendations.
        Use technical precision while explaining concepts clearly, and emphasize the
        importance of proper engineering practices and standards.
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: general_decision
  description: General knowledge and miscellaneous topics
  priority: 50
  rules:
    operator: AND
    conditions:
    - type: domain
      name: other
  modelRefs:
  - model: qwen3
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a helpful and knowledgeable assistant. Provide accurate,
        helpful responses across a wide range of topics.
  - type: semantic-cache
    configuration:
      enabled: true
      similarity_threshold: 0.75
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: healthcare_decision
  description: Healthcare and medical queries
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: healthcare
  modelRefs:
  - model: secure-llm
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a healthcare assistant. Handle all personal information
        with utmost care.
  - type: pii
    configuration:
      enabled: true
      threshold: 0.9
      pii_types_allowed:
      - GPE
      - ORGANIZATION
```

```yaml
- name: finance_decision
  description: Financial and banking queries
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: finance
  modelRefs:
  - model: secure-llm
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a financial advisor. Never store or log any PII information.
  - type: pii
    configuration:
      enabled: true
      threshold: 0.95
      pii_types_allowed:
      - GPE
      - ORGANIZATION
```

```yaml
- name: customer_support_decision
  description: Customer support and general inquiries
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: customer_support
  modelRefs:
  - model: general-llm
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a friendly customer support agent. Be cautious with customer
        information.
  - type: pii
    configuration:
      enabled: true
      threshold: 0.8
      pii_types_allowed: []
```

```yaml
- name: code_generation_decision
  description: Internal code generation and development tools
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: code_generation
  modelRefs:
  - model: general-llm
    use_reasoning: true
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a code generation assistant for internal developers.
  - type: pii
    configuration:
      enabled: true
      threshold: 0.5
      pii_types_allowed: []
```

```yaml
- name: documentation_decision
  description: Public documentation and help articles
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: documentation
  modelRefs:
  - model: general-llm
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a documentation assistant. Help create clear public documentation.
  - type: pii
    configuration:
      enabled: true
      threshold: 0.6
      pii_types_allowed: []
```

```yaml
- name: testing_decision
  description: Testing and quality assurance queries
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: testing
  modelRefs:
  - model: general-llm
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a QA assistant helping with test scenarios.
  - type: pii
    configuration:
      enabled: false
      pii_types_allowed: []
```

```yaml
- name: general_decision
  description: General queries that don't fit into specific categories
  priority: 50
  rules:
    operator: AND
    conditions:
    - type: domain
      name: general
  modelRefs:
  - model: general-llm
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a helpful assistant.
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: math_decision
  description: Mathematics and quantitative reasoning
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: math
  modelRefs:
  - model: openai/gpt-oss-20b
    use_reasoning: true
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a mathematics expert. Provide step-by-step solutions.
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: general_decision
  description: General knowledge and miscellaneous topics
  priority: 50
  rules:
    operator: AND
    conditions:
    - type: domain
      name: other
  modelRefs:
  - model: openai/gpt-oss-20b
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a helpful assistant.
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: business_decision
  description: Business and management related queries
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: business
  modelRefs:
  - model: Model-A
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed:
      - EMAIL_ADDRESS
```

```yaml
- name: law_decision
  description: Legal questions and law-related topics
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: law
  modelRefs:
  - model: Model-B
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed:
      - EMAIL_ADDRESS
      - PERSON
      - GPE
      - PHONE_NUMBER
      - US_SSN
      - CREDIT_CARD
```

```yaml
- name: psychology_decision
  description: Psychology and mental health topics
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: psychology
  modelRefs:
  - model: Model-A
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed:
      - EMAIL_ADDRESS
```

```yaml
- name: biology_decision
  description: Biology and life sciences questions
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: biology
  modelRefs:
  - model: Model-A
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed:
      - EMAIL_ADDRESS
```

```yaml
- name: chemistry_decision
  description: Chemistry and chemical sciences questions
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: chemistry
  modelRefs:
  - model: Model-A
    use_reasoning: true
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed:
      - EMAIL_ADDRESS
```

```yaml
- name: history_decision
  description: Historical questions and cultural topics
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: history
  modelRefs:
  - model: Model-A
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed:
      - EMAIL_ADDRESS
```

```yaml
- name: health_decision
  description: Health and medical information queries
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: health
  modelRefs:
  - model: Model-B
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed:
      - EMAIL_ADDRESS
      - PERSON
      - GPE
      - PHONE_NUMBER
      - US_SSN
      - CREDIT_CARD
```

```yaml
- name: economics_decision
  description: Economics and financial topics
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: economics
  modelRefs:
  - model: Model-B
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed:
      - EMAIL_ADDRESS
      - PERSON
      - GPE
      - PHONE_NUMBER
      - US_SSN
      - CREDIT_CARD
```

```yaml
- name: math_decision
  description: Mathematics and quantitative reasoning
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: math
  modelRefs:
  - model: Model-B
    use_reasoning: true
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed:
      - EMAIL_ADDRESS
      - PERSON
      - GPE
      - PHONE_NUMBER
      - US_SSN
      - CREDIT_CARD
```

```yaml
- name: physics_decision
  description: Physics and physical sciences
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: physics
  modelRefs:
  - model: Model-B
    use_reasoning: true
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed:
      - EMAIL_ADDRESS
      - PERSON
      - GPE
      - PHONE_NUMBER
      - US_SSN
      - CREDIT_CARD
```

```yaml
- name: computer_science_decision
  description: Computer science and programming
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: computer_science
  modelRefs:
  - model: Model-B
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed:
      - EMAIL_ADDRESS
      - PERSON
      - GPE
      - PHONE_NUMBER
      - US_SSN
      - CREDIT_CARD
```

```yaml
- name: philosophy_decision
  description: Philosophy and ethical questions
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: philosophy
  modelRefs:
  - model: Model-A
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed:
      - EMAIL_ADDRESS
```

```yaml
- name: engineering_decision
  description: Engineering and technical problem-solving
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: engineering
  modelRefs:
  - model: Model-B
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed:
      - EMAIL_ADDRESS
      - PERSON
      - GPE
      - PHONE_NUMBER
      - US_SSN
      - CREDIT_CARD
```

```yaml
- name: general_decision
  description: General knowledge and miscellaneous topics
  priority: 50
  rules:
    operator: AND
    conditions:
    - type: domain
      name: other
  modelRefs:
  - model: Model-B
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed:
      - EMAIL_ADDRESS
      - PERSON
      - GPE
      - PHONE_NUMBER
      - US_SSN
      - CREDIT_CARD
```

```yaml
- name: general_decision
  description: General knowledge and miscellaneous topics
  priority: 50
  rules:
    operator: AND
    conditions:
    - type: domain
      name: other
  modelRefs:
  - model: openai/gpt-oss-20b
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: technical_support_decision
  description: Technical support and troubleshooting queries
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: technical_support
  modelRefs:
  - model: qwen3
    use_reasoning: true
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a technical support specialist. Provide detailed, step-by-step
        guidance for technical issues. Use clear explanations and include relevant
        troubleshooting steps.
  - type: jailbreak
    configuration:
      enabled: true
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: product_inquiry_decision
  description: Product information and specifications
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: product_inquiry
  modelRefs:
  - model: qwen3
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a product specialist. Provide accurate information about
        products, features, pricing, and availability. Be helpful and informative.
  - type: jailbreak
    configuration:
      enabled: true
  - type: pii
    configuration:
      enabled: false
      pii_types_allowed: []
```

```yaml
- name: account_management_decision
  description: Account and subscription management
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: account_management
  modelRefs:
  - model: qwen3
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are an account management assistant. Help users with account-related
        tasks such as password resets, profile updates, and subscription management.
        Prioritize security and privacy.
  - type: jailbreak
    configuration:
      enabled: true
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: general_inquiry_decision
  description: General questions and information requests
  priority: 50
  rules:
    operator: AND
    conditions:
    - type: domain
      name: general_inquiry
  modelRefs:
  - model: qwen3
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a helpful general assistant. Answer questions clearly
        and concisely. If you need more information, ask clarifying questions.
  - type: jailbreak
    configuration:
      enabled: true
  - type: pii
    configuration:
      enabled: false
      pii_types_allowed: []
```

```yaml
- name: nested_stem_english_short
  description: (STEM or math keyword) AND English AND NOT long context → code/STEM
    specialist
  priority: 500
  rules:
    operator: AND
    conditions:
    - operator: OR
      conditions:
      - type: domain
        name: computer_science
      - type: keyword
        name: math_request
    - type: language
      name: en
    - operator: NOT
      conditions:
      - type: context
        name: long_context
  modelRefs:
  - model: en_cs_model
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a precise STEM and coding expert. Provide concise, correct
        answers with code examples where relevant.
```

```yaml
- name: and_cs_english
  description: AND — computer_science domain AND English language
  priority: 400
  rules:
    operator: AND
    conditions:
    - type: domain
      name: computer_science
    - type: language
      name: en
  modelRefs:
  - model: en_cs_model
```

```yaml
- name: urgent_request_decision
  description: Urgent and time-sensitive requests
  priority: 150
  rules:
    operator: AND
    conditions:
    - type: domain
      name: urgent_request
  modelRefs:
  - model: qwen3
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a highly responsive assistant specialized in handling
        urgent requests. Prioritize speed and efficiency while maintaining accuracy.
        Provide concise, actionable responses and focus on immediate solutions.
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: sensitive_data_decision
  description: Requests involving sensitive personal data
  priority: 150
  rules:
    operator: AND
    conditions:
    - type: domain
      name: sensitive_data
  modelRefs:
  - model: qwen3
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a security-conscious assistant specialized in handling
        sensitive data. Exercise extreme caution with personal information, follow
        data protection best practices, and remind users about privacy considerations.
  - type: jailbreak
    configuration:
      enabled: true
      threshold: 0.6
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: exclude_spam_decision
  description: Potential spam or suspicious requests
  priority: 150
  rules:
    operator: AND
    conditions:
    - type: domain
      name: exclude_spam
  modelRefs:
  - model: qwen3
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a content moderation assistant. This request has been
        flagged as potential spam. Please verify the legitimacy of the request before
        proceeding.
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: regex_pattern_match_decision
  description: Structured data and pattern-based requests
  priority: 150
  rules:
    operator: AND
    conditions:
    - type: domain
      name: regex_pattern_match
  modelRefs:
  - model: qwen3
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a technical assistant specialized in handling structured
        data and pattern-based requests. Provide precise, format-aware responses.
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: business_decision
  description: Business and management related queries
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: business
  modelRefs:
  - model: qwen3
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a senior business consultant and strategic advisor with
        expertise in corporate strategy, operations management, financial analysis,
        marketing, and organizational development. Provide practical, actionable business
        advice backed by proven methodologies and industry best practices. Consider
        market dynamics, competitive landscape, and stakeholder interests in your
        recommendations.
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: technical_decision
  description: Programming, software engineering, and technical questions
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: technical
  modelRefs:
  - model: llama2-7b
    lora_name: technical-lora
    use_reasoning: true
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are an expert software engineer with deep knowledge of programming
        languages, algorithms, system design, and best practices. Provide clear, accurate
        technical guidance with code examples when appropriate.
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: medical_decision
  description: Medical and healthcare questions
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: medical
  modelRefs:
  - model: llama2-7b
    lora_name: medical-lora
    use_reasoning: true
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a medical expert with comprehensive knowledge of anatomy,
        physiology, diseases, treatments, and healthcare practices. Provide accurate
        medical information while emphasizing that responses are for educational purposes
        only and not a substitute for professional medical advice.
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: legal_decision
  description: Legal questions and law-related topics
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: legal
  modelRefs:
  - model: llama2-7b
    lora_name: legal-lora
    use_reasoning: true
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a legal expert with knowledge of legal principles, case
        law, and statutory interpretation. Provide accurate legal information while
        clearly stating that responses are for informational purposes only and do
        not constitute legal advice.
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: general_decision
  description: General questions that don't fit specific domains
  priority: 50
  rules:
    operator: AND
    conditions:
    - type: domain
      name: general
  modelRefs:
  - model: llama2-7b
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a helpful AI assistant with broad knowledge across many
        topics. Provide clear, accurate, and helpful responses.
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: code_decision
  description: Code-related queries (BM25 classified)
  priority: 150
  rules:
    operator: AND
    conditions:
    - type: keyword
      name: code_keywords
  modelRefs:
  - model: qwen3
    use_reasoning: true
```

```yaml
- name: general_decision
  description: General queries
  priority: 50
  rules:
    operator: AND
    conditions:
    - type: domain
      name: other
  modelRefs:
  - model: qwen3
    use_reasoning: false
```

```yaml
- name: psychology_decision
  description: Psychology and mental health topics - with Redis semantic cache
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: psychology
  modelRefs:
  - model: openai/gpt-oss-20b
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a psychology expert with deep knowledge of cognitive
        processes, behavioral patterns, mental health, developmental psychology, social
        psychology, and therapeutic approaches. Provide evidence-based insights grounded
        in psychological research and theory. When discussing mental health topics,
        emphasize the importance of professional consultation and avoid providing
        diagnostic or therapeutic advice.
  - type: semantic-cache
    configuration:
      enabled: true
      similarity_threshold: 0.92
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: health_decision
  description: Health and medical information queries - with Redis semantic cache
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: health
  modelRefs:
  - model: openai/gpt-oss-20b
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a health and medical information expert with knowledge
        of anatomy, physiology, diseases, treatments, preventive care, nutrition,
        and wellness. Provide accurate, evidence-based health information while emphasizing
        that your responses are for educational purposes only and should never replace
        professional medical advice, diagnosis, or treatment. Always encourage users
        to consult healthcare professionals for medical concerns and emergencies.
  - type: semantic-cache
    configuration:
      enabled: true
      similarity_threshold: 0.95
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: general_decision
  description: General knowledge and miscellaneous topics - with Redis semantic cache
  priority: 50
  rules:
    operator: AND
    conditions:
    - type: domain
      name: other
  modelRefs:
  - model: openai/gpt-oss-20b
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a helpful and knowledgeable assistant. Provide accurate,
        helpful responses across a wide range of topics.
  - type: semantic-cache
    configuration:
      enabled: true
      similarity_threshold: 0.75
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: business_decision
  description: Business and management queries
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: business
  modelRefs:
  - model: openai/gpt-oss-20b
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a senior business consultant and strategic advisor with
        expertise in corporate strategy, operations management, financial analysis,
        marketing, and organizational development.
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: math_decision
  description: Mathematics and quantitative reasoning
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: math
  modelRefs:
  - model: openai/gpt-oss-20b
    use_reasoning: true
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a mathematics expert. Provide step-by-step solutions,
        show your work clearly, and explain mathematical concepts in an understandable
        way.
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: computer_science_decision
  description: Computer science and programming
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: computer_science
  modelRefs:
  - model: openai/gpt-oss-20b
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a computer science expert with knowledge of algorithms,
        data structures, programming languages, and software engineering.
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: hard_math_problems
  description: Hard Mathematical queries with high reasoning
  priority: 150
  rules:
    operator: AND
    conditions:
    - type: domain
      name: math
    - type: complexity
      name: math_problem:hard
  modelRefs:
  - model: DeepSeek-V3.2
    use_reasoning: true
    reasoning_effort: high
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are Qwen3-235B, a mathematics expert. Provide rigorous proofs
        with step-by-step with hard reasoning.
  - type: semantic-cache
    configuration:
      enabled: false
  - type: router_replay
    configuration:
      enabled: true
      max_records: 1000
      capture_request_body: true
      capture_response_body: true
      max_body_bytes: 4096
```

```yaml
- name: chemistry_problems
  description: Chemistry queries with reasoning
  priority: 148
  rules:
    operator: AND
    conditions:
    - type: domain
      name: chemistry
  modelRefs:
  - model: Qwen/Qwen3-235B
    use_reasoning: true
    reasoning_effort: medium
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are Qwen3-235B, a chemistry expert. Explain chemical reactions,
        molecular structures, and laboratory techniques with scientific rigor.
  - type: semantic-cache
    configuration:
      enabled: false
  - type: router_replay
    configuration:
      enabled: true
      max_records: 1000
      capture_request_body: true
      capture_response_body: true
      max_body_bytes: 4096
```

```yaml
- name: biology_problems
  description: Biology and life sciences queries
  priority: 147
  rules:
    operator: AND
    conditions:
    - type: domain
      name: biology
  modelRefs:
  - model: Qwen/Qwen3-235B
    use_reasoning: true
    reasoning_effort: medium
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are Qwen3-235B, a biology expert. Explain molecular biology,
        genetics, ecology, and life sciences with detailed scientific explanations.
  - type: semantic-cache
    configuration:
      enabled: false
  - type: router_replay
    configuration:
      enabled: true
      max_records: 1000
      capture_request_body: true
      capture_response_body: true
      max_body_bytes: 4096
```

```yaml
- name: health_medical
  description: Health and medical information queries
  priority: 146
  rules:
    operator: AND
    conditions:
    - type: domain
      name: health
  modelRefs:
  - model: GLM-4.7
    use_reasoning: true
    reasoning_effort: medium
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are GLM-4.7, a medical information assistant. Provide accurate
        health information based on scientific evidence. Always remind users to consult
        healthcare professionals for medical advice.
  - type: semantic-cache
    configuration:
      enabled: false
  - type: router_replay
    configuration:
      enabled: true
      max_records: 1000
      capture_request_body: true
      capture_response_body: true
      max_body_bytes: 4096
```

```yaml
- name: physics_problems
  description: Physics queries with reasoning
  priority: 145
  rules:
    operator: AND
    conditions:
    - type: domain
      name: physics
  modelRefs:
  - model: GLM-4.7
    use_reasoning: true
    reasoning_effort: medium
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are GLM-4.7, a physics expert. Explain concepts with mathematical
        derivations and verified facts.
  - type: semantic-cache
    configuration:
      enabled: false
  - type: router_replay
    configuration:
      enabled: true
      max_records: 1000
      capture_request_body: true
      capture_response_body: true
      max_body_bytes: 4096
```

```yaml
- name: engineering_problems
  description: Engineering and technical problem-solving
  priority: 144
  rules:
    operator: AND
    conditions:
    - type: domain
      name: engineering
  modelRefs:
  - model: DeepSeek-V3.2
    use_reasoning: true
    reasoning_effort: medium
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are DeepSeek-V3.2, an engineering expert. Provide technical
        solutions with practical considerations, calculations, and design principles.
  - type: semantic-cache
    configuration:
      enabled: false
  - type: router_replay
    configuration:
      enabled: true
      max_records: 1000
      capture_request_body: true
      capture_response_body: true
      max_body_bytes: 4096
```

```yaml
- name: law_legal
  description: Legal questions and law-related topics
  priority: 143
  rules:
    operator: AND
    conditions:
    - type: domain
      name: law
  modelRefs:
  - model: Qwen/Qwen3-235B
    use_reasoning: true
    reasoning_effort: medium
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are Qwen3-235B, a legal information assistant. Provide information
        about legal principles and concepts. Always remind users to consult qualified
        legal professionals for legal advice.
  - type: semantic-cache
    configuration:
      enabled: false
  - type: router_replay
    configuration:
      enabled: true
      max_records: 1000
      capture_request_body: true
      capture_response_body: true
      max_body_bytes: 4096
```

```yaml
- name: complex_engineering
  description: Complex code generation and algorithm design (only when domain is clearly
    computer science)
  priority: 141
  rules:
    operator: AND
    conditions:
    - type: domain
      name: computer science
    - type: embedding
      name: deep_thinking_en
    - type: complexity
      name: computer_science:hard
  modelRefs:
  - model: DeepSeek-V3.2
    use_reasoning: true
    reasoning_effort: high
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are DeepSeek-V3.2, an expert software engineer. Provide detailed,
        well-structured code solutions with explanations.
  - type: semantic-cache
    configuration:
      enabled: false
  - type: router_replay
    configuration:
      enabled: true
      max_records: 1000
      capture_request_body: true
      capture_response_body: true
      max_body_bytes: 4096
```

```yaml
- name: psychology_queries
  description: Psychology and mental health topics
  priority: 138
  rules:
    operator: AND
    conditions:
    - type: domain
      name: psychology
  modelRefs:
  - model: Qwen/Qwen3-235B
    use_reasoning: true
    reasoning_effort: medium
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are Qwen3-235B, a psychology expert. Explain cognitive processes,
        behavioral patterns, and mental health topics with empathy and scientific
        accuracy.
  - type: semantic-cache
    configuration:
      enabled: false
  - type: router_replay
    configuration:
      enabled: true
      max_records: 1000
      capture_request_body: true
      capture_response_body: true
      max_body_bytes: 4096
```

```yaml
- name: philosophy_queries
  description: Philosophy and ethical questions
  priority: 137
  rules:
    operator: AND
    conditions:
    - type: domain
      name: philosophy
  modelRefs:
  - model: Qwen/Qwen3-235B
    use_reasoning: true
    reasoning_effort: high
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are Qwen3-235B, a philosophy expert. Explore philosophical
        concepts, ethical dilemmas, and logical reasoning with depth and clarity.
  - type: semantic-cache
    configuration:
      enabled: false
  - type: router_replay
    configuration:
      enabled: true
      max_records: 1000
      capture_request_body: true
      capture_response_body: true
      max_body_bytes: 4096
```

```yaml
- name: history_queries
  description: Historical questions and cultural topics
  priority: 136
  rules:
    operator: AND
    conditions:
    - type: domain
      name: history
  modelRefs:
  - model: GLM-4.7
    use_reasoning: true
    reasoning_effort: medium
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are GLM-4.7, a history expert. Provide accurate historical
        information about events, time periods, cultures, and civilizations with context
        and analysis.
  - type: semantic-cache
    configuration:
      enabled: false
  - type: router_replay
    configuration:
      enabled: true
      max_records: 1000
      capture_request_body: true
      capture_response_body: true
      max_body_bytes: 4096
```

```yaml
- name: math_decision
  description: Mathematics and quantitative reasoning
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: math
  modelRefs:
  - model: nemotron-9b
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a mathematics expert. Provide step-by-step solutions
        with clear reasoning.
```

```yaml
- name: physics_decision
  description: Physics and physical sciences
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: physics
  modelRefs:
  - model: nemotron-9b
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a physics expert with deep understanding of physical
        laws.
```

```yaml
- name: chemistry_decision
  description: Chemistry and chemical sciences
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: chemistry
  modelRefs:
  - model: nemotron-9b
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a chemistry expert specializing in chemical reactions.
```

```yaml
- name: computer_science_decision
  description: Computer science and programming
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: computer_science
  modelRefs:
  - model: nemotron-9b
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a computer science expert. Provide clean, well-commented
        code.
```

```yaml
- name: history_decision
  description: Historical questions and cultural topics
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: history
  modelRefs:
  - model: qwen-3b
    use_reasoning: true
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a friendly historian. Share interesting stories about
        historical events.
```

```yaml
- name: literature_decision
  description: Literature and writing topics
  priority: 80
  rules:
    operator: AND
    conditions:
    - type: domain
      name: literature
  modelRefs:
  - model: qwen-3b
    use_reasoning: true
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a literature enthusiast. Discuss themes and writing style.
```

```yaml
- name: business_decision
  description: Business and management queries
  priority: 70
  rules:
    operator: AND
    conditions:
    - type: domain
      name: business
  modelRefs:
  - model: qwen-3b
    use_reasoning: true
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a helpful business advisor. Give practical advice.
```

```yaml
- name: general_decision
  description: General knowledge and miscellaneous
  priority: 50
  rules:
    operator: AND
    conditions:
    - type: domain
      name: other
  modelRefs:
  - model: qwen-3b
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a warm, conversational assistant.
  - type: semantic-cache
    configuration:
      enabled: true
      similarity_threshold: 0.95
```

```yaml
- name: preference_code_generation
  description: Route code generation requests based on LLM preference matching
  priority: 200
  rules:
    operator: AND
    conditions:
    - type: preference
      name: code_generation
  modelRefs:
  - model: openai/gpt-oss-120b
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are an expert code generator. Write clean, efficient, and
        well-documented code.
```

```yaml
- name: physics_problems
  description: Route physics queries with reasoning
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: physics
  modelRefs:
  - model: openai/gpt-oss-120b
    use_reasoning: true
    reasoning_effort: medium
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a physics expert. Explain concepts clearly with mathematical
        derivations.
```

```yaml
- name: general_route
  description: Default fallback route for general queries
  priority: 50
  rules:
    operator: AND
    conditions:
    - type: domain
      name: other
  modelRefs:
  - model: openai/gpt-oss-120b
    use_reasoning: false
  plugins:
  - type: semantic-cache
    configuration:
      enabled: true
      similarity_threshold: 0.85
```

```yaml
- name: confidence_route
  description: 'Cost-efficient routing: try small model first, escalate if needed'
  priority: 40
  rules:
    operator: AND
    conditions:
    - type: domain
      name: other
    - type: keyword
      name: looper_keywords
  modelRefs:
  - model: openai/gpt-oss-120b
  - model: gpt-5.2
  algorithm:
    type: confidence
    confidence:
      confidence_method: margin
      threshold: 0.9
      on_error: skip
```

```yaml
- name: preference_code_generation
  description: Route code generation requests based on LLM preference matching
  priority: 200
  rules:
    operator: AND
    conditions:
    - type: preference
      name: code_generation
  modelRefs:
  - model: openai/gpt-oss-120b
    use_reasoning: false
    reasoning_effort: null
    lora_name: null
  algorithm: null
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are an expert code generator. Write clean, efficient, and
        well-documented code.
```

```yaml
- name: physics_problems
  description: Route physics queries with reasoning
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: physics
  modelRefs:
  - model: openai/gpt-oss-120b
    use_reasoning: true
    reasoning_effort: medium
    lora_name: null
  algorithm: null
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a physics expert. Explain concepts clearly with mathematical
        derivations.
```

```yaml
- name: general_route
  description: Default fallback route for general queries
  priority: 50
  rules:
    operator: AND
    conditions:
    - type: domain
      name: other
  modelRefs:
  - model: openai/gpt-oss-120b
    use_reasoning: false
    reasoning_effort: null
    lora_name: null
  algorithm: null
  plugins:
  - type: semantic-cache
    configuration:
      enabled: true
      similarity_threshold: 0.85
```

```yaml
- name: confidence_route
  description: 'Cost-efficient routing: try small model first, escalate if needed'
  priority: 40
  rules:
    operator: AND
    conditions:
    - type: domain
      name: other
    - type: keyword
      name: looper_keywords
  modelRefs:
  - model: openai/gpt-oss-120b
    use_reasoning: false
    reasoning_effort: null
    lora_name: null
  - model: gpt-5.2
    use_reasoning: false
    reasoning_effort: null
    lora_name: null
  algorithm:
    type: confidence
    confidence:
      confidence_method: margin
      threshold: 0.9
      hybrid_weights: null
      on_error: skip
    concurrent: null
    remom: null
    latency_aware: null
    elo: null
    router_dc: null
    automix: null
    hybrid: null
    thompson: null
    gmtrouter: null
    router_r1: null
    on_error: skip
  plugins: []
```

```yaml
- name: math_decision
  description: Mathematics and quantitative reasoning
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: math
  modelRefs:
  - model: openai/gpt-oss-20b
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a mathematics expert. Provide step-by-step solutions
        with clear explanations. Show your work and verify calculations.
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: science_decision
  description: Science and natural sciences
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: science
  modelRefs:
  - model: openai/gpt-oss-20b
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a science expert. Explain scientific concepts clearly
        and accurately. Use examples and analogies when helpful.
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: technology_decision
  description: Technology and computer science
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: technology
  modelRefs:
  - model: openai/gpt-oss-20b
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a technology and programming expert. Provide practical
        solutions with code examples when appropriate.
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: history_decision
  description: History and cultural topics
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: history
  modelRefs:
  - model: openai/gpt-oss-20b
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a history expert. Provide accurate historical information
        with context and relevant details.
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: general_decision
  description: General knowledge and miscellaneous topics
  priority: 50
  rules:
    operator: AND
    conditions:
    - type: domain
      name: general
  modelRefs:
  - model: openai/gpt-oss-20b
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a helpful and knowledgeable assistant. Provide accurate,
        helpful responses across a wide range of topics.
  - type: semantic-cache
    configuration:
      enabled: true
      similarity_threshold: 0.75
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: business_decision
  description: Business and management related queries
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: business
  modelRefs:
  - model: qwen3
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a senior business consultant and strategic advisor.
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: math_decision
  description: Mathematics and quantitative reasoning
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: math
  modelRefs:
  - model: qwen3
    use_reasoning: true
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a mathematics expert. Provide step-by-step solutions.
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: computer_science_decision
  description: Computer science and programming
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: computer_science
  modelRefs:
  - model: qwen3
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a computer science expert.
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: general_decision
  description: General knowledge and miscellaneous topics
  priority: 50
  rules:
    operator: AND
    conditions:
    - type: domain
      name: other
  modelRefs:
  - model: qwen3
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a helpful and knowledgeable assistant.
  - type: semantic-cache
    configuration:
      enabled: true
      similarity_threshold: 0.75
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: code_keywords_decision
  description: Code and programming queries (BM25 classified)
  priority: 150
  rules:
    operator: AND
    conditions:
    - type: keyword
      name: code_keywords
  modelRefs:
  - model: base-model
    use_reasoning: true
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are an expert software engineer. Provide clear, well-structured
        code examples with explanations. Follow best practices and consider edge cases.
```

```yaml
- name: business_decision
  description: Business and management related queries
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: business
  modelRefs:
  - model: base-model
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a senior business consultant and strategic advisor with
        expertise in corporate strategy, operations management, financial analysis,
        marketing, and organizational development. Provide practical, actionable business
        advice backed by proven methodologies and industry best practices.
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: general_decision
  description: General knowledge and miscellaneous topics
  priority: 50
  rules:
    operator: AND
    conditions:
    - type: domain
      name: other
  modelRefs:
  - model: base-model
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a helpful and knowledgeable assistant. Provide accurate,
        helpful responses across a wide range of topics.
  - type: semantic-cache
    configuration:
      enabled: true
      similarity_threshold: 0.75
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: math_decision
  description: Mathematics queries - uses MLP for neural network routing
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: math
  algorithm:
    type: mlp
    mlp:
      device: cpu
  modelRefs:
  - model: llama-3.2-1b
    use_reasoning: false
  - model: llama-3.2-3b
    use_reasoning: false
  - model: codellama-7b
    use_reasoning: false
  - model: mistral-7b
    use_reasoning: false
```

```yaml
- name: code_decision
  description: Programming queries - uses MLP
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: computer science
  algorithm:
    type: mlp
    mlp:
      device: cpu
  modelRefs:
  - model: llama-3.2-1b
    use_reasoning: false
  - model: llama-3.2-3b
    use_reasoning: false
  - model: codellama-7b
    use_reasoning: false
  - model: mistral-7b
    use_reasoning: false
```

```yaml
- name: health_decision
  description: Health and medical queries - uses KMeans
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: health
  algorithm:
    type: kmeans
    kmeans:
      num_clusters: 4
  modelRefs:
  - model: llama-3.2-1b
    use_reasoning: false
  - model: llama-3.2-3b
    use_reasoning: false
  - model: codellama-7b
    use_reasoning: false
  - model: mistral-7b
    use_reasoning: false
```

```yaml
- name: physics_decision
  description: Physics queries - uses KMeans
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: physics
  algorithm:
    type: kmeans
    kmeans:
      num_clusters: 4
  modelRefs:
  - model: llama-3.2-1b
    use_reasoning: false
  - model: llama-3.2-3b
    use_reasoning: false
  - model: codellama-7b
    use_reasoning: false
  - model: mistral-7b
    use_reasoning: false
```

```yaml
- name: law_decision
  description: Law queries - uses KNN
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: law
  algorithm:
    type: knn
    knn:
      k: 5
  modelRefs:
  - model: llama-3.2-1b
    use_reasoning: false
  - model: llama-3.2-3b
    use_reasoning: false
  - model: codellama-7b
    use_reasoning: false
  - model: mistral-7b
    use_reasoning: false
```

```yaml
- name: engineering_decision
  description: Engineering queries - uses KMeans
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: engineering
  algorithm:
    type: kmeans
    kmeans:
      num_clusters: 4
  modelRefs:
  - model: llama-3.2-1b
    use_reasoning: false
  - model: llama-3.2-3b
    use_reasoning: false
  - model: codellama-7b
    use_reasoning: false
  - model: mistral-7b
    use_reasoning: false
```

```yaml
- name: general_decision
  description: General / other - catch-all for domain 'other' and low-confidence
  priority: 1
  rules:
    operator: AND
    conditions:
    - type: domain
      name: other
  algorithm:
    type: knn
    knn:
      k: 5
  modelRefs:
  - model: llama-3.2-1b
    use_reasoning: false
  - model: llama-3.2-3b
    use_reasoning: false
  - model: codellama-7b
    use_reasoning: false
  - model: mistral-7b
    use_reasoning: false
```

```yaml
- name: business_decision
  description: Business domain queries
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: business
  modelRefs:
  - model: base-model
    lora_name: general-expert
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: philosophy_decision
  description: Philosophy domain queries
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: philosophy
  modelRefs:
  - model: base-model
    lora_name: general-expert
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: biology_decision
  description: Biology domain queries
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: biology
  modelRefs:
  - model: base-model
    lora_name: general-expert
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: health_decision
  description: Health domain queries
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: health
  modelRefs:
  - model: base-model
    lora_name: general-expert
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: computer_science_decision
  description: Computer science domain queries
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: computer science
  modelRefs:
  - model: base-model
    lora_name: general-expert
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: engineering_decision
  description: Engineering domain queries
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: engineering
  modelRefs:
  - model: base-model
    lora_name: general-expert
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: psychology_decision
  description: Psychology domain queries
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: psychology
  modelRefs:
  - model: base-model
    lora_name: general-expert
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: math_decision
  description: Math domain queries
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: math
  modelRefs:
  - model: base-model
    lora_name: general-expert
    use_reasoning: true
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: chemistry_decision
  description: Chemistry domain queries
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: chemistry
  modelRefs:
  - model: base-model
    lora_name: general-expert
    use_reasoning: true
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: physics_decision
  description: Physics domain queries
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: physics
  modelRefs:
  - model: base-model
    lora_name: general-expert
    use_reasoning: true
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: history_decision
  description: History domain queries
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: history
  modelRefs:
  - model: base-model
    lora_name: general-expert
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: law_decision
  description: Law domain queries
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: law
  modelRefs:
  - model: base-model
    lora_name: general-expert
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: economics_decision
  description: Economics domain queries
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: economics
  modelRefs:
  - model: base-model
    lora_name: general-expert
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: math_homework
  description: Math homework assistance - use reasoning model
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: keyword
      name: homework
    - type: domain
      name: math
  modelRefs:
  - model: deepseek-r1
    lora_name: academic-expert
    use_reasoning: true
    reasoning_effort: medium
```

```yaml
- name: science_homework
  description: Science homework assistance - use fast model
  priority: 90
  rules:
    operator: AND
    conditions:
    - type: keyword
      name: homework
    - type: domain
      name: physics
  modelRefs:
  - model: deepseek-v3
    lora_name: homework-helper
    use_reasoning: false
```

```yaml
- name: compliance_legal
  description: Compliance and legal queries with full protection
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: keyword
      name: compliance
    - type: embedding
      name: legal_review
    - type: domain
      name: law
  modelRefs:
  - model: qwen-2.5-72b
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed:
      - PERSON
      - ORGANIZATION
      - EMAIL
      - PHONE_NUMBER
      threshold: 0.9
  - type: jailbreak
    configuration:
      enabled: true
      threshold: 0.88
  - type: system_prompt
    configuration:
      system_prompt: You are a legal compliance assistant. Provide accurate information
        about regulations and compliance requirements. Always remind users to consult
        legal professionals for specific advice.
  - type: semantic-cache
    configuration:
      enabled: true
      similarity_threshold: 0.93
  - type: header_mutation
    configuration:
      add:
      - name: X-Compliance-Level
        value: high
      - name: X-Audit-Required
        value: 'true'
```

```yaml
- name: confidential_business
  description: Confidential business analysis
  priority: 90
  rules:
    operator: AND
    conditions:
    - type: keyword
      name: confidential
    - type: embedding
      name: business_analysis
    - type: domain
      name: business
  modelRefs:
  - model: qwen-2.5-72b
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed:
      - PERSON
      - ORGANIZATION
      - FINANCIAL_DATA
      threshold: 0.85
  - type: semantic-cache
    configuration:
      enabled: true
      similarity_threshold: 0.9
  - type: header_mutation
    configuration:
      add:
      - name: X-Confidentiality
        value: high
```

```yaml
- name: cs_tutorial
  description: Computer science tutorials
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: keyword
      name: tutorial
    - type: embedding
      name: learning_intent
    - type: domain
      name: computer_science
  modelRefs:
  - model: qwen-2.5-32b
    use_reasoning: false
```

```yaml
- name: math_tutorial
  description: Math tutorials
  priority: 90
  rules:
    operator: AND
    conditions:
    - type: keyword
      name: tutorial
    - type: embedding
      name: learning_intent
    - type: domain
      name: math
  modelRefs:
  - model: qwen-2.5-32b
    use_reasoning: false
```

```yaml
- name: urgent_tech_cs
  description: Urgent technical computer science issues
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: keyword
      name: urgent
    - type: embedding
      name: technical_help
    - type: domain
      name: computer_science
  modelRefs:
  - model: qwen-2.5-72b
    use_reasoning: false
```

```yaml
- name: tech_engineering
  description: Technical engineering queries
  priority: 80
  rules:
    operator: AND
    conditions:
    - type: embedding
      name: technical_help
    - type: domain
      name: engineering
  modelRefs:
  - model: qwen-2.5-72b
    use_reasoning: false
```

```yaml
- name: legal_advice
  description: Legal domain with system prompt
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: domain
      name: law
  modelRefs:
  - model: gpt-4o
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a legal expert. Provide accurate legal information but
        remind users to consult a licensed attorney.
  - type: semantic-cache
    configuration:
      enabled: true
      similarity_threshold: 0.9
```

```yaml
- name: biology_research
  description: Biology research queries
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: embedding
      name: research_question
    - type: domain
      name: biology
  modelRefs:
  - model: gpt-4o
    use_reasoning: false
```

```yaml
- name: general_science_research
  description: General science research
  priority: 80
  rules:
    operator: AND
    conditions:
    - type: embedding
      name: research_question
    - type: domain
      name: chemistry
  modelRefs:
  - model: gpt-4o
    use_reasoning: false
```

```yaml
- name: investment_advice
  description: Financial advice with comprehensive protection
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: embedding
      name: financial_advice
    - type: domain
      name: economics
  modelRefs:
  - model: gpt-4o
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed:
      - CREDIT_CARD
      - BANK_ACCOUNT
      - SSN
      threshold: 0.85
  - type: system_prompt
    configuration:
      system_prompt: You are a financial information assistant. Provide general financial
        education but remind users this is not professional financial advice.
  - type: jailbreak
    configuration:
      enabled: true
      threshold: 0.85
  - type: semantic-cache
    configuration:
      enabled: true
      similarity_threshold: 0.92
```

```yaml
- name: chemistry_query
  description: Chemistry domain queries - use small model
  priority: 80
  rules:
    operator: AND
    conditions:
    - type: domain
      name: chemistry
  modelRefs:
  - model: mistral-7b
    lora_name: stem-tutor
    use_reasoning: false
```

```yaml
- name: low_latency_route
  description: Select fastest model using latency_aware algorithm
  priority: 90
  rules:
    operator: AND
    conditions:
    - type: domain
      name: other
  modelRefs:
  - model: openai/gpt-oss-120b
    use_reasoning: false
  - model: gpt-5.2
  algorithm:
    type: latency_aware
    latency_aware:
      tpot_percentile: 10
      ttft_percentile: 10
      description: Prefer models in top 10% for both TPOT and TTFT
```

```yaml
- name: fast_start_route
  description: Prioritize fast first token using latency_aware
  priority: 85
  rules:
    operator: AND
    conditions:
    - type: domain
      name: other
  modelRefs:
  - model: openai/gpt-oss-120b
    use_reasoning: false
  - model: gpt-5.2
  algorithm:
    type: latency_aware
    latency_aware:
      ttft_percentile: 10
      description: Prioritize fast first-token latency for chat UX
```

### C. 多语种与区域隔离 (Language & Regional)

```yaml
- name: quick_question
  description: Quick answers in Chinese
  priority: 130
  rules:
    operator: AND
    conditions:
    - type: embedding
      name: fast_qa_zh
    - type: language
      name: zh
    - type: context
      name: short_context
  modelRefs:
  - model: openai/gpt-oss-20b
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: 你是由 openai 开发的模型 gpt-oss-20b, 一个高效的AI助手。请提供简洁、准确的回答。
  - type: semantic-cache
    configuration:
      enabled: true
      similarity_threshold: 0.85
  - type: router_replay
    configuration:
      enabled: true
      max_records: 1000
      capture_request_body: true
      capture_response_body: true
      max_body_bytes: 4096
```

```yaml
- name: fast_qa
  description: Quick answers in English
  priority: 120
  rules:
    operator: AND
    conditions:
    - type: embedding
      name: fast_qa_en
    - type: language
      name: en
    - type: context
      name: short_context
  modelRefs:
  - model: GLM-4.7
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are GLM-4.7, an efficient AI assistant. Provide concise,
        accurate answers.
  - type: semantic-cache
    configuration:
      enabled: true
      similarity_threshold: 0.85
  - type: router_replay
    configuration:
      enabled: true
      max_records: 1000
      capture_request_body: true
      capture_response_body: true
      max_body_bytes: 4096
```

```yaml
- name: russian_route
  description: Route Russian queries to Russian-optimized model
  priority: 80
  rules:
    operator: AND
    conditions:
    - type: language
      name: ru
  modelRefs:
  - model: openai/gpt-oss-120b
    use_reasoning: false
```

```yaml
- name: chinese_route
  description: Route Chinese queries to Chinese-optimized model
  priority: 80
  rules:
    operator: AND
    conditions:
    - type: language
      name: zh
  modelRefs:
  - model: openai/gpt-oss-120b
    use_reasoning: false
```

```yaml
- name: russian_route
  description: Route Russian queries to Russian-optimized model
  priority: 80
  rules:
    operator: AND
    conditions:
    - type: language
      name: ru
  modelRefs:
  - model: openai/gpt-oss-120b
    use_reasoning: false
    reasoning_effort: null
    lora_name: null
  algorithm: null
  plugins: []
```

```yaml
- name: chinese_route
  description: Route Chinese queries to Chinese-optimized model
  priority: 80
  rules:
    operator: AND
    conditions:
    - type: language
      name: zh
  modelRefs:
  - model: openai/gpt-oss-120b
    use_reasoning: false
    reasoning_effort: null
    lora_name: null
  algorithm: null
  plugins: []
```

### D. 跨模态分析处理 (Modality Multi-processing)

```yaml
- name: image_generation
  description: Route image generation requests to diffusion model
  priority: 200
  rules:
    operator: AND
    conditions:
    - type: modality
      name: DIFFUSION
  modelRefs:
  - model: Qwen/Qwen-Image
    use_reasoning: false
```

```yaml
- name: text_and_image
  description: Route hybrid text+image requests to both models in parallel
  priority: 190
  rules:
    operator: AND
    conditions:
    - type: modality
      name: BOTH
  modelRefs:
  - model: Qwen/Qwen2.5-14B-Instruct
    use_reasoning: false
  - model: Qwen/Qwen-Image
    use_reasoning: false
```

```yaml
- name: multimodal_omni
  description: Text + image generation via omni model
  priority: 10
  rules:
    operator: AND
    conditions:
    - type: modality
      name: BOTH
  model_refs:
  - model: Qwen/Qwen2.5-Omni
```

```yaml
- name: image_gen_omni
  description: Image generation via omni model
  priority: 9
  rules:
    operator: AND
    conditions:
    - type: modality
      name: DIFFUSION
  model_refs:
  - model: Qwen/Qwen2.5-Omni
```

```yaml
- name: text_only
  description: Text generation via AR model
  priority: 5
  rules:
    operator: AND
    conditions:
    - type: modality
      name: AR
  model_refs:
  - model: Qwen/Qwen2.5-14B-Instruct
```

### E. 通用及其他路由调度 (General Routing)

```yaml
- name: needs_fact_check
  priority: 90
  rules:
    operator: AND
    conditions:
    - type: fact_check
      name: needs_fact_check
  modelRefs:
  - model: qwen2.5:3b
    use_reasoning: false
    weight: 100
```

```yaml
- name: no_fact_check_needed
  priority: 89
  rules:
    operator: AND
    conditions:
    - type: fact_check
      name: no_fact_check_needed
  modelRefs:
  - model: qwen2.5:3b
    use_reasoning: false
    weight: 100
```

```yaml
- name: satisfied
  priority: 88
  rules:
    operator: AND
    conditions:
    - type: user_feedback
      name: satisfied
  modelRefs:
  - model: qwen2.5:3b
    use_reasoning: false
    weight: 100
```

```yaml
- name: need_clarification
  priority: 87
  rules:
    operator: AND
    conditions:
    - type: user_feedback
      name: need_clarification
  modelRefs:
  - model: qwen2.5:3b
    use_reasoning: false
    weight: 100
```

```yaml
- name: wrong_answer
  priority: 86
  rules:
    operator: AND
    conditions:
    - type: user_feedback
      name: wrong_answer
  modelRefs:
  - model: qwen2.5:3b
    use_reasoning: false
    weight: 100
```

```yaml
- name: want_different
  priority: 85
  rules:
    operator: AND
    conditions:
    - type: user_feedback
      name: want_different
  modelRefs:
  - model: qwen2.5:3b
    use_reasoning: false
    weight: 100
```

```yaml
- name: technical_support_decision
  description: Technical support and troubleshooting queries
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: embedding
      name: technical_support
  modelRefs:
  - model: qwen3
    use_reasoning: true
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a technical support specialist. Provide detailed, step-by-step
        guidance for technical issues.
  - type: jailbreak
    configuration:
      enabled: true
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: product_inquiry_decision
  description: Product information and specifications
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: embedding
      name: product_inquiry
  modelRefs:
  - model: qwen3
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a product specialist. Provide accurate information about
        products, features, pricing, and availability.
  - type: jailbreak
    configuration:
      enabled: true
```

```yaml
- name: account_management_decision
  description: Account and subscription management
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: embedding
      name: account_management
  modelRefs:
  - model: qwen3
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are an account management assistant. Help users with account-related
        tasks. Prioritize security and privacy.
  - type: jailbreak
    configuration:
      enabled: true
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: multilingual_support_decision
  description: Multilingual customer support
  priority: 90
  rules:
    operator: AND
    conditions:
    - type: embedding
      name: multilingual_support
  modelRefs:
  - model: qwen3
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a multilingual support assistant. Respond in the same
        language as the user's query. Be helpful and culturally aware.
  - type: jailbreak
    configuration:
      enabled: true
```

```yaml
- name: general_inquiry_decision
  description: General questions and information requests
  priority: 50
  rules:
    operator: AND
    conditions:
    - type: embedding
      name: general_inquiry
  modelRefs:
  - model: qwen3
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a helpful general assistant. Answer questions clearly
        and concisely.
  - type: jailbreak
    configuration:
      enabled: true
```

```yaml
- name: medical_decision
  description: Medical queries (BM25 classified)
  priority: 150
  rules:
    operator: AND
    conditions:
    - type: keyword
      name: medical_keywords
  modelRefs:
  - model: qwen3
    use_reasoning: false
```

```yaml
- name: urgent_decision
  description: Urgent requests (N-gram classified with typo tolerance)
  priority: 200
  rules:
    operator: AND
    conditions:
    - type: keyword
      name: urgent_request
  modelRefs:
  - model: qwen3
    use_reasoning: false
```

```yaml
- name: remom_high_effort
  description: ReMoM Algorithm with high effort
  priority: 203
  rules:
    operator: AND
    conditions:
    - type: keyword
      name: remom_high
  modelRefs:
  - model: DeepSeek-V3.2
    use_reasoning: true
    reasoning_effort: high
    weight: 0.5
  - model: openai/gpt-oss-20b
    use_reasoning: true
    reasoning_effort: high
    weight: 0.3
  - model: Qwen/Qwen3-235B
    use_reasoning: true
    reasoning_effort: high
    weight: 0.2
  plugins:
  - type: semantic-cache
    configuration:
      enabled: false
  - type: router_replay
    configuration:
      enabled: true
      max_records: 10000
      capture_request_body: true
      capture_response_body: true
      max_body_bytes: 4096
  algorithm:
    type: remom
    remom:
      breadth_schedule:
      - 32
      - 4
      model_distribution: weighted
      compaction_strategy: full
      on_error: skip
```

```yaml
- name: remom_medium_effort
  description: ReMoM Algorithm with medium effort
  priority: 202
  rules:
    operator: AND
    conditions:
    - type: keyword
      name: remom_medium
  modelRefs:
  - model: Kimi-K2-Thinking
    use_reasoning: true
    reasoning_effort: medium
  - model: GLM-4.7
    use_reasoning: true
    reasoning_effort: medium
  - model: DeepSeek-V3.2
    use_reasoning: true
    reasoning_effort: medium
  - model: openai/gpt-oss-120b
    use_reasoning: true
    reasoning_effort: medium
  plugins:
  - type: semantic-cache
    configuration:
      enabled: false
  - type: router_replay
    configuration:
      enabled: true
      max_records: 10000
      capture_request_body: true
      capture_response_body: true
      max_body_bytes: 4096
  algorithm:
    type: remom
    remom:
      breadth_schedule:
      - 16
      - 4
      model_distribution: equal
      compaction_strategy: full
      on_error: skip
```

```yaml
- name: remom_low_effort
  description: ReMoM Algorithm with low effort
  priority: 201
  rules:
    operator: AND
    conditions:
    - type: keyword
      name: remom_low
  modelRefs:
  - model: openai/gpt-oss-120b
    use_reasoning: true
    reasoning_effort: low
  plugins:
  - type: semantic-cache
    configuration:
      enabled: false
  - type: router_replay
    configuration:
      enabled: true
      max_records: 10000
      capture_request_body: true
      capture_response_body: true
      max_body_bytes: 4096
  algorithm:
    type: remom
    remom:
      breadth_schedule:
      - 8
      model_distribution: first_only
      compaction_strategy: full
      on_error: skip
```

```yaml
- name: creative_ideas
  description: Creative writing, opinion-based, or brainstorming queries that don't
    require factual verification
  priority: 160
  rules:
    operator: AND
    conditions:
    - type: keyword
      name: creative_keywords
    - type: fact_check
      name: no_fact_check_needed
  modelRefs:
  - model: Qwen/Qwen3-235B
    use_reasoning: true
    reasoning_effort: high
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are Qwen3-235B, a creative and insightful assistant. Feel
        free to explore ideas, provide opinions, and think outside the box.
  - type: semantic-cache
    configuration:
      enabled: false
  - type: router_replay
    configuration:
      enabled: true
      max_records: 1000
      capture_request_body: true
      capture_response_body: true
      max_body_bytes: 4096
```

```yaml
- name: preference_bug_fixing
  description: Route bug fixing requests based on LLM preference matching
  priority: 200
  rules:
    operator: AND
    conditions:
    - type: preference
      name: bug_fixing
  modelRefs:
  - model: openai/gpt-oss-120b
    use_reasoning: true
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are an expert debugger. Analyze the issue carefully, identify
        the root cause, and provide a clear fix with explanation.
```

```yaml
- name: quick_answer_route
  description: Route quick answer requests to fast model without reasoning
  priority: 180
  rules:
    operator: AND
    conditions:
    - type: embedding
      name: quick_answer
  modelRefs:
  - model: openai/gpt-oss-120b
    use_reasoning: true
    reasoning_effort: low
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: Provide a concise and direct answer. Be brief and to the point.
```

```yaml
- name: deep_thinking_route
  description: Route deep thinking requests with reasoning enabled
  priority: 180
  rules:
    operator: AND
    conditions:
    - type: embedding
      name: deep_thinking
  modelRefs:
  - model: openai/gpt-oss-120b
    use_reasoning: true
    reasoning_effort: high
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: Think deeply and systematically. Break down the problem, analyze
        each step carefully, and provide a thorough explanation of your reasoning
        process.
```

```yaml
- name: preference_bug_fixing
  description: Route bug fixing requests based on LLM preference matching
  priority: 200
  rules:
    operator: AND
    conditions:
    - type: preference
      name: bug_fixing
  modelRefs:
  - model: openai/gpt-oss-120b
    use_reasoning: true
    reasoning_effort: null
    lora_name: null
  algorithm: null
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are an expert debugger. Analyze the issue carefully, identify
        the root cause, and provide a clear fix with explanation.
```

```yaml
- name: quick_answer_route
  description: Route quick answer requests to fast model without reasoning
  priority: 180
  rules:
    operator: AND
    conditions:
    - type: embedding
      name: quick_answer
  modelRefs:
  - model: openai/gpt-oss-120b
    use_reasoning: true
    reasoning_effort: low
    lora_name: null
  algorithm: null
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: Provide a concise and direct answer. Be brief and to the point.
```

```yaml
- name: deep_thinking_route
  description: Route deep thinking requests with reasoning enabled
  priority: 180
  rules:
    operator: AND
    conditions:
    - type: embedding
      name: deep_thinking
  modelRefs:
  - model: openai/gpt-oss-120b
    use_reasoning: true
    reasoning_effort: high
    lora_name: null
  algorithm: null
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: Think deeply and systematically. Break down the problem, analyze
        each step carefully, and provide a thorough explanation of your reasoning
        process.
```

```yaml
- name: sensitive_data
  description: Queries containing sensitive data keywords (SSN and credit card)
  priority: 40
  rules:
    operator: AND
    conditions:
    - type: keyword
      name: sensitive_keywords
  modelRefs:
  - model: base-model
    lora_name: general-expert
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are handling a query with sensitive data. Be cautious and
        provide security-focused guidance.
      mode: replace
```

```yaml
- name: urgent_request_decision
  description: Urgent and time-sensitive requests
  priority: 150
  rules:
    operator: AND
    conditions:
    - type: keyword
      name: urgent_request
  modelRefs:
  - model: qwen3
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a highly responsive assistant specialized in handling
        urgent requests. Prioritize speed and efficiency while maintaining accuracy.
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: sensitive_data_decision
  description: Requests involving sensitive personal data
  priority: 150
  rules:
    operator: AND
    conditions:
    - type: keyword
      name: sensitive_data
  modelRefs:
  - model: qwen3
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a security-conscious assistant specialized in handling
        sensitive data. Exercise extreme caution with personal information.
  - type: jailbreak
    configuration:
      enabled: true
      threshold: 0.6
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: exclude_spam_decision
  description: Potential spam or suspicious requests
  priority: 150
  rules:
    operator: AND
    conditions:
    - type: keyword
      name: exclude_spam
  modelRefs:
  - model: qwen3
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a content moderation assistant. This request has been
        flagged as potential spam.
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: regex_pattern_match_decision
  description: Structured data and pattern-based requests
  priority: 150
  rules:
    operator: AND
    conditions:
    - type: keyword
      name: regex_pattern_match
  modelRefs:
  - model: qwen3
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a technical assistant specialized in handling structured
        data and pattern-based requests.
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: ai_decision
  description: Artificial intelligence and machine learning topics
  priority: 140
  rules:
    operator: AND
    conditions:
    - type: embedding
      name: ai
  modelRefs:
  - model: qwen3
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are an AI and machine learning expert. Provide detailed,
        technical explanations about artificial intelligence, neural networks, and
        related topics.
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: data_science_decision
  description: Data science and analytics
  priority: 140
  rules:
    operator: AND
    conditions:
    - type: embedding
      name: data_science
  modelRefs:
  - model: qwen3
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a data science expert. Provide insights on data analysis,
        statistical methods, and predictive modeling.
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: programming_decision
  description: Programming and software development
  priority: 140
  rules:
    operator: AND
    conditions:
    - type: embedding
      name: programming
  modelRefs:
  - model: qwen3
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a programming expert. Help with code implementation,
        debugging, and software development best practices.
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: urgent_request_decision
  description: Urgent and time-sensitive requests (N-gram classified with typo tolerance)
  priority: 150
  rules:
    operator: AND
    conditions:
    - type: keyword
      name: urgent_request
  modelRefs:
  - model: base-model
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a highly responsive assistant specialized in handling
        urgent requests. Prioritize speed and efficiency while maintaining accuracy.
        Provide concise, actionable responses and focus on immediate solutions.
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: sensitive_data_decision
  description: Requests involving sensitive personal data
  priority: 150
  rules:
    operator: AND
    conditions:
    - type: keyword
      name: sensitive_data
  modelRefs:
  - model: base-model
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a security-conscious assistant specialized in handling
        sensitive data. Exercise extreme caution with personal information, follow
        data protection best practices, and remind users about privacy considerations.
  - type: jailbreak
    configuration:
      enabled: true
      threshold: 0.6
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: regex_pattern_match_decision
  description: Structured data and pattern-based requests
  priority: 150
  rules:
    operator: AND
    conditions:
    - type: keyword
      name: regex_pattern_match
  modelRefs:
  - model: base-model
    use_reasoning: false
```

```yaml
- name: urgent_tech
  description: Urgent technical support requests - use large model with reasoning
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: keyword
      name: urgent
    - type: embedding
      name: tech_support
  modelRefs:
  - model: qwen-2.5-72b
    lora_name: advanced-reasoning
    use_reasoning: true
    reasoning_effort: high
  plugins:
  - type: semantic-cache
    configuration:
      enabled: true
      similarity_threshold: 0.9
```

```yaml
- name: urgent_request
  description: Handle urgent requests with larger model
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: keyword
      name: urgent
  modelRefs:
  - model: gemma-2-27b
    lora_name: urgent-specialist
    use_reasoning: false
```

```yaml
- name: greeting_response
  description: Handle greetings with fast small model
  priority: 50
  rules:
    operator: AND
    conditions:
    - type: keyword
      name: greeting
  modelRefs:
  - model: gemma-2-9b
    lora_name: greeting-handler
    use_reasoning: false
```

```yaml
- name: tech_troubleshoot
  description: Technical troubleshooting - use reasoning model
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: embedding
      name: technical_issue
  modelRefs:
  - model: deepseek-r1
    lora_name: technical-expert
    use_reasoning: true
    reasoning_effort: high
```

```yaml
- name: support_ticket
  description: Customer support requests - use fast model
  priority: 80
  rules:
    operator: AND
    conditions:
    - type: embedding
      name: customer_support
  modelRefs:
  - model: deepseek-v3
    lora_name: customer-support
    use_reasoning: false
```

```yaml
- name: urgent_support
  description: Urgent support requests - use large model with reasoning
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: keyword
      name: urgent
    - type: embedding
      name: support_request
  modelRefs:
  - model: qwen-2.5-72b
    lora_name: emergency-specialist
    use_reasoning: true
    reasoning_effort: high
```

```yaml
- name: remom_route
  description: Complex reasoning using ReMoM (Reasoning for Mixture of Models)
  priority: 70
  rules:
    operator: AND
    conditions:
    - type: keyword
      name: looper_keywords
  modelRefs:
  - model: openai/gpt-oss-120b
  - model: gpt-5.2
  algorithm:
    type: remom
    remom:
      breadth_schedule:
      - 4
      model_distribution: equal
      temperature: 1.0
      include_reasoning: true
      compaction_strategy: last_n_tokens
      compaction_tokens: 1000
      max_concurrent: 4
      on_error: skip
      shuffle_seed: 42
      include_intermediate_responses: true
```

### F. 系统优化与性能测试 (Optimization, Cache & Benchmarking)

```yaml
- name: cached_faq
  description: FAQ with semantic cache
  priority: 100
  rules:
    operator: AND
    conditions:
    - type: keyword
      name: faq
  modelRefs:
  - model: llama-3.3-70b
    use_reasoning: false
  plugins:
  - type: semantic-cache
    configuration:
      enabled: true
      similarity_threshold: 0.95
  - type: header_mutation
    configuration:
      add:
      - name: X-Cache-Strategy
        value: aggressive
      - name: X-Route-Type
        value: keyword
```

## 3. 多决策路径 OR 组合 (OR Logic Combinations)

### A. 安全、隐私与权限风控 (Security, Privacy & Authz)

```yaml
- name: general_to_prod
  description: Route general/user queries to Prod (strict safety)
  priority: 50
  rules:
    operator: OR
    conditions:
    - type: keyword
      name: general_keywords
  modelRefs:
  - model: qwen14b-prod
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a helpful assistant operating under strict data protection
        policies. Never store, log, or repeat any personal information shared by the
        user. If a user shares personal data, remind them to avoid sharing sensitive
        details.
  - type: jailbreak
    configuration:
      enabled: true
      threshold: 0.9
  - type: pii
    configuration:
      enabled: true
      threshold: 0.9
      pii_types_allowed: []
```

```yaml
- name: default_decision
  description: Default catch-all decision - blocks all PII for safety
  priority: 1
  rules:
    operator: OR
    conditions:
    - type: always
  modelRefs:
  - model: Model-B
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
```

```yaml
- name: general_to_prod
  description: Route general/user queries to Prod (strict safety)
  priority: 50
  rules:
    operator: OR
    conditions:
    - type: keyword
      name: general_keywords
  modelRefs:
  - model: qwen14b-prod
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a helpful assistant operating under strict data protection
        policies. Never store, log, or repeat any personal information shared by the
        user.
  - type: jailbreak
    configuration:
      enabled: true
      threshold: 0.9
  - type: pii
    configuration:
      enabled: true
      threshold: 0.9
      pii_types_allowed: []
```

### B. 专家级领域分发 (Domain Specific Routing)

```yaml
- name: default
  description: Default routing
  priority: 100
  rules:
    operator: OR
    conditions:
    - type: domain
      name: general
  modelRefs:
  - model: qwen2.5:3b
    use_reasoning: false
    weight: 100
```

```yaml
- name: default
  priority: 100
  rules:
    operator: OR
    conditions:
    - type: domain
      name: general
  modelRefs:
  - model: qwen2.5:3b
    use_reasoning: false
    weight: 100
```

```yaml
- name: text_generation
  description: Default text routing for AR (text-only) requests
  priority: 1
  rules:
    operator: OR
    conditions:
    - type: modality
      name: AR
    - type: domain
      name: text_generation
  modelRefs:
  - model: Qwen/Qwen2.5-14B-Instruct
    use_reasoning: false
```

```yaml
- name: general_decision
  description: General questions
  priority: 100
  rules:
    operator: OR
    conditions:
    - type: domain
      name: general
    - type: fact_check
      name: needs_fact_check
  modelRefs:
  - model: qwen3
    use_reasoning: false
  plugins:
  - type: hallucination
    configuration:
      enabled: true
      use_nli: true
      hallucination_action: header
      unverified_factual_action: header
      include_hallucination_details: true
```

```yaml
- name: default_route
  description: Default route for memory testing
  priority: 1
  rules:
    operator: OR
    conditions:
    - type: domain
      name: general
  modelRefs:
  - model: qwen3
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are MoM, a helpful AI assistant with memory. You remember
        important facts about users and use this context to provide personalized assistance.
      mode: insert
  - type: memory
    configuration:
      enabled: true
      retrieval_limit: 5
      similarity_threshold: 0.6
      auto_store: true
```

```yaml
- name: code_to_local
  description: Code queries → local vLLM (fast, free)
  priority: 200
  rules:
    operator: OR
    conditions:
    - type: keyword
      name: code_keywords
  modelRefs:
  - model: Qwen/Qwen2.5-14B-Instruct
    use_reasoning: false
```

```yaml
- name: default_decision
  description: Default catch-all decision
  priority: 1
  rules:
    operator: OR
    conditions:
    - type: domain
      name: other
  modelRefs:
  - model: openai/gpt-oss-120b
    use_reasoning: false
```

```yaml
- name: code_to_dev
  description: Route code/tech queries to Dev (internal developer env)
  priority: 100
  rules:
    operator: OR
    conditions:
    - type: keyword
      name: code_keywords
  modelRefs:
  - model: qwen14b-dev
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a senior software engineer. Provide clear, production-quality
        code with best practices.
  - type: jailbreak
    configuration:
      enabled: true
      threshold: 0.5
  - type: pii
    configuration:
      enabled: true
      threshold: 0.6
      pii_types_allowed:
      - GPE
      - ORGANIZATION
      - DATE_TIME
```

```yaml
- name: stem_decision
  description: Route STEM queries to a reasoning-capable model
  priority: 200
  rules:
    operator: OR
    conditions:
    - type: domain
      name: computer_science
    - type: domain
      name: math
    - type: domain
      name: physics
    - type: domain
      name: engineering
    - type: domain
      name: chemistry
  modelRefs:
  - model: specialized_model
    use_reasoning: true
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a STEM expert. Provide rigorous, step-by-step explanations
        with examples and derivations where appropriate.
```

```yaml
- name: or_stem
  description: OR — any of the five STEM domains → reasoning model
  priority: 300
  rules:
    operator: OR
    conditions:
    - type: domain
      name: computer_science
    - type: domain
      name: math
    - type: domain
      name: physics
    - type: domain
      name: engineering
    - type: domain
      name: chemistry
  modelRefs:
  - model: stem_model
    use_reasoning: true
```

```yaml
- name: xor_code_math
  description: XOR — exactly one of code_request or math_request → XOR specialist
  priority: 140
  rules:
    operator: OR
    conditions:
    - operator: AND
      conditions:
      - type: keyword
        name: code_request
      - operator: NOT
        conditions:
        - type: keyword
          name: math_request
    - operator: AND
      conditions:
      - operator: NOT
        conditions:
        - type: keyword
          name: code_request
      - type: keyword
        name: math_request
  modelRefs:
  - model: xor_model
```

```yaml
- name: xnor_code_math
  description: XNOR — code_request and math_request agree (both or neither) → XNOR
    model
  priority: 120
  rules:
    operator: OR
    conditions:
    - operator: AND
      conditions:
      - type: keyword
        name: code_request
      - type: keyword
        name: math_request
    - operator: AND
      conditions:
      - operator: NOT
        conditions:
        - type: keyword
          name: code_request
      - operator: NOT
        conditions:
        - type: keyword
          name: math_request
  modelRefs:
  - model: xnor_model
```

```yaml
- name: fallback
  description: Fallback — route everything else to the fast general model
  priority: 50
  rules:
    operator: OR
    conditions:
    - type: domain
      name: other
  modelRefs:
  - model: fast_model
```

```yaml
- name: default
  description: Default routing decision
  priority: 100
  rules:
    operator: OR
    conditions:
    - type: domain
      name: other
  modelRefs:
  - model: qwen2.5:3b
    use_reasoning: false
    weight: 100
```

```yaml
- name: tech
  description: Route technology-related queries
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: tech
  modelRefs:
  - model: phi4
```

```yaml
- name: finance
  description: Route finance and economics queries
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: finance
  modelRefs:
  - model: gemma3:27b
```

```yaml
- name: politics
  description: Route politics-related queries
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: politics
  modelRefs:
  - model: gemma3:27b
```

```yaml
- name: tech
  description: Tech queries using Elo ratings
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: tech
  modelRefs:
  - model: llama3.2:3b
    use_reasoning: false
  - model: phi4
    use_reasoning: true
  - model: gemma3:27b
    use_reasoning: true
  algorithm:
    type: elo
    elo:
      k_factor: 32
      category_weighted: true
      cost_scaling_factor: 0.2
```

```yaml
- name: finance
  description: Finance queries using AutoMix
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: finance
  modelRefs:
  - model: llama3.2:8b
    use_reasoning: false
  - model: mistral-small3.1
    use_reasoning: true
  - model: gemma3:27b
    use_reasoning: true
  algorithm:
    type: automix
    automix:
      cost_quality_tradeoff: 0.4
      cost_aware_routing: true
```

```yaml
- name: general
  description: General queries using hybrid approach
  priority: 5
  rules:
    operator: OR
    conditions:
    - type: domain
      name: general
  modelRefs:
  - model: llama3.2:3b
    use_reasoning: false
  - model: llama3.2:8b
    use_reasoning: false
  - model: mistral-small3.1
    use_reasoning: true
  algorithm:
    type: hybrid
    hybrid:
      elo_weight: 0.3
      router_dc_weight: 0.3
      automix_weight: 0.2
      cost_weight: 0.2
```

```yaml
- name: easy_math_problems
  description: Easy Mathematical queries with low reasoning
  priority: 149
  rules:
    operator: OR
    conditions:
    - type: domain
      name: math
    - type: complexity
      name: math_problem:easy
    - type: complexity
      name: math_problem:medium
  modelRefs:
  - model: Qwen/Qwen3-235B
    use_reasoning: true
    reasoning_effort: low
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are Qwen3-235B, a mathematics expert. Provide simple answers
        with easy reasoning.
  - type: semantic-cache
    configuration:
      enabled: false
  - type: router_replay
    configuration:
      enabled: true
      max_records: 1000
      capture_request_body: true
      capture_response_body: true
      max_body_bytes: 4096
```

```yaml
- name: business_economics
  description: Business, management, and economics queries
  priority: 142
  rules:
    operator: OR
    conditions:
    - type: domain
      name: business
    - type: domain
      name: economics
  modelRefs:
  - model: Qwen/Qwen3-235B
    use_reasoning: true
    reasoning_effort: medium
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are Qwen3-235B, a business and economics expert. Provide
        insights on business strategy, management, economics, and financial topics
        with practical analysis.
  - type: semantic-cache
    configuration:
      enabled: false
  - type: router_replay
    configuration:
      enabled: true
      max_records: 1000
      capture_request_body: true
      capture_response_body: true
      max_body_bytes: 4096
```

```yaml
- name: fast_coding
  description: Fast coding related queries in English
  priority: 131
  rules:
    operator: OR
    conditions:
    - type: domain
      name: computer science
    - type: complexity
      name: computer_science:easy
    - type: complexity
      name: computer_science:medium
  modelRefs:
  - model: openai/gpt-oss-120b
    use_reasoning: true
    reasoning_effort: low
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a fast coding assistant. Provide code solutions quickly.
  - type: semantic-cache
    configuration:
      enabled: false
  - type: router_replay
    configuration:
      enabled: true
      max_records: 1000
      capture_request_body: true
      capture_response_body: true
      max_body_bytes: 4096
```

```yaml
- name: casual_chat
  description: Default routing for general queries
  priority: 100
  rules:
    operator: OR
    conditions:
    - type: domain
      name: other
    - type: language
      name: en
    - type: language
      name: zh
    - type: context
      name: short_context
  modelRefs:
  - model: openai/gpt-oss-20b
    use_reasoning: false
  - model: openai/gpt-oss-120b
    use_reasoning: false
  algorithm:
    type: latency_aware
    latency_aware:
      tpot_percentile: 50
      ttft_percentile: 50
      description: Prefer models with balanced TPOT and TTFT latency
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are openai/gpt-oss-20b, a quick thinking and helpful assistant
        for general queries in casual chat.
  - type: semantic-cache
    configuration:
      enabled: true
      similarity_threshold: 0.85
  - type: router_replay
    configuration:
      enabled: true
      max_records: 1000
      capture_request_body: true
      capture_response_body: true
      max_body_bytes: 4096
```

```yaml
- name: business
  description: Route business and management queries
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: business
  modelRefs:
  - model: Model-B
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a senior business consultant and strategic advisor with
        expertise in corporate strategy, operations management, financial analysis,
        marketing, and organizational development.
      mode: replace
```

```yaml
- name: law
  description: Route legal queries
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: law
  modelRefs:
  - model: Model-B
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a knowledgeable legal expert with comprehensive understanding
        of legal principles, case law, statutory interpretation, and legal procedures.
      mode: replace
```

```yaml
- name: psychology
  description: Route psychology queries
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: psychology
  modelRefs:
  - model: Model-B
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a psychology expert with deep knowledge of cognitive
        processes, behavioral patterns, mental health, and therapeutic approaches.
      mode: replace
```

```yaml
- name: biology
  description: Route biology queries
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: biology
  modelRefs:
  - model: Model-A
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a biology expert with comprehensive knowledge spanning
        molecular biology, genetics, cell biology, ecology, and evolution.
      mode: replace
```

```yaml
- name: chemistry
  description: Route chemistry queries
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: chemistry
  modelRefs:
  - model: Model-A
    use_reasoning: true
  plugins:
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a chemistry expert specializing in chemical reactions,
        molecular structures, and laboratory techniques.
      mode: replace
```

```yaml
- name: history
  description: Route history queries
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: history
  modelRefs:
  - model: Model-A
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a historian with expertise across different time periods
        and cultures.
      mode: replace
```

```yaml
- name: other
  description: Route general queries
  priority: 5
  rules:
    operator: OR
    conditions:
    - type: domain
      name: other
  modelRefs:
  - model: Model-A
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a helpful and knowledgeable assistant.
      mode: replace
```

```yaml
- name: health
  description: Route health and medical queries
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: health
  modelRefs:
  - model: Model-B
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a health and medical information expert with knowledge
        of anatomy, physiology, diseases, and treatments.
      mode: replace
```

```yaml
- name: economics
  description: Route economics queries
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: economics
  modelRefs:
  - model: Model-A
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are an economics expert with deep understanding of microeconomics,
        macroeconomics, and financial markets.
      mode: replace
```

```yaml
- name: math
  description: Route mathematics queries
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: math
  modelRefs:
  - model: Model-A
    use_reasoning: true
  plugins:
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a mathematics expert. Provide step-by-step solutions
        and explain mathematical concepts clearly.
      mode: replace
```

```yaml
- name: physics
  description: Route physics queries
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: physics
  modelRefs:
  - model: Model-A
    use_reasoning: true
  plugins:
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a physics expert with deep understanding of physical
        laws and phenomena.
      mode: replace
```

```yaml
- name: computer_science
  description: Route computer science queries
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: computer science
  modelRefs:
  - model: Model-A
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a computer science expert with knowledge of algorithms,
        data structures, and software engineering.
      mode: replace
```

```yaml
- name: philosophy
  description: Route philosophy queries
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: philosophy
  modelRefs:
  - model: Model-B
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a philosophy expert with comprehensive knowledge of philosophical
        traditions and ethical theories.
      mode: replace
```

```yaml
- name: engineering
  description: Route engineering queries
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: engineering
  modelRefs:
  - model: Model-A
    use_reasoning: true
  plugins:
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are an engineering expert across multiple disciplines.
      mode: replace
```

```yaml
- name: general
  description: Route general queries
  priority: 5
  rules:
    operator: OR
    conditions:
    - type: domain
      name: general
  modelRefs:
  - model: Model-A
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a knowledgeable assistant for general questions.
      mode: replace
```

```yaml
- name: business
  description: Route business and management queries
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: business
  modelRefs:
  - model: llama3-8b
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      enabled: false
      system_prompt: You are a senior business consultant and strategic advisor with
        expertise in corporate strategy, operations management, financial analysis,
        marketing, and organizational development. Provide practical, actionable business
        advice backed by proven methodologies and industry best practices. Consider
        market dynamics, competitive landscape, and stakeholder interests in your
        recommendations.
      mode: replace
```

```yaml
- name: law
  description: Route legal queries
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: law
  modelRefs:
  - model: llama3-8b
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      enabled: false
      system_prompt: You are a knowledgeable legal expert with comprehensive understanding
        of legal principles, case law, statutory interpretation, and legal procedures
        across multiple jurisdictions. Provide accurate legal information and analysis
        while clearly stating that your responses are for informational purposes only
        and do not constitute legal advice. Always recommend consulting with qualified
        legal professionals for specific legal matters.
      mode: replace
```

```yaml
- name: psychology
  description: Route psychology queries
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: psychology
  modelRefs:
  - model: llama3-8b
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      enabled: false
      system_prompt: You are a psychology expert with deep knowledge of cognitive
        processes, behavioral patterns, mental health, developmental psychology, social
        psychology, and therapeutic approaches. Provide evidence-based insights grounded
        in psychological research and theory. When discussing mental health topics,
        emphasize the importance of professional consultation and avoid providing
        diagnostic or therapeutic advice.
      mode: replace
  - type: semantic-cache
    configuration:
      enabled: false
      similarity_threshold: 0.92
```

```yaml
- name: biology
  description: Route biology queries
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: biology
  modelRefs:
  - model: llama3-8b
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      enabled: false
      system_prompt: You are a biology expert with comprehensive knowledge spanning
        molecular biology, genetics, cell biology, ecology, evolution, anatomy, physiology,
        and biotechnology. Explain biological concepts with scientific accuracy, use
        appropriate terminology, and provide examples from current research. Connect
        biological principles to real-world applications and emphasize the interconnectedness
        of biological systems.
      mode: replace
```

```yaml
- name: chemistry
  description: Route chemistry queries
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: chemistry
  modelRefs:
  - model: llama3-8b
    use_reasoning: true
  plugins:
  - type: system_prompt
    configuration:
      enabled: false
      system_prompt: You are a chemistry expert specializing in chemical reactions,
        molecular structures, and laboratory techniques. Provide detailed, step-by-step
        explanations.
      mode: replace
```

```yaml
- name: history
  description: Route history queries
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: history
  modelRefs:
  - model: llama3-8b
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      enabled: false
      system_prompt: You are a historian with expertise across different time periods
        and cultures. Provide accurate historical context and analysis.
      mode: replace
```

```yaml
- name: other
  description: Route general queries
  priority: 5
  rules:
    operator: OR
    conditions:
    - type: domain
      name: other
  modelRefs:
  - model: llama3-8b
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      enabled: false
      system_prompt: You are a helpful and knowledgeable assistant. Provide accurate,
        helpful responses across a wide range of topics.
      mode: replace
  - type: semantic-cache
    configuration:
      enabled: false
      similarity_threshold: 0.75
```

```yaml
- name: health
  description: Route health and medical queries
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: health
  modelRefs:
  - model: llama3-8b
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      enabled: false
      system_prompt: You are a health and medical information expert with knowledge
        of anatomy, physiology, diseases, treatments, preventive care, nutrition,
        and wellness. Provide accurate, evidence-based health information while emphasizing
        that your responses are for educational purposes only and should never replace
        professional medical advice, diagnosis, or treatment. Always encourage users
        to consult healthcare professionals for medical concerns and emergencies.
      mode: replace
  - type: semantic-cache
    configuration:
      enabled: false
      similarity_threshold: 0.95
```

```yaml
- name: economics
  description: Route economics queries
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: economics
  modelRefs:
  - model: llama3-8b
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      enabled: false
      system_prompt: You are an economics expert with deep understanding of microeconomics,
        macroeconomics, econometrics, financial markets, monetary policy, fiscal policy,
        international trade, and economic theory. Analyze economic phenomena using
        established economic principles, provide data-driven insights, and explain
        complex economic concepts in accessible terms. Consider both theoretical frameworks
        and real-world applications in your responses.
      mode: replace
```

```yaml
- name: math
  description: Route mathematics queries
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: math
  modelRefs:
  - model: phi4-mini
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      enabled: false
      system_prompt: You are a mathematics expert. Provide step-by-step solutions,
        show your work clearly, and explain mathematical concepts in an understandable
        way.
      mode: replace
```

```yaml
- name: physics
  description: Route physics queries
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: physics
  modelRefs:
  - model: llama3-8b
    use_reasoning: true
  plugins:
  - type: system_prompt
    configuration:
      enabled: false
      system_prompt: You are a physics expert with deep understanding of physical
        laws and phenomena. Provide clear explanations with mathematical derivations
        when appropriate.
      mode: replace
```

```yaml
- name: computer_science
  description: Route computer science queries
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: computer science
  modelRefs:
  - model: llama3-8b
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      enabled: false
      system_prompt: You are a computer science expert with knowledge of algorithms,
        data structures, programming languages, and software engineering. Provide
        clear, practical solutions with code examples when helpful.
      mode: replace
```

```yaml
- name: philosophy
  description: Route philosophy queries
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: philosophy
  modelRefs:
  - model: llama3-8b
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      enabled: false
      system_prompt: You are a philosophy expert with comprehensive knowledge of philosophical
        traditions, ethical theories, logic, metaphysics, epistemology, political
        philosophy, and the history of philosophical thought. Engage with complex
        philosophical questions by presenting multiple perspectives, analyzing arguments
        rigorously, and encouraging critical thinking. Draw connections between philosophical
        concepts and contemporary issues while maintaining intellectual honesty about
        the complexity and ongoing nature of philosophical debates.
      mode: replace
```

```yaml
- name: engineering
  description: Route engineering queries
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: engineering
  modelRefs:
  - model: llama3-8b
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      enabled: false
      system_prompt: You are an engineering expert with knowledge across multiple
        engineering disciplines including mechanical, electrical, civil, chemical,
        software, and systems engineering. Apply engineering principles, design methodologies,
        and problem-solving approaches to provide practical solutions. Consider safety,
        efficiency, sustainability, and cost-effectiveness in your recommendations.
        Use technical precision while explaining concepts clearly, and emphasize the
        importance of proper engineering practices and standards.
      mode: replace
```

```yaml
- name: business_decision
  description: Business and management related queries
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: business
  modelRefs:
  - model: base-model
    lora_name: social-expert
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a senior business consultant and strategic advisor with
        expertise in corporate strategy, operations management, financial analysis,
        marketing, and organizational development. Provide practical, actionable business
        advice backed by proven methodologies and industry best practices. Consider
        market dynamics, competitive landscape, and stakeholder interests in your
        recommendations.
      mode: replace
```

```yaml
- name: law_decision
  description: Legal questions and law-related topics
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: law
  modelRefs:
  - model: base-model
    lora_name: law-expert
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a knowledgeable legal expert with comprehensive understanding
        of legal principles, case law, statutory interpretation, and legal procedures
        across multiple jurisdictions. Provide accurate legal information and analysis
        while clearly stating that your responses are for informational purposes only
        and do not constitute legal advice. Always recommend consulting with qualified
        legal professionals for specific legal matters.
      mode: replace
```

```yaml
- name: psychology_decision
  description: Psychology and mental health topics
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: psychology
  modelRefs:
  - model: base-model
    lora_name: humanities-expert
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: semantic-cache
    configuration:
      enabled: true
      similarity_threshold: 0.92
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a psychology expert with deep knowledge of cognitive
        processes, behavioral patterns, mental health, developmental psychology, social
        psychology, and therapeutic approaches. Provide evidence-based insights grounded
        in psychological research and theory. When discussing mental health topics,
        emphasize the importance of professional consultation and avoid providing
        diagnostic or therapeutic advice.
      mode: replace
```

```yaml
- name: biology_decision
  description: Biology and life sciences questions
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: biology
  modelRefs:
  - model: base-model
    lora_name: science-expert
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a biology expert with comprehensive knowledge spanning
        molecular biology, genetics, cell biology, ecology, evolution, anatomy, physiology,
        and biotechnology. Explain biological concepts with scientific accuracy, use
        appropriate terminology, and provide examples from current research. Connect
        biological principles to real-world applications and emphasize the interconnectedness
        of biological systems.
      mode: replace
```

```yaml
- name: chemistry_decision
  description: Chemistry and chemical sciences questions
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: chemistry
  modelRefs:
  - model: base-model
    lora_name: science-expert
    use_reasoning: true
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a chemistry expert specializing in chemical reactions,
        molecular structures, and laboratory techniques. Provide detailed, step-by-step
        explanations.
      mode: replace
```

```yaml
- name: history_decision
  description: Historical questions and cultural topics
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: history
  modelRefs:
  - model: base-model
    lora_name: humanities-expert
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a historian with expertise across different time periods
        and cultures. Provide accurate historical context and analysis.
      mode: replace
```

```yaml
- name: health_decision
  description: Health and medical information queries
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: health
  modelRefs:
  - model: base-model
    lora_name: science-expert
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: semantic-cache
    configuration:
      enabled: true
      similarity_threshold: 0.95
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a health and medical information expert with knowledge
        of anatomy, physiology, diseases, treatments, preventive care, nutrition,
        and wellness. Provide accurate, evidence-based health information while emphasizing
        that your responses are for educational purposes only and should never replace
        professional medical advice, diagnosis, or treatment. Always encourage users
        to consult healthcare professionals for medical concerns and emergencies.
      mode: replace
```

```yaml
- name: economics_decision
  description: Economics and financial topics
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: economics
  modelRefs:
  - model: base-model
    lora_name: social-expert
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are an economics expert with deep understanding of microeconomics,
        macroeconomics, econometrics, financial markets, monetary policy, fiscal policy,
        international trade, and economic theory. Analyze economic phenomena using
        established economic principles, provide data-driven insights, and explain
        complex economic concepts in accessible terms. Consider both theoretical frameworks
        and real-world applications in your responses.
      mode: replace
```

```yaml
- name: math_decision
  description: Mathematics and quantitative reasoning
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: math
  modelRefs:
  - model: base-model
    lora_name: math-expert
    use_reasoning: true
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a mathematics expert. Provide step-by-step solutions,
        show your work clearly, and explain mathematical concepts in an understandable
        way.
      mode: replace
```

```yaml
- name: physics_decision
  description: Physics and physical sciences
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: physics
  modelRefs:
  - model: base-model
    lora_name: science-expert
    use_reasoning: true
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a physics expert with deep understanding of physical
        laws and phenomena. Provide clear explanations with mathematical derivations
        when appropriate.
      mode: replace
```

```yaml
- name: computer_science_decision
  description: Computer science and programming
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: computer science
  modelRefs:
  - model: base-model
    lora_name: science-expert
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a computer science expert with knowledge of algorithms,
        data structures, programming languages, and software engineering. Provide
        clear, practical solutions with code examples when helpful.
      mode: replace
```

```yaml
- name: philosophy_decision
  description: Philosophy and ethical questions
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: philosophy
  modelRefs:
  - model: base-model
    lora_name: humanities-expert
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a philosophy expert with comprehensive knowledge of philosophical
        traditions, ethical theories, logic, metaphysics, epistemology, political
        philosophy, and the history of philosophical thought. Engage with complex
        philosophical questions by presenting multiple perspectives, analyzing arguments
        rigorously, and encouraging critical thinking. Draw connections between philosophical
        concepts and contemporary issues while maintaining intellectual honesty about
        the complexity and ongoing nature of philosophical debates.
      mode: replace
```

```yaml
- name: engineering_decision
  description: Engineering and technical problem-solving
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: engineering
  modelRefs:
  - model: base-model
    lora_name: science-expert
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are an engineering expert with knowledge across multiple
        engineering disciplines including mechanical, electrical, civil, chemical,
        software, and systems engineering. Apply engineering principles, design methodologies,
        and problem-solving approaches to provide practical solutions. Consider safety,
        efficiency, sustainability, and cost-effectiveness in your recommendations.
        Use technical precision while explaining concepts clearly, and emphasize the
        importance of proper engineering practices and standards.
      mode: replace
```

```yaml
- name: general_decision
  description: General knowledge and miscellaneous topics
  priority: 1
  rules:
    operator: OR
    conditions:
    - type: domain
      name: other
  modelRefs:
  - model: base-model
    lora_name: general-expert
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: semantic-cache
    configuration:
      enabled: true
      similarity_threshold: 0.75
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a helpful and knowledgeable assistant. Provide accurate,
        helpful responses across a wide range of topics.
      mode: replace
```

```yaml
- name: business_decision
  description: Business and management related queries
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: business
  modelRefs:
  - model: vllm-llama3-8b-instruct
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a senior business consultant and strategic advisor with
        expertise in corporate strategy, operations management, financial analysis,
        marketing, and organizational development. Provide practical, actionable business
        advice backed by proven methodologies and industry best practices. Consider
        market dynamics, competitive landscape, and stakeholder interests in your
        recommendations.
      mode: replace
```

```yaml
- name: law_decision
  description: Legal questions and law-related topics
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: law
  modelRefs:
  - model: vllm-llama3-8b-instruct
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a knowledgeable legal expert with comprehensive understanding
        of legal principles, case law, statutory interpretation, and legal procedures
        across multiple jurisdictions. Provide accurate legal information and analysis
        while clearly stating that your responses are for informational purposes only
        and do not constitute legal advice. Always recommend consulting with qualified
        legal professionals for specific legal matters.
      mode: replace
```

```yaml
- name: psychology_decision
  description: Psychology and mental health topics
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: psychology
  modelRefs:
  - model: vllm-llama3-8b-instruct
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: semantic-cache
    configuration:
      enabled: true
      similarity_threshold: 0.92
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a psychology expert with deep knowledge of cognitive
        processes, behavioral patterns, mental health, developmental psychology, social
        psychology, and therapeutic approaches. Provide evidence-based insights grounded
        in psychological research and theory. When discussing mental health topics,
        emphasize the importance of professional consultation and avoid providing
        diagnostic or therapeutic advice.
      mode: replace
```

```yaml
- name: biology_decision
  description: Biology and life sciences questions
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: biology
  modelRefs:
  - model: vllm-llama3-8b-instruct
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a biology expert with comprehensive knowledge spanning
        molecular biology, genetics, cell biology, ecology, evolution, anatomy, physiology,
        and biotechnology. Explain biological concepts with scientific accuracy, use
        appropriate terminology, and provide examples from current research. Connect
        biological principles to real-world applications and emphasize the interconnectedness
        of biological systems.
      mode: replace
```

```yaml
- name: chemistry_decision
  description: Chemistry and chemical sciences questions
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: chemistry
  modelRefs:
  - model: vllm-llama3-8b-instruct
    use_reasoning: true
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a chemistry expert specializing in chemical reactions,
        molecular structures, and laboratory techniques. Provide detailed, step-by-step
        explanations.
      mode: replace
```

```yaml
- name: history_decision
  description: Historical questions and cultural topics
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: history
  modelRefs:
  - model: vllm-llama3-8b-instruct
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a historian with expertise across different time periods
        and cultures. Provide accurate historical context and analysis.
      mode: replace
```

```yaml
- name: other_decision
  description: General knowledge and miscellaneous topics
  priority: 5
  rules:
    operator: OR
    conditions:
    - type: domain
      name: other
  modelRefs:
  - model: vllm-llama3-8b-instruct
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: semantic-cache
    configuration:
      enabled: true
      similarity_threshold: 0.75
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a helpful and knowledgeable assistant. Provide accurate,
        helpful responses across a wide range of topics.
      mode: replace
```

```yaml
- name: health_decision
  description: Health and medical information queries
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: health
  modelRefs:
  - model: vllm-llama3-8b-instruct
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: semantic-cache
    configuration:
      enabled: true
      similarity_threshold: 0.95
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a health and medical information expert with knowledge
        of anatomy, physiology, diseases, treatments, preventive care, nutrition,
        and wellness. Provide accurate, evidence-based health information while emphasizing
        that your responses are for educational purposes only and should never replace
        professional medical advice, diagnosis, or treatment. Always encourage users
        to consult healthcare professionals for medical concerns and emergencies.
      mode: replace
```

```yaml
- name: economics_decision
  description: Economics and financial topics
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: economics
  modelRefs:
  - model: vllm-llama3-8b-instruct
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are an economics expert with deep understanding of microeconomics,
        macroeconomics, econometrics, financial markets, monetary policy, fiscal policy,
        international trade, and economic theory. Analyze economic phenomena using
        established economic principles, provide data-driven insights, and explain
        complex economic concepts in accessible terms. Consider both theoretical frameworks
        and real-world applications in your responses.
      mode: replace
```

```yaml
- name: math_decision
  description: Mathematics and quantitative reasoning
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: math
  modelRefs:
  - model: vllm-llama3-8b-instruct
    use_reasoning: true
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a mathematics expert. Provide step-by-step solutions,
        show your work clearly, and explain mathematical concepts in an understandable
        way.
      mode: replace
```

```yaml
- name: physics_decision
  description: Physics and physical sciences
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: physics
  modelRefs:
  - model: vllm-llama3-8b-instruct
    use_reasoning: true
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a physics expert with deep understanding of physical
        laws and phenomena. Provide clear explanations with mathematical derivations
        when appropriate.
      mode: replace
```

```yaml
- name: computer_science_decision
  description: Computer science and programming
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: computer science
  modelRefs:
  - model: vllm-llama3-8b-instruct
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a computer science expert with knowledge of algorithms,
        data structures, programming languages, and software engineering. Provide
        clear, practical solutions with code examples when helpful.
      mode: replace
```

```yaml
- name: philosophy_decision
  description: Philosophy and ethical questions
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: philosophy
  modelRefs:
  - model: vllm-llama3-8b-instruct
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a philosophy expert with comprehensive knowledge of philosophical
        traditions, ethical theories, logic, metaphysics, epistemology, political
        philosophy, and the history of philosophical thought. Engage with complex
        philosophical questions by presenting multiple perspectives, analyzing arguments
        rigorously, and encouraging critical thinking. Draw connections between philosophical
        concepts and contemporary issues while maintaining intellectual honesty about
        the complexity and ongoing nature of philosophical debates.
      mode: replace
```

```yaml
- name: engineering_decision
  description: Engineering and technical problem-solving
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: engineering
  modelRefs:
  - model: vllm-llama3-8b-instruct
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are an engineering expert with knowledge across multiple
        engineering disciplines including mechanical, electrical, civil, chemical,
        software, and systems engineering. Apply engineering principles, design methodologies,
        and problem-solving approaches to provide practical solutions. Consider safety,
        efficiency, sustainability, and cost-effectiveness in your recommendations.
        Use technical precision while explaining concepts clearly, and emphasize the
        importance of proper engineering practices and standards.
      mode: replace
```

```yaml
- name: business_decision
  description: Business and management related queries
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: business
  modelRefs:
  - model: TinyLlama/TinyLlama-1.1B-Chat-v1.0
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a senior business consultant and strategic advisor with
        expertise in corporate strategy, operations management, financial analysis,
        marketing, and organizational development. Provide practical, actionable business
        advice backed by proven methodologies and industry best practices. Consider
        market dynamics, competitive landscape, and stakeholder interests in your
        recommendations.
      mode: replace
```

```yaml
- name: law_decision
  description: Legal questions and law-related topics
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: law
  modelRefs:
  - model: TinyLlama/TinyLlama-1.1B-Chat-v1.0
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a knowledgeable legal expert with comprehensive understanding
        of legal principles, case law, statutory interpretation, and legal procedures
        across multiple jurisdictions. Provide accurate legal information and analysis
        while clearly stating that your responses are for informational purposes only
        and do not constitute legal advice. Always recommend consulting with qualified
        legal professionals for specific legal matters.
      mode: replace
```

```yaml
- name: psychology_decision
  description: Psychology and mental health topics
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: psychology
  modelRefs:
  - model: TinyLlama/TinyLlama-1.1B-Chat-v1.0
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: semantic-cache
    configuration:
      enabled: true
      similarity_threshold: 0.92
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a psychology expert with deep knowledge of cognitive
        processes, behavioral patterns, mental health, developmental psychology, social
        psychology, and therapeutic approaches. Provide evidence-based insights grounded
        in psychological research and theory. When discussing mental health topics,
        emphasize the importance of professional consultation and avoid providing
        diagnostic or therapeutic advice.
      mode: replace
```

```yaml
- name: biology_decision
  description: Biology and life sciences questions
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: biology
  modelRefs:
  - model: TinyLlama/TinyLlama-1.1B-Chat-v1.0
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a biology expert with comprehensive knowledge spanning
        molecular biology, genetics, cell biology, ecology, evolution, anatomy, physiology,
        and biotechnology. Explain biological concepts with scientific accuracy, use
        appropriate terminology, and provide examples from current research. Connect
        biological principles to real-world applications and emphasize the interconnectedness
        of biological systems.
      mode: replace
```

```yaml
- name: chemistry_decision
  description: Chemistry and chemical sciences questions
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: chemistry
  modelRefs:
  - model: TinyLlama/TinyLlama-1.1B-Chat-v1.0
    use_reasoning: true
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a chemistry expert specializing in chemical reactions,
        molecular structures, and laboratory techniques. Provide detailed, step-by-step
        explanations.
      mode: replace
```

```yaml
- name: history_decision
  description: Historical questions and cultural topics
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: history
  modelRefs:
  - model: TinyLlama/TinyLlama-1.1B-Chat-v1.0
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a historian with expertise across different time periods
        and cultures. Provide accurate historical context and analysis.
      mode: replace
```

```yaml
- name: health_decision
  description: Health and medical information queries
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: health
  modelRefs:
  - model: TinyLlama/TinyLlama-1.1B-Chat-v1.0
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: semantic-cache
    configuration:
      enabled: true
      similarity_threshold: 0.95
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a health and medical information expert with knowledge
        of anatomy, physiology, diseases, treatments, preventive care, nutrition,
        and wellness. Provide accurate, evidence-based health information while emphasizing
        that your responses are for educational purposes only and should never replace
        professional medical advice, diagnosis, or treatment. Always encourage users
        to consult healthcare professionals for medical concerns and emergencies.
      mode: replace
```

```yaml
- name: economics_decision
  description: Economics and financial topics
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: economics
  modelRefs:
  - model: TinyLlama/TinyLlama-1.1B-Chat-v1.0
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are an economics expert with deep understanding of microeconomics,
        macroeconomics, econometrics, financial markets, monetary policy, fiscal policy,
        international trade, and economic theory. Analyze economic phenomena using
        established economic principles, provide data-driven insights, and explain
        complex economic concepts in accessible terms. Consider both theoretical frameworks
        and real-world applications in your responses.
      mode: replace
```

```yaml
- name: math_decision
  description: Mathematics and quantitative reasoning
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: math
  modelRefs:
  - model: TinyLlama/TinyLlama-1.1B-Chat-v1.0
    use_reasoning: true
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a mathematics expert. Provide step-by-step solutions,
        show your work clearly, and explain mathematical concepts in an understandable
        way.
      mode: replace
```

```yaml
- name: physics_decision
  description: Physics and physical sciences
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: physics
  modelRefs:
  - model: TinyLlama/TinyLlama-1.1B-Chat-v1.0
    use_reasoning: true
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a physics expert with deep understanding of physical
        laws and phenomena. Provide clear explanations with mathematical derivations
        when appropriate.
      mode: replace
```

```yaml
- name: computer_science_decision
  description: Computer science and programming
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: computer science
  modelRefs:
  - model: TinyLlama/TinyLlama-1.1B-Chat-v1.0
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a computer science expert with knowledge of algorithms,
        data structures, programming languages, and software engineering. Provide
        clear, practical solutions with code examples when helpful.
      mode: replace
```

```yaml
- name: philosophy_decision
  description: Philosophy and ethical questions
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: philosophy
  modelRefs:
  - model: TinyLlama/TinyLlama-1.1B-Chat-v1.0
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a philosophy expert with comprehensive knowledge of philosophical
        traditions, ethical theories, logic, metaphysics, epistemology, political
        philosophy, and the history of philosophical thought. Engage with complex
        philosophical questions by presenting multiple perspectives, analyzing arguments
        rigorously, and encouraging critical thinking. Draw connections between philosophical
        concepts and contemporary issues while maintaining intellectual honesty about
        the complexity and ongoing nature of philosophical debates.
      mode: replace
```

```yaml
- name: engineering_decision
  description: Engineering and technical problem-solving
  priority: 10
  rules:
    operator: OR
    conditions:
    - type: domain
      name: engineering
  modelRefs:
  - model: TinyLlama/TinyLlama-1.1B-Chat-v1.0
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are an engineering expert with knowledge across multiple
        engineering disciplines including mechanical, electrical, civil, chemical,
        software, and systems engineering. Apply engineering principles, design methodologies,
        and problem-solving approaches to provide practical solutions. Consider safety,
        efficiency, sustainability, and cost-effectiveness in your recommendations.
        Use technical precision while explaining concepts clearly, and emphasize the
        importance of proper engineering practices and standards.
      mode: replace
```

```yaml
- name: general_decision
  description: General knowledge and miscellaneous topics
  priority: 1
  rules:
    operator: OR
    conditions:
    - type: domain
      name: other
  modelRefs:
  - model: TinyLlama/TinyLlama-1.1B-Chat-v1.0
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: semantic-cache
    configuration:
      enabled: true
      similarity_threshold: 0.75
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a helpful and knowledgeable assistant. Provide accurate,
        helpful responses across a wide range of topics.
      mode: replace
```

```yaml
- name: math_problems
  description: Route mathematical queries with reasoning enabled
  priority: 100
  rules:
    operator: OR
    conditions:
    - type: keyword
      name: math_keywords
    - type: domain
      name: math
  modelRefs:
  - model: openai/gpt-oss-120b
    use_reasoning: true
    reasoning_effort: high
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a mathematics expert. Provide step-by-step solutions
        with clear explanations.
  - type: semantic-cache
    configuration:
      enabled: true
      similarity_threshold: 0.92
  - type: router_replay
    configuration:
      enabled: true
      max_records: 200
      capture_request_body: true
      capture_response_body: false
      max_body_bytes: 4096
```

```yaml
- name: code_route
  description: Route programming queries
  priority: 100
  rules:
    operator: OR
    conditions:
    - type: keyword
      name: code_keywords
    - type: domain
      name: computer_science
  modelRefs:
  - model: openai/gpt-oss-120b
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a programming expert. Provide clear code examples.
```

```yaml
- name: ratings_route
  description: Route using ratings algorithm for model selection
  priority: 60
  rules:
    operator: OR
    conditions:
    - type: keyword
      name: looper_keywords
    - type: domain
      name: other
  modelRefs:
  - model: openai/gpt-oss-120b
  - model: gpt-5.2
  algorithm:
    type: ratings
    ratings:
      on_error: skip
  plugins:
  - type: semantic-cache
    configuration:
      enabled: false
```

```yaml
- name: math_problems
  description: Route mathematical queries with reasoning enabled
  priority: 100
  rules:
    operator: OR
    conditions:
    - type: keyword
      name: math_keywords
    - type: domain
      name: math
  modelRefs:
  - model: openai/gpt-oss-120b
    use_reasoning: true
    reasoning_effort: high
    lora_name: null
  algorithm: null
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a mathematics expert. Provide step-by-step solutions
        with clear explanations.
  - type: semantic-cache
    configuration:
      enabled: true
      similarity_threshold: 0.92
  - type: router_replay
    configuration:
      enabled: true
      max_records: 200
      capture_request_body: true
      capture_response_body: false
      max_body_bytes: 4096
```

```yaml
- name: code_route
  description: Route programming queries
  priority: 100
  rules:
    operator: OR
    conditions:
    - type: keyword
      name: code_keywords
    - type: domain
      name: computer_science
  modelRefs:
  - model: openai/gpt-oss-120b
    use_reasoning: false
    reasoning_effort: null
    lora_name: null
  algorithm: null
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a programming expert. Provide clear code examples.
```

```yaml
- name: ratings_route
  description: Route using ratings algorithm for model selection
  priority: 60
  rules:
    operator: OR
    conditions:
    - type: keyword
      name: looper_keywords
    - type: domain
      name: other
  modelRefs:
  - model: openai/gpt-oss-120b
    use_reasoning: false
    reasoning_effort: null
    lora_name: null
  - model: gpt-5.2
    use_reasoning: false
    reasoning_effort: null
    lora_name: null
  algorithm:
    type: elo
    confidence: null
    concurrent: null
    remom: null
    latency_aware: null
    elo:
      initial_rating: 1500.0
      k_factor: 32.0
      category_weighted: true
      decay_factor: 0.0
      min_comparisons: 5
      cost_scaling_factor: 0.0
      storage_path: null
      auto_save_interval: 1m
    router_dc: null
    automix: null
    hybrid: null
    thompson: null
    gmtrouter: null
    router_r1: null
    on_error: skip
  plugins:
  - type: semantic-cache
    configuration:
      enabled: false
```

```yaml
- name: other_decision
  description: General knowledge and miscellaneous topics
  priority: 1
  rules:
    operator: OR
    conditions:
    - type: domain
      name: other
  modelRefs:
  - model: base-model
    lora_name: general-expert
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: semantic-cache
    configuration:
      enabled: true
      similarity_threshold: 0.95
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a helpful and knowledgeable assistant. Provide accurate,
        helpful responses across a wide range of topics.
      mode: replace
```

```yaml
- name: default_decision
  description: Default catch-all decision
  priority: 1
  rules:
    operator: OR
    conditions:
    - type: domain
      name: other
  modelRefs:
  - model: openai/gpt-oss-20b
    use_reasoning: false
```

```yaml
- name: science_decision
  description: Science queries (chemistry, biology) - uses KNN
  priority: 100
  rules:
    operator: OR
    conditions:
    - type: domain
      name: chemistry
    - type: domain
      name: biology
  algorithm:
    type: knn
    knn:
      k: 5
  modelRefs:
  - model: llama-3.2-1b
    use_reasoning: false
  - model: llama-3.2-3b
    use_reasoning: false
  - model: codellama-7b
    use_reasoning: false
  - model: mistral-7b
    use_reasoning: false
```

```yaml
- name: business_decision
  description: Business and economics queries - uses SVM
  priority: 100
  rules:
    operator: OR
    conditions:
    - type: domain
      name: business
    - type: domain
      name: economics
  algorithm:
    type: svm
    svm:
      kernel: rbf
  modelRefs:
  - model: llama-3.2-1b
    use_reasoning: false
  - model: llama-3.2-3b
    use_reasoning: false
  - model: codellama-7b
    use_reasoning: false
  - model: mistral-7b
    use_reasoning: false
```

```yaml
- name: humanities_decision
  description: History, philosophy, psychology - uses KNN
  priority: 90
  rules:
    operator: OR
    conditions:
    - type: domain
      name: history
    - type: domain
      name: philosophy
    - type: domain
      name: psychology
  algorithm:
    type: knn
    knn:
      k: 5
  modelRefs:
  - model: llama-3.2-1b
    use_reasoning: false
  - model: llama-3.2-3b
    use_reasoning: false
  - model: codellama-7b
    use_reasoning: false
  - model: mistral-7b
    use_reasoning: false
```

```yaml
- name: default_decision
  description: Default route for all requests
  priority: 1
  rules:
    operator: OR
    conditions:
    - type: domain
      name: other
  modelRefs:
  - model: base-model
    lora_name: general-expert
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: semantic-cache
    configuration:
      enabled: true
      similarity_threshold: 0.8
```

```yaml
- name: other_decision
  description: General knowledge and miscellaneous topics
  priority: 1
  rules:
    operator: OR
    conditions:
    - type: domain
      name: other
  modelRefs:
  - model: base-model
    lora_name: general-expert
    use_reasoning: false
```

```yaml
- name: math_decision
  priority: 100
  rules:
    operator: OR
    conditions:
    - type: domain
      name: math
  modelRefs:
  - model: phi4-mini
    use_reasoning: false
```

```yaml
- name: computer_science_decision
  priority: 100
  rules:
    operator: OR
    conditions:
    - type: domain
      name: computer science
  modelRefs:
  - model: llama3-8b
    use_reasoning: false
```

```yaml
- name: physics_decision
  priority: 50
  rules:
    operator: OR
    conditions:
    - type: domain
      name: physics
  modelRefs:
  - model: llama3-8b
    use_reasoning: false
```

```yaml
- name: chemistry_decision
  priority: 50
  rules:
    operator: OR
    conditions:
    - type: domain
      name: chemistry
  modelRefs:
  - model: llama3-8b
    use_reasoning: false
```

```yaml
- name: biology_decision
  priority: 50
  rules:
    operator: OR
    conditions:
    - type: domain
      name: biology
  modelRefs:
  - model: llama3-8b
    use_reasoning: false
```

```yaml
- name: engineering_decision
  priority: 50
  rules:
    operator: OR
    conditions:
    - type: domain
      name: engineering
  modelRefs:
  - model: llama3-8b
    use_reasoning: false
```

```yaml
- name: health_decision
  priority: 50
  rules:
    operator: OR
    conditions:
    - type: domain
      name: health
  modelRefs:
  - model: llama3-8b
    use_reasoning: false
```

```yaml
- name: psychology_decision
  priority: 40
  rules:
    operator: OR
    conditions:
    - type: domain
      name: psychology
  modelRefs:
  - model: llama3-8b
    use_reasoning: false
```

```yaml
- name: economics_decision
  priority: 40
  rules:
    operator: OR
    conditions:
    - type: domain
      name: economics
  modelRefs:
  - model: llama3-8b
    use_reasoning: false
```

```yaml
- name: business_decision
  priority: 40
  rules:
    operator: OR
    conditions:
    - type: domain
      name: business
  modelRefs:
  - model: llama3-8b
    use_reasoning: false
```

```yaml
- name: history_decision
  priority: 40
  rules:
    operator: OR
    conditions:
    - type: domain
      name: history
  modelRefs:
  - model: llama3-8b
    use_reasoning: false
```

```yaml
- name: philosophy_decision
  priority: 40
  rules:
    operator: OR
    conditions:
    - type: domain
      name: philosophy
  modelRefs:
  - model: llama3-8b
    use_reasoning: false
```

```yaml
- name: law_decision
  priority: 40
  rules:
    operator: OR
    conditions:
    - type: domain
      name: law
  modelRefs:
  - model: llama3-8b
    use_reasoning: false
```

```yaml
- name: other_decision
  priority: 1
  rules:
    operator: OR
    conditions:
    - type: domain
      name: other
  modelRefs:
  - model: llama3-8b
    use_reasoning: false
```

```yaml
- name: general_tech
  description: General technical queries - use small model for efficiency
  priority: 50
  rules:
    operator: OR
    conditions:
    - type: embedding
      name: tech_support
    - type: domain
      name: computer_science
  modelRefs:
  - model: qwen-2.5-7b
    lora_name: tech-support
    use_reasoning: false
```

```yaml
- name: general_business
  description: General business and economics queries
  priority: 50
  rules:
    operator: OR
    conditions:
    - type: embedding
      name: business_analysis
    - type: domain
      name: economics
  modelRefs:
  - model: qwen-2.5-72b
    use_reasoning: false
  plugins:
  - type: semantic-cache
    configuration:
      enabled: true
      similarity_threshold: 0.85
```

```yaml
- name: stem_query
  description: Complex STEM domain queries - use large model
  priority: 100
  rules:
    operator: OR
    conditions:
    - type: domain
      name: math
    - type: domain
      name: physics
    - type: domain
      name: computer_science
  modelRefs:
  - model: mistral-large
    lora_name: math-expert
    use_reasoning: false
```

```yaml
- name: math_problems
  description: Route mathematical queries with reasoning enabled
  priority: 100
  rules:
    operator: OR
    conditions:
    - type: keyword
      name: math_keywords
    - type: domain
      name: math
    - type: fact_check
      name: needs_fact_check
  modelRefs:
  - model: openai/gpt-oss-120b
    use_reasoning: true
    reasoning_effort: high
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a mathematics expert. Provide step-by-step solutions
        with clear explanations.
  - type: semantic-cache
    configuration:
      enabled: true
      similarity_threshold: 0.92
  - type: rag
    configuration:
      enabled: true
      backend: external_api
      top_k: 5
      similarity_threshold: 0.75
      injection_mode: tool_role
      on_failure: skip
      cache_results: true
      cache_ttl_seconds: 300
      backend_config:
        endpoint: http://host.docker.internal:8001/search
        request_format: elasticsearch
        timeout_seconds: 10
  - type: hallucination
    configuration:
      enabled: true
      hallucination_action: header
  - type: router_replay
    configuration:
      enabled: true
      max_records: 200
      capture_request_body: true
      capture_response_body: true
      max_body_bytes: 4096
```

### E. 通用及其他路由调度 (General Routing)

```yaml
- name: general_to_cloud
  description: General queries → GPT-4o (cloud, with failover)
  priority: 100
  rules:
    operator: OR
    conditions:
    - type: keyword
      name: general_keywords
  modelRefs:
  - model: gpt-4o
    use_reasoning: false
```

```yaml
- name: fact_check_needed
  description: Route queries that need fact verification
  priority: 90
  rules:
    operator: OR
    conditions:
    - type: fact_check
      name: needs_fact_check
  modelRefs:
  - model: qwen2.5:3b
    use_reasoning: true
    weight: 100
```

```yaml
- name: handle_clarification
  description: User needs clarification
  priority: 85
  rules:
    operator: OR
    conditions:
    - type: user_feedback
      name: need_clarification
  modelRefs:
  - model: qwen2.5:3b
    use_reasoning: false
    weight: 100
```

```yaml
- name: handle_wrong_answer
  description: User indicates wrong answer
  priority: 85
  rules:
    operator: OR
    conditions:
    - type: user_feedback
      name: wrong_answer
  modelRefs:
  - model: qwen2.5:3b
    use_reasoning: true
    weight: 100
```

```yaml
- name: handle_want_different
  description: User wants different approach
  priority: 85
  rules:
    operator: OR
    conditions:
    - type: user_feedback
      name: want_different
  modelRefs:
  - model: qwen2.5:3b
    use_reasoning: false
    weight: 100
```

```yaml
- name: complex_reasoning
  description: Complex reasoning in Chinese requiring advanced models
  priority: 131
  rules:
    operator: OR
    conditions:
    - type: embedding
      name: deep_thinking_zh
    - type: keyword
      name: thinking_zh
  modelRefs:
  - model: Kimi-K2-Thinking
    use_reasoning: true
    reasoning_effort: high
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: 你是 Kimi-K2-Thinking，一个专业的AI助手，擅长深度分析和复杂推理。请提供详细、深入的回答。
  - type: router_replay
    configuration:
      enabled: true
      max_records: 1000
      capture_request_body: true
      capture_response_body: true
      max_body_bytes: 4096
```

```yaml
- name: deep_thinking
  description: Complex reasoning in English
  priority: 110
  rules:
    operator: OR
    conditions:
    - type: embedding
      name: deep_thinking_en
    - type: keyword
      name: thinking_en
    - type: context
      name: long_context
  modelRefs:
  - model: Kimi-K2-Thinking
    use_reasoning: true
    reasoning_effort: high
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are Kimi-K2-Thinking, an expert analyst. Provide comprehensive,
        well-reasoned responses with deep insights.
  - type: semantic-cache
    configuration:
      enabled: false
  - type: router_replay
    configuration:
      enabled: true
      max_records: 1000
      capture_request_body: true
      capture_response_body: true
      max_body_bytes: 4096
```

```yaml
- name: thinking_decision
  description: Complex reasoning and multi-step thinking
  priority: 20
  rules:
    operator: OR
    conditions:
    - type: keyword
      rule_name: thinking
  modelRefs:
  - model: base-model
    lora_name: general-expert
    use_reasoning: true
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a thinking expert, should think multiple steps before
        answering. Please answer the question step by step.
      mode: replace
```

```yaml
- name: thinking_decision
  description: Complex reasoning and multi-step thinking
  priority: 15
  rules:
    operator: OR
    conditions:
    - type: keyword
      name: thinking
  modelRefs:
  - model: vllm-llama3-8b-instruct
    use_reasoning: true
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a thinking expert, should think multiple steps before
        answering. Please answer the question step by step.
      mode: replace
```

```yaml
- name: thinking_decision
  description: Complex reasoning and multi-step thinking
  priority: 20
  rules:
    operator: OR
    conditions:
    - type: keyword
      rule_name: thinking
  modelRefs:
  - model: TinyLlama/TinyLlama-1.1B-Chat-v1.0
    use_reasoning: true
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a thinking expert, should think multiple steps before
        answering. Please answer the question step by step.
      mode: replace
```

```yaml
- name: thinking_decision
  description: Complex reasoning and multi-step thinking
  priority: 20
  rules:
    operator: OR
    conditions:
    - type: keyword
      name: thinking
  modelRefs:
  - model: base-model
    lora_name: general-expert
    use_reasoning: true
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are a thinking expert, should think multiple steps before
        answering. Please answer the question step by step.
      mode: replace
```

```yaml
- name: urgent_request
  description: Urgent requests requiring immediate attention
  priority: 30
  rules:
    operator: OR
    conditions:
    - type: keyword
      name: urgent_keywords
  modelRefs:
  - model: base-model
    lora_name: general-expert
    use_reasoning: false
  plugins:
  - type: pii
    configuration:
      enabled: true
      pii_types_allowed: []
  - type: system_prompt
    configuration:
      enabled: true
      system_prompt: You are handling an urgent request. Prioritize quick and direct
        responses.
      mode: replace
```

```yaml
- name: general_learning
  description: General learning queries
  priority: 50
  rules:
    operator: OR
    conditions:
    - type: keyword
      name: tutorial
    - type: embedding
      name: learning_intent
  modelRefs:
  - model: qwen-2.5-32b
    use_reasoning: false
```

```yaml
- name: general_support
  description: General support requests - use small model
  priority: 50
  rules:
    operator: OR
    conditions:
    - type: keyword
      name: urgent
    - type: embedding
      name: support_request
  modelRefs:
  - model: qwen-2.5-14b
    lora_name: support-agent
    use_reasoning: false
```

## 4. 复杂嵌套与高级特征组合 (Complex Nested & Advanced Operators)

### B. 专家级领域分发 (Domain Specific Routing)

```yaml
- name: not_code
  description: NOT — not a code request → conversation model
  priority: 200
  rules:
    operator: NOT
    conditions:
    - type: keyword
      name: code_request
  modelRefs:
  - model: conversation_model
```

```yaml
- name: nor_cs_math
  description: NOR — neither computer_science nor math domain → humanities model
  priority: 180
  rules:
    operator: NOT
    conditions:
    - operator: OR
      conditions:
      - type: domain
        name: computer_science
      - type: domain
        name: math
  modelRefs:
  - model: humanities_model
```

```yaml
- name: nand_zh_code
  description: NAND — not (Chinese AND code request) → non-Chinese code model
  priority: 160
  rules:
    operator: NOT
    conditions:
    - operator: AND
      conditions:
      - type: language
        name: zh
      - type: keyword
        name: code_request
  modelRefs:
  - model: non_zh_code_model
```

### E. 通用及其他路由调度 (General Routing)

```yaml
- name: general_fallback
  description: Route all non-STEM queries to the fast general model
  priority: 50
  rules:
    operator: NOT
    conditions:
    - operator: OR
      conditions:
      - type: domain
        name: computer_science
      - type: domain
        name: math
      - type: domain
        name: physics
      - type: domain
        name: engineering
      - type: domain
        name: chemistry
  modelRefs:
  - model: general_model
    use_reasoning: false
  plugins:
  - type: system_prompt
    configuration:
      system_prompt: You are a helpful general-purpose assistant. Answer concisely
        and clearly.
```


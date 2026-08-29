# 🚀 LLM Gateway with LiteLLM & LangChain

Comprehensive documentation and code examples for building a production-ready **LLM Gateway** using **LiteLLM** and **LangChain**.

---

## 📌 Overview

An **LLM Gateway** acts as a smart middleware layer sitting between your applications (Chatbots, RAG systems, AI Agents) and multiple Large Language Model providers (OpenAI, Anthropic, Google Gemini, Groq, local models, etc.).

It solves key production challenges by providing a single, unified interface for routing, automatic fallbacks, caching, cost tracking, rate limiting, and observability.

```
                    ┌─────────────────────────────┐
                    │       Your Application      │
                    │  (Chatbot, RAG, Agent, etc) │
                    └──────────────┬──────────────┘
                                   │
                                   ▼
                    ┌─────────────────────────────┐
                    │       LLM GATEWAY           │
                    │  • Routing & Load Balancing │
                    │  • Automatic Fallbacks      │
                    │  • Caching                   │
                    │  • Rate Limiting & Timeouts │
                    │  • Cost Tracking            │
                    │  • Observability & Logging  │
                    │  • Security Guardrails      │
                    └──────┬─────┬─────┬─────┬────┘
                           │     │     │     │
                           ▼     ▼     ▼     ▼
                        OpenAI Claude Gemini Groq
```

---

## 🌟 Key Features

- **Unified API**: Call 100+ LLM providers using a single `completion()` interface.
- **Automatic Fallbacks**: Transparently failover to secondary providers (e.g., `gpt-4o-mini` → `groq/llama-3.3-70b-versatile` → `claude-3-5-sonnet`) during outages or rate limits.
- **Cost Tracking & Optimization**: Calculate exact USD costs per API request automatically using built-in pricing databases.
- **Caching**: Cache repeated queries locally (or in Redis) to save costs and reduce latency.
- **Smart Routing & Load Balancing**: Support strategies like `least-busy`, `latency-based-routing`, and `simple-shuffle`.
- **Observability & Logging**: Hook custom success/failure callbacks to record input/output tokens, latency, cost, and user IDs.
- **LangChain Integration**: Seamlessly plug `ChatLiteLLM` into LangChain Expression Language (LCEL) chains and agents.
- **Security Guardrails**: Implement pre-call and post-call hooks for PII redaction, prompt injection blocking, and forbidden topic filtering.

---

## 🛠️ Installation & Setup

### Prerequisites
- Python `>= 3.10`

### 1. Install Dependencies

Install the required packages using `pip`:

```bash
pip install litellm langchain langchain-community langchain-openai langchain-litellm python-dotenv
```

### 2. Environment Configuration

Create a `.env` file in the root directory and add your API keys:

```env
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
GROQ_API_KEY=gsk_...
GEMINI_API_KEY=...
```

Verify your environment setup in Python:

```python
import os
from dotenv import load_dotenv

load_dotenv()

print("OpenAI Key:", "✅ Loaded" if os.getenv("OPENAI_API_KEY") else "❌ Missing")
print("Groq Key:",   "✅ Loaded" if os.getenv("GROQ_API_KEY") else "❌ Missing")
```

---

## 📖 Practical Usage & Code Examples

### 1. Simple Unified API Call

Swap providers seamlessly by changing the `model` string parameter:

```python
from litellm import completion

# OpenAI Call
response_openai = completion(
    model="gpt-4o-mini",
    messages=[{"role": "user", "content": "Explain RAG in one sentence."}]
)
print("🔵 OpenAI:", response_openai.choices[0].message.content)

# Groq Call (Ultra-fast inference)
response_groq = completion(
    model="groq/llama-3.3-70b-versatile",
    messages=[{"role": "user", "content": "Explain RAG in one sentence."}]
)
print("🟢 Groq:", response_groq.choices[0].message.content)
```

---

### 2. Automatic Fallbacks

Ensure high availability by specifying backup models if the primary provider fails or encounters rate limits:

```python
from litellm import completion

response = completion(
    model="gemini/gemini-1.5-flash",
    messages=[{"role": "user", "content": "What is an LLM Gateway?"}],
    fallbacks=[
        "gpt-4o-mini",
        "groq/llama-3.3-70b-versatile"
    ]
)

print("Response:", response.choices[0].message.content[:200])
print("Model that actually answered:", response.model)
```

---

### 3. Real-Time Cost Tracking

Track exact usage metrics and token pricing per call:

```python
from litellm import completion, completion_cost

response = completion(
    model="gpt-4o-mini",
    messages=[{"role": "user", "content": "Write a haiku about AI."}]
)

cost = completion_cost(completion_response=response)

print("Response:", response.choices[0].message.content)
print(f"Prompt Tokens: {response.usage.prompt_tokens}")
print(f"Completion Tokens: {response.usage.completion_tokens}")
print(f"Total Cost: ${cost:.8f}")
```

---

### 4. Response Caching

Cache identical prompts in-memory to drastically accelerate response times and eliminate duplicate costs:

```python
import time
import litellm
from litellm import completion
from litellm.caching import Cache

# Enable local in-memory cache (Redis is supported for production)
litellm.cache = Cache(type="local")

prompt = "What does LLM stand for? Answer in one line."

# First call - API request
start = time.time()
r1 = completion(model="gpt-4o-mini", messages=[{"role": "user", "content": prompt}], caching=True)
print(f"First call (API): {time.time() - start:.2f}s")

# Second call - Served from cache
start = time.time()
r2 = completion(model="gpt-4o-mini", messages=[{"role": "user", "content": prompt}], caching=True)
print(f"Second call (Cache): {time.time() - start:.4f}s")
```

---

### 5. Smart Router & Load Balancing

Route requests based on predefined aliases or load-balancing strategies (`least-busy`, `latency-based-routing`, `simple-shuffle`):

```python
import os
from litellm import Router

model_list = [
    {
        "model_name": "fast-cheap",
        "litellm_params": {
            "model": "groq/llama-3.3-70b-versatile",
            "api_key": os.getenv("GROQ_API_KEY")
        }
    },
    {
        "model_name": "smart-coding",
        "litellm_params": {
            "model": "gpt-4o",
            "api_key": os.getenv("OPENAI_API_KEY")
        }
    }
]

router = Router(model_list=model_list)

fast_res = router.completion(model="fast-cheap", messages=[{"role": "user", "content": "Summarize AI."}])
code_res = router.completion(model="smart-coding", messages=[{"role": "user", "content": "Write Python code for quicksort."}])
```

---

### 6. Observability & Callbacks

Set up global audit logging for tracking requests across teams or users:

```python
import litellm
from litellm import completion

call_logs = []

def log_success(kwargs, completion_response, start_time, end_time):
    call_logs.append({
        "model": kwargs.get("model"),
        "prompt": kwargs["messages"][-1]["content"][:60],
        "input_tokens": completion_response.usage.prompt_tokens,
        "output_tokens": completion_response.usage.completion_tokens,
        "latency_sec": round((end_time - start_time).total_seconds(), 2),
        "user": kwargs.get("user", "anonymous")
    })

litellm.success_callback = [log_success]

# Execute a tagged call
completion(
    model="gpt-4o-mini",
    messages=[{"role": "user", "content": "What is RAG?"}],
    user="alice"
)

print(call_logs)
```

---

### 7. LangChain Integration with Fallbacks

Integrate LiteLLM with LangChain Expression Language (LCEL):

```python
from langchain_litellm import ChatLiteLLM
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

# Configure models with fallback capabilities
primary = ChatLiteLLM(model="gpt-x")  # Intentional invalid model to trigger fallback
fallback_1 = ChatLiteLLM(model="gpt-4o-mini", temperature=0.2)
fallback_2 = ChatLiteLLM(model="groq/llama-3.3-70b-versatile", temperature=0.2)

robust_llm = primary.with_fallbacks([fallback_1, fallback_2])

prompt = ChatPromptTemplate.from_messages([
    ("system", "You are an expert AI tutor. Reply concisely."),
    ("user", "{question}")
])

chain = prompt | robust_llm | StrOutputParser()

response = chain.invoke({"question": "Explain fine-tuning in 2 sentences."})
print(response)
```

---

### 8. Security Guardrails

#### PII Redaction
Scrub sensitive information (emails, phones, Aadhaar, PAN, SSN) before sending prompts to external providers:

```python
import re
import litellm
from litellm import completion

PII_PATTERNS = {
    "EMAIL": r"[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}",
    "PHONE": r"(\+91[\-\s]?)?[6-9]\d{9}",
    "PAN":   r"\b[A-Z]{5}\d{4}[A-Z]\b",
}

def pii_input_guardrail(kwargs):
    messages = kwargs.get("messages", [])
    for msg in messages:
        if msg.get("role") == "user":
            content = msg["content"]
            for label, pattern in PII_PATTERNS.items():
                content = re.sub(pattern, f"<{label}_REDACTED>", content)
            msg["content"] = content

litellm.input_callback = [pii_input_guardrail]
```

#### Prompt Injection Guardrail
Block malicious attempts to bypass system prompts:

```python
import re
import litellm

INJECTION_PATTERNS = [
    r"ignore (all |the )?(previous|prior|above) (instructions?|prompts?|rules?)",
    r"you are (now |a )?(DAN|jailbroken|unrestricted)",
]
INJECTION_REGEX = [re.compile(p, re.IGNORECASE) for p in INJECTION_PATTERNS]

def injection_guardrail(kwargs):
    messages = kwargs.get("messages", [])
    for msg in messages:
        if msg.get("role") == "user":
            content = msg["content"]
            for regex in INJECTION_REGEX:
                if regex.search(content):
                    raise ValueError("Blocked: Prompt injection attempt detected.")

litellm.input_callback = [injection_guardrail]
```

---

## 🛡️ Production Best Practices

1. **Use Redis Caching**: Shared cache across API replicas that survives process restarts.
2. **Virtual Keys & Rate Limits**: Issue virtual keys per team/user to restrict token spend and rate limit heavy users.
3. **Observability Integration**: Export logs to platforms like Langfuse, Helicone, Arize, or OpenTelemetry.
4. **Timeouts & Retries**: Always set `timeout` and `num_retries` to avoid hanging requests.
5. **Model Pinning**: Always pin specific model versions (e.g., `gpt-4o-2024-08-06`) to prevent upstream regressions.
6. **Config as Code**: Store your proxy configuration in a version-controlled `config.yaml` file.

---

## 📊 LLM Gateway Comparison Matrix

| Gateway | Type | Best For |
|---|---|---|
| **LiteLLM** | Open-source | Swiss army knife: 100+ providers, lightweight, python/proxy |
| **Portkey** | SaaS / OSS | Advanced enterprise observability & prompt management |
| **Helicone** | SaaS / OSS | One-line drop-in proxy with clean analytics dashboard |
| **Cloudflare AI Gateway** | SaaS | One-click edge caching and rate limiting |
| **Kong AI Gateway** | Enterprise | API Gateway extensions for enterprise microservices |
| **OpenRouter** | SaaS | Consolidated unified billing across 100+ AI models |

---

## 📁 Repository Structure

```
.
├── README.md               # Project documentation & reference guide
├── llm_gateway_work.ipynb  # Interactive Jupyter notebook with complete tutorial & demos
└── pyproject.toml          # Project configuration & python setup
```

---

## 📜 License & Credits

Tutorial created by **Krish Naik | KRISHAI Technologies**.  
Built using [LiteLLM](https://github.com/BerriAI/litellm) and [LangChain](https://github.com/langchain-ai/langchain).
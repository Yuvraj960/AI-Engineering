# Module 01: Zero-Shot and Few-Shot Prompting

## 1. Executive Summary & Core Concepts

Prompt Engineering is the discipline of structuring text queries and contextual data to steer Large Language Models (LLMs) toward accurate, aligned, and deterministic outputs. At the heart of prompt engineering lies **In-Context Learning (ICL)**: the ability of transformer-based autoregressive models to adapt to tasks at inference time without modifying underlying model weights (parameters).

### 1.1 Transformer Architecture & Mechanistic Foundations
Modern LLMs (e.g., GPT-4, Claude 3, Llama 3) operate on the **Transformer Decoder Architecture** utilizing multi-head self-attention mechanisms. When a prompt is passed into an LLM:

1. **Tokenization & Embedding**: Text is parsed into discrete tokens and mapped into high-dimensional vector spaces ($d_{\text{model}}$).
2. **Self-Attention**: Each token attends to all preceding tokens via Attention matrices ($Q, K, V$):
   $$\text{Attention}(Q, K, V) = \text{softmax}\left(\frac{QK^T}{\sqrt{d_k}}\right)V$$
3. **Contextual In-Context Learning**: Demonstrations provided in the prompt create "task vectors" across attention layers, shifting the conditional probability distribution $P(w_{t} \mid w_1, w_2, \dots, w_{t-1})$ toward the desired format and task domain.

---

## 2. Zero-Shot Prompting

### 2.1 Definition & Under the Hood Mechanics
**Zero-Shot Prompting** relies entirely on the pre-trained parametric knowledge of the LLM. No input-output examples are provided in the context window. The model must rely on its instruction-tuning (RLHF/DPO) to infer task intent solely from the prompt instructions.

### 2.2 Ideal Use Cases & Limitations
* **Ideal For**: General text classification, basic summarization, language translation, open-ended ideation, and tasks where domain-specific formatting is standardized.
* **Limitations**: High sensitivity to instruction wording; prone to hallucination on complex, multi-step tasks or niche data formats.

### 2.3 Prompt Pattern & Implementation
```text
[Task Instruction]: Classify the sentiment of the text into Positive, Negative, or Neutral.
[Text]: "The API response latency spiked to 450ms after the deployment, causing client timeout errors."
[Constraint]: Return ONLY the class label.
```

---

## 3. One-Shot and Few-Shot Prompting

### 3.1 Definition & In-Context Mechanics
* **One-Shot Prompting**: Providing exactly one exemplar (input $\rightarrow$ output pair) to guide the format and semantic task mapping.
* **Few-Shot Prompting**: Providing multiple ($k = 2 \text{ to } 10+$) exemplars before presenting the target query.

Few-shot prompting conditions the transformer's attention heads to recognize patterns in structure, tone, constraints, and operational logic.

### 3.2 Anatomy of a High-Performance Few-Shot Prompt
A production-grade few-shot prompt consists of four primary blocks:
1. **System Instruction**: High-level goal and strict behavioral boundaries.
2. **Exemplars (Task Context)**: Standardized $Input \rightarrow Output$ pairs representing diverse edge cases.
3. **Delimiter Formatting**: Clear markup (e.g., XML tags, Markdown headers) separating exemplars.
4. **Target Query**: The live input requiring execution.

---

## 4. Engineering Principles & Best Practices

### 4.1 Exemplar Selection Strategies
* **Diversity**: Include positive, negative, edge, and ambiguous cases.
* **Relevance**: Use semantic search (vector embeddings) to dynamically retrieve the $k$-most relevant exemplars for the input query (Dynamic Few-Shot In-Context Learning).
* **Label Distribution**: Ensure balanced target labels to prevent majority-class bias.

### 4.2 Formatting & Label Calibration
* **Label Order Bias**: LLMs exhibit recency and frequency bias. Keep example order balanced or randomly shuffled.
* **Delimiters**: Use explicit structural tags (`<example>`, `### Input`, `### Output`) to prevent prompt injection and task confusion.

---

## 5. Production Python Implementation

The following script demonstrates zero-shot, static few-shot, and dynamic few-shot prompting patterns using Python and Pydantic validation.

```python
import os
from typing import List, Literal
from pydantic import BaseModel, Field

# Mocking an OpenAI or Anthropic API Client Call structure
class SentimentAnalysisRequest(BaseModel):
    text: str
    mode: Literal["zero_shot", "few_shot"]

class SentimentResponse(BaseModel):
    sentiment: Literal["POSITIVE", "NEGATIVE", "NEUTRAL"]
    confidence_score: float = Field(ge=0.0, le=1.0)
    reasoning: str

# 1. Zero-Shot Prompt Construction
def build_zero_shot_prompt(input_text: str) -> str:
    return f"""You are an expert NLP classifier. 
Classify the sentiment of the following support ticket text.

Text: "{input_text}"

Respond strictly in JSON matching this structure:
{{
  "sentiment": "POSITIVE" | "NEGATIVE" | "NEUTRAL",
  "confidence_score": float,
  "reasoning": "brief explanation"
}}
"""

# 2. Few-Shot Prompt Construction
FEW_SHOT_EXEMPLARS = [
    {
        "input": "The deployment went smoothly and system memory usage dropped by 15%.",
        "output": '{"sentiment": "POSITIVE", "confidence_score": 0.98, "reasoning": "Smooth deployment and reduced memory usage indicate optimal state."}'
    },
    {
        "input": "Database query connection timed out during peak load hours.",
        "output": '{"sentiment": "NEGATIVE", "confidence_score": 0.95, "reasoning": "Database timeout directly impacts system availability."}'
    },
    {
        "input": "Scheduled maintenance will occur tonight at 02:00 UTC.",
        "output": '{"sentiment": "NEUTRAL", "confidence_score": 0.90, "reasoning": "Informational notification with no positive or negative impact."}'
    }
]

def build_few_shot_prompt(input_text: str, exemplars: List[dict]) -> str:
    prompt_parts = [
        "You are an expert NLP classifier. Classify sentiment based on the provided examples.\n"
    ]
    
    # Append Exemplars
    for idx, ex in enumerate(exemplars, 1):
        prompt_parts.append(f"<example_{idx}>")
        prompt_parts.append(f"Input: {ex['input']}")
        prompt_parts.append(f"Output: {ex['output']}")
        prompt_parts.append(f"</example_{idx}>\n")
        
    # Append Live Query
    prompt_parts.append("<target_query>")
    prompt_parts.append(f"Input: {input_text}")
    prompt_parts.append("Output:")
    prompt_parts.append("</target_query>")
    
    return "\n".join(prompt_parts)

# Execution Sandbox
if __name__ == "__main__":
    test_text = "The new UI layout is crisp, though the search bar occasionally lags."
    
    zero_shot_p = build_zero_shot_prompt(test_text)
    few_shot_p = build_few_shot_prompt(test_text, FEW_SHOT_EXEMPLARS)
    
    print("--- GENERATED ZERO-SHOT PROMPT ---")
    print(zero_shot_p)
    print("\n--- GENERATED FEW-SHOT PROMPT ---")
    print(few_shot_p)
```

---

## 6. Summary Comparison Matrix

| Feature / Metric | Zero-Shot Prompting | Few-Shot Prompting |
| :--- | :--- | :--- |
| **Token Consumption** | Low (Minimal context overhead) | High (Increases linearly with $k$ examples) |
| **Deterministic Output** | Moderate to Low | High (Exemplars enforce strict syntax) |
| **Complex Logic Handling** | Poor | Superior |
| **Out-of-Distribution Handling** | Relies entirely on pre-training | Can be steered via specialized domain exemplars |
| **Primary Risk** | Hallucinations, formatting drift | Example ordering bias, context window bloat |
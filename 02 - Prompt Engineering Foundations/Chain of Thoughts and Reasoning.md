# Module 02: Chain-of-Thought (CoT) & Advanced Reasoning Frameworks

## 1. Theoretical Foundations & Mechanistic Overview

Large Language Models (LLMs) operate as **autoregressive, next-token prediction engines**. When presented with a prompt, an LLM estimates the joint probability distribution of a token sequence by factoring it into a product of conditional probabilities:

$$P(w_1, w_2, \dots, w_N) = \prod_{i=1}^{N} P(w_i \mid w_1, w_2, \dots, w_{i-1})$$

When an LLM is asked to solve complex logic, arithmetic, or multi-step algorithmic problems in a direct mapping fashion ($Input \rightarrow Final\ Answer$), it must perform all intermediate computational steps within a single forward pass through its transformer layers. 

### 1.1 Why Standard Prompting Fails on Complex Tasks
Transformer layers perform a fixed amount of computation per token output. Forcing a direct answer limits the depth of computation to $O(1)$ operations relative to the difficulty of the problem context. 

### 1.2 How Chain-of-Thought Works
**Chain-of-Thought (CoT)** prompting shifts the inference dynamics by encouraging the model to generate intermediate steps (reasoning tokens) *before* emitting the final answer. 

1. **Expands Compute at Inference Time**: Each generated intermediate token adds an extra forward pass, increasing total inference compute proportional to the complexity of the task.
2. **Attentional Grounding**: Subsequent tokens attend back to the previously generated reasoning tokens ($w_1, \dots, w_{k-1}$), effectively using the context window as a scratchpad or working memory.
3. **Reduces Error Propagation**: Breaking down complex tasks into discrete sub-problems lowers the conditional error probability at each step.

---

## 2. Primary Reasoning Paradigms

```
+-------------------------------------------------------------------------+
|                         Prompting Taxonomies                             |
+-------------------------------------------------------------------------+
|  1. Direct Output       : [Query] -> [Final Output]                      |
|  2. Zero-Shot CoT       : [Query] -> "Let's think..." -> [Steps] -> [Ans]  |
|  3. Few-Shot CoT        : [Exemplars w/ Steps] -> [Query] -> [Steps]    |
|  4. Self-Consistency    : Sample N CoT paths -> Majority Vote           |
|  5. Tree-of-Thoughts    : Search tree (BFS/DFS) -> Evaluate State       |
|  6. ReAct Agent Loop    : Thought -> Action -> Observation -> Final Ans   |
+-------------------------------------------------------------------------+
```

### 2.1 Zero-Shot CoT
* **Mechanism**: Appending a trigger phrase such as `"Let's think step by step."` or `"Break down your reasoning before answering."` to the prompt.
* **Effect**: Instructs the model's instruction-tuned layers to transition into a step-by-step reasoning pattern without needing specific input-output exemplars.

### 2.2 Few-Shot CoT
* **Mechanism**: Providing explicit exemplars where input queries are paired with step-by-step reasoning chains followed by the final answer.
* **Effect**: Establishes strict behavioral patterns, structural formatting, and specific reasoning methods tailored to a target domain.

---

## 3. Advanced Reasoning Paradigms & Architectures

### 3.1 Self-Consistency (CoT-SC)
Instead of greedy decoding ($\text{Temperature} = 0$), Self-Consistency samples multiple distinct reasoning paths by setting $\text{Temperature} > 0$ (e.g., $T = 0.7$). The final answer is selected via **majority voting** or marginalizing over the generated reasoning paths.

```
                  ┌──> Reasoning Path A ──> Ans: 42
                  │
[User Prompt] ────┼──> Reasoning Path B ──> Ans: 42  ──> [Majority Vote] ──> Final: 42
                  │
                  └──> Reasoning Path C ──> Ans: 15
```

### 3.2 Tree-of-Thoughts (ToT)
Tree-of-Thoughts generalizes CoT by considering a tree of intermediate thoughts, where each thought represents a coherent text sequence that serves as an intermediate step toward a solution.
* **Exploration Strategies**: Breadth-First Search (BFS) or Depth-First Search (DFS).
* **State Evaluation**: The model evaluates its own intermediate thoughts (e.g., assigning a score or classification: `Sure`, `Maybe`, `Impossible`).

### 3.3 ReAct (Reasoning + Acting)
ReAct tightly couples **reasoning trace generation** with **action execution** (tool calls to external APIs, databases, or search engines).

```
Loop:
  1. Thought    : Model analyzes current state and determines what information is needed.
  2. Action     : Model emits a structured tool call (e.g., Search[query]).
  3. Observation: Environment executes tool and feeds result back into prompt context.
```

---

## 4. Production Python Implementation

The following Python script demonstrates:
1. **Self-Consistency Voting Engine**: Running multi-path inference and aggregating answers.
2. **ReAct Agent Execution Loop**: Simulating tool execution and iterative reasoning.

```python
import json
import re
from collections import Counter
from typing import List, Dict, Any, Callable

# =====================================================================
# 1. Self-Consistency Voting Engine
# =====================================================================

def mock_llm_call(prompt: str, temperature: float = 0.7) -> str:
    """
    Simulates sampling different reasoning paths from an LLM.
    In production, this calls an API (OpenAI, Anthropic, Ollama, etc.).
    """
    import random
    
    responses = [
        "Step 1: John has 5 apples.\nStep 2: He gives 2 to Mary (5 - 2 = 3).\nStep 3: He buys 4 more (3 + 4 = 7).\nFINAL ANSWER: 7",
        "Step 1: Start with 5 apples.\nStep 2: Subtract 2 apples given to Mary = 3.\nStep 3: Add 4 apples bought = 7.\nFINAL ANSWER: 7",
        "Step 1: Total apples bought = 5 + 4 = 9.\nStep 2: Subtract 2 given away = 7.\nFINAL ANSWER: 7",
        "Step 1: 5 - 2 = 3 apples.\nStep 2: He dropped one so 3 - 1 = 2.\nStep 3: Add 4 = 6.\nFINAL ANSWER: 6"  # Flawed path
    ]
    return random.choice(responses)

def extract_final_answer(cot_output: str) -> str:
    """Extracts answer value following 'FINAL ANSWER:' delimiter."""
    match = re.search(r"FINAL ANSWER:\s*(.+)$", cot_output, re.MULTILINE)
    if match:
        return match.group(1).strip()
    return cot_output.strip()

def run_self_consistency(prompt: str, num_samples: int = 5) -> Dict[str, Any]:
    """Runs Self-Consistency by sampling N paths and running majority voting."""
    results = []
    answers = []
    
    for i in range(num_samples):
        path = mock_llm_call(prompt, temperature=0.7)
        ans = extract_final_answer(path)
        results.append({"path_id": i + 1, "reasoning": path, "answer": ans})
        answers.append(ans)
        
    # Majority Vote Computation
    vote_counts = Counter(answers)
    majority_answer, count = vote_counts.most_common(1)[0]
    
    return {
        "final_selected_answer": majority_answer,
        "confidence": count / num_samples,
        "all_sampled_paths": results,
        "vote_distribution": dict(vote_counts)
    }

# =====================================================================
# 2. ReAct Agent Engine
# =====================================================================

class ReActAgent:
    def __init__(self, tools: Dict[str, Callable[[str], str]]):
        self.tools = tools
        
    def run(self, user_query: str, max_steps: int = 3) -> str:
        context = f"Question: {user_query}\nSolve using Thought, Action, and Observation steps.\n"
        
        for step in range(1, max_steps + 1):
            print(f"\n--- [ReAct Step {step}] ---")
            
            # Simulate Model Generation
            if step == 1:
                llm_output = (
                    "Thought: I need to check the current weather in Tokyo.\n"
                    "Action: get_weather['Tokyo']"
                )
            elif step == 2:
                llm_output = (
                    "Thought: The weather is 18°C with light rain. I have sufficient info to answer.\n"
                    "Final Answer: The current weather in Tokyo is 18°C with light rain."
                )
            else:
                break
                
            print(llm_output)
            context += f"\n{llm_output}"
            
            # Check for Final Answer
            if "Final Answer:" in llm_output:
                return llm_output.split("Final Answer:")[1].strip()
                
            # Parse and Execute Action
            action_match = re.search(r"Action:\s*(\w+)\['(.+)'\]", llm_output)
            if action_match:
                tool_name, tool_arg = action_match.groups()
                if tool_name in self.tools:
                    obs = self.tools[tool_name](tool_arg)
                    obs_str = f"Observation: {obs}"
                    print(obs_str)
                    context += f"\n{obs_str}"
                else:
                    context += f"\nObservation: Tool '{tool_name}' not found."
                    
        return "Failed to resolve within max steps."

# Helper Tool Function
def mock_get_weather(city: str) -> str:
    weather_data = {"Tokyo": "18°C, Light Rain", "New York": "22°C, Sunny"}
    return weather_data.get(city, "City not found")

# =====================================================================
# Execution Sandbox
# =====================================================================
if __name__ == "__main__":
    print("==================================================")
    print("1. RUNNING SELF-CONSISTENCY DEMO")
    print("==================================================")
    sc_result = run_self_consistency("John has 5 apples, gives 2 to Mary, buys 4 more. How many?", num_samples=5)
    print(f"Majority Answer: {sc_result['final_selected_answer']}")
    print(f"Confidence Score: {sc_result['confidence'] * 100:.1f}%")
    print(f"Vote Distribution: {sc_result['vote_distribution']}")

    print("\n==================================================")
    print("2. RUNNING REACT AGENT DEMO")
    print("==================================================")
    tools = {"get_weather": mock_get_weather}
    agent = ReActAgent(tools=tools)
    final_ans = agent.run("What is the weather in Tokyo?")
    print(f"\nFinal Agent Output: {final_ans}")
```

---

## 5. Architectural & Behavioral Tradeoffs

| Metric / Dimension | Standard Direct Prompting | Zero-Shot CoT | Few-Shot CoT | Self-Consistency (CoT-SC) | Tree-of-Thoughts (ToT) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Inference Latency** | Lowest ($O(1)$) | Low | Moderate | High ($N \times \text{Latency}$) | Extremely High |
| **Token Cost** | Lowest | Low-Moderate | Moderate | High | Highest |
| **Logic Accuracy** | Poor on multi-step | Moderate | High | Very High | Maximum |
| **Implementation Complexity** | Minimal | Minimal | Low | Moderate (Aggregator needed) | High (Search algorithms) |
| **Deterministic Reliability** | Low | Low-Moderate | Moderate-High | High | Very High |

---

## 6. Interview & Assessment Questions

### Q1: Why does adding intermediate reasoning tokens improve accuracy in autoregressive models?
**Answer**: Autoregressive transformers perform a fixed amount of computation per output token. Forcing a direct answer limits computation to a single forward pass. Generating reasoning tokens increases the total number of forward passes, providing additional compute steps and storing intermediate context within the self-attention mechanism.

### Q2: What is the primary operational trade-off of Self-Consistency (CoT-SC)?
**Answer**: The main tradeoff is increased latency and token cost. Sampling $N$ reasoning paths increases token consumption and computation costs by $N\times$. However, it substantially improves task accuracy by eliminating errors caused by greedy decoding along a single flawed reasoning path.

### Q3: How does ReAct prevent hallucinations compared to pure Chain-of-Thought?
**Answer**: Pure CoT relies exclusively on parametric memory, which can lead to plausible but incorrect hallucinations on factual or dynamic queries. ReAct grounds reasoning steps by executing actions against external tools (e.g., search engines or APIs) and incorporating the real-world observations back into context.
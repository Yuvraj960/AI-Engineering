# Module 10: Object-Oriented Programming (OOPs)

Welcome to **Module 10: Object-Oriented Programming (OOPs)**. Object-Oriented Programming is the foundational paradigm for designing complex, scalable software systems. In the context of **AI Engineering**, OOP is indispensable. Modern Deep Learning frameworks like **PyTorch** (`torch.nn.Module`), **TensorFlow/Keras** (`tf.keras.Model`), data handling libraries like **Pandas/Polars**, and AI agent orchestration systems (e.g., **LangChain**, **LlamaIndex**, **AutoGEN**) heavily rely on class-based architectures, custom magic methods, dynamic attribute evaluation, and abstract blueprints.

This module delivers a deep dive into Python’s object model, covering encapsulation, inheritance, polymorphism, abstraction (`abc`), magic methods, operator overloading, and object lifecycle management.

---

## 🎯 Learning Objectives

By completing this module, you will be able to:

1. **Describe** how the 4 pillars of OOP (Encapsulation, Inheritance, Polymorphism, Abstraction) structure AI/ML pipelines and modular software architectures.
2. **Explain** Python’s underlying object creation, namespace management, and method-binding semantics (`self`, `cls`).
3. **Apply** encapsulation using public, protected (`_`), and private (`__`) name-mangling attributes alongside managed properties (`@property`).
4. **Implement** single, multi-level, hierarchical, and multiple inheritance patterns using `super()` and understanding Method Resolution Order (MRO).
5. **Analyze** and override magic (dunder) methods to customize object behavior, representation (`__repr__`, `__str__`), sizing (`__len__`), item access (`__getitem__`), and callable execution (`__call__`).
6. **Construct** custom mathematical objects (such as Tensors or Vectors) by overloading standard arithmetic (`+`, `-`, `*`) and comparison operators.
7. **Evaluate** abstract base classes using Python’s `abc` module to design standardized interfaces for AI model wrappers, evaluation metrics, and data loaders.

---

## 🧠 Core Topics & Deep-Dive Concepts

### 1. Classes, Objects, and Python's Object Model

In Python, **everything is an object**—including integers, functions, and classes themselves. A **Class** serves as a blueprint that defines the structure and behavior of its **Instances (Objects)**.

#### Instance Variables vs. Class Variables

* **Instance Variables**: Defined inside methods (typically `__init__`) using `self.variable_name`. Unique to each instance.
* **Class Variables**: Declared directly within the class body. Shared across all instances of the class.

```python
class NeuralNetworkConfig:
    # Class Variable (Shared across all model instances)
    FRAMEWORK = "PyTorch"
    DEFAULT_LEARNING_RATE = 0.001

    def __init__(self, model_name: str, hidden_layers: list[int]):
        # Instance Variables (Unique per model instance)
        self.model_name = model_name
        self.hidden_layers = hidden_layers
        self.trained_epochs = 0

# Instantiation
config_a = NeuralNetworkConfig("ResNet50", [512, 256])
config_b = NeuralNetworkConfig("BERT-Base", [768, 768, 768])

print(config_a.FRAMEWORK)  # Output: PyTorch
print(config_b.model_name) # Output: BERT-Base
```

#### Method Types: Instance, Class (`@classmethod`), and Static (`@staticmethod`)

1. **Instance Methods**: Take `self` as the first argument; access and modify instance state.
2. **Class Methods (`@classmethod`)**: Take `cls` as the first argument; access or modify class-level state. Frequently used as **factory constructors**.
3. **Static Methods (`@staticmethod`)**: Do not receive `self` or `cls`; behave like utility functions isolated within the class namespace.

```python
class ModelLoader:
    def __init__(self, model_path: str):
        self.model_path = model_path

    # Instance method
    def load_weights(self):
        return f"Loading weights from {self.model_path}"

    # Class method as alternative factory constructor
    @classmethod
    from_pretrained(cls, pretrained_name: str):
        path = f"https://huggingface.co/models/{pretrained_name}.bin"
        return cls(model_path=path)

    # Static method
    @staticmethod
    def validate_path(path: str) -> bool:
        return path.endswith(".bin") or path.endswith(".pt")

# Usage
model = ModelLoader.from_pretrained("gpt2-medium")
print(model.load_weights())
print(ModelLoader.validate_path("model.pt"))
```

---

### 2. Encapsulation & Data Hiding

Encapsulation restricts direct access to an object's internal state to prevent unintended corruption and enforce validation contracts.

#### Access Modifiers in Python

* **Public**: `attribute` — Accessible from anywhere.
* **Protected**: `_attribute` — Convention indicating "internal use only" (subclasses and class instance).
* **Private**: `__attribute` — Triggers **Name Mangling** (renamed internally to `_ClassName__attribute`) to prevent direct external access and accidental subclass overrides.

```python
class Hyperparameters:
    def __init__(self, learning_rate: float, batch_size: int):
        self.public_label = "Optimization Config"
        self._protected_factor = 0.9          # Protected convention
        self.__learning_rate = learning_rate  # Private (Name-mangled)

    # Property Getter
    @property
    def learning_rate((self) -> float:
        return self.__learning_rate

    # Property Setter with Validation
    @learning_rate.setter
    def learning_rate(self, value: float):
        if value <= 0 or value > 1.0:
            raise ValueError("Learning rate must be between 0.0 and 1.0")
        self.__learning_rate = value

hp = Hyperparameters(0.01, 32)
hp.learning_rate = 0.005  # Managed set via property
print(hp.learning_rate)   # Output: 0.005

# Direct access to private attribute raises AttributeError
# print(hp.__learning_rate) 
# Name mangled access (discouraged in production):
print(hp._Hyperparameters__learning_rate) # Output: 0.005
```

---

### 3. Inheritance, Method Overriding, and MRO

Inheritance allows child classes to derive attributes and behavior from parent classes, promoting code reusability.

#### Method Resolution Order (MRO) and `super()`

When working with multiple inheritance, Python uses the **C3 Linearization Algorithm** to determine the lookup order for methods and attributes. You can inspect a class's lookup order using `ClassName.mro()` or `ClassName.__mro__`.

```python
class BaseComponent:
    def __init__(self, name: str):
        self.name = name

    def execute(self):
        print(f"Executing base component: {self.name}")

class TrainableComponent(BaseComponent):
    def __init__(self, name: str, epochs: int):
        # Call parent constructor cleanly
        super().__init__(name)
        self.epochs = epochs

    # Method Overriding
    def execute(self):
        super().execute() # Execute parent logic
        print(f"Training for {self.epochs} epochs...")

component = TrainableComponent("LLM Finetuner", 5)
component.execute()
print(TrainableComponent.mro())
```

---

### 4. Abstraction using Abstract Base Classes (`abc`)

Abstraction hides internal implementation complexity while defining mandatory structural interfaces. In AI frameworks, abstraction is used to create base abstractions for **Samplers**, **Loss Functions**, **Optimizers**, and **Vector Store Connectors**.

```python
from abc import ABC, abstractmethod

class BaseLLMClient(ABC):
    """Abstract Base Class for LLM Provider API wrappers."""

    def __init__(self, model_name: str, api_key: str):
        self.model_name = model_name
        self.api_key = api_key

    @abstractmethod
    def generate(self, prompt: str, temperature: float = 0.7) -> str:
        """Mandatory abstract method to generate text responses."""
        pass

    @abstractmethod
    def embed(self, text: str) -> list[float]:
        """Mandatory abstract method to compute text embeddings."""
        pass

class OpenAIClient(BaseLLMClient):
    def generate(self, prompt: str, temperature: float = 0.7) -> str:
        return f"[OpenAI {self.model_name}] Response to: {prompt}"

    def embed(self, text: str) -> list[float]:
        return [0.012, -0.043, 0.891, 0.112]

# Direct instantiation of Abstract Class raises TypeError:
# client = BaseLLMClient("base", "key") # TypeError: Can't instantiate abstract class BaseLLMClient...

client = OpenAIClient("gpt-4o", "sk-proj-xxx")
print(client.generate("Explain OOP in Python"))
```

---

### 5. Magic (Dunder) Methods and Operator Overloading

Magic methods (prefixed and suffixed with double underscores `__`) allow custom objects to interact seamlessly with Python's built-in operators, functions, and syntaxes (`+`, `len()`, `[]`, `()`, `str()`).

| Magic Method | Built-in Trigger / Operation | AI Engineering Relevance |
| :--- | :--- | :--- |
| `__init__(self, ...)` | Instance initialization | Initializing layer weights, model specs |
| `__repr__(self)` | Developer string representation | Clear debugging of model architecture |
| `__str__(self)` | Readable user string representation | Displaying formatted model summaries |
| `__len__(self)` | `len(obj)` | Querying Dataset length (`DataLoader`) |
| `__getitem__(self, idx)` | Indexing `obj[idx]` or slicing | Accessing specific Dataset samples |
| `__call__(self, *args)` | Function execution `obj(*args)` | Forward pass in Neural Networks (`model(x)`) |
| `__add__`, `__sub__`, `__mul__` | Arithmetic operators `+`, `-`, `*` | Custom Tensor operations & loss scaling |
| `__eq__`, `__lt__` | Comparisons `==`, `<` | Comparing metrics/checkpoints |

#### Deep Learning Analogy: PyTorch's `__call__` and `__getitem__`

```python
class SimpleVector:
    """Custom Vector object demonstrating operator overloading."""
    def __init__(self, components: list[float]):
        self.components = components

    def __repr__(self) -> str:
        return f"SimpleVector({self.components})"

    def __len__(self) -> int:
        return len(self.components)

    def __getitem__(self, idx: int) -> float:
        return self.components[idx]

    # Operator Overloading for Vector Addition (+)
    def __add__(self, other: 'SimpleVector') -> 'SimpleVector':
        if len(self) != len(other):
            raise ValueError("Vectors must be of equal length for addition.")
        combined = [a + b for a, b in zip(self.components, other.components)]
        return SimpleVector(combined)

    # Callable behavior (like PyTorch nn.Module forward pass)
    def __call__(self, scale_factor: float) -> 'SimpleVector':
        return SimpleVector([c * scale_factor for c in self.components])

v1 = SimpleVector([1.0, 2.0, 3.0])
v2 = SimpleVector([4.0, 5.0, 6.0])

print(f"Vector Representation: {v1}")   # __repr__
print(f"Vector Length: {len(v1)}")         # __len__
print(f"Element Access v1[0]: {v1[0]}")     # __getitem__
print(f"Vector Addition v1 + v2: {v1 + v2}") # __add__
print(f"Callable Scaling v1(2.5): {v1(2.5)}") # __call__
```

---

### 6. Object Lifecycle: Creation (`__new__`) vs Initialization (`__init__`) vs Cleanup (`__del__`)

* `__new__(cls, *args, **kwargs)`: Responsible for **creating and returning a new raw memory instance** of the class. It is a static method (implicitly).
* `__init__(self, *args, **kwargs)`: Responsible for **initializing** the created instance attributes.
* `__del__(self)`: Destructor called when an object's reference count reaches 0 or during garbage collection. (Used sparingly; context managers are preferred for explicit resource cleanup).

```python
class SingletonModelRegistry:
    """Demonstrating __new__ to enforce a Single Instance (Singleton) for Model Registry."""
    _instance = None

    def __new__(cls, *args, **kwargs):
        if cls._instance is None:
            print("Allocating memory for Singleton Registry...")
            cls._instance = super().__new__(cls)
        return cls._instance

    def __init__(self):
        # Prevent re-initialization if already exists
        if not hasattr(self, 'initialized'):
            self.registry = {}
            self.initialized = True

reg1 = SingletonModelRegistry()
reg2 = SingletonModelRegistry()

print(reg1 is reg2) # Output: True (Exact same memory reference)
```

---

## 🛠️ Practical Hands-On Project: AI Pipeline Abstraction & Neural Layer Emulator

This end-to-end production-grade script simulates a **Modular AI Neural Network Framework** (similar to PyTorch). It incorporates abstract base classes, inheritance, property encapsulation, operator overloading for loss functions, and dataset item access via magic methods.

```python
# filename: ai_framework_emulator.py

from abc import ABC, abstractmethod
import math
import random
from typing import List, Union

# ==========================================
# 1. Abstract Base Class for Model Layers
# ==========================================

class Layer(ABC):
    """Abstract Base Class defining the contract for all neural network layers."""
    
    def __init__(self, name: str):
        self.name = name
        self._trainable = True

    @property
    def trainable(self) -> bool:
        return self._trainable

    @trainable.setter
    def trainable(self, status: bool):
        self._trainable = status

    @abstractmethod
    def forward(self, inputs: List[float]) -> List[float]:
        """Compute the layer forward pass."""
        pass

    @abstractmethod
    def get_parameter_count(self) -> int:
        """Return total trainable parameters."""
        pass

    def __call__(self, inputs: List[float]) -> List[float]:
        """Allows calling the layer instance directly like a function: layer(x)"""
        return self.forward(inputs)


# ==========================================
# 2. Dense (Fully Connected) Layer Implementation
# ==========================================

class DenseLayer(Layer):
    """Dense Neural Network Layer with linear transformation and ReLU activation."""
    
    def __init__(self, input_dim: int, output_dim: int, name: str = "Dense"):
        super().__init__(name)
        self.input_dim = input_dim
        self.output_dim = output_dim
        # Random weight matrix initialization (output_dim x input_dim)
        self.weights = [[random.uniform(-0.5, 0.5) for _ in range(input_dim)] for _ in range(output_dim)]
        self.biases = [0.0] * output_dim

    def forward(self, inputs: List[float]) -> List[float]:
        if len(inputs) != self.input_dim:
            raise ValueError(f"Input size mismatch. Expected {self.input_dim}, got {len(inputs)}")

        outputs = []
        for i in range(self.output_dim):
            # Compute dot product: W * X + b
            linear_sum = sum(w * x for w, x in zip(self.weights[i], inputs)) + self.biases[i]
            # Apply ReLU Activation: max(0, x)
            activated = max(0.0, linear_sum)
            outputs.append(activated)
        return outputs

    def get_parameter_count(self) -> int:
        return (self.input_dim * self.output_dim) + self.output_dim

    def __repr__(self) -> str:
        return f"DenseLayer(in={self.input_dim}, out={self.output_dim}, params={self.get_parameter_count()})"


# ==========================================
# 3. Custom Evaluation Metric with Operator Overloading
# ==========================================

class LossMetric:
    """Metric container supporting arithmetic operations to aggregate batch losses."""
    
    def __init__(self, value: float, batch_size: int = 1):
        self.value = value
        self.batch_size = batch_size

    def __add__(self, other: 'LossMetric') -> 'LossMetric':
        total_batch = self.batch_size + other.batch_size
        weighted_loss = ((self.value * self.batch_size) + (other.value * other.batch_size)) / total_batch
        return LossMetric(weighted_loss, total_batch)

    def __repr__(self) -> str:
        return f"LossMetric(average_loss={self.value:.4f}, total_samples={self.batch_size})"


# ==========================================
# 4. Custom Dataset Class with Indexing Magic Methods
# ==========================================

class AIDataset:
    """Dataset class emulating PyTorch Dataset (__len__ and __getitem__)."""
    
    def __init__(self, features: List[List[float]], labels: List[float]):
        if len(features) != len(labels):
            raise ValueError("Features and labels must have identical sample counts.")
        self._features = features
        self._labels = labels

    def __len__(self) -> int:
        return len(self._features)

    def __getitem__(self, idx: int) -> dict:
        return {
            "feature": self._features[idx],
            "label": self._labels[idx]
        }


# ==========================================
# 5. Execution Pipeline
# ==========================================

if __name__ == "__main__":
    random.seed(42)

    print("=== 1. Initializing AIDataset ===")
    mock_features = [[1.2, 0.5, -0.8], [0.1, 2.3, 0.9], [-1.1, -0.4, 1.5]]
    mock_labels = [1.0, 0.0, 1.0]
    dataset = AIDataset(mock_features, mock_labels)

    print(f"Dataset Size (__len__): {len(dataset)}")
    print(f"First Sample (__getitem__): {dataset[0]}\n")

    print("=== 2. Building Sequential Neural Architecture ===")
    layer1 = DenseLayer(input_dim=3, output_dim=4, name="Hidden_1")
    layer2 = DenseLayer(input_dim=4, output_dim=1, name="Output_Layer")

    print(f"Layer 1 Specs: {layer1}")
    print(f"Layer 2 Specs: {layer2}")
    
    pipeline: List[Layer] = [layer1, layer2]

    print("\n=== 3. Executing Forward Pass ===")
    sample_input = dataset[0]["feature"]
    x = sample_input
    print(f"Initial Input Tensor: {x}")
    
    for layer in pipeline:
        x = layer(x)  # Triggers __call__ -> forward()
        print(f"Output after {layer.name}: {x}")

    print("\n=== 4. Aggregating Loss Metrics via Operator Overloading ===")
    batch1_loss = LossMetric(value=0.45, batch_size=32)
    batch2_loss = LossMetric(value=0.25, batch_size=32)
    
    total_loss = batch1_loss + batch2_loss  # Triggers __add__
    print(f"Batch 1: {batch1_loss}")
    print(f"Batch 2: {batch2_loss}")
    print(f"Aggregated Metric (+ operator): {total_loss}")
```

---

## ❓ Technical Interview & Conceptual Exam Questions

1. **How does Python's `super()` method work during Multiple Inheritance, and what is MRO?**
   * **Answer**: `super()` doesn't simply call the immediate parent class; it follows the class's **Method Resolution Order (MRO)** calculated via the C3 Linearization algorithm. In multiple inheritance hierarchy, `super()` delegates calls to the next sibling or parent in the MRO list, avoiding duplicate method calls (such as double initialization in diamond inheritance scenarios).

2. **Why does PyTorch mandate inheriting from `torch.nn.Module` and implementing `forward()` instead of overriding `__call__` directly?**
   * **Answer**: `torch.nn.Module` overrides the `__call__` magic method internally. When a user calls `model(inputs)`, `__call__` handles internal framework tasks—such as executing forward pre-hooks, registration of gradients, tracing, and forward post-hooks—before invoking the custom user-defined `forward()` method. Overriding `__call__` directly would bypass these vital AI pipeline mechanics.

3. **What is the difference between `__repr__` and `__str__`, and which one should be prioritized in AI library development?**
   * **Answer**: `__str__` provides a friendly, readable string representation aimed at end-users (triggered by `print()` or `str()`). `__repr__` aims to provide an unambiguous representation intended for developers and debugging (triggered by interactive shells or `repr()`). In AI libraries, prioritizing `__repr__` is essential because if `__str__` is omitted, Python defaults to `__repr__`, ensuring clear model topology output during debugging.

4. **What are Abstract Base Classes (`ABC`), and how do they enforce API design contracts in LLM frameworks?**
   * **Answer**: Abstract Base Classes derived from `abc.ABC` allow developers to define interface contracts containing methods decorated with `@abstractmethod`. If a subclass fails to implement any `@abstractmethod`, Python prevents instantiation at runtime with a `TypeError`. Frameworks like LangChain use ABCs to enforce that custom LLM wrappers or Vector DB adapters implement mandatory methods like `generate()` or `add_documents()`.

---

## 📚 Curated Resources & References

### Official Documentation & Technical Specifications

* [Python 3 Documentation – Tutorial: Classes](https://docs.python.org/3/tutorial/classes.html)
* [Python 3 Documentation – `abc` Module (Abstract Base Classes)](https://docs.python.org/3/library/abc.html)
* [Python Data Model – Special Method Names (Dunder Methods)](https://docs.python.org/3/reference/datamodel.html#special-method-names)

### AI Engineering Context & Design Patterns

* [PyTorch Documentation – `torch.nn.Module` Design Architecture](https://pytorch.org/docs/stable/generated/torch.nn.Module.html)
* [Real Python – Comprehensive Guide to OOP in Python 3](https://realpython.com/python3-object-oriented-programming/)
* [Real Python – Operator Overloading and Magic Methods](https://realpython.com/operator-function-overloading/)
# Module 06: Key-Value Pairs - Dictionaries

Welcome to the **Key-Value Pairs - Dictionaries** module of the AI Engineering track. A Python dictionary (`dict`) is an associative, mutable collection that maps unique keys to arbitrary values. Dictionaries provide fast $O(1)$ average-time lookup and insertion, making them an essential tool for organizing structured records, system settings, and JSON-like hierarchical datasets.

## 🎯 Learning Objectives

By completing this module, you will be able to:

1. **Describe** the core structure and advantages of Python dictionaries over sequential collections like lists or tuples.

2. **Explain** the rules governing keys and values, including key uniqueness and immutability (hashability) requirements.

3. **Apply** essential dictionary operations to create, update, retrieve, and delete key-value pairs safely.

4. **Analyze & Utilize** built-in dictionary view methods (`keys()`, `values()`, `items()`, `get()`, `pop()`, `clear()`) for efficient iteration and data retrieval.

5. **Design & Implement** nested dictionary structures to model complex, hierarchical systems such as application configurations and inventory catalogs.

## 🧠 Core Topics & In-Depth Notes

### 1. Structure, Syntax & Key Immutability Rules

A Python **dictionary** is defined using curly braces `{}` with `key: value` pairs separated by commas, or initialized via the `dict()` constructor.

```python
# Standard literal initialization
model_config = {
    "architecture": "Transformer",
    "layers": 12,
    "learning_rate": 0.001,
    "is_trained": True
}

# Constructor initialization
user_profile = dict(username="alex_dev", role="Admin", status="Active")
```

#### Key Rules vs. Value Rules

* **Keys MUST be Hashable (Immutable):** Keys can be strings, numbers, or tuples containing immutable objects. Mutable objects like lists or dictionaries **cannot** be used as keys.
* **Keys MUST be Unique:** Duplicate keys are not allowed. If duplicate keys are declared, the last value overwrites previous assignments.
* **Values are Unrestricted:** Values can be of any data type (e.g., lists, nested dicts, functions, `None`) and can contain duplicates.
* **Insertion Order Preservation:** Since Python 3.7+, standard dictionaries preserve key insertion order.

```python
# Valid dictionary keys
valid_dict = {
    "string_key": 100,
    42: "numeric_key",
    (1, 2): "tuple_key"
}

# Invalid dictionary key (Raises TypeError: unhashable type: 'list')
# invalid_dict = {[1, 2]: "list_key"}
```

---

### 2. Accessing, Inserting, and Modifying Data

Elements can be accessed using direct key indexing `dict[key]` or safely via the `get()` method.

```python
config = {"host": "localhost", "port": 8080}

# 1. Direct Indexing (Fast, but raises KeyError if missing)
port = config["port"]  # 8080
# timeout = config["timeout"]  # Raises KeyError!

# 2. Safe Retrieval with .get()
timeout = config.get("timeout", 30)  # Returns default 30 if missing
ssl_enabled = config.get("ssl")      # Returns None if default omitted

# 3. Adding/Updating Pairs
config["port"] = 9090                # Updates existing key
config["timeout"] = 60               # Inserts new key-value pair

# 4. Bulk Updates using .update()
config.update({"timeout": 120, "ssl": True, "environment": "Production"})
```

---

### 3. Key Dictionary Methods & Deletion Mechanics

| Method / Operator | Description | Behavior on Missing Key |
| :--- | :--- | :--- |
| `get(key, default)` | Retrieves value for `key`. | Returns `default` (or `None`) without throwing an error. |
| `pop(key, default)` | Removes `key` and returns its value. | Returns `default` if provided; raises `KeyError` if omitted. |
| `popitem()` | Removes and returns the last inserted `(key, value)` tuple. | Raises `KeyError` if dictionary is empty. |
| `del dict[key]` | Statement to delete `key` and its value. | Raises `KeyError`. |
| `clear()` | Empties all key-value pairs from the dictionary. | N/A (Leaves dictionary empty). |

```python
inventory = {"apples": 50, "bananas": 30, "oranges": 20}

# Removing items safely
banana_count = inventory.pop("bananas", 0)  # Removes 'bananas' and returns 30
missing_count = inventory.pop("grapes", 0)  # Returns 0 safely

# Using del statement
del inventory["oranges"]

print(inventory)  # {'apples': 50}
```

---

### 4. Iterating over Dictionaries with View Objects

Python provides dynamic **view objects** (`keys()`, `values()`, and `items()`) that reflect live updates to the dictionary.

```python
metrics = {"epoch_1": 0.85, "epoch_2": 0.89, "epoch_3": 0.93}

# Iterating over Keys (Default iteration behavior)
print("--- Keys ---")
for epoch in metrics:
    print(epoch)

# Iterating over Values
print("--- Values ---")
for accuracy in metrics.values():
    print(f"Accuracy: {accuracy}")

# Iterating over Key-Value Pairs (Tuple Unpacking)
print("--- Items ---")
for epoch, accuracy in metrics.items():
    print(f"{epoch} -> {accuracy * 100:.1f}%")
```

---

### 5. Nested Dictionaries & Hierarchical Data Modeling

Nested dictionaries store structured datasets, such as multi-tenant user accounts, catalog hierarchies, or JSON API payloads.

```python
system_catalog = {
    "server_alpha": {
        "ip": "10.0.0.1",
        "roles": ["web", "auth"],
        "specs": {"cpu_cores": 8, "ram_gb": 32}
    },
    "server_beta": {
        "ip": "10.0.0.2",
        "roles": ["database"],
        "specs": {"cpu_cores": 16, "ram_gb": 64}
    }
}

# Accessing nested fields
beta_ram = system_catalog["server_beta"]["specs"]["ram_gb"]
print(f"Server Beta RAM: {beta_ram} GB")  # 64 GB

# Modifying nested field
system_catalog["server_alpha"]["roles"].append("cache")
```

---

## 🛠️ Practical Hands-On Project: Product Inventory & Word Frequency Analyzer

This practical application demonstrates dictionary updates, nested structure management, dynamic aggregation, and safe key lookups.

```python
# filename: inventory_and_text_analyzer.py

def analyze_word_frequency(text: str) -> dict[str, int]:
    """Calculates occurrence frequencies for each word in a string."""
    words = text.lower().replace(".", "").replace(",", "").split()
    frequency_table = {}
    
    for word in words:
        # Increment count safely using .get()
        frequency_table[word] = frequency_table.get(word, 0) + 1
        
    return frequency_table


def manage_inventory_system():
    """Simulates inventory tracking with nested store data."""
    store_inventory = {
        "E101": {"name": "Wireless Mouse", "price": 25.99, "stock": 45},
        "E102": {"name": "Mechanical Keyboard", "price": 89.99, "stock": 15},
        "E103": {"name": "HD Monitor", "price": 179.50, "stock": 8}
    }

    print("=== Initial Inventory Catalog ===")
    for item_id, details in store_inventory.items():
        print(f"ID: {item_id} | Name: {details['name']} | Stock: {details['stock']} | Price: ${details['price']:.2f}")

    # Process batch stock updates
    stock_updates = [("E101", -5), ("E103", 10), ("E104", 20)]  # E104 is a new item

    print("\n=== Processing Batch Updates ===")
    for item_id, qty_change in stock_updates:
        if item_id in store_inventory:
            store_inventory[item_id]["stock"] += qty_change
            print(f"Updated {item_id} stock by {qty_change}. New total: {store_inventory[item_id]['stock']}")
        else:
            print(f"Warning: Item {item_id} not found in catalog. Skipping update.")

    # Calculate total warehouse stock valuation
    total_value = sum(item["price"] * item["stock"] for item in store_inventory.values())
    print(f"\nTotal Warehouse Valuation: ${total_value:,.2f}")


if __name__ == "__main__":
    print("--- Part 1: Text Frequency Analysis ---")
    sample_text = "Python is powerful. Python dictionaries are fast and flexible."
    freq_results = analyze_word_frequency(sample_text)
    print("Word Frequencies:", freq_results)

    print("\n--- Part 2: Inventory Management System ---")
    manage_inventory_system()
```

---

## ❓ Frequently Asked Exam & Technical Interview Questions

1. **Why are dictionaries faster for lookups compared to lists?**
   * *Answer:* Dictionaries use an internal **hash table** structure that computes the hash value of a key to locate its memory address directly in $O(1)$ average time. Lists require scanning elements sequentially in $O(n)$ time.

2. **What happens when you call `dict.get('missing_key')` vs `dict['missing_key']`?**
   * *Answer:* `dict['missing_key']` raises a `KeyError` exception and interrupts program execution if unhandled. `dict.get('missing_key')` executes safely by returning `None` (or a custom default argument if specified).

3. **Can a list or dictionary be used as a key in a Python dictionary? Why or why not?**
   * *Answer:* No. Dictionary keys must be **hashable** (immutable). Lists and dictionaries are mutable, meaning their contents (and thus their hash values) can change, which breaks hash table integrity. Attempting to use them raises a `TypeError: unhashable type`.

---

## 📚 Curated Resources & References

### Official Documentation & Tutorials
* [Python Official Documentation – Dictionaries](https://docs.python.org/3/tutorial/datastructures.html#dictionaries) – Official language guide on mapping types.
* [Real Python: Practical Guide to Python Dictionaries](https://realpython.com/python-dicts/) – Detailed tutorial covering methods, patterns, and practical implementation tips.

### Interactive Practice Platforms
* [GeeksforGeeks: Python Dictionary](https://www.geeksforgeeks.org/python-dictionary/) – Comprehensive summaries, examples, and exercise tasks.
* [W3Schools Python Dictionaries](https://www.w3schools.com/python/python_dictionaries.asp) – Interactive browser-based exercise playground.
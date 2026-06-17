# Evaluation Spec — Pod Classifier

Complete this spec **before** writing any code for Milestone 3.

Use Plan or Ask mode to think through each blank field. When you're done,
your answers here become the blueprint for `compute_accuracy()` and
`compute_per_class_accuracy()` in `evaluate.py`.

---

## Background: What is evaluation?

After building a classifier, we need to know how well it works. Evaluation answers:
- **Overall:** What fraction of episodes did we classify correctly?
- **Per-class:** Are we better at some labels than others?

Both functions take the same inputs: a list of predicted labels and a list of
ground-truth labels, in the same order.

---

## compute_accuracy(predictions, ground_truth)

### What it does
Returns the fraction of predictions that exactly match the ground truth.

### Inputs

| Parameter | Type | Description |
|---|---|---|
| `predictions` | `list[str]` | Labels predicted by `classify_episode()`, one per episode. |
| `ground_truth` | `list[str]` | The correct labels, in the same order as `predictions`. |

### Output

| Return value | Type | Description |
|---|---|---|
| accuracy | `float` | A value between 0.0 and 1.0. |

---

### Spec fields — fill these in before writing code

**Formula:**

```
accuracy = number of correct predictions / total number of predictions

A prediction is "correct" when predictions[i] == ground_truth[i] (exact string match).
Divide by the total length of the list (i.e., the number of episodes evaluated).
```

---

**Step-by-step logic:**

```
1. If both lists are empty, return 0.0.
2. Count correct: iterate over zip(predictions, ground_truth), count pairs
   where the two values are equal.
3. Divide the count by len(ground_truth).
4. Return the result as a float.
```

---

**Edge case — what if both lists are empty?**

```
Return 0.0. There are no predictions to evaluate, so accuracy is undefined.
Returning 0.0 is a safe default that won't crash callers that expect a float.
```

---

**Worked example:**

```
predictions  = ["interview", "solo", "panel", "interview"]
ground_truth = ["interview", "solo", "solo",  "narrative"]

pos 0: "interview" == "interview" ✓
pos 1: "solo"      == "solo"      ✓
pos 2: "panel"     != "solo"      ✗
pos 3: "interview" != "narrative" ✗

correct = 2, total = 4
compute_accuracy() = 2 / 4 = 0.5
```

---

## compute_per_class_accuracy(predictions, ground_truth)

### What it does
Returns accuracy broken down by each label. For each label in `VALID_LABELS`,
reports how many episodes with that ground-truth label were classified correctly.

### Inputs

| Parameter | Type | Description |
|---|---|---|
| `predictions` | `list[str]` | Labels predicted by `classify_episode()`. |
| `ground_truth` | `list[str]` | Correct labels, in the same order. |

### Output

A `dict` keyed by label. Each value is a dict with three keys:

```python
{
    "interview": {"correct": int, "total": int, "accuracy": float},
    "solo":      {"correct": int, "total": int, "accuracy": float},
    "panel":     {"correct": int, "total": int, "accuracy": float},
    "narrative": {"correct": int, "total": int, "accuracy": float},
}
```

---

### Spec fields — fill these in before writing code

**What does "correct" mean for a given class?**

```
An episode counts as correct for class X when:
  ground_truth[i] == X   AND   predictions[i] == X

Both conditions must be true. The episode must actually belong to class X
(ground truth), and the classifier must have predicted X for it.
```

---

**What does "total" mean for a given class?**

```
"total" for class X is the number of episodes where ground_truth[i] == X,
regardless of what was predicted. It is NOT the total number of all predictions.
It only counts episodes that actually belong to that class.
```

---

**Step-by-step logic:**

```
1. Initialize a dict for each label in VALID_LABELS: {"correct": 0, "total": 0}.
2. Loop over zip(predictions, ground_truth) to get each (predicted, truth) pair.
3. For each pair:
   - Increment total for the ground_truth label (this episode belongs to that class).
   - If predicted == truth, also increment correct for that label.
4. After the loop, compute accuracy for each label:
   - If total > 0: accuracy = correct / total
   - If total == 0: accuracy = 0.0
5. Return the completed dict with all four labels.
```

---

**Edge case — what if a class has no examples in ground_truth (total == 0)?**

```
Set accuracy to 0.0. Dividing by zero is undefined, and 0.0 signals that the
class was not represented in the test set — callers can treat it as "no data"
rather than a meaningful accuracy score.
```

---

**Worked example:**

```
predictions  = ["interview", "interview", "solo", "panel", "panel"]
ground_truth = ["interview", "solo",      "solo", "panel", "narrative"]

Tracing through each pair:
pos 0: truth=interview, pred=interview → interview total+1, correct+1
pos 1: truth=solo,      pred=interview → solo total+1
pos 2: truth=solo,      pred=solo      → solo total+1, correct+1
pos 3: truth=panel,     pred=panel     → panel total+1, correct+1
pos 4: truth=narrative, pred=panel     → narrative total+1

label       correct  total  accuracy
----------  -------  -----  --------
interview     1        1      1.0
solo          1        2      0.5
panel         1        1      1.0
narrative     0        1      0.0
```

---

## Reflection questions (discuss at the checkpoint)

1. Your overall accuracy might be decent even if one class has very low accuracy.
   Why is per-class accuracy a more informative metric than overall accuracy alone?

2. If `panel` episodes consistently get misclassified as `interview`, what does
   that tell you about your training labels or your prompt?

3. You labeled 20 training episodes and evaluated on 20 test episodes (5 per class).
   How might the evaluation results change if you had labeled 100 training episodes?
   What if you had 200 test episodes?

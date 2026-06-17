# Classifier Spec — Pod Classifier

Complete this spec **before** writing any code for Milestone 2.

Use Plan or Ask mode to think through each blank field. When you're done,
your answers here become the blueprint for `build_few_shot_prompt()` and
`classify_episode()` in `classifier.py`.

---

## build_few_shot_prompt(labeled_examples, description)

### What it does
Constructs a prompt string for the LLM that includes the task instructions,
all labeled training examples, and the new episode description to classify.

### Inputs

| Parameter | Type | Description |
|---|---|---|
| `labeled_examples` | `list[dict]` | Each dict has `"title"`, `"description"`, `"label"` (and others). These are the examples you labeled in Milestone 1. |
| `description` | `str` | The episode description to classify. |

### Output

| Return value | Type | Description |
|---|---|---|
| prompt | `str` | A complete prompt string ready to send to the LLM. |

---

### Spec fields — fill these in before writing code

**Task instruction (what should the LLM know about the task?):**

```
You are classifying podcast episodes by their format. Classify the episode
into exactly one of these four labels:

- interview: a conversation between a host and one or more guests
- solo: a single host speaking from memory, experience, or opinion — no guests,
  no assembled external sources
- panel: multiple guests with roughly equal speaking time, often debating or
  discussing a topic together
- narrative: a story assembled from external sources — interviews, archival
  audio, reporting — with a clear narrative arc

Return only the label and your reasoning. Do not explain the taxonomy.
```

---

**How should labeled examples be formatted in the prompt?**

```
Each example should include the episode title, a brief excerpt or the full
description, and the correct label. Separate examples with a blank line or
a delimiter like "---". Include all fields that help the model see why the
label was applied — title and description are both useful; other fields
(like episode ID) are not needed.
```

---

**Example block sketch (write one concrete example):**

```
Title: {title}
Description: {description}
Label: {label}
```

---

**How should the new episode (to be classified) be presented?**

```
Present it in the same format as the labeled examples, but omit the Label
line and replace it with an instruction to classify. For example:

Title: {title}
Description: {description}
Label: ?

Then add a line like: "Classify the episode above. Return your answer in
the format below:" followed by the output format you chose.
```

---

**What output format should you request from the LLM?**

```
Request a two-line structured format:

  Label: <one of: interview, solo, panel, narrative>
  Reasoning: <one sentence explaining why>

Tradeoffs considered:
- JSON would be maximally parseable but models often wrap it in markdown
  code fences, requiring extra cleanup.
- A single label alone is easy to parse but loses the reasoning.
- This two-line format is easy to parse (find the "Label:" line, strip it)
  and keeps reasoning accessible without needing a JSON parser.
```

---

**Edge cases to handle in the prompt:**

```
- Empty labeled_examples: skip the examples block entirely; the prompt still
  works as a zero-shot classifier using just the task instruction.
- Very short description (one sentence): include it as-is — the model can
  still classify it, though with less signal. No special handling needed.
- Description marketing language: the task instruction warns the model to
  classify by structural format, not by how the episode is presented.
```

---

## classify_episode(description, labeled_examples)

### What it does
Classifies a single podcast episode description using the few-shot LLM classifier.
Returns a dict with a label and reasoning.

### Inputs

| Parameter | Type | Description |
|---|---|---|
| `description` | `str` | The episode description to classify. |
| `labeled_examples` | `list[dict]` | Labeled training examples from `load_labeled_examples()`. |

### Output

| Return value | Type | Description |
|---|---|---|
| result | `dict` | Must have keys `"label"` and `"reasoning"`. `"label"` must be one of `VALID_LABELS` or `"unknown"`. |

---

### Spec fields — fill these in before writing code

**Step 1 — Build the prompt:**

```
Call build_few_shot_prompt(labeled_examples, description) and store the
returned string in a variable (e.g., prompt). Pass through both arguments
exactly as received — no modification needed before calling.
```

---

**Step 2 — Send to the LLM:**

```
Call _client.chat.completions.create() with:
  - model: the model name from config (LLM_MODEL)
  - messages: a list with one dict — {"role": "user", "content": prompt}
    (system-design.md shows an optional system message too — either shape works)
  - max_tokens: a reasonable limit (e.g., 200–300) to keep responses concise

Extract the response text from:
  response.choices[0].message.content
```

---

**Step 3 — Parse the response:**

```
Split the response text by newlines. For each line:
- If the line starts with "label:" (case-insensitive after strip()), extract
  the value after the colon, strip whitespace, and lowercase it.
- If the line starts with "reasoning:", extract the value after the colon
  and strip whitespace.

Default label to "unknown" and reasoning to the full raw response text
if no matching lines are found.
```

---

**Step 4 — Validate the label:**

```
After parsing, check: if label not in VALID_LABELS, set label = "unknown".
This catches variants the model might return despite instructions — e.g.,
"Interview" (wrong case), "interviews" (pluralized), or a free-form phrase.
Lowercasing in Step 3 handles simple casing issues; this catches everything else.
```

---

**Step 5 — Handle errors gracefully:**

```
Wrap the entire function body in try/except Exception as e.
What could go wrong:
- Network or API error (timeout, auth failure, rate limit)
- Missing key in response object (unexpected API shape)
- Completely unparseable response (no "Label:" line found → label stays "unknown")

On any exception, return:
  {"label": "unknown", "reasoning": f"Error: {str(e)}"}

This keeps the evaluation loop running across all 20 episodes even if one call fails.
```

---

### Return value structure

```python
{
    "label": str,      # one of VALID_LABELS, or "unknown" if invalid/error
    "reasoning": str,  # brief explanation from the LLM
}
```

---

## Notes on label quality

The classifier is only as good as your labels. If your training examples have
inconsistent or ambiguous labels, the LLM will learn the wrong pattern.

Before implementing the classifier, re-read `data/taxonomy.md` and double-check
any labels you're unsure about. Annotation quality is part of the lab.

---

## Implementation Notes

*Fill this in after implementing and testing both functions.*

**Test: what does the raw LLM response look like for one episode?**

```
Episode tested: [Dr. Priya Nair on Adolescent Mental Health After the Pandemic]
Raw response text: [Label: interview
Reasoning: This episode features a conversation between the host and a single guest, Dr. Priya Nair, discussing her research and expertise on a specific topic.
]
```

**How did you parse the label out of the response?**

```
Iterated over each line of the response text. For each line, called strip()
then checked if it started with "label:" using .lower(). Extracted the substring
after the colon and called .strip().lower() on it. Applied the same prefix check
for "reasoning:". Defaulted label to "unknown" and reasoning to the full raw
text if neither prefix was found.
```

**Did any episodes return `"unknown"`? If so, why?**

```
No. The model returned a valid label from VALID_LABELS for every episode tested.
```

**One thing about the output format that surprised you:**

```
Nothing surprising — the model followed the two-line format instructions precisely,
with no extra explanation, preamble, or markdown wrapping.
```

# Python 3.11 Migration Guide

This document records all findings and changes made to migrate the ASTNN codebase from
Python 3.6/legacy dependencies to Python 3.11-compatible standards.

---

## Environment Baseline

| Item | Original | Updated |
|------|----------|---------|
| Python | 3.6.7 | 3.11+ |
| pandas | 0.20.3 | ≥ 2.0.0 |
| gensim | 3.5.0 | ≥ 4.0.0 |
| scikit-learn | 0.19.1 | ≥ 1.0.0 |
| pytorch | 1.0.0 | ≥ 2.0.0 |
| pycparser | 2.18 | ≥ 2.21 |
| javalang | 0.11.0 | ≥ 0.13.0 |
| numpy | (implicit) | ≥ 1.24.0 |
| tqdm | (implicit) | ≥ 4.64.0 |
| click | (implicit) | ≥ 8.0.0 |

---

## Issues Found and Fixed

### 1. `is` / `is not` Used for Value Comparison (Python 3.8+ SyntaxWarning, error-prone in 3.12)

Python's `is` and `is not` operators test **object identity**, not equality. Using them to
compare integer or string literals has been a `SyntaxWarning` since Python 3.8 and can cause
incorrect behaviour because CPython's small-integer cache is an implementation detail, not a
language guarantee.

**Files changed:**

| File | Line | Before | After |
|------|------|--------|-------|
| `model.py` | 41 | `if node[i][0] is not -1:` | `if node[i][0] != -1:` |
| `model.py` | 47 | `if temp[j][0] is not -1:` | `if temp[j][0] != -1:` |
| `model.py` | 67 | `if i is not -1` | `if i != -1` |
| `prepare_data.py` | 26 | `if name is not 'For':` | `if name != 'For':` |
| `prepare_data.py` | 36 | `elif name is 'Compound':` | `elif name == 'Compound':` |
| `clone/model.py` | 48 | `if temp[j][0] is not -1:` | `if temp[j][0] != -1:` |
| `clone/utils.py` | 69 | `elif name is 'BlockStatement'` | `elif name == 'BlockStatement'` |

---

### 2. gensim 4.0 API Breaking Changes

gensim 4.0 (released 2021) removed and renamed several attributes of `KeyedVectors`:

| Old API | New API | Notes |
|---------|---------|-------|
| `wv.syn0` | `wv.vectors` | Direct access to the embedding matrix |
| `wv.vocab` | `wv.key_to_index` | Mapping from token string to integer index |
| `wv.vocab[t].index` | `wv.key_to_index[t]` | `key_to_index` returns the index directly (int), not a `Vocab` object |
| `Word2Vec(size=N, …)` | `Word2Vec(vector_size=N, …)` | Constructor parameter renamed |

**Files changed:**

- `train.py`: `word2vec.syn0` → `word2vec.vectors` (3 occurrences)
- `pipeline.py`:
  - `Word2Vec(size=size, …)` → `Word2Vec(vector_size=size, …)`
  - `word2vec.vocab` → `word2vec.key_to_index`
  - `word2vec.syn0` → `word2vec.vectors`
  - `vocab[token].index` → `vocab[token]`
- `clone/train.py`: `word2vec.syn0` → `word2vec.vectors` (3 occurrences)
- `clone/pipeline.py`:
  - `Word2Vec(size=size, …)` → `Word2Vec(vector_size=size, …)`
  - `word2vec.vocab` → `word2vec.key_to_index`
  - `word2vec.syn0` → `word2vec.vectors`
  - `vocab[token].index` → `vocab[token]`

---

### 3. pandas 2.0 Breaking Change: `Series.append()` Removed

`DataFrame.append()` and `Series.append()` were deprecated in pandas 1.4 and **removed in
pandas 2.0**. The replacement is `pd.concat()`.

**File changed:**

| File | Before | After |
|------|--------|-------|
| `clone/pipeline.py` line 124 | `pairs['id1'].append(pairs['id2']).unique()` | `pd.concat([pairs['id1'], pairs['id2']]).unique()` |

---

### 4. Deprecated `torch.autograd.Variable` Removed

`torch.autograd.Variable` was deprecated in PyTorch 0.4 (2018). In PyTorch ≥ 1.0 all
`Tensor` objects automatically support autograd; wrapping them in `Variable` is a no-op but
it adds noise and can mask type errors. All `Variable(…)` wrappers have been removed.

**Files changed:** `model.py`, `train.py`, `clone/model.py`, `clone/train.py`

Specifically:
- Removed `from torch.autograd import Variable` imports.
- Replaced `Variable(torch.zeros(…))` with plain `torch.zeros(…)`.
- Replaced `Variable(self.th.LongTensor(…))` with plain `self.th.LongTensor(…)`.
- Replaced `loss_function(output, Variable(labels))` with `loss_function(output, labels)`.

---

### 5. `requirements.txt` Added

A `requirements.txt` file listing all runtime dependencies with minimum versions compatible
with Python 3.11 was added to the repository root.

```
pandas>=2.0.0
gensim>=4.0.0
scikit-learn>=1.0.0
pycparser>=2.21
javalang>=0.13.0
numpy>=1.24.0
tqdm>=4.64.0
click>=8.0.0
torch>=2.0.0
```

Install with:

```bash
pip install -r requirements.txt
```

---

## How to Verify

Run a quick syntax check on all changed Python files:

```bash
python -m py_compile model.py train.py pipeline.py prepare_data.py
python -m py_compile clone/model.py clone/train.py clone/pipeline.py clone/utils.py
```

No output means no syntax errors.

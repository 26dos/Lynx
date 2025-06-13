

# CAR File Content Risk Analysis

## 1. Objective Summary

Identify whether a target CAR file contains high-entropy random data or unstructured junk data, rather than:

- Real uploaded files (e.g., text, images, videos, code, etc.);
- Data with clear semantic or structural features or known file signatures.

---

## 2. Technical Detection Scheme

We evaluate from the following four dimensions:

### 1. Entropy Analysis (Information Entropy)

- **Principle**: Real files tend to be compressible and have lower entropy; random files approach maximum entropy (~8 bits).
- **Steps**:
  - Extract raw data from each block in the CAR file.
  - Calculate Shannon entropy for each or the first few blocks.
  - Threshold: entropy > 7.9 → likely random data.
- **Golang Sample Code**:

```go
func CalculateEntropy(data []byte) float64 {
    freq := make([]float64, 256)
    for _, b := range data {
        freq[b]++
    }
    var entropy float64
    for _, count := range freq {
        if count > 0 {
            p := count / float64(len(data))
            entropy -= p * math.Log2(p)
        }
    }
    return entropy
}
```

---

### 2. File Signature and Magic Number Detection

- **Principle**: Real files typically start with known magic bytes, e.g.:
  - PNG: `89 50 4E 47`
  - PDF: `%PDF`
  - ELF: `7F 45 4C 46`
  - ZIP: `50 4B 03 04`
- **Steps**:
  - Extract the first 4–8 bytes of data from the first few blocks.
  - Match against known file signatures.
  - Matching any common format → indicates real file presence.
- **Tools**: `trid`, `file`, or use built-in magic number libraries.

---

### 3. Duplicate Rate Analysis

- **Principle**: Filler or junk data often contains repetitive blocks, such as:
  - Same block repeated;
  - Multiple blocks with identical content;
  - All-zero blocks.
- **Steps**:
  - Hash each block and track occurrence count.
  - Calculate duplicate ratio = duplicate block count / total blocks.
  - Threshold: > 50% duplicate → high risk.

```go
hashMap := make(map[string]int)
for _, blk := range blocks {
    h := sha256.Sum256(blk)
    hashMap[string(h[:])]++
}
```

---

### 4. CAR DAG Structure Analysis

- **Expected structure of real CAR files**:
  - Deep DAG, many branches and leaf nodes;
  - Not a flat or linear chain.
- **Structure of junk CAR files**:
  - Very shallow (1–2 levels);
  - Blocks directly link to root;
  - No multi-level structure or disordered links.
- **Steps**:
  - Use `go-car` or `go-ipld-prime` to extract DAG structure.
  - Analyze:
    - Child block count per node;
    - Total DAG depth;
    - Branching ratio.

---

## 3. Comprehensive Scoring Strategy

Combine features from each dimension to calculate a trust score:

| Check Dimension     | Metric                            | Weight |
|---------------------|------------------------------------|--------|
| Entropy Analysis     | Entropy > 7.95                     | +40    |
| Duplicate Rate       | Duplicate rate > 50%              | +30    |
| No Magic Signature   | No format detected in first 10 blocks | +15 |
| DAG Anomaly          | DAG depth <= 2                    | +15    |

**Total score > 60 indicates suspicious or filler data.**

---

## 4. Implementation Flow (Illustration)

```
┌───────────────────────────────┐
│  Retrieval client receives CAR │
└──────────────┬────────────────┘
               ▼
┌───────────────────────────────┐
│ Decode CAR and extract blocks │
└──────────────┬────────────────┘
               ▼
┌────────────────────────────────────────┐
│ Analyze each block: entropy + magic + hash │
└────────────────┬───────────────────────┘
                 ▼
     ┌────────────────────────────┐
     │  Generate DAG structure    │
     └────────┬──────────────────┘
              ▼
┌────────────────────────────────────┐
│ Combine scores to assess trust/risk│
└────────────────────────────────────┘
```

---

## 5. Recommended Libraries / Tools

- **CAR Parser in Golang**:  
  [`github.com/ipld/go-car/v2`](https://github.com/ipld/go-car)

- **Entropy Calculation**:  
  Custom or use `github.com/montanaflynn/stats`

- **Magic Number Detection**:  
  `github.com/rakyll/magicmime` or `libmagic` integration

- **DAG Visualization**:  
  `graphviz`, `go-ipld-prime`

---

## 6. Future Expansion

- Integrate ML models: (High entropy + no magic + high repetition → junk)
- Build a trusted CAR feature whitelist
- Maintain a historical database of SPs' real file content ratio

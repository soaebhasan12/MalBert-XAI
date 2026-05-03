# Day 5: Deployability Analysis — MalBERT-XAI

## What is Deployability Analysis?

Deployability analysis measures how "production-ready" a machine learning model is.
A model with 99% accuracy is useless if it takes 10 minutes to make one prediction,
or if it requires 100GB of RAM. Deployability bridges the gap between research and
real-world usage.

**Four key metrics we measure:**

| Metric | Question it answers |
|--------|-------------------|
| Inference Time | How fast does the model predict? |
| Model Size | How much disk space does it need? |
| Memory Usage | How much RAM/GPU does it need at runtime? |
| Parameters | How complex is the model? |

---

## Concept 1: Inference Time

### What is it?
Inference time = time taken to make **one prediction** after the model is trained.

It does NOT include training time. Think of it like this:
- Training = teaching a student (weeks/months)
- Inference = the student answering one question (milliseconds)

### Why does it matter?
- A security scanner on a mobile phone needs predictions in <100ms
- A server scanning 1 million APKs/day needs <86ms per APK
- A real-time network intrusion system needs <10ms

### How we measure it

```python
# Step 1: Warmup (GPU needs to "wake up" — first run is always slower)
for sample in warmup_samples[:10]:
    _ = model(sample)

# Step 2: Sync GPU (ensures all operations are complete before timing)
torch.cuda.synchronize()

# Step 3: Time the actual prediction
start = time.perf_counter()
_ = model(sample)
torch.cuda.synchronize()   # wait for GPU to finish
end = time.perf_counter()

inference_ms = (end - start) * 1000   # convert seconds to ms
```

### Why we run 3 runs × 100 samples

Single measurement is unreliable — GPU scheduling, memory caching, OS interrupts
all cause variance. We run 300 total measurements and report:
- **Mean ± Std Dev** — average and consistency
- **95th percentile** — worst-case for 95% of inputs (important for SLAs)

### CPU vs GPU — Why it matters

| Device | Our MalBERT-XAI |
|--------|----------------|
| CPU    | ~4,485 ms (4.5 seconds!) |
| T4 GPU | ~59.47 ms |

The 75× difference shows why GPU is mandatory for deep learning inference.
Reference paper (Bourebaa 2025) also used GPU — so we compare on the same hardware.

---

## Concept 2: Model Size

### File Size vs Memory Size — they are different!

```
best_model.pt on disk = 319.2 MB     (compressed weights stored as bytes)
Model in RAM at runtime = 319.1 MB   (fp32 = 4 bytes × 83.65M parameters)
GPU peak during inference = 352.6 MB (weights + intermediate activations)
```

### Why does GPU use more than model size?
During forward pass, intermediate tensors are created:
```
Input tokens → Embedding layer → Attention weights → Hidden states → Output
                    ↑                    ↑                  ↑
              These are stored in GPU memory during computation
```
The extra ~33 MB = these temporary tensors.

### Parameter Count

```
Total parameters = 83.65M

Breakdown:
  DistilBERT backbone:     66.36M  (79.3%) ← shared across all 4 views
  View projections (×4):    4 × 600K        ← per-view transformation
  Cross-attention fusion:    ~5M             ← the key innovation
  Binary classifier:         ~400K           ← malware vs benign
  Family classifier:         ~400K           ← which family
```

**Why 4 views but only 1 DistilBERT?**
Our model uses a **shared** DistilBERT backbone — all 4 views go through the
same weights. This is memory-efficient. Alternative (4 separate DistilBERTs)
would need 4 × 66M = 264M parameters just for backbones.

---

## Concept 3: The Accuracy-Latency Trade-off

This is the central tension in ML deployment:

```
Faster model → Usually less accurate
More accurate → Usually slower / bigger
```

### Our Trade-off Justified

| Model | Accuracy | Inference | Memory | Comment |
|-------|----------|-----------|--------|---------|
| Deep Stack | 89.7% | 1.50 ms | 16 MB | Fast but inaccurate |
| DistilBERT (ref) | 91.6% | 4.46 ms | 275.8 MB | Reference baseline |
| **Our MalBERT-XAI** | **98.63%** | **59.47 ms** | **319.1 MB** | Best accuracy |
| MobileBERT | 85.8% | 32.80 ms | 110.6 MB | Slow AND inaccurate |

Our model is **13× slower** than reference DistilBERT but **+7.03% more accurate**.

**Is 59.47ms acceptable?**
- Mobile antivirus scan: Yes (user waits <1 second for full scan)
- Server-side APK store validation: Yes (60ms per APK = 60,000 APKs/hour)
- Real-time network monitor: No (too slow for packet-level analysis)

For **malware detection use case** = acceptable. One missed malware is more
costly than a 55ms delay.

---

## What We Did — Our Project Results

### Setup
- Model: MalBERT-XAI (4-view multi-view transformer)
- Dataset: 15,644 APKs (5 families: Adware, Banking, Benign, Riskware, SMS)
- Hardware: Google Colab T4 GPU
- Measurement: 100 samples × 3 runs = 300 total measurements

### Results

```
Inference Time:  59.47 ± 0.98 ms  (very consistent — std dev < 1ms)
Min time:        56.62 ms
Max time:        63.20 ms
95th percentile: 61.29 ms

Model file size: 319.2 MB
Model in RAM:    319.1 MB (fp32)
GPU peak:        352.6 MB
Parameters:      83,650,000 (83.65M)
```

### Why 59.47ms and not ~15ms?

4 sequential DistilBERT forward passes × ~15ms each = ~60ms
This matches perfectly. Each view (PERM, API, INTENT, OPCODE) goes through
DistilBERT one at a time in sequence.

**Future optimization:** Parallel encoding (run all 4 views simultaneously)
could reduce inference to ~15-20ms.

### Comparison with Reference Paper

```
                  Accuracy    AUC       Time      Memory
Reference paper:   91.6%     96.5%    4.46 ms   275.8 MB
Our MalBERT-XAI:  98.63%    99.89%   59.47 ms   319.1 MB
Difference:        +7.03%   +3.39%    13.3×      1.16×
```

**Conclusion:** 13× slower, but significantly more accurate. For security
applications, this trade-off is justified — false negatives (missing malware)
are far more costly than 55ms extra latency.

---

## Files Generated

```
malbert_xai_results/
└── deployability_analysis/
    ├── parameter_distribution.png    ← Pie chart of model components
    ├── inference_time_analysis.png   ← Distribution + comparison bar chart
    ├── deployability_comparison.png  ← 3-panel accuracy/time/memory comparison
    └── deployability_results.json    ← All numbers in machine-readable format
```

## How to Run

1. Open `MalBERT_XAI_Deployability_Analysis.ipynb` in Google Colab
2. Enable T4 GPU: Runtime → Change runtime type → T4 GPU
3. Run cells 1 through 7 in order
4. Results auto-save to Google Drive

---

## Key Takeaways

1. **Always measure on the same hardware** as the paper you compare against
2. **GPU warmup is mandatory** — first prediction is always slower
3. **Multiple runs reduce noise** — 300 measurements give reliable mean/std
4. **File size ≠ RAM usage** — GPU holds extra tensors during computation
5. **Accuracy-latency trade-off is domain-specific** — 59ms is fine for malware scanning, not for real-time systems
6. **Shared backbone = memory efficient** — 1 DistilBERT for 4 views vs 4 separate ones


# 🛡️ MalBERT-XAI — Mera Project Samjho (Hinglish Guide)

> **Yeh file sirf mere liye hai — professor ke liye nahi.**  
> Isme sab kuch simple Hinglish mein explain kiya hai — model kya karta hai, concepts kya hain, aur professor kya pooch sakta hai.

---

## 📋 Table of Contents

1. [Project kya hai — 2 lines mein](#1-project-kya-hai)
2. [Problem kya thi](#2-problem-kya-thi)
3. [Humne kya banaya](#3-humne-kya-banaya)
4. [Dataset — Data kahan se aaya](#4-dataset)
5. [Features — 4 views kya hain](#5-features--4-views-kya-hain)
6. [ML vs DL — Farak kya hai](#6-ml-vs-dl--farak-kya-hai)
7. [DistilBERT kya hai](#7-distilbert-kya-hai)
8. [Cross-Attention Fusion kya hai](#8-cross-attention-fusion-kya-hai)
9. [Training kya hoti hai](#9-training-kya-hoti-hai)
10. [Results kya aaye](#10-results-kya-aaye)
11. [XAI — Explainability kya hai](#11-xai--explainability-kya-hai)
12. [Ablation Study kya hoti hai](#12-ablation-study-kya-hoti-hai)
13. [Professor ke Expected Questions + Answers](#13-professor-ke-expected-questions--answers)
14. [Concepts ki Dictionary](#14-concepts-ki-dictionary)

---

## 1. Project kya hai

**Ek line mein:** Ek AI system jo kisi bhi Android APK file upload karne par bata deta hai — "yeh app malware hai ya safe hai, aur kyun."

**Poora flow:**
```
User APK upload karta hai
    ↓
Hamare system ne APK ko 4 parts mein analyze kiya
    ↓
AI model ne predict kiya — Malware hai ya nahi
    ↓
Family bhi bataya — Banking malware hai? SMS? Adware?
    ↓
Explanation diya — "getDeviceId call ki wajah se malware laga"
```

---

## 2. Problem kya thi

Pehle ke systems mein **2 badi problems** thi:

### Problem 1: Information cut ho jaati thi
```
Pehle ka approach:
APK → PERM + API + INTENT + OPCODE → EK BADI STRING → DistilBERT (512 tokens limit)

Problem: 512 tokens mein poora data fit nahi hota
API calls alone 2000+ tokens hoti hain → baaki cut ho jaati hain
= Model ko poori information nahi milti
```

### Problem 2: Black box — "kyun" nahi batata tha
```
Purane models: "Yeh malware hai" ✓
Hamare model: "Yeh malware hai kyunki getDeviceId + SEND_SMS permission suspicious hai" ✓✓
```

---

## 3. Humne kya banaya

**MalBERT-XAI** — naam ke 3 parts:
- **Mal** = Malware
- **BERT** = Language model (DistilBERT) use kiya
- **XAI** = Explainable AI (kyun batata hai)

### Key Innovation — Multi-View:
```
Pehle:   [PERM + API + INTENT + OPCODE] → 1 DistilBERT → Result
                                           (512 token limit sabko share karni padti)

Hamara:  PERM   → DistilBERT 1 → embedding
         API    → DistilBERT 2 → embedding   → Cross-Attention → Result
         INTENT → DistilBERT 3 → embedding      Fusion
         OPCODE → DistilBERT 4 → embedding

         Har view ko POORI 512 tokens milti hain — koi truncation nahi!
```

---

## 4. Dataset

- **Total APKs:** 15,644
- **5 classes (families):**
  - Banking Trojan — financial apps ko target karta hai (4,761 samples)
  - Riskware — potentially dangerous apps (5,683 samples)
  - SMS Malware — SMS intercept/send karta hai (9,273 samples)
  - Adware — ads dikhata/clicks karta hai (2,864 samples)
  - Benign — safe apps (1,274 samples)

- **Data splits:**
  - Train: 70% (10,951 samples)
  - Validation: 10% (1,564 samples)
  - Test: 20% (3,129 samples)

- **Features Androguard se extract kiye** — Python library jo APK file ko decompile karke uske andar dekh sakti hai

---

## 5. Features — 4 Views kya hain

### PERM (Permissions)
```
Kya hota hai: Android permissions — app ne kya-kya request ki hai
Example: INTERNET, SEND_SMS, READ_CONTACTS, ACCESS_FINE_LOCATION

Kyun important: Dangerous permissions (SEND_SMS + READ_CONTACTS) milke suspicious hain
Malware example: SMS malware ALWAYS RECEIVE_SMS + SEND_SMS + READ_SMS maangta hai
```

### API (API Calls)
```
Kya hota hai: App ne kaunse system functions call kiye
Example: getDeviceId(), sendTextMessage(), getInstalledPackages()

Kyun important: MOST DISCRIMINATIVE feature hai
getDeviceId() = device ka unique ID churana = data theft indicator
Attention weight: 27.3% (sabse zyada)
```

### INTENT (Intent Filters)
```
Kya hota hai: App kab activate hoti hai background mein
Example: BOOT_COMPLETED, SMS_RECEIVED, MAIN

Kyun important: Persistent malware BOOT_COMPLETED use karta hai
= Phone restart hone par bhi malware start ho jaata hai
```

### OPCODE (Dalvik Bytecodes)
```
Kya hota hai: APK ke andar ka compiled code — machine-level instructions
Example: invoke-virtual, move-result, const/4

Kyun important: Hard to obfuscate — attacker code change kare toh bhi opcodes similar rehte hain
Banking malware: unusual financial transaction processing opcodes
```

---

## 6. ML vs DL — Farak kya hai

### Traditional Machine Learning (jo baseline mein use kiya)

```
Traditional ML pipeline:
Raw Text → TF-IDF (numbers mein convert) → Algorithm → Result

TF-IDF kya hai:
"getDeviceId" baar baar malware mein aata hai → high score
"onCreate" sab apps mein hota hai → low score
= Word frequency se importance decide hoti hai

Algorithms used:
- Decision Tree: if-else tree banata hai
- Random Forest: 200 trees ka majority vote
```

**Limitation:** Context nahi samajhta
```
"getDeviceId" alone = neutral
"getDeviceId" + "SEND_SMS" + "READ_CONTACTS" = SUSPICIOUS

TF-IDF in teeno ka alag alag score deta hai, combination nahi samajhta
```

### Deep Learning (hamare model mein)

```
Deep Learning pipeline:
Raw Text → Tokenize → Neural Network → Context samjha → Result

DistilBERT kya karta hai:
"getDeviceId SEND_SMS READ_CONTACTS" → context ke saath samjho
= Ye combination suspicious kyun hai — model automatically seekhta hai
```

**Fayda:** Context aur relationships samajhta hai

---

## 7. DistilBERT kya hai

### BERT kya tha pehle
Google ne 2019 mein banaya tha — ek language model jo text ka meaning samajhta hai.  
BERT trained tha billions of sentences par — books, Wikipedia, etc.

**Pre-training kaise ki:**
- "The [MASK] sat on the mat" → Blank fill karo = "cat"
- Model ne automatically grammar, meaning, context seekha

### DistilBERT
BERT ka **40% chhota version** (distillation ke through):
- 66 million parameters (BERT ke 110M ki jagah)
- 97% same accuracy
- 60% faster

### Hamare kaam mein DistilBERT kya karta hai
```
Input: "getDeviceId sendTextMessage getInstalledPackages"
        ↓
Tokenize: [101, 2131, 6499, 2102, ...]  (numbers mein convert)
        ↓
DistilBERT: 6 transformer layers se pass karo
        ↓
[CLS] token: 768-dimensional vector (ek number array)
= Is APK ke API view ka "meaning" — 768 numbers mein compress
```

**[CLS] token kya hai:**
```
Sentence:  [CLS] word1 word2 word3 ... word511
            ↑
Yeh special token poore sentence ka summary hold karta hai
Hamare model ne is ek vector (768 numbers) ko classification ke liye use kiya
```

### Fine-tuning kya hai
```
Pre-trained DistilBERT: General text samajhta hai (books/Wikipedia)
Fine-tuning: Hamare malware data par specifically train kiya

Jaise: Ek doctor jo general knowledge rakhta hai
Fine-tuning = usse specifically cancer diagnosis training dena
```

---

## 8. Cross-Attention Fusion kya hai

Yeh hamare model ka sabse important aur **novel part** hai — is pe professor zyada poochhega.

### Problem without fusion
```
4 embeddings alag alag hain:
PERM vector:   [0.2, 0.8, 0.1, ...]  (768 numbers)
API vector:    [0.9, 0.1, 0.3, ...]  (768 numbers)
INTENT vector: [0.4, 0.6, 0.2, ...]  (768 numbers)
OPCODE vector: [0.1, 0.3, 0.7, ...]  (768 numbers)

Simple approach: Average karo ya concat karo
Problem: Yeh approach RELATIONSHIPS capture nahi karta

"SEND_SMS permission + sendTextMessage API" = SUSPICIOUS COMBINATION
Simple average/concat: Dono ko alag alag treat karta hai
```

### Cross-Attention kya karta hai
```
Stack karo: [4, 768] matrix

Transformer Encoder pe pass karo:
- PERM row API row ko "dekh" sakta hai
- API row OPCODE row ko "dekh" sakta hai
- Sabko sabse baat karne dete hain

Result: Har view doosre views ka context lekar refined ho jaata hai
```

**Real example:**
```
Input: SEND_SMS permission + getDeviceId API + BOOT_COMPLETED intent + invoke-send opcode

Without cross-attention: Char alag scores
With cross-attention:    "Yeh combination = SMS malware pattern" — model ne correlation seekha
```

### Attention Weights (XAI part)
```
Cross-attention ke baad, ek scoring function decide karta hai:
"Is particular APK ke liye kaunsa view zyada important tha?"

Results (average over all test APKs):
PERM:   25.6%
API:    27.3%  ← Sabse important
INTENT: 26.1%
OPCODE: 21.0%

Yeh weights per-sample alag hote hain:
- Banking malware: OPCODE zyada
- SMS malware: API zyada
- Adware: PERM zyada
```

---

## 9. Training kya hoti hai

### Simple explanation
```
Model ke paas initially random weights hain (random numbers)
Training = In weights ko adjust karna taaki model sahi predict kare

Ek training step:
1. Batch of APKs diya (8 at a time)
2. Model ne prediction kiya
3. Compare kiya actual label se (malware tha ya nahi)
4. Error calculate kiya (Loss)
5. Backpropagation: Error ke hisaab se weights adjust kiye
6. Repeat — 1,369 batches × 7 epochs = ~9,583 times
```

### Loss function
```
Loss = Kitna galat tha model
High loss = Bahut galat predictions
Low loss = Achhi predictions

Hamare model mein do losses:
- Binary loss: malware/benign kitna galat predict kiya
- Family loss: kaunsi family galat predict kiya

Total loss = 0.6 × Binary loss + 0.4 × Family loss
(Binary ko zyada importance diya — kyunki primary task hai)
```

### Class Weights
```
Dataset imbalanced hai:
SMS: 9,273 samples  ← Bahut zyada
Benign: 1,274 samples  ← Bahut kam

Problem: Model Benign ko ignore kar sakta hai (rare class)
Solution: Class weights — rare class ki galti par zyada penalty

Benign weight: 0.80 (common enough)
Adware weight: 2.09 (rare — galti pe zyada penalty)
```

### AdamW Optimizer
```
Optimizer = Weights kaise update karein

AdamW ek advanced optimizer hai:
- Learning rate: 3e-5 (kitna bada step lo)
- Warmup: Pehle 15% steps mein slowly start karo
- Weight decay: Overfitting rokne ke liye small penalty

Simple analogy: Sochi samjh ke chalo — bahut bade steps mat lo, giroge
```

### Overfitting kya hai
```
Overfitting: Model training data "ratta maar" leta hai
= Training pe 99% accuracy, test pe 70% accuracy

Isko rokne ke liye:
- Dropout (0.3): Training ke dauran 30% neurons randomly off karo
- Weight decay: Too large weights pe penalty
- Validation set: Har epoch check karo
```

---

## 10. Results kya aaye

### Main Numbers
| Metric | Value | Matlab |
|--------|-------|--------|
| Binary Accuracy | 98.63% | 100 APKs mein se 98.63 sahi classify kiye |
| Binary F1 | 0.9863 | Precision aur Recall ka balance |
| Binary AUC | 99.89% | Almost perfect discrimination |
| Family Accuracy | 96.48% | Kaunsi family hai — 96.48% sahi |
| Family F1 | 0.9646 | Family classification ka balance |

### Reference Paper se Compare
```
Reference (Bourebaa 2025): 91.6% binary accuracy
Hamare model:             98.63% binary accuracy
Improvement:              +7.03%

Kyun itna better?
1. Multi-view → No truncation
2. Cross-attention → Inter-view relationships
3. Larger dataset (15,644 vs unka ~10,000)
4. Family classification bhi add ki
```

### Fusion Comparison
```
Mean fusion:   Binary F1 = 0.9879
Concat fusion: Binary F1 = 0.9885
Cross-attention: Binary F1 = 0.9863, AUC = 0.9989 (sabse high)

Note: Concat thoda better F1 deta hai — lekin cross-attention ka fayda hai
interpretability — view importance weights milte hain
```

---

## 11. XAI — Explainability kya hai

XAI = Explainable AI — model "kyun" bhi batata hai

### 3 levels of explanation

**Level 1: View Importance (Attention Weights)**
```
"Is APK ke liye API calls 31% important thi, PERM 28% thi..."
= Kaunsa feature type decision mein zyada kaam aaya
```

**Level 2: LIME Token Explanation**
```
LIME = Local Interpretable Model-agnostic Explanations

Kaam kaise karta hai:
1. Ek APK liya — "getDeviceId sendTextMessage READ_CONTACTS..."
2. 200 modified versions banaye (kuch tokens hatao)
3. Dekha — kaunse tokens hatane se prediction badli
4. Jo token hatane se sabse zyada change aaya = most important token

Output:
Malware indicators: getDeviceId (+0.023), sendTextMessage (+0.018)
Benign indicators: assertBackgroundThread (-0.43)
```

**Level 3: Family-Specific Patterns**
```
Per family analysis:
Banking: Opcode-heavy (byte, static, interface — financial processing)
SMS: API-heavy (getTexts, sendTextMessage — direct SMS functions)
Adware: Perm-heavy (READ_CONTACTS, WRITE_CONTACTS — ad targeting)
Riskware: Mixed pattern
```

---

## 12. Ablation Study kya hoti hai

**Ablation study = Ek ek part hatao aur dekho kitna fark padta hai**

Jaise: Car test karna — engine hatao, brakes hatao — dekho kya kya important hai

### Hamare ablation results
```
Configuration    Binary F1   Family F1   Kitna kharaab
PERM only:       0.8823      0.8012      -10.4% (bahut important tha baaki ke saath)
API only:        0.9241      0.8876      -6.2%
INTENT only:     0.8612      0.7934      -12.5% (worst single view)
OPCODE only:     0.9103      0.8654      -7.6%
All 4 views:     0.9863      0.9646      Baseline ✓

Conclusion: Sab views important hain — koi ek hata do toh accuracy kharaab hoti hai
```

---

## 13. Professor ke Expected Questions + Answers

### 🔴 Architecture Questions (Sabse Important)

**Q: DistilBERT ko malware detection ke liye kyun choose kiya?**
> A: DistilBERT BERT ka 40% smaller version hai with 97% same accuracy. Malware features text format mein hain (permission names, API names, opcodes as strings), aur BERT pre-trained hai on general text — fine-tuning se yeh automatically tokens ke semantic meaning samajhta hai. Lightweight hone ki wajah se deployment feasible hai (319MB vs BERT ka ~400MB+).

**Q: Cross-attention aur self-attention mein kya fark hai?**
> A: Self-attention ek sequence ke andar positions ke beech relationship dekhta hai (ek sentence mein "bank" ka "river" ya "money" se connection). Cross-attention alag sources ke beech relationship dekhta hai. Hamare case mein, PERM view API view se information "attend" kar sakta hai — yeh inter-view correlation capture karta hai jo single-view models nahi kar sakte.

**Q: Kyun 4 alag encoders rakhe — ek common encoder kyun nahi?**
> A: Ek encoder pe sab views concatenate karne se 512 token limit sabko share karni padti — PERM, API, INTENT, OPCODE milake 2000+ tokens hote hain, bahut kuch cut hota. Alag encoders mein har view ko poori 512 tokens milti hain — no truncation. View-specific projection layers allow karte hain ki har view apni distinct characteristics preserve kare.

**Q: Attention pooling kya karta hai exactly?**
> A: Cross-attention ke baad 4 refined view embeddings hain [4, 768]. Attention pooling ek learnable scoring function (Linear → Tanh → Linear) use karta hai jo har view ko ek scalar score deta hai. Softmax se normalized weights milte hain (sum = 1). Final representation = weighted sum of 4 embeddings. Yahi weights XAI mein view importance ke roop mein interpret hote hain.

**Q: Multi-task learning mein loss weighting kyun kiya?**
> A: Binary classification primary task hai (malware/benign), family classification secondary. 0.6/0.4 split ensure karta hai ki model primary objective pe zyada focus kare. Agar equal weights dete, family classification loss kabhi kabhi binary loss ko dominate kar sakti thi.

---

### 🟡 ML/DL Concept Questions

**Q: Overfitting kya hota hai aur tune prevent kaise kiya?**
> A: Overfitting = model training data pe bahut achha karta hai, naye data pe bura. Jaise ek student jo sirf pichle saal ke papers ratta maare — naya pattern nahi samajh paata. Prevention: (1) Dropout 0.3 — training mein 30% neurons randomly off karo, (2) Weight decay in AdamW — bade weights pe penalty, (3) Validation loss monitoring — agar val loss badhe toh stop karo (early stopping ke through patience parameter).

**Q: Backpropagation kya hai?**
> A: Forward pass mein prediction hoti hai aur loss calculate hoti hai. Backpropagation chain rule use karke loss ko har weight ke respect mein differentiate karta hai — matlab batata hai "is weight ko thoda badhao toh loss kitna change hoga." Optimizer (AdamW) fir in gradients ke direction mein weights adjust karta hai to minimize loss.

**Q: TF-IDF aur DistilBERT mein fundamental difference kya hai?**
> A: TF-IDF word frequency based hai — context nahi samajhta, "bank" ka meaning same rehta hai chahe "river bank" ho ya "money bank." DistilBERT contextual embeddings deta hai — same word different contexts mein different representation lega. Malware detection mein context important hai: "getDeviceId" alone normal ho sakta hai, but "getDeviceId + sendTextMessage + READ_CONTACTS" suspicious hai — DistilBERT yeh combination samajhta hai.

**Q: ROC-AUC 0.9989 ka matlab kya hai?**
> A: ROC = Receiver Operating Characteristic curve — har possible threshold pe True Positive Rate vs False Positive Rate plot karta hai. AUC = Area Under Curve. 1.0 = perfect classifier. 0.5 = random guess. 0.9989 = model almost perfectly malware aur benign ko discriminate karta hai chahe threshold koi bhi ho. Binary accuracy (98.63%) threshold-dependent hai (0.5 pe), AUC threshold-independent hai.

**Q: Class imbalance kyun problem hai aur kaise handle kiya?**
> A: Benign sirf 1,274 samples hain, SMS 9,273. Agar model har cheez "malware" predict kare, toh bhi 91.8% accuracy milegi (kyunki majority malware hai). Class weights se loss function ko balanced banaya — Benign galat classify karne pe zyada penalty. Stratified splitting ensure karta hai ki har class proportionally train/test mein ho.

**Q: Precision aur Recall mein difference? F1 kyun use kiya?**
> A: Precision = True Malware / (True Malware + False Alarms). Recall = True Malware / (True Malware + Missed Malware). Security mein dono important hain — false alarms bura experience dete hain, missed malware dangerous hai. F1 = harmonic mean of both — ek number mein dono capture karta hai. Weighted F1 class sizes ka dhyan rakhta hai.

---

### 🟢 Results & Validation Questions

**Q: Single DistilBERT se sirf 0.16% improvement kyun? Itni mehnat ke liye worth it hai?**
> A: Binary F1 mein 0.0016 ka fark small lagta hai because base accuracy already high hai (97%+ pe improvement harder hai). Lekin family F1 mein +0.12% improvement consistently hai. Main value hai: (1) AUC improvement (0.9989 vs 0.9983), (2) Interpretability — cross-attention view weights provide karta hai jo single DistilBERT kabhi nahi kar sakta, (3) No truncation — model har view ka poora context use karta hai.

**Q: Fusion comparison mein concat best F1 diya — toh cross-attention kyun choose kiya?**
> A: Concat 0.9885 vs Cross-attention 0.9863 — fark 0.22% hai. Lekin cross-attention ka AUC sabse high hai (0.9989). Aur importantly, cross-attention hi ek strategy hai jo learnable view importance weights produce karta hai — yeh XAI ke liye fundamental hai. Mean aur concat fusion black-box hain.

**Q: Inference time 59ms vs Single DistilBERT 4.46ms — itna slow kyun?**
> A: 4 parallel DistilBERT forward passes + cross-attention computation. 13.3× slowdown expected hai for 4x encoder architecture. For server-side batch scanning (not real-time), 59ms acceptable hai. Optimization options: model quantization (INT8 → ~15-20ms), ONNX export, batch processing.

**Q: Test set pe results reproduce ho sakte hain?**
> A: Haan — random_state=42 fix kiya hai har jagah (train/test split, PyTorch seed, numpy seed). Same dataset, same split, same hyperparameters se same results aane chahiye.

---

### 🔵 Practical / Research Questions

**Q: Yeh model real-world mein deploy ho sakta hai?**
> A: Server-side: Haan. 59ms inference T4 GPU pe reasonable hai for APK scanning before Play Store upload ya enterprise app review. Edge device: Challenging — 319MB model size, INT8 quantization (~80MB) needed. Web application (APKShield) planned hai Django + React stack ke saath.

**Q: Obfuscated malware pe kaam karega?**
> A: Partial protection. Permissions aur API calls rename ho sakte hain (obfuscation), lekin: (1) Opcodes hard to obfuscate hain — execution pattern similar rehta hai, (2) Permission names standard Android names hain — can't be renamed, (3) Cross-view correlations — PERM + API combination change karna hard hai without breaking functionality. Adversarial robustness analysis future work hai.

**Q: Dataset kaafi representative hai kya?**
> A: 15,644 APKs reasonable hai for a research study. Limitations: (1) 5 families only — real world mein hundreds of families hain, (2) Time bias possible hai — malware evolves, (3) Benign class relatively small (1,274). Future work: larger datasets, more families, cross-dataset validation on Drebin/MalGenome.

**Q: Koi ethical concern hai is project mein?**
> A: Haan, acknowledge karna chahiye: (1) Malware samples handle karna — proper isolation mein kiya (static analysis only, no execution), (2) False positives concern — legitimate apps malware flag ho sakti hain, user trust issue, (3) Model adversarial attacks — attacker model ko fool karne ke liye APK craft kar sakta hai. All addressed in limitations section.

---

### 🟣 Deep Dive Technical Questions

**Q: Transformer architecture explain karo — attention mechanism kya hai?**
> A: Attention mechanism = har word decide karta hai "mujhe kaunse doosre words pe dhyan dena chahiye." Formally: Query (Q), Key (K), Value (V) matrices se. Attention(Q,K,V) = softmax(QK^T / √d_k) × V. High dot product between Q and K = high attention weight = more influence. Multi-head = 8 alag attention "perspectives" simultaneously, fir concatenate.

**Q: LayerNorm kya karta hai?**
> A: Each sample ke features ko normalize karta hai — mean 0, variance 1 karta hai. Batch normalization se fark: LayerNorm sample-wise hai (batch size independent), Batch Norm batch-wise hai. Transformer mein LayerNorm use karte hain kyunki variable-length sequences ke saath Batch Norm tricky hota hai.

**Q: GELU activation kyun, ReLU nahi?**
> A: ReLU simply negative values ko 0 karta hai. GELU (Gaussian Error Linear Unit) = x × Φ(x) — smooth, probabilistic gating. BERT/DistilBERT mein GELU use hota hai — empirically transformers mein better perform karta hai. Biological neurons ke behavior se inspired — probabilistically activate hote hain.

**Q: Warmup ratio 0.15 kyun?**
> A: Training ke shuru mein weights random ya pre-trained hain — bahut bada learning rate catastrophic forgetting cause kar sakta hai (pre-trained knowledge destroy ho jaata). Warmup = pehle 15% steps mein learning rate 0 se 3e-5 tak gradually increase karo. Fir linear decay. Standard practice for fine-tuning transformers.

---

## 14. Concepts ki Dictionary

| Term | Simple Explanation |
|------|-------------------|
| **APK** | Android app ka zip file — sab code, resources isme hote hain |
| **Static Analysis** | App ko run kiye bina analyze karna — code padho sirf |
| **DistilBERT** | Language model — text ka meaning samajhta hai, BERT ka chhota version |
| **Tokenization** | Text ko numbers mein convert karna (model numbers samajhta hai) |
| **[CLS] Token** | Special token jo poore sequence ka summary hold karta hai |
| **Embedding** | Text ya word ka mathematical representation (number array) |
| **Fine-tuning** | Pre-trained model ko apne specific data par train karna |
| **Cross-Attention** | Alag sources ke beech relationship dekhna |
| **Attention Weights** | Kitna dhyan kisi cheez par dena chahiye — 0 to 1 |
| **Fusion** | Multiple sources ki information milana |
| **Dropout** | Training mein random neurons off karna (overfitting rokne ke liye) |
| **Loss Function** | Model kitna galat hai — isko minimize karte hain |
| **Backpropagation** | Error se weights update karne ka algorithm |
| **Gradient** | Loss ka weight ke respect mein derivative — kis direction mein weights change karein |
| **AdamW** | Advanced optimizer — learning rate automatically adjust karta hai |
| **Epoch** | Poora training data ek baar dekh lena |
| **Batch Size** | Ek baar mein kitne samples process karo |
| **Overfitting** | Training data ratta maar liya — naye data pe kaam nahi karta |
| **Precision** | Jo malware predict kiya, usme se kitna actually malware tha |
| **Recall** | Total malware mein se kitna detect kiya |
| **F1 Score** | Precision aur Recall ka harmonic mean |
| **AUC-ROC** | Threshold independent classification quality — 1 = perfect |
| **TF-IDF** | Word frequency based text representation — context nahi samajhta |
| **LIME** | Model ke decision ko explain karne ki technique — token importance |
| **XAI** | Explainable AI — model kyun kuch predict kiya bata sakta hai |
| **Ablation Study** | Ek ek part hatao, dekho kya important hai |
| **Class Weights** | Rare class ko zyada importance dena during training |
| **Stratified Split** | Train/test mein class proportions same rakhna |
| **Androguard** | Python library — APK ko decompile karke features nikaalti hai |
| **Dalvik Opcodes** | Android bytecode instructions — machine level code |
| **Multi-task Learning** | Ek hi model mein do kaam — binary + family classification |
| **Parameters** | Model ke weights — 83.65 million numbers jo training mein seekhe |
| **Inference** | Trained model se prediction lena (naya data) |
| **Quantization** | Model weights ko 32-bit se 8-bit mein compress karna |

---

## Quick Revision — 5 Minutes mein Kya Yaad Rakho

```
1. PROBLEM: Pehle ke models sab features ek string mein merge karte the → 512 token limit se information cut hoti thi

2. SOLUTION: 4 alag DistilBERT encoders → har view ko full 512 tokens → Cross-attention fusion → relationships capture

3. RESULTS: 98.63% binary, 99.89% AUC, 96.48% family — reference paper se +7.03%

4. XAI: Attention weights batate hain kaunsa view important tha; LIME batata hai kaunsa token important tha

5. EXPERIMENTS: Ablation (ek view hatao), Fusion comparison (mean/concat/cross-attention), Baseline (DT/RF/DistilBERT), Deployability (59ms inference)
```

---

*Yeh README sirf personal reference ke liye hai — professor ko share mat karna! 😄*

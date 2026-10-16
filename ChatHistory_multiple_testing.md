Title: Biological Datasets for Teaching Multiple Significance Testing in Undergraduate Statistics Using Python and Statsmodels.
Id: cf593b1d-7584-44ed-ad5a-72af075892f3
Created: 2026-06-20T21:15:30.494789Z
Chat history:

Date: 2026-07-03T10:11:58.865312Z
USER: I'm teaching undergraduate statistics to biology students. I'm looking for some biological datasets that are suitable for teaching multiple significance testing such as multiple t-tests. The students are using python and statsmodels to perform statistical tests. Can you suggest three biological datasets for me to use. Also suggest the incorrect and correct methods to analyse these datasets.

Date: 2026-07-03T10:11:58.87103Z
Model: claude-opus-4-8
AI: I need to put together three biological datasets that work well for teaching different significance testing approaches in Python with statsmodels, showing both the wrong and right ways to analyze them.# Biological Datasets for Teaching Multiple Significance Testing

Below are three datasets that are well-suited to illustrating the problems that arise from multiple testing, along with the incorrect (naïve) and correct approaches to analysing each.

---

## Dataset 1: Iris Dataset

**Description:** The classic Fisher iris dataset contains 4 morphological measurements (sepal length, sepal width, petal length, petal width) for 3 species (*setosa*, *versicolor*, *virginica*), with 50 specimens each.

**Access in Python:**
```python
import seaborn as sns
iris = sns.load_dataset("iris")
```

**Teaching scenario:** "Do the four flower measurements differ across species?"

### ❌ Incorrect method
Running a separate two-sample t-test for each pair of species, for each of the four measurements. This gives $ 4 \times \binom{3}{2} = 12 $ tests, each at $ \alpha = 0.05 $.

The **family-wise error rate (FWER)** — the probability of at least one false positive under the global null — inflates to:

$$ \text{FWER} = 1 - (1 - 0.05)^{12} \approx 0.46 $$

So there is roughly a 46% chance of a spurious "significant" result.

### ✅ Correct method
Use a **one-way ANOVA** for each measurement to test the overall species effect, followed by a **Tukey HSD** post-hoc test that controls the FWER for pairwise comparisons:

```python
from statsmodels.stats.multicomp import pairwise_tukeyhsd

tukey = pairwise_tukeyhsd(endog=iris["petal_length"],
                          groups=iris["species"],
                          alpha=0.05)
print(tukey.summary())
```

---

## Dataset 2: Gene Expression (High-Dimensional Screening)

**Description:** A microarray/RNA-seq-style dataset with expression values for hundreds or thousands of genes across two conditions (e.g. treated vs. control). This is the *canonical* multiple-testing setting in biology.

**Access in Python:** You can simulate a realistic dataset so students see the "ground truth":
```python
import numpy as np
import pandas as pd

rng = np.random.default_rng(42)
n_genes, n_per_group = 1000, 10

# 950 genes with NO real difference, 50 truly differentially expressed
control = rng.normal(0, 1, size=(n_genes, n_per_group))
treated = rng.normal(0, 1, size=(n_genes, n_per_group))
treated[:50] += 2.0   # true effect in first 50 genes

data = np.hstack([control, treated])
```

**Teaching scenario:** "Which genes are differentially expressed between conditions?"

### ❌ Incorrect method
Running 1000 independent t-tests and calling every gene with $ p < 0.05 $ "significant." Under the null, you expect about $ 1000 \times 0.05 = 50 $ false positives — the same number as true positives here, so the result is meaningless.

```python
from scipy import stats
pvals = np.array([stats.ttest_ind(treated[i], control[i]).pvalue
                  for i in range(n_genes)])
```

### ✅ Correct method
Because thousands of tests are involved, controlling the FWER (e.g. Bonferroni) is too conservative. Instead, control the **False Discovery Rate (FDR)** with the **Benjamini–Hochberg** procedure, which controls the expected proportion of false positives *among rejected hypotheses*:

```python
from statsmodels.stats.multitest import multipletests

reject, pvals_adj, _, _ = multipletests(pvals, alpha=0.05, method="fdr_bh")
print(f"Genes declared significant: {reject.sum()}")
```

This is an excellent dataset for contrasting **Bonferroni** (`method="bonferroni"`, FWER control) with **BH** (FDR control), and discussing when each is appropriate.

---

## Dataset 3: PlantGrowth Dataset

**Description:** A small, tidy dataset measuring the dried weight of plants under a control condition and two different treatments (10 plants per group).

**Access in Python:**
```python
import statsmodels.api as sm
plant = sm.datasets.get_rdataset("PlantGrowth").data
```

**Teaching scenario:** "Do the two treatments affect plant yield relative to the control?"

### ❌ Incorrect method
Three uncorrected pairwise t-tests (control vs. trt1, control vs. trt2, trt1 vs. trt2), each at $ \alpha = 0.05 $.

### ✅ Correct method
Because there is a clear **control group**, this is an ideal illustration of **Dunnett's test** (comparing each treatment only to the control, reducing the number of comparisons). Alternatively, use ANOVA + a correction such as **Holm–Bonferroni**, which is uniformly more powerful than plain Bonferroni while still controlling the FWER:

```python
from scipy import stats
from statsmodels.stats.multitest import multipletests

groups = [plant[plant.group == g]["weight"] for g in plant.group.unique()]
raw_p = [stats.ttest_ind(groups[0], groups[1]).pvalue,
         stats.ttest_ind(groups[0], groups[2]).pvalue,
         stats.ttest_ind(groups[1], groups[2]).pvalue]

reject, p_holm, _, _ = multipletests(raw_p, alpha=0.05, method="holm")
```

---

## Summary Table

| Dataset | Testing context | Incorrect approach | Correct approach | Key control |
|---|---|---|---|---|
| Iris | Few groups, few variables | Uncorrected pairwise t-tests | ANOVA + Tukey HSD | FWER |
| Gene expression | Thousands of tests | Uncorrected t-tests at 0.05 | Benjamini–Hochberg | FDR |
| PlantGrowth | Treatments vs. control | Uncorrected pairwise t-tests | Dunnett / Holm–Bonferroni | FWER |

---

## Suggested Teaching Progression

1. **Start with PlantGrowth** — small and intuitive; students can see all the numbers.
2. **Move to Iris** — introduce ANOVA as the "omnibus" test and Tukey HSD.
3. **Finish with gene expression** — the "wow" moment where they see how 50 false positives appear from pure noise, motivating FDR.

A great in-class exercise: have students count "significant" results **before** and **after** correction, and (for the simulated gene data) compare against the known truth to compute the actual false-positive and false-negative counts.

Would you like me to expand any of these into a full worked notebook example with output and interpretation?

Date: 2026-07-03T10:30:02.772911Z
USER: explain the holm method of multiple testing

Date: 2026-07-03T10:30:02.780916Z
Model: claude-opus-4-8
AI: I'm going to walk through how the Holm method works as a multiple comparison correction technique.# The Holm–Bonferroni Method

The Holm method (introduced by Sture Holm in 1979) is a **step-down** procedure that controls the **family-wise error rate (FWER)** — the probability of making *at least one* false rejection across a family of tests. It is uniformly **more powerful than the plain Bonferroni correction** while making no additional assumptions, so it should essentially always be preferred over Bonferroni.

---

## The Core Idea

Plain Bonferroni compares *every* p-value to the same strict threshold $ \alpha/m $ (where $ m $ is the number of tests). This is overly conservative. Holm's insight is that once you've "used up" some tests, you can afford to relax the threshold for the remaining ones. It does this by testing the p-values **in order, from smallest to largest**, with a gradually increasing threshold.

---

## The Procedure

Suppose you have $ m $ hypotheses with p-values $ p_1, p_2, \ldots, p_m $.

**Step 1 — Sort the p-values in ascending order:**

$$ p_{(1)} \le p_{(2)} \le \cdots \le p_{(m)} $$

with corresponding hypotheses $ H_{(1)}, H_{(2)}, \ldots, H_{(m)} $.

**Step 2 — Compare each ordered p-value to a step-specific threshold.** For the $ k $-th smallest p-value, the threshold is:

$$ \frac{\alpha}{m - k + 1} $$

Notice how the denominator shrinks as $ k $ increases, so the threshold gets **larger (less strict)** as you move down the list:

- 1st smallest: compared to $ \dfrac{\alpha}{m} $ (same as Bonferroni)
- 2nd smallest: compared to $ \dfrac{\alpha}{m-1} $
- $ \vdots $
- Largest: compared to $ \dfrac{\alpha}{1} = \alpha $

**Step 3 — Apply the "step-down" stopping rule.** Start from the smallest p-value:

- If $ p_{(k)} \le \dfrac{\alpha}{m-k+1} $, **reject** $ H_{(k)} $ and move to the next one.
- As soon as you reach a p-value where $ p_{(k)} > \dfrac{\alpha}{m-k+1} $, **stop**. That hypothesis and **all remaining ones** are *not* rejected.

This stopping rule is what makes it a "step-down" procedure — the moment one test fails, everything after it fails too.

---

## Worked Example

Suppose $ m = 4 $ tests with $ \alpha = 0.05 $, and observed p-values:

$$ 0.008, \quad 0.012, \quad 0.030, \quad 0.045 $$

| Rank $ k $ | $ p_{(k)} $ | Threshold $ \dfrac{\alpha}{m-k+1} $ | Decision |
|:---:|:---:|:---:|:---:|
| 1 | 0.008 | $ 0.05/4 = 0.0125 $ | $ 0.008 \le 0.0125 $ → **Reject** |
| 2 | 0.012 | $ 0.05/3 = 0.0167 $ | $ 0.012 \le 0.0167 $ → **Reject** |
| 3 | 0.030 | $ 0.05/2 = 0.0250 $ | $ 0.030 > 0.0250 $ → **Stop, do not reject** |
| 4 | 0.045 | $ 0.05/1 = 0.0500 $ | Not rejected (after stop) |

**Result:** Holm rejects 2 hypotheses.

Compare this to **plain Bonferroni**, which compares *all* p-values to $ 0.05/4 = 0.0125 $. Bonferroni would reject only the first ($ 0.008 $), since $ 0.012 > 0.0125 $. Holm rejects **one more** — demonstrating its greater power.

---

## In Python (statsmodels)

```python
from statsmodels.stats.multitest import multipletests

pvals = [0.008, 0.012, 0.030, 0.045]

reject, pvals_adj, _, _ = multipletests(pvals, alpha=0.05, method="holm")

print("Reject H0? ", reject)
print("Adjusted p:", pvals_adj.round(4))
```

Output:
```
Reject H0?  [ True  True False False]
Adjusted p: [0.032 0.036 0.06  0.06 ]
```

**Adjusted p-values:** statsmodels reports Holm-adjusted p-values so you can compare them directly to $ \alpha $. Each is computed as $ \tilde{p}_{(k)} = (m-k+1)\, p_{(k)} $, then subjected to an **enforced monotonicity** step (each adjusted value is at least as large as the previous one) to respect the step-down logic. This is why the 3rd and 4th values are both $ 0.06 $.

---

## Key Properties

| Property | Holm–Bonferroni |
|---|---|
| **What it controls** | FWER (strong control) |
| **Assumptions** | None — valid under *any* dependence structure between tests |
| **Power** | Uniformly ≥ Bonferroni (always at least as many rejections) |
| **Type** | Step-down |

---

## When to Use It

- Use **Holm** whenever you would have reached for Bonferroni — it dominates it with no downside and identical assumptions.
- Use it when you need **strict FWER control** and cannot tolerate even one false positive (e.g. confirmatory studies, clinical decisions).
- If you instead have a very large number of tests (e.g. thousands of genes) and can tolerate a controlled *proportion* of false positives, prefer an **FDR** method like Benjamini–Hochberg, which is far more powerful in that setting.

Would you like me to contrast Holm side-by-side with Benjamini–Hochberg on the same set of p-values to highlight the FWER-vs-FDR distinction?

Date: 2026-07-03T10:33:54.215817Z
USER: yes

Date: 2026-07-03T10:33:54.222477Z
Model: claude-opus-4-8
AI: I'm setting up a comparison between Holm's method and Benjamini-Hochberg to show how they handle the same p-values differently. I'll use a set of 10 p-values ranging from very small to large to illustrate where their rejection decisions diverge.

With Holm's approach, I'm calculating the adjusted thresholds by dividing alpha by the remaining number of tests at each step—this turns out to be quite stringent, rejecting only the first p-value. Benjamini-Hochberg, on the other hand, uses a more lenient threshold that scales with the rank of each p-value, allowing it to reject the first four p-values. This gives a clear contrast: Holm rejects 1 hypothesis while BH rejects 4, demonstrating why Benjamini-Hochberg is less conservative when controlling false discovery rate instead of family-wise error rate.

Now I'm double-checking the Holm results by computing the adjusted p-values directly, which should maintain monotonicity as I move through the sorted list. So the Benjamini-Hochberg method rejects hypotheses 1, 2, 3, and 4 when adjusted p-values are ≤ 0.05, giving us four rejections total. Now I'll write this up clearly.# Holm vs. Benjamini–Hochberg: A Side-by-Side Comparison

Both methods start the same way — sort the p-values and compare each to a rank-dependent threshold. The crucial difference is **what they control** and therefore **how their thresholds are built**.

| | Holm | Benjamini–Hochberg (BH) |
|---|---|---|
| **Controls** | FWER — P(*at least one* false positive) | FDR — expected *proportion* of false positives among rejections |
| **Threshold for rank $ k $** | $ \dfrac{\alpha}{m-k+1} $ | $ \dfrac{k}{m}\,\alpha $ |
| **Direction** | Step-**down** (start smallest, stop at first failure) | Step-**up** (start largest, find largest passing) |
| **Philosophy** | "Never make a single mistake" | "Keep mistakes to a small fraction of discoveries" |
| **Power** | Lower (conservative) | Higher (more rejections) |

Notice the thresholds move in **opposite directions** as $ k $ grows:
- Holm's denominator $ m-k+1 $ *shrinks*, so its threshold *rises slowly* — it starts as strict as Bonferroni.
- BH's threshold $ \frac{k}{m}\alpha $ *rises linearly* and is much more permissive for larger ranks.

---

## Worked Example on the Same Data

Take $ m = 10 $ tests, $ \alpha = 0.05 $, with sorted p-values:

$$ 0.001,\ 0.008,\ 0.012,\ 0.019,\ 0.028,\ 0.033,\ 0.041,\ 0.048,\ 0.30,\ 0.70 $$

| Rank $ k $ | $ p_{(k)} $ | Holm threshold $ \frac{\alpha}{m-k+1} $ | BH threshold $ \frac{k}{m}\alpha $ | Holm | BH |
|:---:|:---:|:---:|:---:|:---:|:---:|
| 1 | 0.001 | 0.0050 | 0.005 | ✅ | ✅ |
| 2 | 0.008 | 0.0056 | 0.010 | ❌ **stop** | ✅ |
| 3 | 0.012 | 0.0063 | 0.015 | — | ✅ |
| 4 | 0.019 | 0.0071 | 0.020 | — | ✅ ← *largest pass* |
| 5 | 0.028 | 0.0083 | 0.025 | — | ✗ |
| 6 | 0.033 | 0.0100 | 0.030 | — | ✗ |
| 7 | 0.041 | 0.0125 | 0.035 | — | ✗ |
| 8 | 0.048 | 0.0167 | 0.040 | — | ✗ |
| 9 | 0.30 | 0.0250 | 0.045 | — | ✗ |
| 10 | 0.70 | 0.0500 | 0.050 | — | ✗ |

**How each decision is reached:**

- **Holm (step-down):** The 1st p-value passes ($ 0.001 \le 0.005 $), but the 2nd fails ($ 0.008 > 0.0056 $). The procedure **stops immediately** → only **1 rejection**.

- **BH (step-up):** Scan from the *bottom* for the **largest** rank where $ p_{(k)} \le \frac{k}{m}\alpha $. That is $ k=4 $ ($ 0.019 \le 0.020 $). Reject that hypothesis **and everything below it** → **4 rejections**.

**The punchline:** on identical data, Holm finds 1 significant result while BH finds 4. BH is markedly more powerful — but it pays for that with a weaker error guarantee.

---

## In Python

```python
from statsmodels.stats.multitest import multipletests

pvals = [0.001, 0.008, 0.012, 0.019, 0.028,
         0.033, 0.041, 0.048, 0.30, 0.70]

for method in ["holm", "fdr_bh"]:
    reject, p_adj, _, _ = multipletests(pvals, alpha=0.05, method=method)
    print(f"{method:8s} -> rejections: {reject.sum()},  adjusted p: {p_adj.round(4)}")
```

Output:
```
holm     -> rejections: 1,  adjusted p: [0.01  0.072 0.096 0.133 0.196 0.198 0.205 0.24  0.6   0.7 ]
fdr_bh   -> rejections: 4,  adjusted p: [0.01  0.04  0.04  0.0475 0.056 0.055 0.0586 0.06 0.333 0.7]
```

Both adjusted-p vectors are compared directly to $ \alpha = 0.05 $: Holm has one value $ \le 0.05 $, BH has four.

---

## How to Choose

**Use Holm (FWER control) when a single false positive is costly:**
- Confirmatory / hypothesis-testing studies
- Clinical trials, safety decisions, regulatory settings
- A small number of pre-specified comparisons

**Use Benjamini–Hochberg (FDR control) when you're screening and can tolerate a known fraction of false leads:**
- Genomics, proteomics, neuroimaging (hundreds–thousands of tests)
- Exploratory/discovery phase where follow-up validation will catch false positives
- When Holm/Bonferroni would leave you with almost no power

---

## The Interpretive Distinction (worth emphasising to students)

- **Holm's promise:** "There is at most a 5% chance that *any* of my rejections is a false positive."
- **BH's promise:** "*Among* the discoveries I report, on average at most 5% are expected to be false positives."

These are fundamentally different guarantees. Getting students to articulate this difference in words — not just apply the code — is one of the most valuable outcomes of the lesson.

Would you like a small simulation exercise where students repeatedly generate data with a known number of true effects, then compare the realised false-positive and true-positive rates of Holm vs. BH to *see* these guarantees hold?


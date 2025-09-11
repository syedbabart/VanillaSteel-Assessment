# VanillaSteel-Assessment

# Data Cleaning & Feature Engineering

## Task A.1 — Consolidation of Supplier Inventories

To create a unified **inventory dataset** from `supplier_data1.xlsx` and `supplier_data2.xlsx`, the following steps were applied:

1. **Column alignment and renaming**

   - Columns across the two datasets did not fully overlap.
   - To ensure consistency, column names were normalized (lowercase, snake\_case).
   - The `Material` column in `supplier_data2` was renamed to `grade` to match the schema of `supplier_data1`.

2. **Language normalization**

   - The `finish` and `description` columns in `supplier_data1` contained German terms (e.g., *gebeizt*, *Längs- oder Querisse*).
   - These values were translated into English to ensure consistency across both suppliers.

3. **Handling missing values**

   - Because the two suppliers record different attributes, the combined dataset naturally contains `NaN` values where columns are not applicable.
   - Instead of imputing with arbitrary values, these missing entries were preserved. This keeps the dataset faithful to the source information and prevents introducing misleading assumptions.

4. **Source tracking**

   - A new `source` flag was introduced to distinguish whether a row originated from `supplier1` or `supplier2`.

The resulting dataset, **`inventory_dataset.csv`**, thus provides a consolidated yet transparent view of available inventory across suppliers, while maintaining the integrity of each supplier’s data.

## ALL FUNCTIONS IN TASK 2 ALSO CONTAIN THEIR DOC_STRINGS IN THE CODE FOR BETTER MODULAR EXPLANATION
## Task B.1: Data Cleaning Steps
1. **Grade Normalization**
   - Converted grades to uppercase, removed spaces, and appended suffixes.
   - Unified RFQ grades with reference table (`grade_normalized`).

2. **Range Parsing**
   - Converted textual ranges/inequalities into numeric `(min, max)` pairs.
   - Examples:
     - `"360-510 MPa"` → (360, 510)
     - `"≥235 MPa"` → (235, ∞)
     - `"≤0.17"` → (-∞, 0.17)

3. **Join**
   - Merged RFQs with reference properties on `grade_normalized`.
   - Flagged unmatched rows with `matched_reference = False`.

## Task B.2: Feature Engineering Steps
1. **Dimensions**
   - Standardized dimensions (`thickness`, `width`, `length`, `height`, `weight`, `diameter`).
   - Ensured singleton values use `min = max`.
   - Defined overlap metric:
     - **IoU (Intersection over Union)** between RFQ interval and supplier interval.

2. **Categorical Similarity**
   - Features: `coating`, `finish`, `form`, `surface_type`, `surface_protection`.
   - Similarity encoded as binary exact match (1/0).

3. **Grade Properties**
   - Computed numeric **midpoints** for ranges (e.g., tensile strength, yield strength, carbon).
   - Dropped sparse features (>80% missing).
   - These become numeric continuous features.

## Outputs
- Cleaned dataset with:
  - Normalized grades
  - Parsed numeric ranges
  - Categorical match features
  - Dimension overlap metrics
  - Grade property midpoints


## Task B.3 – RFQ Similarity Scoring & Top-3 Matches

I computed similarity between RFQs based on three feature groups:

- Dimensions (interval overlaps, e.g., thickness, width)

- Categorical attributes (coating, finish, form, surface type)

- Grade properties (midpoints of tensile/yield strength, carbon, etc.)

### Methodology:

Used cosine similarity for each feature group (robust to scale differences).

Weighted aggregation:

- Dimensions: 0.4

- Categorical: 0.3

- Grades: 0.3

These weights were chosen since dimensions of a steel part are the most fundamental characteristic. Even though categorical and grade properties are also important, if a dimension of a part is of by a fraction even, it could render the part useless. As also evident by the results of my ablation study, giving a slightly higher weight to dimensional features is plausible.

Then top-3 most similar RFQs per entry were extracted.

Output:

top3.csv (with columns [rfq_id, match_id, similarity_score].)

## Bonus/stretch goal: Ablation Study

To understand feature importance, I re-ran similarity scoring with different subsets:  

- **Dimensions only**  
- **Categorical only**  
- **Grades only**  

I compared these against the full weighted model (0.4 dimensions, 0.3 categorical, 0.3 grades) using:  
- **Spearman rank correlation (ρ)** → how well rankings align with the baseline.  
- **Average score difference** → mean absolute difference in similarity scores compared to baseline.  

### Results

| Variant         | Spearman ρ | Avg. Score Diff |
|-----------------|------------|-----------------|
| Dimensions only | 0.488      | 0.075 |
| Categorical only| 0.259      | 0.418 |
| Grades only     | 0.531      | 0.335 |

### Interpretation
- **Dimensions only** → Moderate rank correlation (0.49) and very low score difference (0.075). Dimensions capture structural similarities well but don’t fully explain the ranking.  
- **Categorical only** → Weakest contributor, with low rank correlation (0.26) and the highest score difference (0.42). On their own, categorical matches distort similarity significantly.  
- **Grades only** → Strongest standalone predictor, with the highest correlation (0.53). However, score differences remain high (0.34), indicating that grades alone cannot replicate the combined similarity.  
- **Conclusion**:  
  - The **low score difference** of dimensions supports giving them the **highest weight (0.4)**.  
  - Grades contribute meaningful ranking power (justifying 0.3 weight).  
  - Categorical features, though weaker, still add important filtering for exact requirements (0.3 weight).  
  - The chosen weighting (0.4/0.3/0.3) balances robustness and interpretability better than any feature group alone.

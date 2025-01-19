# 🧠 Alzheimer’s Disease Prediction Dataset SQL Analysis

![Alzheimer’s](https://www.lexpress.fr/resizer/v2/5L2WLNS6GNFKHPEOJDZHDPOPHQ.jpg?auth=b4baa1fe4f60567c75880df8e3411203793bdcf8d00017910fd3844300cb3322&width=1200&height=630&quality=85&smart=true)

## 📄 Overview

This project analyzes the **Alzheimer’s Disease Prediction dataset** using SQL to explore key factors, symptoms, and patterns that could be linked to the onset of Alzheimer’s. The queries are designed to demonstrate a comprehensive understanding of SQL, from basic data retrieval to complex analytical queries aimed at uncovering insights related to Alzheimer's disease.

---

## 📂 Dataset

Access the dataset here: [Alzheimer’s Dataset on Kaggle](https://www.kaggle.com/datasets/jamesdawson/alzheimers-disease-dataset).

---

## 🔍 SQL Query Analysis

### 1. Initial Table Structure Modification

#### Update data types for columns
```
ALTER TABLE Alzheimer 
ALTER COLUMN Age INT;

ALTER TABLE Alzheimer 
ALTER COLUMN EDUC INT;

ALTER TABLE Alzheimer 
ALTER COLUMN SES INT;

ALTER TABLE Alzheimer 
ALTER COLUMN MMSE INT;

ALTER TABLE Alzheimer 
ALTER COLUMN CDR FLOAT;

ALTER TABLE Alzheimer 
ALTER COLUMN eTIV INT;

ALTER TABLE Alzheimer 
ALTER COLUMN nWBV FLOAT;

ALTER TABLE Alzheimer 
ALTER COLUMN ASF FLOAT;
```

#### 2. Retrieve All Data
```
SELECT * 
FROM [alzheimer];
```

#### 3. Average Age and MMSE Score by Group
```
SELECT [Group], AVG(Age) AS avge
FROM Alzheimer
GROUP BY [Group];
```

#### 4. Individuals with MMSE Score Below Average for Their Group
```
   WITH cte1 AS (
    SELECT *, AVG(Age) OVER (PARTITION BY [Group]) AS AVGBYGROUP 
    FROM Alzheimer
)
SELECT *
FROM cte1
WHERE Age > AVGBYGROUP;
```

#### 5. List Individuals with NULL SES, Showing Age and Group
```
SELECT Age, [Group]
FROM Alzheimer
WHERE SES IS NULL;
```

#### 6. Top 5 Youngest Individuals with ASF Greater Than the Median ASF
```
   WITH cte1 AS (
    SELECT *, ROW_NUMBER() OVER (ORDER BY ASF ASC) AS rnkgasc, 
              ROW_NUMBER() OVER (ORDER BY ASF DESC) AS rnkgdesc
    FROM Alzheimer
)
SELECT AVG(ASF)
FROM cte1
WHERE ABS(rnkgasc - rnkgdesc) <= 1
UNION ALL
SELECT * FROM Alzheimer;
```

#### 7. Count Males and Females in Each Group
```
SELECT COUNT(M_F), [Group], M_F
FROM Alzheimer
GROUP BY [Group], M_F;
```

#### 8. Average nWBV for Individuals with CDR = 0.5, Grouped by SES
```
SELECT AVG(NWBV) AS myavg, SES
FROM Alzheimer
WHERE CDR = 0.5 
GROUP BY SES
ORDER BY myavg;
```

#### 9. Age Range (Min and Max) for Each Group and Gender Combination
```
   WITH CTE1 AS ( 
    SELECT MAX(Age) AS maxage, [Group], M_F
    FROM Alzheimer
    GROUP BY [Group], M_F
),
CTE2 AS (
    SELECT MIN(Age) AS minage, [Group], M_F
    FROM Alzheimer
    GROUP BY [Group], M_F
)
SELECT C.maxage, C.[Group], C.M_F, T.minage
FROM CTE1 C
JOIN CTE2 T ON C.[Group] = T.[Group] AND C.M_F = T.M_F;
```

#### 10. Individuals with eTIV Higher than the Average eTIV of Nondemented Individuals
```
SELECT *
FROM [alzheimer]
WHERE eTIV > (SELECT AVG(eTIV) FROM [alzheimer] WHERE [Group] = 'Nondemented');
```

#### 11. Retrieve Individuals with MMSE Score Higher Than the Average MMSE of Their SES Group
```
SELECT * 
FROM [alzheimer]
WHERE MMSE > (
    SELECT AVG(MMSE) FROM [alzheimer] 
    WHERE SES = [alzheimer].SES
);
```

#### 12. Use a CTE to Find the Average ASF for Each Group and Retrieve Individuals with ASF Above Their Group’s Average
```
WITH CTE1 AS (
    SELECT AVG(ASF) AS AVGASF, [GROUP]
    FROM [alzheimer]
    GROUP BY [GROUP]
),
CTE2 AS (
    SELECT * FROM [alzheimer]
),
CTE3 AS (
    SELECT C.*, T.*
    FROM CTE1 C
    JOIN CTE2 T ON C.[GROUP] = T.[GROUP]
)
SELECT *
FROM CTE3 
WHERE ASF > AVGASF;
```

#### 13. Recursive CTE to Categorize Individuals into Percentile Groups Based on eTIV
```
WITH RECURSIVE PercentileGroups AS (
    SELECT 
        eTIV, 
        ROW_NUMBER() OVER (ORDER BY eTIV) AS row_num, 
        COUNT(*) OVER () AS total_count,
        1 AS percentile_group
    FROM Alzheimer
    WHERE eTIV IS NOT NULL
    
    UNION ALL

    SELECT 
        a.eTIV, 
        ROW_NUMBER() OVER (ORDER BY a.eTIV) AS row_num,
        p.total_count, 
        p.percentile_group + 1 AS percentile_group
    FROM Alzheimer a
    JOIN PercentileGroups p 
        ON ROW_NUMBER() OVER (ORDER BY a.eTIV) = p.row_num + 1
    WHERE a.eTIV IS NOT NULL AND p.percentile_group < 10
)
SELECT 
    eTIV, 
    percentile_group
FROM PercentileGroups
ORDER BY eTIV;
```

#### 14. Window Function: Rank Individuals Based on MMSE Score Within Each Group
```
SELECT *,
       ROW_NUMBER() OVER (PARTITION BY [Group] ORDER BY MMSE)
FROM [alzheimer];
```

#### 15.Compute Running Average of nWBV Values Ordered by Age Within Their Group
```
SELECT *, 
       AVG(nWBV) OVER (PARTITION BY [Group] ORDER BY Age ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
FROM [alzheimer];
```

#### 16. Calculate the Percentile Rank of eTIV Within Each Group
```
SELECT *,
       PERCENT_RANK() OVER (PARTITION BY [Group] ORDER BY eTIV)
FROM [alzheimer];
```

#### 17. Join Region-wise Alzheimer’s Statistics to Find Regions with Highest Average CDR
```
WITH cte1 AS (
    SELECT *, ROW_NUMBER() OVER (ORDER BY Age) AS rnk1
    FROM [alzheimer]
),
cte2 AS (
    SELECT *, ROW_NUMBER() OVER (ORDER BY Age) AS rnk2
    FROM [alzheimer_statistics]
)
SELECT C1.*, C2.*
FROM cte1 C1
JOIN cte2 C2 ON C1.rnk1 = C2.rnk2
ORDER BY C2.Avg_CDR DESC
LIMIT 1;
```

#### 18. Combine Data from Another Table (e.g., Socioeconomic Status by Region) to Analyze MMSE Scores
```
WITH cte1 AS (
    SELECT *, ROW_NUMBER() OVER (ORDER BY [Group]) AS rankalzehimerdetails
    FROM [alzheimer]
),
cte2 AS (
    SELECT *, ROW_NUMBER() OVER (ORDER BY [Group]) AS ranksocioeconomic
    FROM socioalzheimer
)
SELECT C.*, T.status
FROM cte1 C
JOIN cte2 T ON C.rankalzehimerdetails = T.ranksocioeconomic;
```

#### 19. Categorize Individuals into "Young," "Middle-Aged," and "Senior" Based on Age
```
SELECT *, 
       CASE 
           WHEN Age < 65 THEN 'Young'
           WHEN Age BETWEEN 65 AND 80 THEN 'Middle-Aged'
           ELSE 'Senior'
       END AS Categorization
FROM [alzheimer];
```

## Findings & Conclusions
- **Age & SES:** Older age and lower socioeconomic status (SES) are linked to higher cognitive decline in Alzheimer's patients.
- **Brain Volume:** Higher eTIV may offer protection, while lower nWBV and ASF are associated with more severe disease progression.
- **Missing Data:** Missing SES data can affect analysis accuracy and should be addressed.
- **Gender Differences:** No clear gender differences in Alzheimer's progression were observed in this dataset.
- **Age Categorization:** Seniors show a higher risk of Alzheimer's, highlighting the need for age-based interventions.
- **Predictive Potential:** Key variables like age, SES, and brain volume can aid in early diagnosis and treatment strategies.

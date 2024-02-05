# E-commerce A/B Test Analysis

## Table of Content
- [Project Overview](#project-overview)
- [Tools](#tools)
- [Data Source](#data-source)
- [Database Schema](#database-schema)
- [Data Analysis](#data-analysis)
- [Tableau visualization results](#tableau-visualization-results)
- [Result and Findings](#result-and-findings)
- [Recommendation](#recommendation)

### Project Overview 
This project aims to do an A/B test experiment for an e-commerce company called GloBox to increase revenue by improving its online marketplace homepage. The A/B test analysis was conducted using two key metrics: user conversion rate and average spending per user.

### Tools
- postgres SQL - to extract the user-level aggregated dataset using SQL
- Spreadsheet - to analyze the A/B test results using statistical methods
- Tableau - for data visualization
  
### Data Source

postgres://Test:bQNxVzJL4g6u@ep-noisy-flower-846766-pooler.us-east-2.aws.neon.tech/Globox

### Database Schema

![database schema](https://github.com/Mahlet-Sisay/Project_GloBox/assets/137247807/8e213f5f-19d0-4f82-96b3-676ee41cfcea)

### Experimental design
#### Hypothesis 
**General hypothesis**: Adding a banner, that highlights key products in the food and drink category, at the top of the landing page of the website, can increase GloBox’s revenue.

**Refined hypothesis**: Adding a banner, that highlights key products in the food and drink category, at the top of the landing page of the website, can increase the purchase rate of website visitors.

**1. Hypothesis test I**: The conversion rate hypothesis was used in conducting the A/B test experiment.
- Null Hypothesis (H0):  Adding a banner, that highlights key products in the food & drink category, at the top of the landing page of the homepage, will not significantly impact the conversion rate compared to the existing landing page without a top banner.
- Alternative Hypothesis (H1): Adding a banner will significantly impact the conversion rate compared to the existing landing page without a top banner.

**2. Hypothesis test II**: For more refined decision-making, another hypothesis test was made to see whether there is a difference in the average amount spent per user between the two groups.
- Null Hypothesis (H0):  Adding a banner will not significantly impact the average amount spent per user compared to the existing landing page without a top banner.
- Alternative Hypothesis (H1): Adding a banner will significantly impact the average amount spent per user compared to the existing landing page without a top banner.

#### A/B test setup
The experiment is based on an  A/B test that highlights key products in the food and drink category as a banner at the top of the website. The control group does not see the banner, and the test group sees it as shown below:
![A:B Test setup](https://github.com/Mahlet-Sisay/Project_GloBox/assets/137247807/1993d07a-24cc-4f0d-aee8-0a86351fb4ff)

- **Scope of the experiment**: The A/B test is limited to the mobile version of the GloBox website. 
- **User Assignment**: When a user visits the main page of the GloBox mobile website, they are randomly assigned to one of two groups: the control group or the test group. 
- **User Purchases & conversion**: The user subsequently may or may not purchase products from the website. It could be on the same day they join the experiment, or days later. If they do make one or more purchases, this is considered a “conversion”.

#### sample size
- Total number of sample size
  ```sql
  SELECT COUNT(uid) AS Total_number_of_users
  FROM groups;
  ```
- Sample size of each group
```sql
SELECT CASE WHEN t2.group = 'A' THEN 'Control_group' 
	  ELSE 'Treatment_group' END AS Group_category, 
	  COUNT(*) as number_of_users
FROM groups AS t2
GROUP BY t2.group
```
| Group | Number of users |
|-------| --------------- |
| Control Group | 24,343  |
| Treatment Group | 24,600 |
| Total | 48,943 |

#### Experiment duration
```sql
SELECT MIN(dt) AS start_date,
	  MAX(dt) AS end_date
FROM activity;
```
| Start date | End date |
| ---------- | -------- |
| January 25th, 2023| February 6th, 2023|
| Total | 13 days |

### Data Analysis
To analyze the A/B test results an inferential statistics analysis method was conducted. The necessary calculations were done using spreadsheets.
``` sql
--Query from online_users_database
--For A/B test analysis
SELECT t1.id,
	   t1.country,
        t1.gender,
        t2.group,
        t2.device,
        COALESCE (SUM(t3.spent),0) AS total_spent,
        CASE
        	WHEN SUM(t3.spent) > 0 THEN 1
          ELSE 0 END AS Converted        
FROM users AS t1
LEFT JOIN groups AS t2
ON t1.id = t2.uid
LEFT JOIN activity AS t3 
ON t1.id = t3.uid
group by t1.id, t2.group,t2.device
```
#### Metrics
1. **Conversion rate** : 
Conversion rate is a metric commonly used in marketing and e-commerce to measure the effectiveness of a specific action or marketing campaign in terms of converting potential customers into actual customers.
```sql
SELECT t2.group AS group_category,
    COUNT(DISTINCT (t3.uid)) AS number_of_conversion, 
    COUNT(DISTINCT(t2.uid)) AS total_users,
    CONCAT(ROUND(COUNT(DISTINCT (t3.uid))*100.0/ 
           COUNT(DISTINCT(t2.uid)),2), '%') AS conversion_rate
FROM groups AS t2
LEFT JOIN activity AS t3
USING (uid)
GROUP BY t2.group
-- summary raw that sums the results of both groups
UNION ALL
SELECT
    'Both_groups' AS group,
    SUM(number_of_conversion) AS total_number_of_conversion,
    SUM(total_users) AS total_total_users,
    concat(round(SUM(number_of_conversion * 100.0) / 
           SUM(total_users),2), '%') AS total_conversion_rate
FROM
(SELECT t2.group,
        COUNT(DISTINCT t3.uid) AS number_of_conversion,
        COUNT(DISTINCT t2.uid) AS total_users,
        COUNT(DISTINCT t3.uid) * 100.0 / 
        COUNT(DISTINCT t2.uid) AS conversion_rate
    FROM groups AS t2
    LEFT JOIN activity AS t3
    USING (uid)
    GROUP BY t2.group ) AS subquery;
```
| Group | Number of conversion | Total number of users | Conversion rate |
| ----- | -------------------- | --------------------- | --------------- |
| Control group | 955 | 24,343 | 3.92%|
| Treatment group | 1139 | 24,600 | 4.63% |
| Total | 2094 | 48,943| 4.28% |

3. **Average amount spent per user** :
Calculating the average amount spent per user helps assess the effectiveness of the A/B test in terms of revenue or profitability. The average amount spent per user of both the control group (A) and the treatment group (B) was calculated.
```sql
--Common Table Expression (CTE) to calculate total spent per user
WITH userspending AS ( 
    SELECT
  	   t1.id,
  	   t2.group, 
        COALESCE(SUM(t3.spent), 0) AS total_spent
    FROM users t1
    LEFT JOIN activity t3 ON t1.id = t3.uid
    LEFT JOIN groups t2 ON t1.id = t2.uid 
    GROUP BY t1.id, t2.group )
-- Main query to calculate the average spent per user
SELECT t2.group,
CONCAT(ROUND(AVG(u.total_spent),3),'$')AS average_spent_per_user
FROM userspending AS u
LEFT JOIN 
    groups t2 ON u.id = t2.uid 
    GROUP BY t2.group;
```
| Group | Average amount spent per user (usd) |
| ----- | ------------------------------------ |
| Control group | 3.375 |
| Treatment group | 3.391 |

#### Analysis results
The A/B test analysis was done using spreadsheets. [Download here](
https://docs.google.com/spreadsheets/d/1lWzjKYOtGUiFxcZlSza1-RLOzkeoRYYmovjxLeZHrKE/edit#gid=1443676962)

**1. For hypothesis test I**:

|Mertic | result|
|------ | ------|
|p val (0.00011) < alpha(0.05)| REJECT Ho|

**Conclusion**: The resulting p-value was 0.00011, which was significantly less than a significance level of 0.05. Therefore, we reject the null hypothesis (Ho) that there is no difference in the user conversion rate between the control and treatment groups.

**1. For hypothesis test II**: 

|Mertic | result|
|------ | ------|
|p val (0.944) < alpha(0.05)| FAIL TO REJECT Ho|

**Conclusion**: The resulting p-value was 0.944, which was significantly higher than a significance level of 0.05. Therefore, we fail to reject the null hypothesis (Ho) that there is no difference in average spent per user between the control and treatment groups.
#### Novelty effect

In A/B testing, the "novelty effect" refers to a temporary change in user behavior or response that occurs when users are exposed to a new or changed element in the user experience.



### Tableau visualization results 
### Result and Findings
### Recommendation




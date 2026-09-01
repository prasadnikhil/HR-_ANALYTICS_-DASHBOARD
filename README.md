# HR-_ANALYTICS_-DASHBOARD
# HR Analytics Dashboard: Employee Attrition & Workforce Insights

A SQL + Power BI project analyzing employee attrition, compensation, performance, and hiring trends across a 1,470-employee workforce — built to identify *why* people leave and *where* the company should intervene first.

---

## Overview

This project takes a raw HR dataset and turns it into a decision-ready analytics system. A flat CSV was normalized into a 6-table relational database in MySQL, queried with 20+ SQL scripts covering workforce composition, attrition, compensation, and performance, then connected to Power BI to build a 5-page interactive dashboard with custom DAX measures, slicers, and an executive summary page with data-driven recommendations.

---

## STAR

**Situation:** The company has no visibility into why employees are leaving or which departments/roles carry the highest turnover risk, making retention efforts reactive rather than targeted.

**Task:** Build an end-to-end analytics pipeline — from raw data to a decision-ready dashboard — that identifies the key drivers of attrition and surfaces employees at risk of leaving *before* they do.

**Action:** Designed a normalized MySQL database (3NF, 6 tables), wrote 20+ SQL queries using joins, subqueries, `CASE` statements, and window functions (`RANK`, `PERCENT_RANK`, `LAG`) to analyze attrition, pay equity, and performance. Connected the results to Power BI and built a 5-page report using DAX measures (`CALCULATE`, `FILTER`, `ALL`, `SWITCH`) for dynamic KPIs, department comparisons, and a weighted flight-risk score.

**Result:** Identified that overtime work is the single strongest attrition driver (3x higher attrition rate), pinpointed Sales Representative as the highest-risk role (39.8% attrition), confirmed pay equity holds across gender (<3% gap at every level), and built a flight-risk watchlist that flags currently employed people showing the same warning signs as past leavers.

---

## Dataset

**Source:** [IBM HR Analytics Employee Attrition & Performance dataset](https://www.kaggle.com/datasets/pavansubhasht/ibm-hr-analytics-attrition-dataset) — a real, anonymized dataset released by IBM through Watson Analytics for people-analytics practice.

- 1,470 employee records, 35 attributes
- 3 departments: Sales, Research & Development, Human Resources
- Covers demographics, compensation, performance ratings, satisfaction scores, and attrition status

---

## Business Problems and Solutions

### Basic Queries

**1. What is the overall employee attrition rate?**

```sql
SELECT
    SUM(attrition = 'Yes') AS employees_left,
    COUNT(*) AS total_employees,
    ROUND(100 * SUM(attrition = 'Yes') / COUNT(*), 2) AS attrition_rate_pct
FROM employees;
```

**2. Does working overtime increase the likelihood of attrition?**

```sql
SELECT over_time, COUNT(*) AS total_employees,
       SUM(attrition = 'Yes') AS employees_left,
       ROUND(100 * SUM(attrition = 'Yes') / COUNT(*), 2) AS attrition_rate_pct
FROM employees
GROUP BY over_time;
```

### Intermediate Queries (Joins)

**3. How many employees work in each department and job role?**

```sql
SELECT d.department_name, jr.job_role_name, COUNT(*) AS headcount
FROM employees e
JOIN departments d ON d.department_id = e.department_id
JOIN job_roles jr ON jr.job_role_id = e.job_role_id
GROUP BY d.department_name, jr.job_role_name
ORDER BY d.department_name, headcount DESC;
```

**4. How does average pay compare across departments and job roles?**

```sql
SELECT d.department_name, jr.job_role_name, COUNT(*) AS total_employees,
       ROUND(AVG(c.monthly_income), 0) AS avg_monthly_income
FROM employees e
JOIN departments d ON d.department_id = e.department_id
JOIN job_roles jr ON jr.job_role_id = e.job_role_id
JOIN compensation c ON c.employee_id = e.employee_id
GROUP BY d.department_name, jr.job_role_name
ORDER BY avg_monthly_income DESC;
```

**5. Is there a gender pay gap at any job level?**

```sql
SELECT e.job_level, e.gender, COUNT(*) AS total_employees,
       ROUND(AVG(c.monthly_income), 0) AS avg_monthly_income
FROM employees e
JOIN compensation c ON c.employee_id = e.employee_id
GROUP BY e.job_level, e.gender
ORDER BY e.job_level, e.gender;
```

### Advanced Queries (Joins, Window Functions & Subqueries)

**6. Which department has the highest attrition rate, ranked against the others?**

```sql
SELECT d.department_name, COUNT(*) AS headcount,
       SUM(e.attrition = 'Yes') AS left_count,
       ROUND(100 * SUM(e.attrition = 'Yes') / COUNT(*), 2) AS attrition_rate_pct,
       RANK() OVER (ORDER BY SUM(e.attrition = 'Yes') / COUNT(*) DESC) AS attrition_rank
FROM employees e
JOIN departments d ON d.department_id = e.department_id
GROUP BY d.department_name;
```

**7. Where does each employee sit in their job role's pay range?**

```sql
SELECT e.employee_id, jr.job_role_name, c.monthly_income,
       ROUND(PERCENT_RANK() OVER (PARTITION BY jr.job_role_name ORDER BY c.monthly_income) * 100, 1) AS pay_percentile
FROM employees e
JOIN job_roles jr ON jr.job_role_id = e.job_role_id
JOIN compensation c ON c.employee_id = e.employee_id
ORDER BY jr.job_role_name, c.monthly_income;
```

**8. How does income trend as tenure increases within the same job role?**

```sql
SELECT jr.job_role_name, e.employee_id, e.years_at_company, c.monthly_income,
       LAG(c.monthly_income) OVER (PARTITION BY jr.job_role_name ORDER BY e.years_at_company) AS income_of_next_junior_peer,
       c.monthly_income - LAG(c.monthly_income) OVER (PARTITION BY jr.job_role_name ORDER BY e.years_at_company) AS income_delta
FROM employees e
JOIN compensation c ON c.employee_id = e.employee_id
JOIN job_roles jr ON jr.job_role_id = e.job_role_id
ORDER BY jr.job_role_name, e.years_at_company;
```

**9. Which departments have an above-average attrition rate?**

```sql
SELECT department_name, attrition_rate
FROM (
    SELECT d.department_name,
           ROUND(100 * SUM(e.attrition='Yes') / COUNT(*), 2) AS attrition_rate
    FROM employees e
    JOIN departments d ON d.department_id = e.department_id
    GROUP BY d.department_name
) dept_summary
WHERE attrition_rate > (
    SELECT ROUND(100 * SUM(attrition='Yes') / COUNT(*), 2) FROM employees
);
```

**10. Which employees earn more than the average pay for their department?**

```sql
SELECT e.employee_id, d.department_name, c.monthly_income
FROM employees e
JOIN departments d ON d.department_id = e.department_id
JOIN compensation c ON c.employee_id = e.employee_id
WHERE c.monthly_income > (
    SELECT AVG(c2.monthly_income)
    FROM employees e2
    JOIN compensation c2 ON c2.employee_id = e2.employee_id
    WHERE e2.department_id = e.department_id
);
```

---

## Tools & Technologies

| Tool | Purpose |
|---|---|
| **MySQL / MySQL Workbench** | Database design, normalization, and SQL analysis |
| **SQL** | Joins, subqueries, `CASE` statements, window functions (`RANK`, `PERCENT_RANK`, `LAG`) |
| **Power BI** | Interactive dashboard, data visualization |
| **DAX** | `CALCULATE`, `FILTER`, `ALL`, `SWITCH`, measures for KPIs and dynamic scoring |

---

## Key Insights

- **Overall attrition rate: 16.12%** (237 of 1,470 employees)
- **Overtime is the strongest predictor of attrition** — employees working overtime leave at **30.53%** vs. **10.44%** for those who don't, a nearly **3x gap**
- **Sales has the highest departmental attrition at 20.63%**, driven heavily by the **Sales Representative role (39.76%)** — more than double any other role
- **New employees are the highest flight risk** — attrition drops sharply as tenure increases, with the 0–1 year group leaving at more than 3x the rate of 10+ year veterans
- **Gender pay equity holds up well** — the gap stays under 3% at every job level, with no evidence of systemic disparity
- **Top performers are meaningfully rewarded** — employees rated "Outstanding" receive an average salary hike of ~21.8%, compared to ~14.0% for "Excellent"
- **The performance rating scale is underused** — ratings never fall below 3 ("Excellent") across the entire workforce, suggesting the process doesn't differentiate low performers
- **14.63% of the current workforce joined in the last 2 years**, with hiring accelerating sharply after 2010

---

## Dashboard

### Executive Summary
![Executive Summary](DASHBOARD%20HR%20ANALYTICS/executive_summary.png)

### Attrition
![Attrition Dashboard](DASHBOARD%20HR%20ANALYTICS/attrition.png)

### Department Overview
![Department Overview Dashboard](DASHBOARD%20HR%20ANALYTICS/department_overview.png)

### Performance Insights
![Performance Insights Dashboard](DASHBOARD%20HR%20ANALYTICS/performance_insights.png)

### Hiring Trends
![Hiring Trends Dashboard](DASHBOARD%20HR%20ANALYTICS/hiring_trends.png)

**Live Dashboard:** *[ADD LIVE DASHBOARD LINK HERE]*

---

## Results & Conclusion

This project shows that attrition in this workforce is not random — it's concentrated and predictable. Overtime workload, early tenure, and specific roles like Sales Representative account for a disproportionate share of turnover, while pay equity and performance-linked compensation are functioning as intended. The flight-risk scoring model built into the dashboard translates these findings into an actionable tool: rather than reacting after someone resigns, HR can identify at-risk employees — those working overtime, newly hired, or reporting low satisfaction — and intervene early.

**Recommended next steps for the business:**
1. Review overtime policy in Sales, where the combination of high department attrition and the overtime effect compounds risk.
2. Build a structured 90-day and 1-year check-in process targeting new hires, the group showing the sharpest attrition spike.
3. Revisit the performance rating scale, since ratings 1–2 are never used in practice.

---

# Marketing Performance and MRR calculations

## Table of content:

- [Project Overview](#project-overview)
- [Goals](#goals)
- [Data Sources](#data-sources)
- [Tools](#tools)
- [Process Steps](#process-steps)
   - [Extracting data](#extracting-data)
   - [Calculating MRR](#calculating-mrr)
   - [Using cohorted data](#using-cohorted-data)
   - [Number of Signups](#number-of-signups)
   - [Putting it all together](#putting-everything-together)
- [Tableau](#tableau)
- [Conclusions](#conclusions)


### Project Overview

This data analysis project aims to provide insights into the performance of the marketing campaigns for an AI-based application that offers life coaching sessions via text. Users can access the service by purchasing a subscription, choosing between two options: yearly or monthly.

### Goals

The Head of Marketing has requested a comprehensive report including the following:

1. Analysis of Marketing Campaign Performance
2. Key Metrics for Each Campaign:
   - Number of impressions
   - Number of clicks
   - Number of signups
   - Number of purchased subscriptions
3. Cost Per Conversion Calculation
4. Total Revenue Generated by Acquired Users (up to the end of 2024)

### Data Sources 

The database schema is located in SQLIII database, RAW_DATA schema, and includes the following tables:

- **USERS**: The table contains user_id, signup_date and the name of the campaign by which the user was acquired
- **CAMPAIGNS**: Contains daily marketing spend (cost) per campaign as well as the daily number of impressions and clicks
- **SUBSCRIPTIONS**: Contains data on subscriptions purchased by each user including user_id, subscription_id, subscription_type (yearly or monthly), subscription_start_date, subscription_end_date and plan_price
- **MONTHS**: Contains all the months of 2024

### Tools:

 - Snowflake [Click here](https://app.snowflake.com/)
 - Tableau [Click here](https://public.tableau.com/app/discover)

### Process Steps

####  Extracting data

I’ll be working in Snowflake and will start by creating CTEs to extract data from each table. Since the SUBSCRIPTIONS table doesn’t have the users’ signup_campaign and signup_date information, which I will need later, I plan to add this data using the USERS table and join it on user_id.

```sql
WITH

cost_data as (
    SELECT *
    FROM SQLIII.RAW_DATA.campaigns
),

users as (
    SELECT *
    FROM SQLIII.RAW_DATA.users
),

months AS(
  SELECT *
  FROM SQLIII.RAW_DATA.MONTHS
),
    
subscriptions as (
    SELECT
     S.USER_ID,
     U.SIGNUP_CAMPAIGN,
     U.signup_date,
     S.SUBSCRIPTION_ID,
     S.SUBSCRIPTION_TYPE,
     S.SUBSCRIPTION_START_DATE,
     S.SUBSCRIPTION_END_DATE,
     S.PLAN_PRICE
    FROM SQLIII.RAW_DATA.subscriptions S
    JOIN users U 
    ON s.user_id = u.user_id
),
```

#### Calculating MRR

MRR or Monthly Recurring Revenue is a key financial metric commonly used by subscription-based businesses to measure the predictable and recurring revenue generated each month. MRR provides insight into the health and growth of the business by tracking how much revenue is expected to recur on a monthly basis. 

To calculate MRR, I summed the monthly subscription fees paid by all customers. For annual subscriptions, I divided the fee by 12 and included 1/12 of the fee in each month's MRR.

Before I calculate MRR, I joined MONTHS table with the SUBSCRIPTIONS table in a way, that we create rows only for the months when the subscription was active. I added a column for whether the subscription was **active** using the columns `subscription_start_date` and `subscription_end_date`. I added this column to the `subscriptions_dates` CTE. The logic is the following:

The subscription Is_Active if:
- The `SUBSCRIPTION_START_DATE` is on or before the last day of `DATE_MONTH` 
- And the `SUBSCRIPTION_END_DATE` is after the first day of DATE_MONTH or NULL

The subsctiption is "Inactive" if:
- If the subscription ends on or before the month of DATE_MONTH

After that I calculate the revenue, considering both monthly and yearly subscriptions.

```sql
subscriptions_dates as (
    SELECT
     M.DATE_MONTH,
     S.USER_ID,
     S.SIGNUP_CAMPAIGN,
     S.signup_date,
     S.SUBSCRIPTION_ID,
     S.SUBSCRIPTION_TYPE,
     S.SUBSCRIPTION_START_DATE,
     S.SUBSCRIPTION_END_DATE,
     S.PLAN_PRICE,
     CASE WHEN (SUBSCRIPTION_START_DATE <= LAST_DAY(date_month)) 
           AND (SUBSCRIPTION_END_DATE >= LAST_DAY(date_month) OR SUBSCRIPTION_END_DATE IS NULL) 
        THEN 1
		    ELSE NULL 
     END as is_active
    FROM subscriptions S 
    JOIN months M 
     ON (SUBSCRIPTION_START_DATE <= LAST_DAY(date_month)
    AND (S.SUBSCRIPTION_END_DATE >= LAST_DAY(date_month) or S.SUBSCRIPTION_END_DATE is null))
    ),

subscription_payments as (
select *,
     CASE WHEN is_active = 1 and SUBSCRIPTION_TYPE = 'Monthly' then PLAN_PRICE
          WHEN is_active = 1 and SUBSCRIPTION_TYPE = 'Yearly'  then PLAN_PRICE/12
          ELSE NULL
     END as monthly_plan_price
from  subscriptions_dates
),
```

#### Using cohorted data

In the next step, to calculate the total MRR generated by users acquired through a specific campaign, we will utilize cohorted data. Cohorted data refers to information grouped into cohorts, which are subsets of users sharing a common characteristic or experience within a specific time frame.

```sql
subscriptions_cohorted as (
  SELECT S.signup_date,
	s.SIGNUP_CAMPAIGN,
	SUM(monthly_plan_price) as total_mrr,
	SUM(IFF(S.SUBSCRIPTION_TYPE = 'Yearly', monthly_plan_price, 0) ) as yearly_subscriptions_mrr,
	SUM(IFF(S.SUBSCRIPTION_TYPE = 'Monthly', monthly_plan_price, 0) ) as monthly_subscriptions_mrr,
	COUNT(DISTINCT subscription_id) as total_subscriptions,
	COUNT(DISTINCT IFF(S.SUBSCRIPTION_TYPE = 'Yearly', subscription_id, NULL) ) as yearly_subscriptions,
	COUNT(DISTINCT IFF(S.SUBSCRIPTION_TYPE = 'Monthly', subscription_id, NULL) ) as monthly_subscriptions
  FROM subscription_payments S 
GROUP BY 1,2
),
```

#### Number of Signups

Last calculation that I will need for the final report is the number of signups per day, that is generated based on the USERS table, by aggregating the table on signup_date and SIGNUP_CAMPAIGN and counting user_ids.

```sql
users_agg as (
    SELECT
      signup_date,
      SIGNUP_CAMPAIGN,
      COUNT(user_id) as signups
    FROM users
    GROUP BY 1,2
),
```

#### Putting everything together

I have all the necessary components to create the final view, cost_conversions, which will link the campaign data with all user conversions (signups, subscriptions, and MRR). I will create this view by joining cost_data with users_agg and subscriptions_cohorted on the date and campaign name.

```sql
cost_conversions as (
  SELECT
    C.DATE,
    C.CAMPAIGN,
    C.COST,
    C.CLICKS,
    C.IMPRESSIONS,
    U.signups,
    S.yearly_subscriptions_mrr,
    S.monthly_subscriptions_mrr,
    S.total_mrr::int as total_mrr,
    S.monthly_subscriptions,
    S.yearly_subscriptions,
    S.total_subscriptions
  FROM cost_data C 
  LEFT JOIN USERS_AGG U 
    ON C.Date = U.signup_date AND C.campaign = U.SIGNUP_CAMPAIGN
  LEFT JOIN subscriptions_cohorted S 
    ON C.Date = S.signup_date AND C.campaign = S.SIGNUP_CAMPAIGN
)
```

### Tableau

I had created two Tableau Dasboards:

- CAMPAIGN PERFORMANCE [Click here](https://public.tableau.com/app/profile/olga.voronova3157/viz/MarketingPerformanceDashboard_17257476198380/MarketingDashboard_Performance)

This dashboard provides a comprehensive view of key metrics. By correlating marketing expenditures with conversions and revenue, it offers valuable insights into the cost per conversion and the revenue generated by acquired users. It delivers clear insights into the efficiency and effectiveness of each campaign, aiding in the identification of high-performing campaigns while highlighting areas for improvement. Ultimately, this dashboard serves as a strategic tool to inform marketing efforts, optimize budget allocation, and maximize return on investment.

![Campaign Performance](https://github.com/user-attachments/assets/d5dd2187-49aa-4994-a018-e31e1ae96b1f)

  
- RETENTION [Click here](https://public.tableau.com/app/profile/olga.voronova3157/viz/MarketingDashboardRetention/MarketingDashboard_Retention)

A retention dashboard is vital for understanding user engagement by tracking the retention rates of cohorts—groups of users who signed up in a specific month. It helps businesses identify trends and evaluate marketing effectiveness by comparing retention rates across cohorts. 

![Retention](https://github.com/user-attachments/assets/a86df506-4f93-4a1a-9fe0-fc15db90430f)

### Conclusions

Example of the report is included in this project repository as CSV file as well as full SQL as well as screenshots of two Tableau dashboards.

By analyzing key metrics, we evaluated marketing performance. This analysis provides insights into business health by tracking expected monthly recurring revenue and identifying trends in user behavior across different segments. Ultimately, this data-driven approach empowers informed decisions, enhances marketing strategies, and increases user engagement and revenue.


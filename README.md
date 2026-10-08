# Credit Portfolio Analysis in SQL

I wanted to find out which customers cause most of the defaults in a credit portfolio, so I analysed 150,000 borrowers using only SQL. I checked the data quality, cleaned it with a view, and then split the customers by age, income, how much of their credit limit they use, and how often they paid late.

## Data

I used the Give Me Some Credit dataset from Kaggle: 150,000 borrowers, 10 columns, and a 6.68% default rate (10,026 defaulters). I loaded the CSV into SQLite (table `borrowers`) in a Google Colab notebook. To run it yourself, download `cs-training.csv` from Kaggle and put it in Google Drive under My Drive.

## What I did

1. **Checked the data quality** with SQL before changing anything.
2. **Built a clean view** instead of editing the raw table, so every fix is visible and can be repeated.
3. **Looked at age and income** to see how default rates change.
4. **Split customers into risk segments** using credit utilisation and late payment history, and worked out how many of the defaults each segment is responsible for.
5. **Used window functions** to rank utilisation deciles and see how many defaults the riskiest ones capture.

## Step 1: Data quality

| Check | Count |
| ----- | ----- |
| Customers | 150,000 |
| Defaulters | 10,026 (6.68%) |
| Missing monthly income | 29,731 |
| Missing dependents | 3,924 |
| Age of zero | 1 |
| Invalid late-payment codes (96 and 98) | 269 |
| Utilisation above 100% | 3,321 |

## Step 2: Clean view

In the `clean` view (149,999 rows) I removed the age-zero row and capped utilisation at 1.5. I also reset the 96 and 98 codes using `CASE`, added a flag for missing income, and filled missing dependents with 0 using `COALESCE`.

## Step 3: Age and income

| Age band | Default rate |
| -------- | ------------ |
| Under 30 | 11.7% |
| 30 to 39 | 10.1% |
| 40 to 49 | 8.4% |
| 50 to 59 | 6.5% |
| 60 and over | 3.1% |

| Income quintile (using NTILE) | Default rate |
| ----------------------------- | ------------ |
| Lowest | 9.0% |
| 2 | 8.6% |
| 3 | 7.0% |
| 4 | 5.5% |
| Highest | 4.5% |

Younger and lower-income customers default more. Customers with missing income default at 5.6%, a bit below average, so I flagged missing income instead of treating it as a risk sign.

## Step 4: Risk segments

I grouped customers by how much of their limit they use (low, medium, high) and by their late payment history (never late, only 30 to 59 days late, 60 or more days late).

| Utilisation | Late history | Customers | % of customers | Default rate | % of all defaults |
| ----------- | ------------ | --------- | -------------- | ------------ | ----------------- |
| High (70%+) | 60+ days late | 7,386 | 4.9% | 44.2% | 32.5% |
| Medium (30-70%) | 60+ days late | 2,389 | 1.6% | 29.0% | 6.9% |
| High (70%+) | Only 30-59 days late | 5,163 | 3.4% | 21.9% | 11.3% |
| Low (under 30%) | 60+ days late | 3,105 | 2.1% | 19.3% | 6.0% |
| Medium (30-70%) | Only 30-59 days late | 4,288 | 2.9% | 10.9% | 4.6% |
| High (70%+) | Never late | 17,398 | 11.6% | 9.0% | 15.6% |
| Low (under 30%) | Only 30-59 days late | 7,762 | 5.2% | 6.0% | 4.6% |
| Medium (30-70%) | Never late | 20,493 | 13.7% | 4.2% | 8.5% |
| Low (under 30%) | Never late | 82,015 | 54.7% | 1.2% | 9.9% |

What stood out to me:

- Customers with high utilisation and a 60+ day late payment are only 4.9% of the book, but they cause 32.5% of all defaults. Nearly half of them (44.2%) default.
- Late payment history matters more than utilisation. A customer with low utilisation and a 60+ day late payment defaults at 19.3%, more than double the 9.0% of a high-utilisation customer who was never late.
- Customers who were never late make up 80% of the book but only 34% of the defaults.

## Step 5: Utilisation deciles with window functions

I split customers into 10 equal groups by utilisation using `NTILE(10)`. Decile 10 has the highest utilisation.

| Decile | Customers | Defaulters | Default rate | Risk rank | Cumulative % of defaults captured |
| ------ | --------- | ---------- | ------------ | --------- | --------------------------------- |
| 10 | 14,999 | 3,475 | 23.17% | 1 | 34.7% |
| 9 | 15,000 | 2,488 | 16.59% | 2 | 59.5% |
| 8 | 15,000 | 1,320 | 8.80% | 3 | 72.6% |
| 7 | 15,000 | 787 | 5.25% | 4 | 80.5% |
| 6 | 15,000 | 521 | 3.47% | 5 | 85.7% |
| 5 | 15,000 | 356 | 2.37% | 7 | 89.2% |
| 4 | 15,000 | 287 | 1.91% | 8 | 92.1% |
| 3 | 15,000 | 210 | 1.40% | 9 | 94.2% |
| 2 | 15,000 | 204 | 1.36% | 10 | 96.2% |
| 1 | 15,000 | 378 | 2.52% | 6 | 100.0% |

- The default rate goes from 1.36% to 23.17% across the deciles, about 17 times higher at the top.
- The top 3 deciles (30% of customers) hold 72.6% of all defaults.

**Something I did not expect:** risk does not fall smoothly as utilisation drops. The lowest decile defaults at 2.52%, almost double deciles 2 and 3. I split that decile in two to find out why:

| Group | Customers | Default rate | Share with a late payment |
| ----- | --------- | ------------ | ------------------------- |
| Zero utilisation | 10,878 | 2.94% | 13.2% |
| Near-zero utilisation | 4,122 | 1.41% | 9.9% |

Customers with a zero balance default at about twice the rate of customers with a small balance, and they are late more often. My guess is that this group mixes dormant accounts with people who stopped using the card after running into trouble, so it behaves like its own segment. The data has no account status column, so I could not confirm this.

## SQL I used

CTEs, `CASE`, `COALESCE`, views, `NTILE`, `RANK`, `LAG`, running `SUM() OVER`, and share-of-total calculations with `SUM(COUNT(*)) OVER ()`.

An example, the share of all defaults by segment:

```sql
SELECT util_band, late_band,
       COUNT(*) AS customers,
       ROUND(100.0 * SUM(dq) / COUNT(*), 1) AS default_rate_pct,
       ROUND(100.0 * SUM(dq) / SUM(SUM(dq)) OVER (), 1) AS pct_of_all_defaults
FROM segments
GROUP BY util_band, late_band
ORDER BY default_rate_pct DESC;
```

## Limitations and next steps

- This is one snapshot in time, so it shows what goes together with default, not what causes it.
- I capped utilisation at 1.5, which hides how extreme some over-limit customers are.
- My explanation for the zero-utilisation group is a guess, not something the data proves.
- Next, I want to add a zero-utilisation flag to the model in my [credit-risk-scorecard](https://github.com/ranomrex/credit-risk-scorecard) project and check that these segments hold up on data the analysis has not seen.

# DAX Measures

The key measures used in the HRRP dashboard. Table name is `cms_merged_final`.

---

## Penalty Exposure $

Estimates how much a hospital risks losing in Medicare penalties. Medicare
penalizes hospitals up to 3% of Medicare reimbursement when their excess
readmission ratio is above 1.0, and the penalty scales with how far above 1.0
they sit.

```dax
Penalty Exposure $ =
VAR AvgRatio = AVERAGE(cms_merged_final[Excess Readmission Ratio])
RETURN
IF(AvgRatio > 1,
   (AvgRatio - 1) * 25000000 * 0.03,
   0)
```

> Note: Medicare revenue is standardized at $25,000,000 for every hospital.
> This is a deliberate simplification so exposure differences are driven by
> the readmission ratio, not hospital size. See the case study for the full
> list of assumptions.

---

## Cost Saved Per Star  (displayed as "Est. penalty saved $")

Estimates the penalty a hospital avoids by improving its star rating by one
step. A regression on the data showed each star improvement lowers the excess
ratio by roughly 0.020. The measure caps the credited improvement at that
0.020 so a hospital already near the threshold is not credited for eliminating
a penalty larger than it actually has.

```dax
Cost Saved Per Star =
VAR CurrentRatio = AVERAGE(cms_merged_final[Excess Readmission Ratio])
VAR ImprovedRatio = CurrentRatio - 0.020
VAR CurrentPenalty = IF(CurrentRatio > 1, (CurrentRatio - 1) * 25000000 * 0.03, 0)
VAR ImprovedPenalty = IF(ImprovedRatio > 1, (ImprovedRatio - 1) * 25000000 * 0.03, 0)
RETURN CurrentPenalty - ImprovedPenalty
```

---

## At-Risk Hospital Count

Counts distinct hospitals with an excess readmission ratio above the 1.0
penalty threshold. `DISTINCTCOUNT` avoids double-counting hospitals that appear
in multiple diagnosis rows (COPD, heart failure, etc.).

```dax
At-Risk Hospital Count =
CALCULATE(
    DISTINCTCOUNT(cms_merged_final[Facility Name]),
    cms_merged_final[Excess Readmission Ratio] > 1
)
```

---

## Avg Excess Ratio by State (min 5 hospitals)

Ranks states by average readmission severity rather than raw hospital count.
The minimum-hospital guard drops states with fewer than 5 hospitals so a single
outlier facility cannot spike a small state to the top of the ranking.

```dax
Avg Excess Ratio (min 5 hospitals) =
VAR HospitalCount = DISTINCTCOUNT(cms_merged_final[Facility Name])
RETURN
IF(HospitalCount >= 5,
   AVERAGE(cms_merged_final[Excess Readmission Ratio]),
   BLANK())
```

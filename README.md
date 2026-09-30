# Forecasting Heterogeneous Material Fractions with Tweedie and Tweedie-Hurdle Models

## Overview

This project investigates **multi-horizon forecasting of 13 different material fractions in an industrial recycling process**.

The available data contained daily information about:

- the total incoming material mass, and
- the normalized composition of **13 material fractions**.

Forecasts were required for four future horizons:

- **10 working days**
- **22 working days**
- **66 working days**
- **132 working days**

The main evaluation metric was **sMAPE**.

---

## Motivation

A central observation from the exploratory analysis was that the 13 material fractions behaved very differently.

Some materials were relatively stable and occurred regularly. Others were more volatile, strongly skewed, or contained many exact zero observations. In particular, some sparse materials showed long periods of absence followed by sudden positive values.

Because of this heterogeneity, I did not want to treat every material as if it followed the same statistical pattern.

The main idea behind my approach was:

> **If the distributional behaviour of each material can be understood, a forecasting model can be chosen that better reflects that behaviour.**

This motivated the use of **Tweedie regression**, which is well suited to non-negative, skewed data containing both zeros and positive continuous values.

For the most intermittent materials, I extended this idea further using a **Tweedie-Hurdle model**, which separates the forecasting problem into two questions:

1. **Will the material occur?**
2. **If it occurs, how much will be present?**

---

## Data

The project used two daily data sources:

### Incoming mass

A daily measure of the total incoming material mass was available and used as additional explanatory information.

### Material composition

The second dataset contained the normalized daily shares of the individual material fractions.

Together, the fractions describe the material composition for a given day, with the fraction values summing approximately to one.

The forecasting target was therefore not simply an independent numeric series, but part of a **compositional time-series problem**.


---

# Tweedie Regression

## Why Tweedie?

Tweedie regression is a generalized linear modelling approach designed for non-negative response variables.

For a suitable variance-power parameter, the Tweedie distribution can represent a mixture of:

- exact zero values, and
- positive continuous values.

This makes it particularly suitable for material fractions that are:

- non-negative,
- right-skewed,
- heteroscedastic,
- and sometimes exactly zero.

Instead of assuming a normal distribution for every target, Tweedie regression allows the model to better reflect the observed distribution of the material fractions.

---

## Forecasting Strategy

A separate forecasting problem was considered for each:

- material fraction, and
- forecast horizon.

The model was designed as a **direct multi-lead forecasting approach**. For a given horizon, all future leads were predicted from information available at the same forecast origin.

Predictions from earlier future steps were therefore not recursively used to generate later predictions.

This helps avoid error propagation and ensures that the prediction setup stays consistent with the information that would have been available at the time of forecasting.

---

## Feature Engineering

The forecasting pipeline combined several sources of information.

### Historical material behaviour

Historical material values were used to represent recent and longer-term behaviour, including previous values, recent averages and variability, and longer-term temporal patterns.

### Incoming mass

The total incoming mass was used as an additional signal describing the overall amount of material entering the process. Historical behaviour of the incoming mass was also included.

### Calendar and seasonal information

Calendar and seasonal information was included to represent recurring temporal behaviour and possible effects related to working days, holidays, and annual patterns.

### Composition information

The material fractions are components of the same overall composition.

For this reason, the model also used features describing the state of the complete material mixture, including measures of concentration, diversity, relative importance, and overall composition.

The aim was to give the model information not only about the target material itself, but also about the overall structure of the material stream.

### Intermittency information

For sparse fractions, historical occurrence behaviour was also important.

Features were therefore created to represent concepts such as:

- whether the material was currently absent,
- how long it had been since the material last appeared,
- the most recent positive value,
- and how frequently the material had occurred in recent history.

These features were especially important for materials with frequent zero observations.

---

## Full vs. Reduced Feature Representation

Two different feature representations were evaluated.

### Full feature set

The full representation included historical information from all material fractions, together with mass, calendar, composition, and intermittency information.

### Reduced feature set

The reduced representation focused more strongly on the target material itself while still keeping the relevant mass, calendar, composition, and intermittency information.

The most appropriate representation was selected separately for each forecasting task using time-series validation performance.

---

# Tweedie-Hurdle Model

## Why a Hurdle Model?

Although standard Tweedie regression can statistically represent both zeros and positive continuous values, some materials showed a much stronger intermittent pattern.

This was especially visible for **Fractions 12 and 13**.

For these materials, the forecasting task can naturally be separated into two processes:

### Occurrence

> Will the material appear at all?

### Magnitude

> If it appears, what fraction should be expected?

This motivated the use of a **Tweedie-Hurdle model**.

---

## Stage 1 — Occurrence Model

A logistic regression model estimates the probability that the future fraction will be positive:

\[
P(Y > 0)
\]

This stage focuses only on the **zero / non-zero behaviour** of the material.

---

## Stage 2 — Positive Magnitude Model

For observations where the material is present, a Tweedie regression model estimates the expected positive fraction:

\[
E[Y \mid Y > 0]
\]

The magnitude model is therefore trained only on positive observations.

---

## Final Hurdle Prediction

The occurrence probability is compared with a probability threshold.

If the probability of occurrence is too low, the final prediction is set to zero.

Otherwise, the occurrence probability is combined with the estimated positive magnitude to produce the final forecast.

Conceptually:

```text
Historical information
        │
        ├───────────────┐
        │               │
        ▼               ▼
Occurrence model    Magnitude model
Logistic Regression Tweedie Regression
        │               │
        ▼               ▼
      P(Y>0)       E[Y | Y>0]
        │               │
        └───────┬───────┘
                ▼
         Final Forecast
```

This architecture allows the model to explicitly distinguish between **material absence** and **material quantity**.

---

# Time-Series Validation

A standard random train-test split would not be appropriate for this forecasting problem because it could allow information from the future to influence model selection.

To preserve the chronological structure of the data, I used **expanding-window time-series cross-validation**.

The validation procedure was designed so that:

- training observations always occurred before validation observations,
- observations belonging to the same forecast origin stayed together,
- a horizon-dependent gap was used to reduce the risk of target leakage,
- preprocessing steps were fitted only on the training portion of each fold,
- and model selection was based only on historical data.

After the final model configuration was selected, the model was fitted on the available pre-test development data and kept fixed during the final out-of-sample evaluation.

The test period was then evaluated using rolling forecast blocks, where newly observed values became available only after a forecast block had been completed.

---

# Results

## Standard Tweedie

The Tweedie model improved upon the baseline for **33 of the 52 material-horizon combinations**.

The strongest fraction-level coverage occurred at:

- **10 working days:** 9 of 13 fractions improved
- **66 working days:** 9 of 13 fractions improved

Several fractions showed consistent improvements across all four forecast horizons.

The largest reported individual improvement from the standard Tweedie model was approximately **45.4%**, observed for Fraction 6 at the 132-working-day horizon.

At the same time, the model did not perform equally well for every material, which confirmed the original motivation for using a target-specific modelling strategy.

---

## Tweedie-Hurdle

The Hurdle extension was evaluated specifically for the sparse Fractions 12 and 13.

The strongest improvements were observed for **Fraction 13**.

The Hurdle model improved Fraction 13 at multiple horizons, including:

- approximately **54.0%** improvement at 10 working days,
- approximately **61.2%** improvement at 66 working days,
- approximately **66.8%** improvement at 132 working days.

For Fraction 12, the Hurdle approach was beneficial only at the longest forecast horizon.

This demonstrates that even among sparse materials, the same modelling strategy is not automatically optimal for every target.

---

# Main Takeaway

The main lesson from this project is that **forecasting performance depends strongly on the statistical behaviour of the target material**.

A single forecasting formulation was not equally effective for every fraction.

Standard Tweedie regression was effective for many irregular, non-negative material fractions, while the Tweedie-Hurdle extension was particularly useful for highly intermittent targets where occurrence and magnitude behaved like two different processes.

> **Understanding the distribution and behaviour of the target can be as important as choosing the forecasting algorithm itself.**

---

# Technologies

The project was implemented in Python using tools including:

- pandas
- NumPy
- scikit-learn
- Matplotlib
- generalized linear models
- logistic regression
- time-series cross-validation

---



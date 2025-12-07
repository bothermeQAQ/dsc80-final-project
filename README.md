# Do LCK games really last longer?  
### An analysis of 2018 professional League of Legends matches

**Name:** Yonghao Wang  
**Course:** DSC 80 – Final Project  

---

## Introduction

This project uses the **2018 League of Legends professional match dataset** from Oracle’s Elixir.  
Each row corresponds to either a team or a player in a professional match and includes
detailed in-game and post-game statistics.

The main question I investigate is:

> **Do different professional leagues play significantly longer games,  
> or are game lengths roughly the same across regions?**

In particular, I compare **LCK** (Korea) to other major leagues (LPL, LCS, LEC), and build
a prediction model that tries to predict which team will win based on early-game
statistics (gold, XP, CS, kills, etc. at 10 minutes).

---

## Files in this Repository

- `template.ipynb` – main Jupyter notebook with all steps of the project  
- `2018_LoL_esports_match_data_from_OraclesElixir.csv` – raw match data  
- `README.md` – this project page (rendered by GitHub Pages)

---

## View the Notebook Online

- 📓 **GitHub view:**  
  https://github.com/bothermeQAQ/dsc80-final-project/blob/main/template.ipynb

- 📊 **Rendered via nbviewer (recommended):**  
  https://nbviewer.org/github/bothermeQAQ/dsc80-final-project/blob/main/template.ipynb

---

## Data Cleaning and Exploratory Data Analysis

In the notebook, I first restrict attention to **team-level** rows, where each row
represents a single team in a single professional game (`position == "team"`). This
gives about 13,000 team–game observations across all professional leagues in 2018.

I create a new column

- `gamelength_min = gamelength / 60`

so that game length is measured in **minutes** instead of seconds. I keep the 10-minute
statistics (gold, XP, CS, kills, assists, deaths, etc.) even when some of them are
missing, and handle missingness explicitly in a later section. For most of the later
analysis I also focus on the four major leagues: **LCK, LPL, LCS, and LEC**.

From the exploratory data analysis (EDA), a few key patterns emerge:

- The distribution of **game length** is right-skewed. Most games last between roughly
  25 and 40 minutes, with a long tail of very long games.
- The distribution of **team gold at 10 minutes** is much tighter, suggesting that
  early-game economies vary, but not as dramatically as overall game length.
- When I plot game length by league, the box plots show noticeable differences across
  regions: for example, LCK games are typically longer than LPL games.
- When I compare **gold at 10 minutes for winning vs. losing teams**, the winning
  distribution is clearly shifted to the right. Teams that are ahead in gold at
  10 minutes are more likely to win, although the two distributions still overlap.

To quantify the league differences more precisely, I compute an aggregated summary for
the four major leagues:

- LCK: mean game length ≈ **36.6** minutes, median ≈ 35.6 minutes  
- LPL: mean game length ≈ **34.5** minutes, median ≈ 33.6 minutes  
- LEC: mean game length ≈ **35.0** minutes  
- LCS: mean game length ≈ **33.1** minutes  

These summaries support the idea that LCK tends to play noticeably longer games than
other major leagues.

<!-- Example: you can embed interactive Plotly figures generated in the notebook, e.g.:

<iframe
  src="assets/gamelength_hist.html"
  width="800"
  height="500"
  frameborder="0">
</iframe>

-->

---

## Assessment of Missingness

The column I focus on in the missingness analysis is **`goldat10`**, which records a
team’s total gold at the 10-minute mark. This variable is important for the prediction
task later, so understanding when it is missing is crucial.

Overall, about **18%** of rows in the dataset are missing `goldat10`. I define a binary
indicator

- `goldat10_missing = 1` if `goldat10` is missing, and `0` otherwise

and examine how this missingness relates to other variables.

First, I consider whether the missingness of `goldat10` might be **Not Missing At Random
(NMAR)**. For example, it is plausible that `goldat10` is more likely to be missing in
games where data collection is incomplete or corrupted, which is reflected in the
`datacompleteness` column.

To investigate this more formally, I run **two permutation tests** on the difference in
missing rate:

1. **Potentially related variable – `datacompleteness`**

   I compare the missing rate of `goldat10` between games labeled as `"complete"` and
   `"partial"` in the `datacompleteness` column. The observed difference in missing rate
   is about **0.94** (almost no missingness in `"complete"` games but very high
   missingness in `"partial"` games), and the permutation test yields a p-value close
   to **0**. This provides strong evidence that `goldat10` missingness depends on
   `datacompleteness`.

2. **Potentially unrelated variable – `side`**

   I also test whether `goldat10_missing` is associated with which **side** a team is on
   (`"Blue"` vs `"Red"`). Here, the observed difference in missing rate is essentially
   **0**, and the permutation test p-value is close to **1.0**, suggesting no meaningful
   relationship between missingness and side.

These results support the idea that the missingness of `goldat10` is driven by data
quality (complete vs partial games) rather than by in-game factors like which side the
team is playing on. The mechanism is definitely **not completely random**, and is at
least partly explained by observable game characteristics.

---

## Hypothesis Testing

I conduct a formal hypothesis test to investigate whether **LCK games are longer on
average than games in other major leagues**.

- **Group A:** games from LCK  
- **Group B:** games from all other major leagues (LPL, LCS, LEC)  
- **Quantity of interest:** mean game length in minutes at the team–game level  

I use the following hypotheses:

- **Null hypothesis \(H_0\):** There is **no difference** in mean game length between
  LCK and the other major leagues.
- **Alternative hypothesis \(H_1\):** There **is** a difference in mean game length
  between LCK and the other major leagues.

The test statistic is the **difference in mean game length** between the two groups
(mean of LCK minus mean of non-LCK). I approximate the null distribution using a
permutation test, repeatedly shuffling the league labels and recomputing the difference
in means.

- The observed difference is about **+2.04** minutes (LCK games are longer).  
- The permutation test p-value is essentially **0** (p < 0.001).

Because the p-value is far below 0.05, I **reject** the null hypothesis. There is very
strong statistical evidence that LCK games last longer on average than games in the
other major leagues.

A similar test for LPL vs. others finds that LPL games are about **1.94 minutes
shorter** than other major leagues, again with a p-value close to zero.

---

## Framing a Prediction Problem

The second part of the project is a prediction task. The goal is to predict **which
team will win** a game using only information available at the 10-minute mark.

- **Target:** `result` (1 if the team won the game, 0 otherwise), at the team–game level.
- **Features available at prediction time:** early-game statistics at 10 minutes (gold,
  XP, CS, kills, assists, deaths for both the team and its opponent), plus contextual
  variables such as league and side (blue vs red).
- **Prediction timing:** the model is only allowed to use information from the **first
  10 minutes** of the game; anything that happens after 10 minutes is excluded to avoid
  “peeking into the future”.
- **Evaluation metric:** I use **accuracy** as the primary metric. The win vs loss
  classes are reasonably balanced, and overall prediction accuracy is an intuitive and
  easy-to-interpret summary of model performance.

This setup mimics the real-world problem of trying to guess the eventual winner early in
the game based on the current state and known context.

---

## Baseline Model

As a baseline, I fit a **logistic regression** model on the team-level data. The
features include:

- Early-game gold, XP, and CS at 10 minutes for the team and its opponent  
- Early-game kills, assists, and deaths at 10 minutes for the team and its opponent  
- One-hot encoded indicators for league and side  

I randomly split the data into a training set (80%) and a test set (20%), stratified by
`result`. On this split, the baseline model achieves:

- **Training accuracy:** ≈ **0.673**  
- **Test accuracy:** ≈ **0.688**

This is substantially better than random guessing (which would be around 0.50 accuracy)
and shows that early-game statistics contain useful information about the final outcome.
However, the model treats each team’s raw gold, XP, and CS separately, and does not
explicitly capture how far ahead or behind a team is relative to its opponent.

---

## Final Model

To improve on the baseline, I design a **final model** that incorporates additional
feature engineering and a more flexible classifier.

The main changes are:

- I construct **difference features** that measure the team’s lead or deficit at
  10 minutes, such as  
  - `gold_diff10 = goldat10 − opp_goldat10`  
  - `xp_diff10 = xpat10 − opp_xpat10`  
  - `cs_diff10 = csat10 − opp_csat10`  
  - `kills_diff10 = killsat10 − opp_killsat10`  
  - `deaths_diff10 = deathsat10 − opp_deathsat10`
- I keep the original early-game statistics and contextual variables, and I train a
  **Random Forest classifier** with tuned hyperparameters (e.g., 300 trees, minimum
  samples per split/leaf, etc.).

Using the same type of 80/20 train–test split, the final model achieves:

- **Training accuracy:** ≈ **0.914**  
- **Test accuracy:** ≈ **0.690**

The final model’s test accuracy is slightly higher than the baseline’s (0.690 vs 0.688),
while its training accuracy is much higher. This suggests that the Random Forest with
difference features is more expressive and can fit the data more closely, but many of
the easily exploitable patterns were already captured by the simpler logistic
regression. Still, from an interpretability standpoint, the difference features more
directly encode the idea of a “lead” at 10 minutes, which aligns well with how players
and analysts think about game state.

---

## Fairness Analysis

In the fairness analysis, I ask whether the final model is **equally accurate across
different professional leagues**, focusing on games from the Korean league LCK compared
to games from all other major leagues.

- **Sensitive attribute:** `league`  
- **Group A:** games from **LCK**  
- **Group B:** games from **all non-LCK leagues** (LPL, LCS, LEC, etc.)  
- **Fairness notion:** **accuracy parity** — the model should have similar accuracy for
  both groups on the held-out test set.

Using the test set predictions from the final model, I compute the model’s accuracy
separately for LCK and non-LCK games:

- Accuracy on LCK games: about **0.68**  
- Accuracy on non-LCK games: about **0.69**

The observed difference in accuracy (LCK minus non-LCK) is therefore about **−0.01**,
meaning that the model is slightly less accurate on LCK games.

To make this analysis more formal, I perform a **permutation test** on the difference in
accuracy. Under the null hypothesis that the model has the same accuracy for both
groups, I repeatedly shuffle the LCK / non-LCK labels in the test set and recompute the
difference in accuracy.

- Observed difference in accuracy: ≈ **−0.011**  
- Permutation test p-value: ≈ **0.769**

Because this p-value is much larger than 0.05, I **do not reject** the null hypothesis
that the model has the same accuracy for LCK and non-LCK games. In other words, there is
no strong statistical evidence that the final model treats LCK games differently from
games in other leagues, at least under this accuracy-parity definition.

This does not prove that the model is perfectly fair in every sense, but it suggests
that, in terms of overall accuracy, the model does not have a large or statistically
significant performance gap between LCK and non-LCK games.

---

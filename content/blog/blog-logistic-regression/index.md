---
title: 📈 The Logic Behind Logistic Regression A Start-to-Finish Guide
summary: .
date: 2023-12-19
authors:
  - andreshernandez
tags:
  - Statistics
  - Logistic Regression
image:
  caption: 'Logistic Regression'
---

Ever needed to predict a "Yes" or "No" answer? Welcome to the world of classification. While there are many tools for the job, one of the most fundamental is **Logistic Regression**.

At its core, it's a version of linear regression, but for a specific-but-common problem: when your target variable has only two possible outcomes (a "dichotomous" variable).

For example:
- ✅ Yes or No
- 🏆 Win or Lose
- 👍 True or False
- 🩺 Healthy or Sick

But if we have linear regression, why do we need a whole new method?

### 🤔 Why Not Just Use Linear Regression?

It's a great question! Using standard linear regression for a "Yes/No" problem (usually coded as 1 and 0) breaks down for a few key reasons:

1. It Predicts "Impossible" Values: Linear regression doesn't know the bounds are 0 and 1. It will happily predict a value of 1.5 or -0.2. This doesn't make sense as a probability.
2. The "Straight Line" Problem: Linear regression assumes a straight-line (linear) relationship. But probabilities don't work that way. Think about "hours of study" vs. "passing an exam." Going from 0 to 1 hour of study has a huge impact. Going from 10 to 11 hours (when you're already likely to pass) has a much smaller impact. The relationship isn't a straight line—it's an "S" curve.
3. It Violates Key Assumptions: For the statisticians out there, this method violates the assumptions of normality and homoscedasticity (equal variances).

We need a function that takes any number (from $-\infty$ to $+\infty$) and squishes it into a clean probability between 0 and 1.

### 💡 The 'Magic' Fix: The Logit Function

Instead of predicting probability directly, logistic regression predicts the log-odds of the outcome. This sounds complex, but it's just two simple steps.

#### Step 1: Understanding "Odds"

First, we talk about odds, which you might know from betting. It's just the probability of an event happening divided by the probability of it not happening.
Odds = $p / (1 - p)$

- If the probability of winning ($p$) is 0.5 (a 50/50 chance), the odds are $0.5 / 0.5 = 1$.
- If the probability ($p$) is 0.9 (a 90% chance), the odds are $0.9 / 0.1 = 9$.
- Unlike probability, odds don't have a ceiling. As $p$ gets closer to 1, the odds shoot toward infinity.

#### Step 2: The "Logit" Transformation

Odds go from 0 to $+\infty$. By taking the natural logarithm (ln) of the odds, we get the logit.

##### The Logit Function 

($L_i$):$$L_i = \ln(\frac{p_i}{1-p_i})$$

This one simple trick solves our problem! The logit function takes a probability (0 to 1), turns it into odds (0 to $+\infty$), and then transforms that into a new value that can range from $-\infty$ to $+\infty$.

**This is the key**: We can now use this $L_i$ value as the dependent variable in a normal linear regression equation!

$$L_i = \beta_0 + \beta_1X_1 + \dots + \beta_kX_k$$

##### Getting Back to Probability

Of course, we don't care about the "log-odds"—we want a probability! By solving that equation for $p_i$, we get the famous Sigmoid Function, which is the final logistic regression model.

##### The Logistic Regression Model:

$$p_i = \frac{e^{\beta_0 + \beta_1X_1 + \dots + \beta_kX_k}}{1 + e^{\beta_0 + \beta_1X_1 + \dots + \beta_kX_k}}$$

This equation gives us the "S" curve we needed. It takes our linear model's output and squishes it perfectly between 0 and 1.

#### Decoding the Results: How to Interpret Coefficients

This is the trickiest part for new users. In logistic regression, the coefficient ($\beta_i$) is the change in the log-odds for a one-unit change in the variable $X_i$.

That's not very intuitive. To make it make sense, we take the exponent of the coefficient: $e^{\beta_i}$. This new value is the odds ratio.

Here’s what $e^{\beta_i}$ tells you:

- If $\beta_i = 0$, then $e^{\beta_i} = 1$. The odds don't change.
- If $\beta_i > 0$, then $e^{\beta_i} > 1$. The odds increase.
- If $\beta_i < 0$, then $e^{\beta_i} < 1$. The odds decrease.

#### The Practical "So What?"

To find the percentage change in odds, use this simple formula:

Percentage Change = $(e^{\beta_i} - 1) \times 100\%$

Let's use your examples:

- If $e^{\beta_i} = 1.5$, the percentage change is $(1.5 - 1) \times 100 = 50\%$.
Interpretation: "For a one-unit change in $X_i$, the odds of the outcome occurring increase by 50%."

- If $e^{\beta_i} = 0.5$, the percentage change is $(0.5 - 1) \times 100 = -50\%$.
Interpretation: "For a one-unit change in $X_i$, the odds of the outcome occurring decrease by 50%."


#### 📊 Are These Results Even Significant?

Just because a variable has a coefficient, it doesn't mean it's actually helping the model.
- The Wald Test: This test checks if individual coefficients ($\beta_i$) are significantly different from zero. If not, that variable might just be noise.
- Likelihood-Ratio Test: This is a statistical test used to compare the fit of your new model (with all its variables) to a "null" model (a model with no variables) to see if your model as a whole is a significant improvement.

A Quick Warning: Tests of significance for samples less than 100 are not reliable.

#### 🎯 How Good Is the Model? (Goodness of Fit)
Once you have your model, how do you know if it's any good?
- Standardized Coefficients: If you want to compare the "importance" of different variables (e.g., is "age" or "income" a stronger predictor?), you need to standardize their coefficients, as they are measured on different scales.
- Log-Likelihood: This is the core of how the model is "fit" using Maximum Likelihood Estimation (MLE). The model finds the parameters ($\beta$) that maximize the probability of observing the data you have. The log-likelihood is a negative number, and the closer it is to 0, the better the model's fit.

#### ✅ Conclusion: What You've Learned

That's it! We've untangled the logic of logistic regression. We started with a simple "Yes/No" problem, saw why linear regression fails, and used the logit function to transform the problem.

We learned how to interpret the results as odds ratios (e.g., "a 50% increase in the odds") and how to check if our model is statistically significant.

In the next article, I will provide a real-life example to explore the logistic regression algorithm in Python.

### References

Most of the content was reviewed in detail from:

1. Pampel, F. C. (2000). Quantitative Applications in the Social Sciences: Logistic regression Thousand Oaks, CA: SAGE Publications Ltd doi: 10.4135/9781412984805. URL: http://methods.sagepub.com/book/logistic-regression/n1.xml
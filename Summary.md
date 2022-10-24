---
output:
  pdf_document: default
  html_document: default
---

# Summary For [Robyn](https://facebookexperimental.github.io/Robyn/docs/analysts-guide-to-MMM)

## Feature Selection

1.  [Prophet](https://facebook.github.io/prophet/) for seasonality decomposition.

    -   national holiday list: **`data("dt_prophet_holidays")`** :all the various holidays/events will be aggregated and modelled as one variable

    -   customized event: **`context_vars`** : make sure that no duplicate information across the two sources

2.  [Model window](https://facebookexperimental.github.io/Robyn/docs/features/#continuous-reporting):

    -   use the full data history (e.g. 4 years) to more accurately determine the trend, seasonality and holiday effects.
    -   defining a specific range (e.g. 18 months) for media, organic and contextual variables can be a better option as it more accurately reflects your current business and marketing scenarios
    -   data sparsity: the recommended ratio of data points is 1 independent variable : 10 observations.
    -   if not sure about which model window to use,experiment with several different windows as part of an iterative process.

3.  Model design:

    -   Control the signs of coefficients: directly control the sign of coefficients in the parameter setting to be either 'positive' or 'negative'.
    -   Categorize variables into Paid Media, Organic and Context variables:
        -   paid_media_vars: Any media variables with a clear marketing spend falls into this category.
        -   organic_vars: Any marketing activities without a clear marketing spend fall into this category. Typically this may include newsletters, push notifications, social media posts, etc. ***As organic variables are expected to have similar carryover (adstock) and saturating behavior as paid media variables, similar transformation techniques will be also applied to organic variables.***
        -   context_vars: These include other variables that are not paid or organic media that can help explain the dependent variable. The most common examples of context variables include competitor activity, price & promotional activity, macroeconomic factors like unemployment rate, etc. These variables will not undergo any transformation techniques and are expected to have a direct impact on dependent variables.
    -   When using exposure variables like impressions instead of spend in paid_media_vars, Robyn fits a nonlinear model with ***Michaelis Menten*** function between exposure and spend to establish the spend-exposure relationship.
    -   Rule of Thumb for the number of input variables: for an n \* p data-frame (where n = num of rows to be modelled and p = num of columns), n should be about 7-10x more than p where we recommend going no more than 10x. If unsure, it is worthwhile to try multiple designs as part of an iterative process. *Ridge regressions in Robyn (covered in a later section) deals with intercorrelated explanatory variables and prevents overfitting.*

4.  [Data Transformation Techniques](https://facebookexperimental.github.io/Robyn/docs/features/#variable-transformations):

    -   Adstock reflects the theory that the effects of advertising can lag and decay following an initial exposure.

        -   Geometric: simple but too simple and often not suitable for digital media transformations. It only requires one parameter called 'theta' that can be quite intuitive. For example, an ad-stock of theta = 0.75 means that 75% of the impressions in period 1 were carried over to period 2.

        -   The Weibull survival function / Weibull distribution provides significantly more flexibility in the shape and scale of the distribution. However, Weibull can take more time to run than Geometric, as it optimizes two parameters (i.e. shape and scale) and it can often be difficult to explain to non-technical stakeholders without charting.

        -   Weibull CDF adstock:The shape parameter controls the shape of the decay curve, where the recommended bound is c(0.0001, 2). Note that the larger the shape, the more S-shape and the smaller the shape, the more L-shape.The scale parameter controls the inflexion point of the decay curve. We recommend a very conservative bound of c(0, 0.1), because scale can significantly increase the adstock's half-life.

        -   Weibull PDF adstock:The difference to Weibull CDF is that Weibull PDF offers lagged effects. When shape \> 2, the curve peaks after x = 0 and has NULL slope at x = 0, enabling lagged effect and sharper increase and decrease of adstock, while the scale parameter indicates the limit of the relative position of the peak at x axis.

            (i) When 1 \< shape \< 2, the curve peaks after x = 0 and has infinite positive slope at x = 0, enabling lagged effect and slower increase and decrease of adstock, while scale has the same effect as above;

            (ii) When shape = 1, the curve peaks at x = 0 and reduces to exponential decay, while scale controls the inflexion point;

            (iii) When 0 \< shape \< 1, the curve peaks at x = 0 and has increasing decay, while scale controls the inflexion point.

            (iv) While all possible shapes are relevant, we recommend c(0.0001, 10) as bounds for shape. When only strong lagged effects are of interest, we recommend c(2.0001, 10) as bound for shape.

            (v) When it comes to scale, we recommend a conservative bound of c(0, 0.1) for scale. - Saturation: The theory of saturation entails that each additional unit of advertising exposure increases the response, but at a declining rate.

        -   Saturation:A Hill function is a two-parametric function in Robyn with alpha and gamma. Alpha controls the shape of the curve between exponential and s-shape. We recommend a bound of c(0.5, 3) - note that the larger the alpha, the more S-shape and the smaller the alpha, the more C-shape. Gamma controls the inflexion point. We recommend a bound of c(0.3, 1) - note that the larger the gamma, the later the inflection point in the response curve.

## Modeling Techniques

### Ridge Regression

You can control the amount of penalty by controlling the lambda(λ) parameter in Ridge regression. When lambda increases, the impact of the shrinkage grows, leading to a decreased variance but increased bias. In contrast, when lambda decreases, the impact of the penalty decreases, leading to increased variance but decreased bias.Robyn is taking lambda.1se, the lambda which gives an error that gives one standard error away from the minimum error.

### Model Selection

-   Model fit: Aim to minimize the model's prediction error (NRMSE or normalized root-mean-square error)
-   Business fit: Aim to minimize decomposition distance (DECOMP.RSSD, decomposition root-sum-square distance). The distance accounts for a relationship between spend share and a channel's coefficient decomposition share. If the distance is too far, its result can be too unrealistic - e.g. media activity with the smallest spending gets the largest effect.

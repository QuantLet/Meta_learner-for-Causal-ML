<div style="margin: 0; padding: 0; text-align: center; border: none;">
<a href="https://quantlet.com" target="_blank" style="text-decoration: none; border: none;">
<img src="https://github.com/StefanGam/test-repo/blob/main/quantlet_design.png?raw=true" alt="Header Image" width="100%" style="margin: 0; padding: 0; display: block; border: none;" />
</a>
</div>

```
Name of Quantlet: Causal-Forest implementation

Published in: CATE meets MLE - A tutorial

Description: Causal Forest implementation from the GRF package to estimate the conditional average treatment effect (CATE) via a variety of machine learning (ML) methods.

The data structure has to be the following: 
- y = outcome variable
- d = treatment variable
- covariates = the covariates to map on d or y
- learners = the machine learning methods to use in the [SuperLearner package](https://cran.r-project.org/web/packages/SuperLearner/vignettes/Guide-to-SuperLearner.html).
- df_aux and df_main = the train and test-set. df_aux is used for training the nuisance functions while df_main is used to estimate the CATE.

Keywords: CATE, ML, causal forest, GRF, causal-inference, treatment

Author: Daniel Jacob

See also: https://grf-labs.github.io/grf/index.html

Submitted: 10.02.2021

```

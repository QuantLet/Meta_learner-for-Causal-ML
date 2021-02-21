[<img src="https://github.com/QuantLet/Styleguide-and-FAQ/blob/master/pictures/banner.png" width="888" alt="Visit QuantNet">](http://quantlet.de/)

## [<img src="https://github.com/QuantLet/Styleguide-and-FAQ/blob/master/pictures/qloqo.png" alt="Visit QuantNet">](http://quantlet.de/) **DR-learner** [<img src="https://github.com/QuantLet/Styleguide-and-FAQ/blob/master/pictures/QN2.png" width="60" alt="Visit QuantNet 2.0">](http://quantlet.de/)

```yaml


Name of Quantlet: DR-learner

Published in: 'CATE meets MLE - A tutorial'

Description: Doubly-Robust method to estimate the conditional average treatment effect (CATE) via a variety of machine learning (ML) methods. 

The data structure has to be the following: 

- y = outcome variable
- d = treatment variable
- covariates = the covariates to map on d or y
- learners = the machine learning methods to use in the [SuperLearner package](https://cran.r-project.org/web/packages/SuperLearner/vignettes/Guide-to-SuperLearner.html).
- df_aux and df_main = the train and test-set. df_aux is used for training the nuisance functions while df_main is used to estimate the CATE. 


Keywords: 'CATE, ML, doubly-robust, causal-inference, treatment'

Author: 'Daniel Jacob'

See also: 'https://arxiv.org/abs/2004.14497'

Submitted:  '10.02.2021'

```

### R Code
```r

install.packages("xgboost", repos=c("http://dmlc.ml/drat/", getOption("repos")), type="source")
vec.pac= c("SuperLearner", "gbm", "glmnet","ranger","caret")

lapply(vec.pac, require, character.only = TRUE) 

# df_aux is the training set
# df_main is the test set where the CATE is estimated on
# covariates are the pre-treatment covariates 


#Learner Library for the SuperLearner:
learners <- c( "SL.glmnet","SL.xgboost", "SL.ranger","SL.lm","SL.mean")

#CV Control for the SuperLearner
control <- SuperLearner.CV.control(V=5) # The cross-validation parameter 

R <- 10 # The number of repetitions for cross-fitting and median averaging. 

DR_learner <- function(df_aux,df_main,covariates,learners,R) {
  
    est_cate_DR <- matrix(0,nrow(df_main),R)
  
  
  
    for(r in 1:R){
    
    ind <- createDataPartition(df_aux[,"d"], p = .5, list = FALSE)
    
    df_aux1<- as.data.frame(df_aux[ ind,])
    df_aux2 <- as.data.frame(df_aux[-ind,])
    
    ## DR-learner 
    # Train a classification model to get the propensity scores
    p_mod <- SuperLearner(Y = df_aux1$d, X = df_aux1[,covariates], newX = df_aux2[,covariates], SL.library = learners,
                          verbose = FALSE, method = "method.NNLS", family = binomial(),cvControl = control)
    
    p_hat <- p_mod$SL.predict
    p_hat = ifelse(p_hat<0.025, 0.025, ifelse(p_hat>.975,.975, p_hat)) # Overlap bounding
    
    # Prop-Score for df_main
    p_hat_main <- predict(p_mod,df_main[,covariates])$pred
    
    # Split the training data into treatment and control observations
    aux_1 <- df_aux1[which(df_aux1$d==1),]
    aux_0 <- df_aux1[which(df_aux1$d==0),]
    
    # Train a regression model for the treatment observations
    m1_mod <- SuperLearner(Y = aux_1$y, X = aux_1[,covariates], newX = df_aux2[,covariates], SL.library = learners,
                           verbose = FALSE, method = "method.NNLS",cvControl = control)
    
    m1_hat <- m1_mod$SL.predict
    
    # Train a regression model for the control observations
    m0_mod <- SuperLearner(Y = aux_0$y, X = aux_0[,covariates], newX = df_aux2[,covariates], SL.library = learners,
                           verbose = FALSE, method = "method.NNLS",cvControl = control)
    
    m0_hat <- m0_mod$SL.predict
    
    
    
    # Apply the doubly-robust estimator 
    y_mo <- (m1_hat - m0_hat) + ((df_aux2$d*(df_aux2$y -m1_hat))/p_hat) - ((1-df_aux2$d)*(df_aux2$y - m0_hat)/(1-p_hat))
    
    
    
    
    
    a  <- tryCatch({
      dr_mod <- SuperLearner(Y = y_mo, X = df_aux2[,covariates], newX = df_main[,covariates], SL.library = learners,
                             verbose = FALSE, method = "method.NNLS",cvControl = control)
      
      score_dr <- dr_mod$SL.predict
      a <- score_dr
      
      
    },error=function(e){
      
      mean_score <- mean(y_mo)
      score_dr <- rep.int(mean_score, times = nrow(df_main))
      a <- score_dr
      return(a)
    })
    
    score_dr <- a
    
    ######################
    
    est_cate_DR[,r] <- score_dr 
  }
  
  return(apply(est_cate_DR,1,median))
}


```

automatically created on 2021-02-21
[<img src="https://github.com/QuantLet/Styleguide-and-FAQ/blob/master/pictures/banner.png" width="888" alt="Visit QuantNet">](http://quantlet.de/)

## [<img src="https://github.com/QuantLet/Styleguide-and-FAQ/blob/master/pictures/qloqo.png" alt="Visit QuantNet">](http://quantlet.de/) **X-learner** [<img src="https://github.com/QuantLet/Styleguide-and-FAQ/blob/master/pictures/QN2.png" width="60" alt="Visit QuantNet 2.0">](http://quantlet.de/)

```yaml


Name of Quantlet: X-learner

Published in: 'CATE meets MLE - A tutorial'

Description: X-learner method to estimate the conditional average treatment effect (CATE) via a variety of machine learning (ML) methods.

The data structure has to be the following: 

- y = outcome variable
- d = treatment variable
- covariates = the covariates to map on d or y
- learners = the machine learning methods to use in the [SuperLearner package](https://cran.r-project.org/web/packages/SuperLearner/vignettes/Guide-to-SuperLearner.html).
- df_aux and df_main = the train and test-set. df_aux is used for training the nuisance functions while df_main is used to estimate the CATE. 


Keywords: 'CATE, ML, X-learner, causal-inference, treatment'

Author: 'Daniel Jacob'

See also: 'https://www.pnas.org/content/116/10/4156'

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

X_learner <- function(df_aux,df_main,covariates,learners,R){

  est_cate_X <- matrix(0,nrow(df_main),R)
  
  
  
    for(r in 1:R){
    
    ind <- createDataPartition(df_aux[,"d"], p = .5, list = FALSE)
    
    df_aux1<- as.data.frame(df_aux[ ind,])
    df_aux2 <- as.data.frame(df_aux[-ind,])
  
  p_mod <- SuperLearner(Y = df_aux$d, X = df_aux[,covariates], newX = df_main[,covariates], SL.library = learners,
                        verbose = FALSE, method = "method.NNLS", family = binomial(),cvControl = control)
  
  # Prop-Score for df_main
  p_hat_main <- p_mod$predictions
  p_hat_main = ifelse(p_hat_main<0.025, 0.025, ifelse(p_hat_main>.975,.975, p_hat_main)) # Overlap bounding
  
  aux_1 <- df_aux1[which(df_aux1$d==1),]
  aux_0 <- df_aux1[which(df_aux1$d==0),]
  
  m1_mod <- SuperLearner(Y = aux_1$y, X = aux_1[,covariates], newX = df_aux2[,covariates], SL.library = learners,
                         verbose = FALSE, method = "method.NNLS",cvControl = control)
  
  m1_hat <- m1_mod$SL.predict
  
  m0_mod <- SuperLearner(Y = aux_0$y, X = aux_0[,covariates], newX = df_aux2[,covariates], SL.library = learners,
                         verbose = FALSE, method = "method.NNLS",cvControl = control)
  
  m0_hat <- m0_mod$SL.predict
  
  
  
  
  tau1 <- df_aux2[which(df_aux2$d==1),"y"] - m0_hat[which(df_aux2$d==1),]
  
  tau0 <- m1_hat[which(df_aux2$d==0),]  -  df_aux2[which(df_aux2$d==0),"y"]
  
  
  a1  <- tryCatch({
    tau1_mod <- SuperLearner(Y = tau1, X = df_aux2[which(df_aux2$d==1),covariates], newX = df_main[,covariates], SL.library = learners,
                             verbose = FALSE, method = "method.NNLS",cvControl = control)
    
    score_tau1 <- tau1_mod$SL.predict
    a1 <- score_tau1
    
    
  },error=function(e){
    
    mean_score <- mean(tau1)
    score_tau1 <- rep.int(mean_score, times = nrow(df_main))
    a1 <- score_tau1
    return(a1)
  })
  
  score_tau1 <- a1
  
  a0  <- tryCatch({
    tau0_mod <- SuperLearner(Y =tau0, X = df_aux2[which(df_aux2$d==0),covariates], newX = df_main[,covariates], SL.library = learners,
                             verbose = FALSE, method = "method.NNLS",cvControl = control)
    
    score_tau0 <- tau0_mod$SL.predict
    a0 <- score_tau0
    
    
  },error=function(e){
    
    mean_score <- mean(tau0)
    score_tau0 <- rep.int(mean_score, times = nrow(df_main))
    a0 <- score_tau0
    return(a0)
  })
  
  score_tau0 <- a0
  
  
  
  tau_weight <- p_hat_main*score_tau0 + (1-p_hat_main)*score_tau1
  
  
    est_cate_X[,r] <- tau_weight 
  }
  
  return(apply(est_cate_X,1,median))
  }

```

automatically created on 2021-02-21
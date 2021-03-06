German Credit Classification
================

This document demonstrates the classification of bank loans into good or bad loans using machine learning techniques. The data used here is the Statlog German Credit Data, a widely studied data set in machine learning.

I trained three first-level models using support vector machine with gaussian kernel, gradient boosted decision tree and elastic net. The three models are then blended together using logistic regression to create the final model

A weighted accuracy is used to take into account the cost matrix that comes with the data set. Using a particular seed for reproducibility, the 10-fold cross-validation estimate of the weighted accuracy of the final model is 0.78. The unweighted accuracy is 0.79. Precision and recall are respectively 0.66 and 0.77.

### Data pre-processing

The version of the German Credit Data included in R's default data sets is already one-hot encoded. This data set includes 61 predictors and 1 output (Good or Bad). In the pre-processing step, I eliminated the linearly dependent predictors and the near zero and zero-variance predictors. The rest of the predictors are scaled and centered before being used.

``` r
library(caret)
```

    ## Loading required package: lattice

    ## Loading required package: ggplot2

``` r
library(corrplot)
data(GermanCredit)
# check if missing data
any(sapply(GermanCredit,function(x)length(which(is.na(x)==T)))!=0)
```

    ## [1] FALSE

``` r
X <- GermanCredit[,-10]
Y <- GermanCredit[,10]

#### Delete linearly dependent ####
X_ldinfo <- findLinearCombos(X)
X <- X[,-X_ldinfo$remove]

#### Preprocess control ####
preproc <- c("nzv","zv","scale","center")
```

### Define a custom performance metric

While the goal here is to predict the nature of the loans (good loan or bad loan) as accurately as possible, it seems clear that erroneously predicting a risky (bad) loan as good loan entails more serious consequences than the other way around. This fact is reflected in the cost matrix that comes with the data set. The cost matrix looks like the following where the rows are the actual outputs and the columns are the predictions.

``` r
cost_m <- matrix(c(0,1,5,0),ncol=2,byrow=TRUE)
colnames(cost_m) <- c("Good","Bad")
rownames(cost_m) <- c("Good","Bad")
cost_m <- as.table(cost_m)
cost_m
```

    ##      Good Bad
    ## Good    0   1
    ## Bad     5   0

To take into account the different costs associated with the two types of error, I created a weighed accuracy which is equivalent to giving 5 times more weights to the positive cases. Note that the positive cases here refer to the bad loans since they are cases where the event of interest (loan default) is present. This weighted accuracy is used to choose the hyperparameters and to evaluate the performance of the models along with other metrics.

``` r
weightedAccu <- function(obs,pred,positive,false_pos_cost,false_neg_cost) {
return(1-(sum(ifelse(obs==positive,
                     false_neg_cost*(obs!=pred),
                     false_pos_cost*(obs!=pred)))/
            (sum(obs==positive)*false_neg_cost+sum(obs!=positive))))  
}

customM <- function(data, lev = NULL, model = NULL) {
  accu_w <- weightedAccu(obs=data[,"obs"],pred=data[,"pred"],
                         positive="Bad",
                         false_pos_cost=1,
                         false_neg_cost=5)
  accu <- mean(data[, "obs"]==data[, "pred"])
  precis <- (sum(data[, "pred"]=="Bad"&data[, "obs"]=="Bad")/sum(data[, "pred"]=="Bad"))
  recall <- (sum(data[, "pred"]=="Bad"&data[, "obs"]=="Bad")/sum(data[, "obs"]=="Bad"))
  specific <- (sum(data[, "pred"]=="Good"&data[, "obs"]=="Good")/sum(data[, "obs"]=="Good"))
  c(WeightedAccuracy=accu_w,Accuracy= accu,Precision= precis,Recall= recall,Specificity = specific)
}
```

### Training control

The German credit Data is unbalanced. As loan default is a relatively rare event, only 30% of the observations in the data set are bad loans. Up-sampling the data is one way to compensate for this imbalance so that the models trained on the data set will be able to learn the characteristics of both types of loans.

``` r
#### Training control ####
fitControl<- trainControl(
  method = "repeatedcv",
  number=10,
  repeats= 5,
  summaryFunction=customM)

fitControlUp <- trainControl(
  method = "repeatedcv",
  number = 10,
  repeats=5,
  sampling="up",
  summaryFunction=customM)
```

### Data spliting

I split the data into a training set and a "blending" set (which is actually another training set). Given the small size of the data set, I did not save any additional data for testing but used directly the cross-validation estimates. As suggested in *Applied Predictive Modeling*, "when the number of samples is not large, a strong case can be made that a test set should be avoided because every sample may be needed for model building. Additionally, the size of the test set may not have sufficient power or precision to make reasonable judgements.(...)Resampling methods, such as cross-validation, can be used to produce appropriate estimates of model performance using the training set."

For reproducibility, a seed is set for the steps of data splitting and cross-validation in this document. In real training, I believe it would be better to do without seeds so that the trained first-level models can have more variabilities coming from the different training and validation sets choice.

``` r
seed <- 3000
set.seed(seed)
Train_indx <- createDataPartition(GermanCredit$Class,p=.75,list=F)
Data_train <- cbind(X[Train_indx,],Class=Y[Train_indx])
Data_blend <- cbind(X[-Train_indx,],Class=Y[-Train_indx])
```

### First-level models training

I chose three models that are based on different principles in the hope that they will produce less correlated results which will be more useful when blended together.

Support vector machine does classification by finding an optimal hyperplane that separates the data with the largest margin. The trained model will be determined only by a subset of the whole data used in training. These data points that really "matter" to the model are the support vectors. They are data points that either sit on the boundary of the hyperplane or cross the boundary (misclassified points, we are of course talking about soft margin SVM here). Gradient boosted decision tree is an aggregation of many individually weak decision trees. An individual decision tree classifies by finding recursively an optimal cut point in one of the predictors. The optimal cut point is a value such that when splitting the data into two according to this value, the outcome (label) of the data in the same group are as homogenous as possible. The aggregation is done such that the base models (here, decision trees) are added one at a time to minimize some kinds of loss function. Finally, elastic net is just a general linear model with a mixed amount of L1 and L2 regularization.

``` r
#### SVM radial  ####
set.seed(seed)

svmradial<-  train(Class ~ ., data = Data_train, 
                   method = "svmRadial",
                   trControl=fitControlUp,
                   preProcess= preproc,
                   metric="WeightedAccuracy")
```

    ## Loading required package: kernlab

    ## 
    ## Attaching package: 'kernlab'

    ## The following object is masked from 'package:ggplot2':
    ## 
    ##     alpha

``` r
#### Gradient boosting ####
set.seed(seed)
xgb<-  train(Class ~ ., data = Data_train, 
             method = "xgbTree",
             preProcess=preproc,
             trControl=fitControlUp,
             metric="WeightedAccuracy")
```

    ## Loading required package: xgboost

    ## Loading required package: plyr

``` r
#### Elastic net ####
set.seed(seed)
enetgrid <-  expand.grid(alpha= seq(0.1,1,0.05),
                        lambda=seq(0.01,0.1,0.01))

enet<-  train(Class ~ ., data = Data_train, 
              method = "glmnet",
              preProcess=preproc,
              trControl=fitControlUp,
              tuneGrid= enetgrid,
              metric="WeightedAccuracy")
```

    ## Loading required package: glmnet

    ## Loading required package: Matrix

    ## Loading required package: foreach

    ## Loaded glmnet 2.0-10

### Base models comparison

The performance of the three models as summarized as follows.

``` r
modelList <- list(RadialSVM=svmradial,
                 Xgb1 =xgb,
                 Enet=enet)
resamps<- resamples(modelList)
bwplot(resamps)
```

![](mlGermanCredit_md_files/figure-markdown_github/unnamed-chunk-7-1.png)

``` r
summary(resamps)
```

    ## 
    ## Call:
    ## summary.resamples(object = resamps)
    ## 
    ## Models: RadialSVM, Xgb1, Enet 
    ## Number of resamples: 50 
    ## 
    ## Accuracy 
    ##                Min.   1st Qu.    Median      Mean   3rd Qu.      Max. NA's
    ## RadialSVM 0.6000000 0.6710526 0.7000356 0.6997110 0.7227632 0.8000000    0
    ## Xgb1      0.5866667 0.6544737 0.6933333 0.6967666 0.7333333 0.8421053    0
    ## Enet      0.5066667 0.5618919 0.5893860 0.5906988 0.6133333 0.6933333    0
    ## 
    ## Precision 
    ##                Min.   1st Qu.    Median      Mean   3rd Qu.      Max. NA's
    ## RadialSVM 0.3939394 0.4725877 0.5000000 0.5023744 0.5307904 0.6333333    0
    ## Xgb1      0.3783784 0.4523460 0.5000000 0.5003354 0.5508899 0.7037037    0
    ## Enet      0.3333333 0.3883773 0.4090909 0.4106193 0.4276786 0.5000000    0
    ## 
    ## Recall 
    ##                Min.   1st Qu.    Median      Mean   3rd Qu.      Max. NA's
    ## RadialSVM 0.5217391 0.6818182 0.7391304 0.7330040 0.7826087 0.9130435    0
    ## Xgb1      0.5454545 0.6403162 0.7272727 0.7109881 0.7727273 0.9090909    0
    ## Enet      0.6818182 0.7751976 0.8260870 0.8268775 0.8695652 0.9565217    0
    ## 
    ## Specificity 
    ##                Min.   1st Qu.    Median      Mean   3rd Qu.      Max. NA's
    ## RadialSVM 0.5660377 0.6415094 0.6923077 0.6854209 0.7169811 0.8076923    0
    ## Xgb1      0.5471698 0.6538462 0.6923077 0.6907547 0.7358491 0.8490566    0
    ## Enet      0.3962264 0.4339623 0.4905660 0.4896009 0.5260341 0.6346154    0
    ## 
    ## WeightedAccuracy 
    ##                Min.   1st Qu.    Median      Mean   3rd Qu.      Max. NA's
    ## RadialSVM 0.5987654 0.6731759 0.7177914 0.7178764 0.7541399 0.8502994    0
    ## Xgb1      0.5925926 0.6561949 0.7121156 0.7044892 0.7400794 0.8395062    0
    ## Enet      0.6012270 0.6826347 0.7200068 0.7194574 0.7572805 0.8203593    0

We can also compare the most important predictors of the three models. Checking account statut (no checking account) and the duration in month appear to be the most important predictors across models.

``` r
plot(varImp(svmradial),top="10",main="svm")
```

![](mlGermanCredit_md_files/figure-markdown_github/unnamed-chunk-8-1.png)

``` r
plot(varImp(xgb),top="10",main="xgb")
```

![](mlGermanCredit_md_files/figure-markdown_github/unnamed-chunk-8-2.png)

``` r
plot(varImp(enet),top="10",main="elastic net")
```

![](mlGermanCredit_md_files/figure-markdown_github/unnamed-chunk-8-3.png)

The following plot presents the correlations between the three models.

``` r
corrplot(modelCor(resamps),method="number")
```

![](mlGermanCredit_md_files/figure-markdown_github/unnamed-chunk-9-1.png)

For the purpose of model blending, uncorrelated first-level models are more desirable. Sometimes the results of the first-level models can be highly correlated. Highly correlated predictors in logistic regression make the coefficient estimate of the individual predictors unreliable but do not influence the predictive power of the model as a whole. Since we are using logistic regression to blend the first-level models only to increase the predictive power and that we are not interested in how each of the first-level models contributes to the final model, this should not be a problem here.

### Final model

The predictions of the base models on the blending set are put together with the output to be used to train the final model using logistic regression. Note that the data set used to blend the models here is not up-sampled as it was with the data used to train the first-level models This is because I want to use cross-validation to estimate the performance of the final model so I want the data to be as close as possible to what the model would see "in the wild".

``` r
basemodel <- lapply(modelList,predict,newdata=Data_blend)
Data_ensem<- do.call(cbind.data.frame, basemodel)
Data_ensem$Class <- Data_blend$Class
set.seed(seed)
blend<-  train(Class ~ ., data = Data_ensem, 
                    method = "glm",
                    trControl=fitControl)
blend$results
```

    ##   parameter WeightedAccuracy Accuracy Precision    Recall Specificity
    ## 1      none        0.7860621   0.7999 0.6607018 0.7753571   0.8115686
    ##   WeightedAccuracySD AccuracySD PrecisionSD  RecallSD SpecificitySD
    ## 1          0.1059413 0.08897237   0.1478231 0.1439051     0.1068566

We can compare the performance of the final model shown above with a baseline model which classifies all loans as good. Since about 0.7 of the data are good loans, the baseline model would achieve a 0.7 unweighted accuracy. But if we take into account the cost matrix, the baseline model will have a weighted accuracy of 0.32.

Here, our final model has a cross-validation weighted accuracy of 0.78 and unweighted accuracy of 0.79. Precision and recall are respectively around 0.66 and 0.77. Note that the results are susceptible to change when a different seed or no seed is used. However, for most of the time, the final model represents a significant improvement compared to the baseline model. This is especially in terms of weighted accuracy.

### Reference

-   Kuhn Max, Johnson Kjell (2013). Applied predictive Modelling. Springer.

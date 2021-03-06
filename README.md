---
title: "PML - Prediction Assignment Writeup"
author: "SSV"
date: "September 24, 2015"
output: html_document
---

This is an analysis of a dataset[^1] provided by the [HAR group](http://groupware.les.inf.puc-rio.br/har).

The analysis aims at reproducing their prediction of the variable "classe". For that purpose we will build a model, training with a subset of training  data and do our own testing against the complement training We will also use the model to predict 20 test cases provided as testing sample.


##Environment

```{r}
library(caret)
library(rpart)
library(rpart.plot)
#library(klaR)
library(rattle)
library(randomForest)

#set up working directory
setwd("G:/writeup")

set.seed(61610)

```

##Data Cleaning

The training data for this project are available [here](https://d396qusza40orc.cloudfront.net/predmachlearn/pml-training.csv)
The test data are available [here](https://d396qusza40orc.cloudfront.net/predmachlearn/pml-testing.csv)
We assume the files are already availabel in the working directory

```{r}
training <- read.csv("pml-training.csv",na.strings=c("NA",""))
testing <- read.csv("pml-testing.csv",na.strings=c("NA",""))
```

There is a significant number of observations with NA values
```{r}
dim(training)
dim(na.omit(training))
```

Further analysis reveals that the NA are present only in columns entirely filled with NA values.
We can still filter, eliminating the variables with 10% or more NA
```{r}
#identifies the NA and then selects the columns with less than 10% NA
isna<-apply(training, 2, function(x) is.na(x))
na10<-apply(isna, 2, function(x) sum(x, na.rm=TRUE)<dim(training)[1]*0.1)
training<-training[, na10]
testing<-testing[, na10]
```
By eliminating those variables there are no NA left
```{r}
#in reality the NA are present only in columns of only NA and nowhere else
dim(na.omit(training))[1]==dim(training)[1]
```

We can then eliminate variables related to timestamps and bio info
```{r}
names(training[,1:10])
training<-training[,-c(1:7)]
testing<-testing[,-c(1:7)]
dim(training)
dim(testing)
```


##Cross Validation
Cross validation is properly applied by the train function of the caret package, given the opportune parameters.
We nonetheless do our own validation by splitting the training set into a pair training/validation.

##Data Split
For validation we split (random subsampling) the training sample in two parts: 60% for training, 40% for validation.
```{r}
#for the sake of computation time, having already built the model several time with the appropriate split, we use a 20-80 split instead of 60-40
inTrain <- createDataPartition(y=training$classe, p=0.20, list=FALSE)
subtrain  <- training[inTrain,]
subval  <- training[-inTrain,]
dim(subtrain)
dim(subtrain)[1]/dim(training)[1]
```

##Random Forest model with cross validation
We use the random forest algorithm with cross validation k = 5, again to keep the computation as short as possible, having already built the model several times with 10 folds
```{r}
# train the model 
model<- train(classe~.,data=subtrain,method="rf", trControl=trainControl(method="cv",number=5), prox=TRUE,allowParallel=TRUE)

model
```

Let's evaluate on the validation sample
```{r}
# make predictions
predictions <- predict(model,subval)
# summarize results
cmat<-confusionMatrix(predictions, subval$classe)
```
So, with a 60-40 split we would expect an error rate very similar to the one found in the HAR analysis (Correctly Classified Instances 	164662 	99.4144 %).
The validation sample confirms an accuracy of:
```{r}
cmat$overall[1]
```


We could decrease the computational load by picking a subset of the variables based on their predictive value or GINI ranking

```{r}
varImp(model)
```
Using only the 20 most important variables can lead to a model as accurate as the one with  53 variables, while reducing the cpu load and computation time considerably.

##Prediction
Now we can apply the model (with 53 variables) to our 20 samples test.

```{r}
result<-predict(model,testing)
result
```
And then generate the files for the submission.

```{r}
pml_write_files = function(x){
  n = length(x)
  for(i in 1:n){
    filename = paste0("problem_id_",i,".txt")
    write.table(x[i],file=filename,quote=FALSE,row.names=FALSE,col.names=FALSE)
  }
}

pml_write_files(result)
```

##Conclusion
The conclusion is that the HAR group most probably used the random forest method with the subset of the 53 predictors that we identified here or with the 20 most important variables identified above, and a 10 fold cross validation. That would leadd to an accuracy very similarr to the one indicated in thier analysis. However, albeit less accurate, our test with 53 variables, 20% training set and 5 folds cross validation, produces the same result on the testing sample of 20, making 100% of correct predictions.


[^1]Velloso, E.; Bulling, A.; Gellersen, H.; Ugulino, W.; Fuks, H. Qualitative Activity Recognition of Weight Lifting Exercises. Proceedings of 4th International Conference in Cooperation with SIGCHI (Augmented Human '13) . Stuttgart, Germany: ACM SIGCHI, 2013.

---
title: "Practical Machine Learning Project"
subtitle: Week 04
output:
  pdf_document: default
  html_document:
    df_print: paged
---

### *Background*

The goal of your project is to predict the manner in which they did the exercise. This is the “classe” variable in the training set. You may use any of the other variables to predict with. You should create a report describing how you built your model, how you used cross validation, what you think the expected out of sample error is, and why you made the choices you did. You will also use your prediction model to predict 20 different test cases.


### *Libraries to be used*

```{r, message=FALSE}
library(ggplot2)
library(caret)
library(randomForest)
library(rpart)
library(rpart.plot)
library(corrplot)
library(dplyr)
```


### *Acquiring the datasets*

```{r}

pmltraining_url <- "https://d396qusza40orc.cloudfront.net/predmachlearn/pml-training.csv"
pmltesting_url <-"https://d396qusza40orc.cloudfront.net/predmachlearn/pml-testing.csv"

download.file(pmltraining_url, "pml-training.csv", method = "curl")
download.file(pmltesting_url, "pml-testing.csv", method = "curl")

```


### *Loading the datasets* 
Loading datasets to "train" and "test" variables and identify "NA" and "#DIV/0!"

```{r}
trainRaw <- read.csv("pml-training.csv", na.strings = c("NA", "#DIV/0!", ""))
testRaw <- read.csv("pml-testing.csv", na.strings = c("NA", "#DIV/0!", ""))
```


Viewing the new dataset structure and dimensions

```{r}
#str(trainRaw)
#str(testRaw)

dim(trainRaw) 
dim(testRaw) 
```

The dim show us that the raw train data contains 19622 observations and 160 variables. It also shows that the test raw data contains 20 observations and 160 variables.



Converting trainRaw$classe to factor. This is done to avoid the error during model accuracy, because variable data 'classe' needs to be factor.

```{r, message=FALSE}
trainRaw$classe <- as.factor(trainRaw$classe)
```

Viewing the variable classe before cleaning the datasets.

```{r}
table(trainRaw$classe)
```


### *Datasets cleaning*

Removing the first seven columns as they don't have useful data.

```{r}
trainLessCol <- trainRaw[,-seq(1:7)]
testLessCol <- testRaw[,-seq(1:7)]
```


Removing columns with NAs on both train and test datasets. 

```{r}
trainCleanData <- trainLessCol[,colSums(is.na(trainLessCol)) == 0]
testCleanData <- testLessCol[,colSums(is.na(testLessCol)) == 0]
```


### Dataset partitions

The already processed and clean train dataset will be split into two datasets. 
*1.-* into 70% for pure training.
*2.-* into 30% for validation.

```{r}
set.seed(101)      # For reproducibility purpose

inTrain <- createDataPartition(trainCleanData$classe, p=0.70, list=F)
trainData <- trainCleanData[inTrain, ]
validationData <- trainCleanData[-inTrain, ]
```


Viewing the new datasets dimensions.

```{r}
# Dimension for the trainData
dim(trainData)

# Dimension for the validationData
dim(validationData)
```

Checking to see if there are too many variable that could be correlated with corrplot.

```{r}
corr_train_data <- select_if(trainCleanData, is.numeric)
corrplot(
    cor(corr_train_data),
    method = "color",
    tl.pos = "n",
    insig = "blank"
)
```
![Correlation Matrix](/images/correlation-variables.png)
The graphs does show that there are some variable that are correlated, however, I don't think it will affect too much the model


### Data Modeling

##### **Prediction Models**



##### *Decision Tree*

```{r}
modelDT <- rpart(classe ~ ., data=trainData, method="class")
# Plot the Decision Tree
prp(modelDT)
```

![Decision Tree](/images/decision-tree.png)

Prediction and Confusion Matrix for Decision Tree Model.
```{r}
predictDT <- predict(modelDT, validationData, type = "class")
# Test results on our validationData data set:
confusionMatrix(predictDT, validationData$classe)
```

Checking for the accuracy of the Decision Tree model.

```{r}
modelAccuracy <- postResample(predictDT, validationData$classe)
modelAccuracy
accuracyDT <- 1 - as.numeric(confusionMatrix(validationData$classe, predictDT)$overall[1])
accuracyDT
```

The Decision Tree model accurate is 73.03% and the estimated out-of-sample error is 26.96%.


##### *Gradient Boosted Model*

```{r}
modelGbm <- train(classe~., method="gbm", data=trainData, verbose=FALSE)
```


```{r}
predictGbm <- predict(modelGbm, validationData)
confusionMatrix(validationData$classe, predictGbm)
```

Checking for the accuracy of the Gradient Boosted model.

```{r}
modelAccuracy <- postResample(predictGbm, validationData$classe)
modelAccuracy
accuracyGbm <- 1 - as.numeric(confusionMatrix(validationData$classe, predictGbm)$overall[1])
accuracyGbm
```

The Gradient Boosted model accurate is 96.31% and the estimated out-of-sample error is 3.68%.


###### *Random Forest*



```{r}
controlRf <- trainControl(method="cv", 5)
modelRf <- train(classe ~ ., data=trainData, method="rf", trControl=controlRf, ntree=250)
modelRf
```


```{r}
predictRf <- predict(modelRf, validationData)
confusionMatrix(validationData$classe, predictRf)
```

Checking for the accuracy of the Random Forest model.

```{r}
modelAccuracy <- postResample(predictRf, validationData$classe)
modelAccuracy
accuracyRF <- 1 - as.numeric(confusionMatrix(validationData$classe, predictRf)$overall[1])
accuracyRF
```

The Random Forest model accurate is 99.45% and the estimated out-of-sample error is 0.54%.


#### Which Prediction Model was better


```{r}
accuracyAll <- rbind(confusionMatrix(predictDT, validationData$classe)$overall["Accuracy"],
confusionMatrix(predictGbm, validationData$classe)$overall["Accuracy"],
confusionMatrix(predictRf, validationData$classe)$overall["Accuracy"])
row.names(accuracyAll) <- c("DT", "Gbm", "RF")

accuracyAll
```

We can see the the better model from the three that we have tested is Random Forest Model, however, Gradient Boosted Model could be a second choice.

```{r}
predictfinal <- predict(modelRf, testCleanData[, -length(names(testCleanData))])
predictfinal
```






























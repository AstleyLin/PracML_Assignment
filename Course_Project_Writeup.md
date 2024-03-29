

# Practical Machine Learning: Assignment 1

## Background
Using devices such as Jawbone Up, Nike FuelBand, and Fitbit it is now possible to collect a large amount of data about 
personal activity relatively inexpensively. These type of devices are part of the quantified self movement - a group 
of enthusiasts who take measurements about themselves regularly to improve their health, to find patterns in their 
behavior, or because they are tech geeks. One thing that people regularly do is quantify how much of a particular 
activity they do, but they rarely quantify how well they do it. 

The goal is to analyze data from accelerometers on the belt, forearm, arm, and dumbell of six participants. They were asked to perform barbell lifts correctly and incorrectly in five different ways. For more information see the ��Weight Lifting Exercises Dataset�� in the following location: [http://groupware.les.inf.puc-rio.br/har]
(see the section on the Weight Lifting Exercise Dataset). 

  
## Data Processing
Reading in the training and testing datasets which was provided by [https://d396qusza40orc.cloudfront.net/predmachlearn/pml-training.csv] and [https://d396qusza40orc.cloudfront.net/predmachlearn/pml-testing.csv] indivdiually.

```r
dfTesting <- read.csv("pml-testing.csv")
dfAnalysis <- read.csv("pml-training.csv")
dim(dfAnalysis); dim(dfTesting)
```

```
[1] 19622   160
```

```
[1]  20 160
```
There are 19622 rows and 160 columns in training data set `dfAnalysis`. By executing the `summary()` to check out the data, we can observe it contains many missing values. We should utilize some skill to clean up the data.  
First, remove "#DIV/0!" words and space characters which was existed in many filds.

```r
vtClass <- sapply(dfAnalysis, FUN=class)
CheckCol <- colnames(dfAnalysis)[is.element(vtClass, "factor")]
CheckCol <- CheckCol[-c(1:3, length(CheckCol))]
for(col in CheckCol) dfAnalysis[,col] <- as.numeric(gsub(" ", "", gsub("#DIV/0!", "", dfAnalysis[,col])))
```
Second, remove the columns which contain NA's in almost whole filed. By summary the data we can find out this type column have at least 19200 NA's so that we can utilize it to be the removing cirteria. And we do the same removing condition to both training and testing data set to keep them the same field to be comparable.

```r
mxNA <- sapply(dfAnalysis, FUN=is.na, simplify=T)
vtNA <- apply(mxNA, MARGIN=2, FUN=sum)
dfAnalysis <- dfAnalysis[, colnames(dfAnalysis)[vtNA < 19200]]
dfTesting <- dfTesting[, colnames(dfTesting)[vtNA < 19200]]
dim(dfAnalysis); dim(dfTesting)
```

```
[1] 19622    60
```

```
[1] 20 60
```
After clean up the data set, the column numbers left to 60

## Exploratory data analyses
Plot a correlation matrix to check correlation between all contiuous variables. The light color means low correlation and the stronger color means hihg correlation. We can see the stronger color amost distributed in topright coner and some distributed in topleft and bottomright, and negative correlation (red color) is much than positive (blue) in general speaking bys visualized judgement 

```r
library(caret); set.seed(32343)
vtExCOl <- c("X","user_name","user_name","raw_timestamp_part_1","raw_timestamp_part_2","cvtd_timestamp","new_window")
vtCol <- colnames(dfAnalysis)[!is.element(colnames(dfAnalysis), vtExCOl)]
vtNum <- sapply(dfAnalysis, FUN=is.numeric)
library(corrplot)
corrplot(cor(dfAnalysis[,vtNum]), order="FPC", method="square", type="upper", tl.cex=0.8, tl.col=rgb(0, 0, 0))
```

![plot of chunk unnamed-chunk-6](figure/unnamed-chunk-6.png) 

## Build the training/ validating dataset and extract important variables 
In order to utilize cross validation to make sure the model is robust, we depart the cleaned data into training one (dfTraining) and validating one (dfValidate). The dfTraining contains 13737 rows and the dfValidate contains 5885 rows which have nearly 7:3 ratio satisifies the partition condition in `createDataPartition()` argument `p`.

```r
library(caret); set.seed(32343)
vtTrain <- createDataPartition(y=dfAnalysis$classe, p=0.7, list=F)
dfTesting <- dfTesting[, vtCol[-length(vtCol)]]
dfValidate <- dfAnalysis[-vtTrain, vtCol]
dfTraining <- dfAnalysis[vtTrain, vtCol]
dim(dfTraining)
```

```
[1] 13737    54
```

```r
dim(dfValidate)
```

```
[1] 5885   54
```

There are 54 numerical columns in this data set, in order to enforce the pridictor's ability we utilize the preprocessing method "Principle Compoent" to mapping the original 54 variables into less mainly components which are independently with each other. We set the variance explanation threshold to be 90%, and then get the PCA model saved to `lsModelPCA`. Then mapping both training and validation data set to be PCA filds. Later we will utilize the PCA fields to do model training rather than original data set. 

```r
lsModelPCA <- preProcess(dfTraining[, -ncol(dfTraining)], method="pca", thresh=0.9)
dfTrainPCA <- predict(lsModelPCA, dfTraining[, -ncol(dfTraining)])
dfValidPCA <- predict(lsModelPCA, dfValidate[, -ncol(dfValidate)])
```

## Model training
We choose the random forest to be the model training method and specify the use of a cross validation method when applying the random forest routine in the `trainControl()` parameter. The method seems to take long time to complete, while essentially increasing some accuracy.

```r
set.seed(32343)
library(randomForest)
if(!is.element("modelFit", ls())) 
    modelFit <- train(dfTraining$classe ~ ., data=dfTrainPCA, method="rf", 
                   trControl=trainControl(method = "cv", number = 4), importance = TRUE)
modelFit$finalModel
```

```

Call:
 randomForest(x = x, y = y, mtry = param$mtry, importance = TRUE) 
               Type of random forest: classification
                     Number of trees: 500
No. of variables tried at each split: 2

        OOB estimate of  error rate: 2.22%
Confusion matrix:
     A    B    C    D    E class.error
A 3874   11   10    8    3    0.008193
B   33 2582   36    2    5    0.028593
C    3   35 2336   21    1    0.025042
D    3    6   83 2155    5    0.043073
E    0   12   14   14 2485    0.015842
```
As the result of model `modelFit` represented, it reach the 97.78% accuracy, the model seems have high accuracy.

To do more advanced understanding of important varialbes (here variable means PCA component). BY ploting the importance, looking from the top to the bottom on the y-axis, it means each  principal components in order from most important to least important regards to degree which x-axis shows.

```r
varImpPlot(modelFit$finalModel, sort = TRUE, type = 1, pch = 19, col = 1, cex = 1, 
           main = "Importance of the Principal Components")
```

![plot of chunk unnamed-chunk-10](figure/unnamed-chunk-10.png) 


## Cross Validation Testing and Out-of-Sample Error Estimate
Utilize validating dataset to estimate Out-of-Sample error, the error is 0.0253 which is very close to trainin model. 

```r
predictions <- predict(modelFit, newdata=dfValidPCA)
lsConfus <- confusionMatrix(predictions, reference=dfValidate$classe)
print(lsConfus$table)
```

```
          Reference
Prediction    A    B    C    D    E
         A 1659   22    2    1    0
         B    1 1098   19    0    2
         C    3   15  994   39    6
         D    7    2   11  917    6
         E    4    2    0    7 1068
```

```r
print(lsConfus$overall["Accuracy"])
```

```
Accuracy 
  0.9747 
```


## Predicted Results of testing datasets
Utilize the PCA model to transform original testing data into principle component and then input to model, we can get the result of prediction.

```r
dfTestPCA <- predict(lsModelPCA, dfTesting)
predictions <- predict(modelFit, newdata=dfTestPCA)
print(predictions)
```

```
 [1] B A B A A E D B A A B C B A E E A B B B
Levels: A B C D E
```










![Workshop Data Science](../commonimages/workshop_logo.png)

# Lab: Predict house sales prices through Regression

This lab will guide you through a practical example of how to train and apply a regression model. You have access to a dataset with house sale data. The dataset includes the Sale Price and many attributes that describe some aspect of the house, including size, area, features, et cetera. You will use this dataset to build a regression model that can be used to estimate what a house with certain characteristics might sell for.

There are many applications of regression in business. The main use cases are around forecasting and optimization. For example, predicting future demand for products, estimating the optimal price for a product and fine-tuning manufacturing and delivery processes. In other words, the principles that you learn with this exercise are applicable to those business scenarios as well.

## Objectives

- Become familiar with Data Exploration, Data Preparation, Model training and Evaluation techniques.
- Become familiar with Python and its popular ML libraries, in particular Scikit Learn.
- Become familiar with the Oracle Data Science service.

# Prerequisites

You require the following:

- An Oracle Cloud Tenancy
- A Data Science Project + Notebook

Please follow the [prerequisites](../prereq/lab.md) first in case you don't have these yet.

# Upload House Sales Data

Download the dataset with the house prices and characteristics.

- Download [The training dataset](./data/housesales.csv) (the dataset is public)

Click on the link, then use the `"Raw"` button and then right click `"Save As"`. Make sure to save these with extension `CSV`. Some browsers try to convert this to Excel format, which is incorrect.

Review the datasets. The column names are explained [here](./data/data_description.txt). This document also explains the possible list-of-values for categorical attributes.

# Prepare your notebook

- Open the notebook (in OCI) that you created in the prerequisites

![Open Notebook](./images/opennotebook2.png)
  
- Upload the dataset by dragging it to the left panel

![Upload Dataset](./images/uploaddataset.png)

- Jupyter is a notebook environment that allows on-the-fly Python execution. It allows you to mix Python code with text for documentation. Create a new Jupyter notebook by clicking on the button that shows `"Python 3"`.

![Create Notebook](./images/createjupyternotebook.png)

# Data Exploration

The goals of this are to:

- Identify what data you have and what is usable
- Identify issues such as missing values, skewed distributions, colinearity and outliers
- Form ideas of new features that could be calculated (Feature Engineering). Later on we'll use the knowledge that we gain during Data Exploration to decide which data transformations to perform.

## Import the required Python libraries

`Pandas` is a library that's used for working with data, e.g. displaying it and doing data transformations.

- `Numpy` is a library to work with arrays and for calculations.
- `Sklearn` is the standard Python library for machine learning. It provides many unsupervised and supervised learning algorithms.
- `Seaborn` is a Python data visualization library based on matplotlib.

  ```python
  import pandas as pd
  import matplotlib.pyplot as plot
  import seaborn as sns
  import numpy as np
  from scipy.stats import norm
  from sklearn.preprocessing import StandardScaler
  from sklearn.model_selection import train_test_split
  from sklearn import linear_model
  from sklearn.metrics import mean_squared_error
  from scipy import stats
  import warnings
  warnings.filterwarnings('ignore')
  %matplotlib inline
  ```

  **Everytime you see a piece of code as above, please paste it into the notebook and click the run icon.**

  ![Run Script](./images/runscript.png)
  
  `This should execute without errors.`

## Load the training data
  
We'll split this into train and test later. All these type of data operations are handled by the Pandas library (named `"pd"` here)
  
```python
alldata = pd.read_csv('./housesales.csv')
```

.

### Check how much data we have

Is the dataset large enough to train a model?

```python
alldata.shape
```

`Conclusion`: There are 1460 rows, which at first sight looks like a good enough to train an initial model. The data is not extensive enough for production use but would help establish a good baseline. We also see that there are 80 input features that we can potentially use to predict our target variable.

.

### Review the target attribute: `SalePrice`

Before we look at the input features, let's have a close look at our Target attribute: `SalePrice`.
First, let's display some values. The `[:10]` selects only the first 10 rows.

```python
alldata['SalePrice'][:10]
```

`Conclusion:` *SalePrice* is an integer and has numbers in the range that we would expect from house prices. There are no empty values.

.

### Which columns will we select as input features for our model?

Let's start by listing all the columns.

```python
alldata.columns
```

`Conclusion`: There are many input features that we could potentially use. Some of these columns are self explanatory, others not so much. Have a look at [data_scription.txt](./data/data_description.txt) for background.

.

### Which columns would we select intuitively?

To start with, let's first take an intuitive approach. Ask yourself:

- `"Which of these columns is likely of value to predict Sale Price?"`
- `"Which factors of a house would I look at yourself when making a buying decision?"`

Imagine that we've decide that we believe the following features are relevant:

- `GrLivArea`: size of living area
- `TotalBsmtSF`: size of basement
- `OverallQual`: overall quality category
- `YearBuilt`: year house was built
- `MSZoning`: A=Agriculture, C=Commercial, RH=Residential High Density, et cetera

.

### Are the columns we selected intuitively also correlated with Sale Price?

Let's test our theories that these input features are correlated with Sale Price. We'll start with the numerical variables first: `GrLivArea`, `TotalBsmtSF`, `OverallQual` and `YearBuilt`.
Because all of these are numerical continuous attributes, we can use scatter plots here.

```python
plot.scatter(alldata.GrLivArea, alldata.SalePrice)
plot.xlabel("GrLivArea")
plot.ylabel("SalePrice")
plot.show()
plot.scatter(alldata.TotalBsmtSF, alldata.SalePrice)
plot.xlabel("TotalBsmtSF")
plot.ylabel("SalePrice")
plot.show()
plot.scatter(alldata.OverallQual, alldata.SalePrice)
plot.xlabel("OverallQual")
plot.ylabel("SalePrice")
plot.show()
plot.scatter(alldata.YearBuilt, alldata.SalePrice)
plot.xlabel("YearBuilt")
plot.ylabel("SalePrice")
plot.show()
```

Conclusion:

- `GrLivArea`: There is a linear correlation with `SalePrice`; We can draw a straight line from the bottom-left to the top-right.
- `TotalBsmtSF`: There appears to be an exponential relationship between TotalBsmtSF and SalePrice. In other words, higher basement sizes lead to exponentially higher prices.
- `OveralQual`: As expected, we see higher sales prices when overal quality perception is higher.
- `YearBuilt`: This relationship is a little less obvious, but there's a trend for higher prices for more recent construction.

Therefore all of these attributes appear to be of predictive value and we want to keep them in our training set.

On another note: We see several `outliers`. In particular, the attribute `GrLivArea` shows that there are some houses with exceptionally large living areas given their price.

### Is the categorical attribute `MSZoning` also correlated with `SalePrice`?

This attribute contains the type of property (`A=Agriculture`, `C=Commercial`, `RH=Residential High Density`, et cetera). Correlation between a categorical and a continuous attribute (SalePrice) can be visualized as a boxplot.

```python
var = 'MSZoning'
data = pd.concat([alldata['SalePrice'], alldata[var]], axis=1)
f, ax = plot.subplots(figsize=(8, 6))
fig = sns.boxplot(x=var, y="SalePrice", data=data)
fig.axis(ymin=0, ymax=800000);
```

`Conclusion`: The boxplots for the various types of properties look very different. Therefore we can conclude that also this attribute appears to be of significance to predict Sale Price, and we will want to include it in our training set.

.

### A different approach: Systematically checking for correlation between input features and our target

It becomes clear that manually investigating all of the attributes this way is very time consuming. Therefore, let's take a more systematic approach. Ideally, we would like to see the correlation between all input attributes and the target `SalePrice`.

```python
corr = alldata.corr(method='spearman')
corr.sort_values(["SalePrice"], ascending = False, inplace = True)
corr.SalePrice
```

`Conclusion`: Our initial intuition of thinking that `TotalBsmtSF`, `OverallQual`, `GrLivArea` and `YearBuilt` are of importance, was correct. However, there are other features listed as high correlation, such as `GarageCars`, `GarageArea` and `Fullbath`.

Here's an explanation of the features that are most correlated with our target:

- `OverallQual`: Rates the overall material and finish of the house (1 = Very Poor, 10 = Very Excellent)
- `GrLivArea`: Above grade (ground) living area square feet
- `GarageCars`: Size of garage in car capacity
- `GarageArea`: Size of garage in square feet
- `TotalBsmtSF`: Total square feet of basement area
- `1stFlrSF`: First Floor square feet
- `FullBath`: Full bathrooms above grade
- `TotRmsAbvGrd`: Total rooms above grade (does not include bathrooms)
- `YearBuilt`: Original construction date

.

### What is the relationship between GarageCars and GarageArea?

Of the top correlated attributes, two are related to the garage(s) of the house. It appears that GarageArea and GarageCars are very similar types of information. This phenomenon is called `colinearity`. Let's test if GarageArea and GarageCars are correlated.

```python
var = 'GarageCars'
data = pd.concat([alldata['GarageArea'], alldata[var]], axis=1)
f, ax = plot.subplots(figsize=(8, 6))
fig = sns.boxplot(x=var, y="GarageArea", data=data)
fig.axis(ymin=0, ymax=1500);
```

Indeed, there's a clear pattern here. It's logical that a bigger size of garage (GarageArea) will allow for more cars to park (GarageCars) and maybe even suggest bigger house too. We will look at how to use this colinearity later on.

.

### How can we deal with Colinearity in a systematic way?

To do this, we would have the check the correlation between all attributes (!) There's a visualization that can help us do this: `The Correlation Matrix Heatmap`

```python
corrmat = alldata.corr()
f, ax = plot.subplots(figsize=(12, 9))
sns.heatmap(corrmat, vmax=.8, square=True);
```

How do we interpret this chart?
We see that all attributes are listed on the vertical and the horizontal axis. Bright colors (white) mean high correlation. The diagonal line shows the correlation of each attribute with itself, and hence is correlated. If you check the very bottom "line", you see the correlation of all input features with `SalePrice`. The other bright spots indicate attributes that might contain similar information. For example, we see the correlation between `GarageArea` and `GarageCars`.

**Which other correlations do you see? Can you explain them?**

.

### Do we have Missing Data?

Many machine learning models are unable to handle missing data. We have to find out how much data is missing. At the moment of replacing missing data, we should first find out whether that would cause some kind of distortion in the data and have a negative effect on model quality.
Therefore, we have to see if the missing data follows certain patterns.

Check how many Empty values each column has, and compute the percentage of the total number of rows.

```python
total = alldata.isnull().sum().sort_values(ascending=False)
percent = (alldata.isnull().sum()/alldata.isnull().count()).sort_values(ascending=False)
missing_data = pd.concat([total, percent], axis=1, keys=['Total', 'Percent'])
missing_data.head(20)
```

`Conclusion`: There are a lot of attributes with missing values. This is especially the case for attributes `PoolQC`, `MiscFeature`, `Alley`, `Fence` and `FireplaceQu`.

.

### Let's investigate the attributes with most missing values in detail

The description of the attribute `PoolQC` (pool quality) is as follows:

- `Ex - Excellent`
- `Gd - Good`
- `TA - Average/Typical`
- `Fa - Fair`
- `NA - No Pool`

It seems sensible to assume that most houses don't have pools and therefore missing values simply mean `"No Pool"`. Therefore we make a note to replace those missing values with `"NA"`.
Similarly, we make a note to replace the missing values for `MiscFeature`, `Alley`, `Fence` and `FireplaceQu` with `"NA"`.

The next attribute with many hidden values is `LotFrontage`, which means `"Linear feet of street connected to property"`. This is a continuous measure, and we choose that we will replace missing values by with the mean of the existing values.

.

### Check the distribution of the Target variable

It's important that the target variable follows Normal Probability distribution. If it does not, this will negatively impact the model's performance. We can check for this using a histogram, and including a normal probability plot. The Seaborn library does this for us.

```python
sns.distplot(alldata['SalePrice'], fit=norm);
fig = plot.figure()
```

`Conclusion`: This deviates from the Normal Distribution. You see that it is positively skewed. The regression algorithms that we will use later on has problems with such a distribution. We will have to address this problem.

# Data Preparation

During Data Exploration, we have realized that several changes must be made to he dataset. Data Preparation is a logical result of Data Exploration; we will now take action based on the insights that we gained earlier.

.

### Update missing values

In the previous topic (identifying Missing Values) we made the decision that:

- We want to replace the missing values of `PoolQC`, `MiscFeature`, `Alley`, `Fence` and `FireplaceQu` with `"NA"`.
- We will replace missing values of `LotFrontage` with the mean of the existing values. Let's do this:

```python
alldata = alldata.fillna({"PoolQC": "NA"})
alldata = alldata.fillna({"MiscFeature": "NA"})
alldata = alldata.fillna({"Alley": "NA"})
alldata = alldata.fillna({"Fence": "NA"})
alldata = alldata.fillna({"FireplaceQu": "NA"})
meanlot = alldata['LotFrontage'].mean()
alldata = alldata.fillna({"LotFrontage": meanlot})
alldata = alldata.dropna()
```

.

### Handling Outliers

Do you remember that the scatter chart for `GrLivArea` showed several outliers?
Let's remove these two outliers, by identifying the houses with the highest `GrLivArea`.

Show the IDs of the houses with the highest GrLivArea.

```python
alldata.sort_values(by = 'GrLivArea', ascending = False)[:2]
```

Now remove them and plot the chart again to check that the outliers have disappeared.

```python
alldata = alldata.drop(alldata[alldata['Id'] == 1299].index)
alldata = alldata.drop(alldata[alldata['Id'] == 524].index)
plot.scatter(alldata.GrLivArea, alldata.SalePrice)
plot.xlabel("GrLivArea")
plot.ylabel("SalePrice")
plot.show()
```

We could check for outliers in other attributes, but this is sufficient for our exercise.

### Handling the skewed distribution of the Target variable

Do you remember that the histogram of `SalePrice` showed a positive skew? We can solve this problem by converting the target variable. In case of positive skewness, use a -log- transformation to make the variable fit normal distribution. Let's make the log transformation and show the histogram again to check the result:

```python
alldata['SalePrice'] = np.log(alldata['SalePrice'])
sns.distplot(alldata['SalePrice'], fit=norm);
fig = plot.figure()
```

`Conclusion`: Now the sales price follows a normal distribution.

.

### Removing irrelevant features

In any case we will remove the ID column, which does not carry any predictive value. We will also remove the attributes that showed very low correlation with `SalePrice` during Data Exploration. Note that this is a fairly brute approach, but it is again sufficient for our exercise.

```python
alldata.drop("Id", axis = 1, inplace = True)
alldata.drop("BsmtFullBath", axis = 1, inplace = True)
alldata.drop("BsmtUnfSF", axis = 1, inplace = True)
alldata.drop("ScreenPorch", axis = 1, inplace = True)
alldata.drop("MoSold", axis = 1, inplace = True)
alldata.drop("3SsnPorch", axis = 1, inplace = True)
alldata.drop("PoolArea", axis = 1, inplace = True)
alldata.drop("MSSubClass", axis = 1, inplace = True)
alldata.drop("YrSold", axis = 1, inplace = True)
alldata.drop("BsmtFinSF2", axis = 1, inplace = True)
alldata.drop("MiscVal", axis = 1, inplace = True)
alldata.drop("LowQualFinSF", axis = 1, inplace = True)
alldata.drop("OverallCond", axis = 1, inplace = True)
alldata.drop("KitchenAbvGr", axis = 1, inplace = True)
alldata.drop("EnclosedPorch", axis = 1, inplace = True)
```

### Convert categorical values to numbers

Most ML algorithms can only work with numbers. Therefore we should convert categories to numbers first.

For all attributes we will assume that they are Nominal (as opposed to Ordinal), meaning that there's no order/sequence in the values that it can take. The go-to method to encode Nominal categorical values is Onehot Encoding. This will convert each separate value of a category into its own column that can take a value of 1 or 0. The Pandas `get_dummies` function does OneHot encoding.

```python
alldata = pd.get_dummies(alldata)
alldata.head()
```

`Conclusion`: For example, see the `SaleType` column that has been converted into `SaleType_ConLw`, `SaleType_New`, et cetera. The dataset now only has numerical values.

# Building the model

We will build a simple Linear Regression model. We will use the Scikit-Learn library for this.

## Separating Target and Input Features

Scikit Learn expects that we deliver the data for training in two parts:

1. A dataset with a single column, the target, in this case Sale Price. We will place this in variable "y".
2. A dataset with all the input columns, in this case all columns apart from Sale Price. We will place this in variable "X".

Prepare the data:

```python
y = np.log(alldata.SalePrice)
X = alldata.drop(['SalePrice'], axis=1)
```

.

## Separating Train and Test data so we can validate the model later

After building the model, we will want to test its performance against new data. It's important that this data has not been seen before during the model training. To achieve this we have to reserve part of our dataset for testing, which will be removed from the training phase.

We'll reserve 20% of the total dataset for testing. The random_state variable is for initializng the randomizer. By hardcoding it here we make sure that we select the same records everytime that we run the script.

```python
X_train, X_test, y_train, y_test = train_test_split(X, y, random_state=63, test_size=.20)
```

Now we have 4 variables:

- `X_train`: The input features of the training dataset.
- `X_test`: The input features of the test dataset.
- `y_train`: The target of the training dataset.
- `y_train`: The target of the test dataset.

.

## Build the model

Now we're ready to build the model (on the training data only). Building the model is also called "fitting".

```python
lr = linear_model.LinearRegression()
model = lr.fit(X_train, y_train)
```

# Verifying the performance of the model

How accurate is our model?

We will use the Test dataset for this. First, apply the predictions on the Test dataset

```python
y_predicted = model.predict(X_test)
```

.

## An intuitive, visual approach to verification

To verify the quality of the predictions, let's first use an intuitive visual approach.

For this we will display in one graph:

- The actual `SalePrice` (according to the original data in the Test dataset)
- The predicted `SalePrice` (the value according to our model)

We're plotting this as a scatter.

```python
actual_values = y_test
plot.scatter(y_predicted, y_test)
plot.xlabel('Predicted Sale Price')
plot.ylabel('Actual Sale Price')
plot.title('Comparing Predicted and Actual Sale Prices')
plot.show()
```

`Conclusion`: In ideal circumstances we'd like to see a perfectly straight line. A straight line would mean that Predicted and Actual Sale Prices are the same. In our case we do see an approximation of a straight line, with the exception of a few outliers.

.

## A measurable, mathematical approach to verification

How can we express the accuracy of the model in a more mathematical way? For that we use a quality metric, in this case we use [RMSE](https://en.wikipedia.org/wiki/Root-mean-square_deviation).

In essence `RMSE` measures the distance between the predictions and the actual values. A lower value for RMSE means a higher accuracy.

RMSE by itself is not easy to interpret, but it can be used to compare different versions of a model, to see whether a change you've made has resulted in an improvement. Scikit-Learn has a function to calculate RMSE.

```python
print('RMSE: ', mean_squared_error(y_test, y_predicted))
```

# Bonus exercise (optional): Try to improve the model by engineering a new input feature

Feature Engineering is the process of creating/updating input features using Domain Knowledge. The goal is to calculate / derive new features that have a higher predictive significance.

.

## Investigate the garage attributes

Remember how we saw that there are a few very similar attributes for garage, namely GarageArea and GarageCars? We will try to remove the colinearity of these by combining them into one attribute. After this, we will check to see if the model accuracy has improved. Let's check the correlation of the two attributes with Sale Price.

```python
plot.scatter(alldata.GarageArea, alldata.SalePrice)
plot.xlabel("GarageArea")
plot.ylabel("SalePrice")
plot.show()
print ('Correlation of GarageArea with SalePrice: ', alldata['GarageArea'].corr(alldata['SalePrice']))
plot.scatter(alldata.GarageCars, alldata.SalePrice)
plot.xlabel("GarageCars")
plot.ylabel("SalePrice")
plot.show()
print ('Correlation of GarageCars with SalePrice: ', alldata['GarageCars'].corr(alldata['SalePrice']))
```

These correlations give us a baseline for a new metric that we could create.

Ask yourself, how can you best express the quality/availability of parking?

- Is it the number of parking places
- Is it the total parking space
- Is it the space per parking place

We'll try to come up with a single metric for parking by -multiplying- `GarageArea` and `GarageCars`.

```python
alldata['GarageArea_x_Car'] = alldata.GarageArea*alldata.GarageCars
plot.scatter(alldata.GarageArea_x_Car, alldata.SalePrice)
plot.xlabel("GarageArea_x_Car")
plot.ylabel("SalePrice")
plot.show()
print ('Correlation of GarageArea_x_Car with SalePrice: ', alldata['GarageArea_x_Car'].corr(alldata['SalePrice']))
```

`Conclusion`: The newly engineered feature seems to delivers a higher correlation than `GarageArea` or `GarageCars` alone.

.

## Remove the original garage attributes, rebuild the model and compare

With the new attribute in place, let's train the model again, and compare its performance with the original model.
<!--alldata.drop("GarageArea", axis = 1, inplace = True)
alldata.drop("GarageCars", axis = 1, inplace = True)-->

```python
y = np.log(alldata.SalePrice)
X = alldata.drop(['SalePrice'], axis=1)
X_train, X_test, y_train, y_test = train_test_split(X, y, random_state=63, test_size=.20)
lr = linear_model.LinearRegression()
model = lr.fit(X_train, y_train)
y_predicted = model.predict(X_test)
print('RMSE: ', mean_squared_error(y_test, y_predicted))
```

**We have managed to improve the RMSE. Our model is performing better with our new garage feature.**

<!--
# Bonus Exercise (optional)
Pick another algorithm to train on this data, and compare its performance with the LinearRegression algorithm.
-->

# Conclusion

- You have made your first steps with Data Exploration, Data Preparation, Model training and Evaluation.
- You have learned the basics of Python and Scikit Learn.
- You have learned how to provision and work with the Oracle Data Science cloud service.  
- And, hopefully, you have been inspired to apply Machine Learning to many more situations!

# Follow-up questions

![Lyudmil Pelov](../commonimages/lyudmil.png)

[lyudmil.x.pelov@oracle.com](mailto:lyudmil.x.pelov@oracle.com)

![Jeroen Klosterman](../commonimages/jeroen.png)

[jeroen.kloosterman@oracle.com](mailto:jeroen.kloosterman@oracle.com)

---
title: "How to apply preprocessing steps in a pipeline only to specific features"
author: "Moritz Körber"
date: "11 October 2019"
---

The situation: You have a pipeline to standardize and automate preprocessing. Your data set contains features of at least two different data types that require different preprocessing steps. For example, categorical features may need to be converted into dummy variables but continuous features may need to be standardized. sci-kit learn has got you covered here since version 0.20! The function [ColumnTransformer](https://scikit-learn.org/stable/modules/generated/sklearn.compose.ColumnTransformer.html) allows you to create column-specific pipeline steps! In this post, I show you how to use the function and talk about the advantages of preprocessing with a pipeline a bit. Let's get started!

First, load the necessary libraries:


```python
import pandas as pd
from sklearn.compose import ColumnTransformer
from sklearn.impute import SimpleImputer
from sklearn.linear_model import LogisticRegression
from sklearn.metrics import classification_report
from sklearn.model_selection import GridSearchCV, RepeatedStratifiedKFold
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import OneHotEncoder, StandardScaler
```

We will be working with the Titanic data set.


```python
titanic = pd.read_csv('./titanic.csv')

titanic.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>pclass</th>
      <th>survived</th>
      <th>name</th>
      <th>sex</th>
      <th>age</th>
      <th>sibsp</th>
      <th>parch</th>
      <th>ticket</th>
      <th>fare</th>
      <th>cabin</th>
      <th>embarked</th>
      <th>boat</th>
      <th>body</th>
      <th>home.dest</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>0</td>
      <td>3</td>
      <td>0</td>
      <td>Mahon Miss. Bridget Delia</td>
      <td>female</td>
      <td>NaN</td>
      <td>0</td>
      <td>0</td>
      <td>330924</td>
      <td>7.8792</td>
      <td>NaN</td>
      <td>Q</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>1</td>
      <td>1</td>
      <td>0</td>
      <td>Clifford Mr. George Quincy</td>
      <td>male</td>
      <td>NaN</td>
      <td>0</td>
      <td>0</td>
      <td>110465</td>
      <td>52.0000</td>
      <td>A14</td>
      <td>S</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>Stoughton MA</td>
    </tr>
    <tr>
      <td>2</td>
      <td>3</td>
      <td>0</td>
      <td>Yasbeck Mr. Antoni</td>
      <td>male</td>
      <td>27.0</td>
      <td>1</td>
      <td>0</td>
      <td>2659</td>
      <td>14.4542</td>
      <td>NaN</td>
      <td>C</td>
      <td>C</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>3</td>
      <td>3</td>
      <td>1</td>
      <td>Tenglin Mr. Gunnar Isidor</td>
      <td>male</td>
      <td>25.0</td>
      <td>0</td>
      <td>0</td>
      <td>350033</td>
      <td>7.7958</td>
      <td>NaN</td>
      <td>S</td>
      <td>13 15</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>4</td>
      <td>3</td>
      <td>0</td>
      <td>Kelly Mr. James</td>
      <td>male</td>
      <td>34.5</td>
      <td>0</td>
      <td>0</td>
      <td>330911</td>
      <td>7.8292</td>
      <td>NaN</td>
      <td>Q</td>
      <td>NaN</td>
      <td>70.0</td>
      <td>NaN</td>
    </tr>
  </tbody>
</table>
</div>



## Advantages of preprocessing with a pipeline

We would like to predict whether a passenger has survived based on the available data. Before we train our model, some preprocessing has to be done. Why should we include preprocessing in our machine learning pipeline? Isn't it easier to do everything beforehand, say with pandas?

First of all, it is convenient and makes the preprocessing steps and their order explicit, transparent and replicable. But there are three way more substantial reasons:

1) It allows to include the preprocessing steps in the hyperparameter tuning (I'll come back to that in another post).

2) It saves you from making the mistake of using any test data for model training or decisions on the model (e. g., classifier parameters), also known as *data leakage*. This pitfall lurks, for example, when you use scalers or are imputing missing values. Avoiding this is crucial to obtaining valid model performance estimates. The preprocessing steps declared in the pipeline are guaranteed to be only performed based on training data (or training folds in cross-validation).

3) It guarantees that your data is always preprocessed the same way. This is important, for example, if a categorical feature has a category in the test set that does not occur in the training set. I'll give you an example: Let's say your training data contains a feature `review_status`, which indicates whether a transaction has already been reviewed. It may feature the following two categories:


```python
review_status = ['not reviewed', 'reviewed']
```

However, in your test data, there is one more category, `'externally reviewed'`, which does not appear in the training set. Now if you use `pandas.get_dummies()`, you will encounter two problems:

1) If novel data comes in observation by observation, using `pandas.get_dummies()` simply makes no sense.

2) You end up with one additional feature/column in the test set compared to the training set. But your model is trained on the training set and does not know this column. Vice versa, if the category is missing in the test set, your model expects one more feature. `OneHotEncoder()`, as all pipeline steps, first calls the `.fit()` method and then the `.transform()` method on the training set but only `.transform()` on the test set. Thus, the categories are derived only from the unique categories in the training set! You can explictly declare what happens if an unknown category is encountered by setting the `handle_unknown` parameter: `handle_unknown = 'error'` throws an error if an unknown category is encountered, while `handle_unknown = 'ignore'` makes the transformer ignore the category. Hence, once fit, `OneHotEncoder()` produces the same output every time it is applied to new data. And this is easy for our model to digest.

## Creating a ColumnTransformer

Okay, let's create a preprocessing pipeline now. We wish to create dummy variables for the categorical features and to standardize the continuous features. For this purpose, we put everything in a `ColumnTransformer`. We begin with the categoricals: First, we need to name the step: `'onehot'`. Then we need to specify the transformer, here `OneHotEncoder()`. Lastly, we need to indicate which columns should be transformed, here done by giving column names `['pclass', 'sex', 'embarked']` but other forms (e.g., indices) also do work.


```python
('onehot', OneHotEncoder(), ['pclass', 'sex', 'embarked'])
```




    ('onehot', OneHotEncoder(categorical_features=None, categories=None, drop=None,
                   dtype=<class 'numpy.float64'>, handle_unknown='error',
                   n_values=None, sparse=True), ['pclass', 'sex', 'embarked'])



The same is done with `StandardScaler()`. Since we have a few missing values in our features, we may implement an imputer as well. Luckily, sci-kit learn provides us with a simple imputer. At last, we need to tell the `ColumnTransformer` what happens to the features that are not selected for transformation in `remainder`. You can choose to just leave them as they are with `remainder = 'passthrough'`, to drop them, as I did, with `remainder = 'drop'` or to pass them to another estimator. Here is the finished `ColumnTransformer`:


```python
preprocessor = ColumnTransformer(
    [
        ('imputer', SimpleImputer(strategy = 'constant', fill_value = 'missing'),
          ['pclass', 'sex', 'embarked']),
        ('onehot', OneHotEncoder(), ['pclass', 'sex', 'embarked']),
        ('imputer', SimpleImputer(strategy = 'median'),
          ['age', 'sibsp', 'parch', 'fare']),
        ('scaler', StandardScaler(), ['age', 'sibsp', 'parch', 'fare'])
    ],
    remainder = 'drop'
)
```

This code looks a bit ugly. I prefer to split these lines into two sub-transformers, one for categorical features and one for numerical features.


```python
# transformer for categorical features
categorical_features = ['pclass', 'sex', 'embarked']
categorical_transformer = Pipeline(
    [
        ('imputer_cat', SimpleImputer(strategy = 'constant', fill_value = 'missing')),
        ('onehot', OneHotEncoder(handle_unknown = 'ignore'))
    ]
)
```

Now, the steps for the numerical features:


```python
# transformer for numerical features
numeric_features = ['age', 'sibsp', 'parch', 'fare']
numeric_transformer = Pipeline(
    [
        ('imputer_num', SimpleImputer(strategy = 'median')),
        ('scaler', StandardScaler())
    ]
)
```

We combine them in a single `ColumnTransformer` again.


```python
preprocessor = ColumnTransformer(
    [
        ('categoricals', categorical_transformer, categorical_features),
        ('numericals', numeric_transformer, numeric_features)
    ],
    remainder = 'drop'
)
```

Next, we create the machine learning pipeline and include the column transformer as a step. `make_pipeline()` may shorten the code, but I find this representation clearer.


```python
pipeline = Pipeline(
    [
        ('preprocessing', preprocessor),
        ('clf', LogisticRegression())
    ]
)
```

Now that we have all preprocessing steps set, we can move on to hyperparameter tuning and estimation of model performance. I pass the candidate parameters as a dictionary here. Since we'll feed a pipeline to `GridSearchCV()` later, we need to indicate what step a parameter belongs to. Adding 'clf__' for our pipeline step `('clf', LogisticRegression())` does the trick here.


```python
params = {
    'clf__solver': ['liblinear'],
    'clf__penalty': ['l1', 'l2'],
    'clf__C': [0.01, 0.1, 1, 10, 100],
    'clf__random_state': [42]
}
```

We still need to define a cross-validation strategy. I go for `RepeatedStratifiedKFold()` with $k = 5$. That means stratified 5-fold cross-validation repeated two times with a shuffling of the observations between the two repetitions.


```python
rskf = RepeatedStratifiedKFold(n_splits = 5, n_repeats = 2, random_state = 42)
```

Next, we create the `GridSearchCV()` object by filling in the steps above and choosing a scoring metric:


```python
cv = GridSearchCV(
  pipeline,
  params,
  cv = rskf,
  scoring = ['f1', 'accuracy'],
  refit = 'f1',
  n_jobs = -1
  )
```

Split the data into features (X) and target (y):


```python
X = titanic.drop('survived', axis = 1)
y = titanic.survived
```

And, finally, execute and evaluate!


```python
cv.fit(X, y)

print(f'Best F1-score: {cv.best_score_:.3f}\n')
print(f'Best parameter set: {cv.best_params_}\n')
print(f'Scores: {classification_report(y, cv.predict(X))}')
```

    Best F1-score: 0.712

    Best parameter set: {'clf__C': 10, 'clf__penalty': 'l1', 'clf__random_state': 42, 'clf__solver': 'liblinear'}

    Scores:               precision    recall  f1-score   support

               0       0.82      0.85      0.83       809
               1       0.74      0.69      0.72       500

        accuracy                           0.79      1309
       macro avg       0.78      0.77      0.77      1309
    weighted avg       0.79      0.79      0.79      1309



Our best results, an F1-score of 0.712, were achieved with an inverse of regularization strength (*C*) of 10 and L1 penalty.

Find the complete code in one single file down below. Happy coding!

<script src="https://gist.github.com/moritzkoerber/81c82f37e68b1140e406ac468a205f3f.js"></script>

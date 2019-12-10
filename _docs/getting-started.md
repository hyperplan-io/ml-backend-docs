---
title: Getting Started
tags: 
 - machine-learning 
 - python
description: Getting started with Hyperplan
---

# Getting Started

The goal of Hyperplan is to help you manage and serve your machine learning projects. Why should you use it ?

* You want to enhance your productivity: Write reusable code accross projects.
* You want to easily extend your application: Hook functions let you register side effects to execute on the data that flows through your algorithms.
* You want your application to be testable: Hyperplan is primarily made out of simple testable functions.

## Project 

A project is the answer to a is a set of algorithms that achieve the same goal. 
* It is uniquely identified.
* It contains a number of prediction functions
* It holds preprocessing and post processing functions
```python
PROJECT_A = Project(
          "id",
          prediction_functions=[
              lambda x: np.array([[2]])
          ]
      )
```

### Prediction function
The prediction function takes three parameters as input. The features, the type of features (application/json, image/jpeg ...) and a dictionary of metadatas (user identifier, entity id etc).
```python
def predict(self, features: np.array, feature_type: str, metadata: Dict) -> np.ndarray:
```

It consists of four major steps: algorithm selection, preprocessing, inference and post processing. These steps can be enriched with hooks (side effects) before and after inference.
### Selection function
The selection function selects which algorithm to execute. It takes as input the list of algorithms available and the feature data. It needs to return one of the prediction functions.
```python
Callable[[List[PredictionFunction], np.ndarray], PredictionFunction]
```

### Pre-Processing

The pre-processing function allows you to make modifications on the raw data before it is passed to any algorithm. This is useful if you accept different types of data and want to have consistent feature passed to your algorithms. This function is different from the algorithm specific pre-processing function, which prepares the data for a specific model.

```python
# A function that takes an array of feature data along with the type of data (application/json, image/jpeg ...) and returns a new array of feature data.
Callable[[np.ndarray, str], np.ndarray]
```

### Post-Processing
The post-processing function is executed on the data produced by the algorithm (labels).

```python
# A function that takes an array of label data and returns a new one.
Callable[[np.ndarray], np.ndarray]
```

## Hooks
Hooks are functions that execute before or after a prediction has been made. They receive raw unprocessed data as input (without preprocessing and postprocessing). They are used to extend the functionalities of your application. They may help you store, publish or collect metrics on your algorithms. 

The signature of a pre hook is as follows.
```python
# An effectful function that takes an array of feature data and a dictionary of metadatas and returns nothing.
# the array of feature data is raw (not preprocessed)
Callable[[np.ndarray, Dict], None]
```
It is a function that takes two arrays as input: the features and the metadatas. The signature of a post hook is as follows.
```python
# An effectful function that takes an array of feature data, an array of label data, a dictionary of metadatas and returns nothing.
# the arrays of both feature and label data are raw (not pre or post processed)
Callable[[np.ndarray, np.ndarray, Dict], None]
```
It is a function that takes three arrays as input: the features, the labels and the metadatas.

To register a hook, you can call the `register_pre_hook` and `register_post_hook` functions.
```python
# Register a hook that will insert the predictions in a SQlite database.
PROJECT_A.register_post_hook(sqlite_hook('example.db'))
```

### Example: SQlite hook

```python
# A function that performs an effect (SQL table creation) and returns a hook function.
def sqlite_hook(filename) -> Callable[[np.ndarray, np.ndarray, Dict], None]:
      conn = sqlite3.connect(filename)
      conn.execute('''
          CREATE TABLE IF NOT EXISTS predictions(
              id BLOB,
              features BLOB,
              labels BLOB,
              examples BLOB,
              created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
              updated_at TIMESTAMP
          )
      ''')
      conn.execute('''
          CREATE TABLE IF NOT EXISTS entities(
              prediction_id BLOB,
              key TEXT,
              VALUE TEXT
          )
      ''')
      return partial(insert_prediction, conn)
```
It is an effectful function that execute a SQL query to create the table and returns a partially applied function with the following signature.
```python
def insert_prediction(conn, features, metadata, labels):
```

## Serving 

### Rest Api
```python
MlRestApp([(PROJECT_A, '/projecta', ['application/json'])]).start()
```
Your project will be served on the route `/projecta` and will accept content type `application/json` only. The content of the body will be passed to your preprocessing function along with the type of data.


## Putting it all together
Lets write a simple project that we will serve on a Http Rest Api.
```python
if __name__ == "__main__":
      PROJECT_A = Project(
          "id",
          prediction_functions=[
              lambda features: [ [x*2 for x in X] for X in features]
          ]
      )

      PROJECT_A.register_post_hook(sqlite_hook('example.db'))
      print(
          'Test prediction: {}'.format(
              PROJECT_A.predict([[1]], 'numpy', {'userId': '42'})
          )
      )

      MlRestApp([(PROJECT_A, '/projecta', ['application/json'])]).start()

```

# Udacity: Intro to Machine Learning
Course: https://classroom.udacity.com/courses/ud120

## Lesson 2: Naive Bayes

### From Scatterplots to Predictions

One of the essential questions in machine learning is, what can we say about a new data point that we've never seen before, given the past data.

### From Scatterplots to Decision Surfaces

Decision Surface: Machine learning will attempt to define a boundary such that a new data point will fall on one side of the line or the other.

Linear decision surface: A decision surface defined by a straight line through the graph.

Google "sklearn naive bayes" and find [this page](https://scikit-learn.org/stable/modules/generated/sklearn.naive_bayes.GaussianNB.html#sklearn.naive_bayes.GaussianNB). Then run the code from the first example in Python 3.

```python
# Set up training points
import numpy as np
X = np.array([[-1, -1], [-2, -1], [-3, -2], [1, 1], [2, 1], [3, 2]])
Y = np.array([1, 1, 1, 2, 2, 2])

# Import the GaussianNB library
from sklearn.naive_bayes import GaussianNB
# Classifier
clf = GaussianNB()
# Call the fit function on the classifier, which takes two arguments, X: features and Y: labels
clf.fit(X, Y)

# Ask the classifier to predict which class this new point falls within
print(clf.predict([[-0.8, -1]]))

clf_pf = GaussianNB()
clf_pf.partial_fit(X, Y, np.unique(Y))

print(clf_pf.predict([[-0.8, -1]]))
```

Quiz: Calculating NB Accuracy
```python
def NBAccuracy(features_train, labels_train, features_test, labels_test):
    """ compute the accuracy of your Naive Bayes classifier """
    ### import the sklearn module for GaussianNB
    from sklearn.naive_bayes import GaussianNB

    ### create classifier
    clf = GaussianNB()

    ### fit the classifier on the training features and labels
    clf.fit(features_train, labels_train)

    ### use the trained classifier to predict labels for the test features
    pred = clf.predict(features_test)

    ### calculate and return the accuracy on the test data
    ### this is slightly different than the example, 
    ### where we just print the accuracy
    ### you might need to import an sklearn module
    accuracy = clf.score(features_test, labels_test)
    return accuracy #=> {"accuracy": "0.884"}
```

One of the important things we just did was we trained and tested on two different sets of data. This is important in ML. If you don't you'll risk over fitting to the training data. So you should always save about 10% of your data to use as your testing set.
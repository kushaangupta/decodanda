# Decodanda
<hr>

Decodanda ([dog-latin](https://en.wikipedia.org/wiki/Dog_Latin) for "to be decoded") is a best-practices-made-easy Python package for decoding neural data.

Decodanda is designed to expose a user-friendly and flexible interface while implementing the best practices to avoid the most common pitfalls in population activity decoding.

Some of the best practices implemented in Decodanda are:
- Balancing classes
- Cross validation
- Creation of pseudo-population data
- Null model to test significance of the performance
- When handling multiple variables, ```Decodanda``` balances the data to disentangle the individual variables and avoid the confounding effects of correlated conditions.

Please refer to [examples.ipynb](https://github.com/lposani/decodanda/blob/master/examples.ipynb) for some usage examples.

For a guided explanation of some of the best practices implemented by Decodanda, you can refer to [my teaching material](https://tinyurl.com/ArtDecod) for the Advanced Theory Course in Neuroscience at Columbia University.

For any feedback, please contact me through [this module](https://forms.gle/iifpsAAPuRBbYzxJ6).


Have fun!
## Getting started

### Install
<hr>
To install Decodanda, run

```bash
python3 setup.py install
```

from the home folder of the package. It is recommended to use a [virtual environment](https://packaging.python.org/en/latest/tutorials/installing-packages/#creating-and-using-virtual-environments) to manage packages and dependencies.

### Decoding one variable from neural activity
<hr>

All decoding functions are implemented as methods of the ```Decodanda``` class. 
The constructor of this class takes two main objects:

- ```data```: a single dictionary, or a list of dictionaries, containing the data to analyze. 
In the case of N neurons and T trials (or time bins), each data dictionary must contain:
  - the **neural data**: 
  
    ```TxN``` array, under the ```raster``` key (you can specify a different one)
  - the **values of the variables** we want to decode

    ```Tx1``` array per variable
  - the **trial** number (independent samples for cross validation): 
     
    ```Tx1``` array


- ```conditions```: a dictionary specifying what variable(s) and what values we want to decode from the data.

For example, if we want to decode the variable ```stimulus```, which takes values ```A, B``` from simultaneous recordings of N neurons x T trials we will have:
```python
from decodanda import Decodanda

data = {
    'raster': [[0, 1, ..., 0], ..., [0, 0, ..., 1]],   # <TxN array>, neural activations 
    'stimulus': ['A', 'A', 'B', ..., 'B'],             # <Tx1 array>, labels
    'trial':  [1, 2, 3, ..., T],                       # <Tx1 array>, trial number
}

conditions = {
    'stimulus': ['A', 'B']
}

dec = Decodanda(
        data=data,
        conditions=conditions)

```

To perform the cross-validated decoding analysis for `stimulus`, call the ```decode()``` method

```python
performances, null = dec.decode(
                        training_fraction=0.5,  # fraction of trials used for training
                        cross_validations=10,   # number of cross validation folds
                        nshuffles=20)           # number of null model iterations
```
which outputs
```text
>>> performances
{'stimulus': 0.84}  # mean decoding performance over the cross_validations folds
>>> null
{'stimulus' [0.55, 0.43, 0.57 ... 0.52]}  # nshuffles (20) values
```



<br/>

### Decoding multiple variables from neural activity
<hr>

It often happens that different conditions in an experiment are correlated to each other. 
For example, in a simple stimulus to action association task (```stimulus: A``` -> ```action: left```; ```stimulus: B``` -> ```action: right```), the action performed by a trained subject would clearly be correlated to the presented stimulus.

This correlation makes it hard to drawn conclusions from the results of a decoding analysis, as a good performance for one variable could in fact be due to the recorded activity responding to the other variable: in the example above, one would be able to decode ```action``` from a brain region that only represents ```stimulus```, and viceversa.

To disentangle variables from each other and avoid the confounding effect of correlated conditions, Decodanda implements a multi-variable balanced sampling in the ```decode()``` function.
In practice, the traning and testing data are sampled by making sure that all the values of the other variables are balanced. 

Taking from the example above, the ```decode()``` function will first create training and testing data for ```stimulus: A``` and ```stimulus: B```, each containing a 50%-50% mix of ```action: left``` and ```action: right```. 
It will then run the cross-validated decoding analysis as explained above.
The result will therefore be informative on whether the recorded activity responds to the variable ```stimulus``` alone, independently on ```action```.

To make Decodanda balance two or more variables, the ```conditions``` dictionary must contain two or more ```key: values``` associations, each representing a variable name and its two values.

```python
from decodanda import Decodanda

data = {
    'raster': [[0, 1, ..., 0], ..., [0, 0, ..., 1]],   # <TxN array>, neural activations 
    'stimulus': ['A', 'A', 'B', ..., 'B'],             # <Tx1 array>, values of the stimulus variable
    'action': ['left', 'left', 'right', ..., 'left'],  # <Tx1 array>, values of the action variable
    'trial':  [1, 2, 3, ..., T],                       # <Tx1 array>, trial number
}

conditions = {
    'stimulus': ['A', 'B'],
    'action': ['left', 'right']
}

dec = Decodanda(
        data=data,
        conditions=conditions)


# The decode() method will now perform a cross-validated decoding analysis for both variables.
# The multi-variable balanced sampling will ensure that each variable is decoded independently on the other variables.

performances, null = dec.decode(
                        training_fraction=0.5,  # fraction of trials used for training
                        cross_validations=10,   # number of cross validation folds
                        nshuffles=20)           # number of null model iterations
```
returns:
```text
>>> performances
{'stimulus': 0.84, 'action': 0.53}  # mean decoding performance over the cross_validations folds
>>> null
{'stimulus' [0.55, 0.45 ... 0.52], 'action': [0.54, 0.48, ..., 0.49]}  # 2 x nshuffles (20) values
```
From which we can deduce that ```stimulus``` is represented in the neural activity, while we are not able to decode ```action``` better than chance with a linear classifier approach.
### Balance data for different decoding analyses to compare performances
<hr>

Say we have a simple experiment where the subject is presented to four stimuli: ```[A, B, C, D]```.
We are interested in examining whether the recorded activity responds to A vs. B, and if this distinction is stronger than C vs. D.

As a first analysis, we could use ```Decodanda``` to decode ```stimulus: A``` from ```stimulus: B``` and get a performance, and compare it with the one we obtain by decoding ```stimulus: C``` from ```stimulus: D```:

```python
dec_AB = Decodanda(
  data=data,
  conditions={'stimulus': ['A', 'B']})

dec_CD = Decodanda(
  data=data,
  conditions={'stimulus': ['C', 'D']})

decoding_params = {
  'training_fraction': 0.5,  # fraction of trials used for training
  'cross_validations': 10,   # number of cross validation folds
  'nshuffles': 20            # number of null model iterations
}
perf_AB, null_AB = dec_AB.decode(**decoding_params)
perf_CD, null_CD = dec_CD.decode(**decoding_params)

```
```
>>> print(perfAB, perfCD)
0.89, 0.75
```
However, in our data stimulus A and stimulus B are presented 100 times, while C and D only 10 times. Therefore, the two decoding performances are not directly comparable.

To 

### Decoding from pseudo-populations data 
<hr>

TODO

### CCGP 
<hr>

TODO

### `Decodanda()` constructor parameters

| parameter                   | type                                                                                                                                                                      | description                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
|-----------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `data`                      | `dict` or `list` of dicts                                                                                                                                                 | The data used to decode, organized in dictionaries. Each data dictionary object must contain <br/> - one or more variables and values we want to decode, each in the format <br/> `<var name>: <Tx1 array of values>` <br/> -`raster: <TxN array>`<br/> the neural features from which we want to decode the variable values <br/> - a `trial: <Tx1 array>`<br/> the number that specify which chunks of data are considered independent for cross validation <br/> <br/> if more than one data dictionaries are passed to the constructor, `Decodanda` will create pseudo-population data by combining trials from the different dictionaries. |
| `conditions`                | `dict`                                                                                                                                                                    | A dictionary that specifies which values for which variables of `data` we want to decode, in the form `{key: [value1, value2]}` <br/><br/>If more than one variable is specified, `Decodanda` will balance all conditions during each decoding analysis to disentangle the variables and avoid confounding correlations.                                                                                                                                                                                                                                                                                                                        |
| `classifier`                | Possibly a `scikit-learn` classifier, but any object that exposes `.fit()`, `.predict()`, and `.score()` methods should work. <br/><br/> default: `sklearn.svm.LinearSVC` | The classifier used for all decoding analyses                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| `neaural_attr`              | `string` <br/><br/> default: `'raster'`                                                                                                                                   | The key of the neural features in the `data` dictionary                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| `trial_attr`                | `string` or `None` <br/><br/> default: `'trial'`                                                                                                                          | The key of the trial attribute in the `data` dictionary. Each different trial is considered as an independent sample to be used in the cross validation routine, i.e., vectors with the same trial number always goes in either the training or the testing batch. If `None`: each contiguous chunk of the same values of all variables will be considered an individual trial.                                                                                                                                                                                                                                                                 |
| `trial chunk`               | `int` or `None` <br/><br/>default: `None`                                                                                                                                 | Only used when `trial_attr=None`. The maximum number of consecutive data points with the same value of all variables that are numbered with the same trial number.                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| `exclude_contiguous_chunks` | `bool`<br/><br/>default: `False`                                                                                                                                          | Only used when `trial_attr=None` and `trial_chunks != None`. Discards trials, defined as chunks of `trial_chunk` data each with the same variable values, that are consecutive in time. Useful to avoid decoding temporal artifacts when there are long auto-correlation times in the neural activations (e.g., calcium imaging)                                                                                                                                                                                                                                                                                                                |
| `min_data_per_condition`    | `int`<br/><br/>default: 2                                                                                                                                                 | The minimum number of data points per each *condition*, defined as a specific combination of variable values, that a data set needs to have to be included in the analysis. In the case of pseudo-simultaneous data, datasets that do not meet this criterion will be excluded from the analysis. If no datasets meet the criterion, the constructor will raise an error.                                                                                                                                                                                                                                                                       |
| `min_trials_per_condition`  | `int`<br/><br/>default: 2                                                                                                                                                 | The minimum number of unique trial numbers per each *condition*, defined as a specific combination of variable values, that a data set needs to have to be included in the analysis. In the case of pseudo-simultaneous data, datasets that do not meet this criterion will be excluded from the analysis. If no datasets meet the criterion, the constructor will raise an error.                                                                                                                                                                                                                                                              |
| `exclude_silent`            | `bool`<br/><br/>default: `False`                                                                                                                                          | Excludes all silent population vectors (only zeros) from the analysis.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| `verbose`                   | `bool`<br/><br/>default: `False`                                                                                                                                          | If `True`, prints most operations and analysis results.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| `fault_tolerance`           | `bool`<br/><br/>default: `False`                                                                                                                                          | If `True`, raises a warning instead of an error when no datasets meet the inclusion criteria.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |

<br/>

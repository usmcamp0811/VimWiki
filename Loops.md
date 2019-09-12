---
jupyter:
  jupytext:
    text_representation:
      extension: .md
      format_name: markdown
      format_version: '1.1'
      jupytext_version: 1.2.1
  kernelspec:
    display_name: py3.7
    language: python
    name: py3.7
---

# How to Parallelize For Loops with Dask

I am writing this to demonstrate how to parallelize a for loop where you might have multiple lists that
need to be appended to during the loop. 

This cell is a function that contains a `for loop` that we parallelized with Dask. The `for loop` is suppose
to add data to three different arrays.

```python
import pandas as pd
import numpy as np
from dask import delayed
import time


def function_with_loop():
    """
    A Function that has a really long `for loop` that needs to be parallelized
    """
    A = np.random.randint(100)
    B = np.random.randint(100)

    list_of_something = []
    # do the loop
    for a in range(A):
        for b in range(B):
            # do the thing that is happening in the loop
            lazy_out = do_something(b)  # returns a delayed object
            # add the delayed object to a list so we can combine it later
            list_of_something.append(lazy_out)
    # declare the containers for holding out loops outputs
    array1 = []
    array2 = []
    array3 = []

    # another loop
    for thing in list_of_something:
        # need another delayed function to combine delayed things
        arrays = combine_things(thing, (array1, array2, array3))

    return arrays
```

This is the code that would normally be inside the `for loop`, but we break it out into 
this function and wrap it in `dask.delayed`. Another thing to take note of is that it
would normally output a tuple of three arrays, but I chose to use a `dict` in order to
make things a little more readable. 

```python
@delayed
def do_something(x):
    """
    A notional function that needs to be done on a large amount of data

    returns: dict of 3 arrays
    """
    array1 = []
    array2 = []
    array3 = []

    for i in range(x):
        time.sleep(np.random.randint(4))
        array1.append(np.random.random_sample())
        array2.append(np.random.random_sample())
        array3.append(np.random.random_sample())

    # could be a tuple of lists or any other combo
    # but I like a dictionary because its more explicit
    return dict(array1=array1, array2=array2, array3=array3)
```

This is function is the trick to being able to parallelize the updating of more than one array/list in
a `for loop`. 

```python
@delayed
def combine_things(things, arrays):
    """
    A helper function for combining the multiple outputs of our delayed function
    """
    # divide up our buckets
    array1, array2, array3 = arrays[0], arrays[1], arrays[2]
    array1.extend(things["array1"])
    array2.extend(things["array2"])
    array3.extend(things["array3"])
    # return the three arrays
    return array1, array2, array3
```

This is how we distribute this code across a cluster. 

```python
from distributed import Client, LocalCluster
cluster = LocalCluster()
client = Client(cluster)

output = function_with_loop()

output = output.compute()

print(output)
```


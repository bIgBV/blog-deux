+++
title = "Remember kids `dict.copy()` is a shallow copy"
date = 2016-02-27

[taxonomies]
categories = ["programming", "python"]
+++

So imagine you need to generate a few thousand models for creating some default
data. Now you the data you are generating is the pretty much all the same for
all these models and you decide to keep the common default data in
a dictionary:

```py
just_some_data = {
    'nothing': 'to',
    'see': 'here',
    'we': {
        'need': [],
        'to': go.deeper
    }
}
```

Simple enough right ? Well, you realise that you need to modify the list in the
nested dictionary by having a single default item unique to each model being
generated. No big deal, you just make a copy of the dictionary and modify the
required data structure.

```py
just_some_data = {
    'nothing': 'to',
    'see': 'here',
    'we': {
        'need': [],
        'to': go.deeper
    }
}

# Loop to generate models
for in range(a_few_thousand):
    copied_data = just_some_data.copy() # This will totally work
    copied_data['we']['need'].append(random_cool_data)
    generate_model_with_default_data(copied_data)
```

You go thorugh it once thinking all is well, and run the script. You notice
that each model takes longer to create than the last one. Wondering what's
wrong you stop the script and have a look at the models being created.

To your horror you realise that for some inexplicable reason the `copy()` did
not work! The list in the nested dictionary of the last model created has all
the unique values of all the previous models created.

So off you go frantically searching google and StackOverflow for answers. And
then you are hit by the fact that **dict.copy() is a shallow copy**. Therefore
when the `copy()` was called on the dictionary only the top level keys and
values were copied. The nested structures were still references to the
structures in the original dictionary. So each time you were appending to the
inner list, the object was being appended to the origin list.

After a bit of StackOverflow prospecting, you learn about the
[copy](https://docs.python.org/3/library/copy.html) module in python which
has the method `deepcopy` which as the name suggests copies the entire object
over, including nested structures. But, keep in mind, you probably shouldn't be
copying highly nested structures since it would have a performance impact.

So after fixing the script and running it for real this time, you learn the
valuable lesson of python and it's data structures.

**Disclaimer** This happened to a friend of mine, definitely not me... ha ha.

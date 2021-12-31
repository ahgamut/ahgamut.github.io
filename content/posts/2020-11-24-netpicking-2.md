---
categories: machine-learning
date: "2020-11-24T00:00:00Z"
slug: netpicking-2
title: 'Netpicking Part 2: Generating the networks'
katex: true
aliases: "/machine-learning/2020/11/24/netpicking-2"
---

In [Netpicking Part 1][part1], I described a dilemma in picking a neural network for MNIST. I went through
summary stats for 1001 different generated networks. This post explains how I generated these networks. 

## Representing a Neural Network ##

The MNIST problem requires finding a function 

$$ \mathit{f} : \R^{784} \rightarrow \{0,1,2,3,4,5,6,7,8,9\} $$ 

such that $ \mathit{f} $ performs well on a target dataset. I can solve this as a **multi-class
classification problem** using neural networks, and constrain the space of functions $ \mathit{f} $:

1. Every $ \mathit{f} $ must accept an input of size `784`
2. Every $ \mathit{f} $ must provide an output of size `10` for each input
3. $ \mathit{f} $ is trained using cross-entropy loss i.e. the output goes through a `SoftMax` layer

A neural network can be represented as:

* a parameterized function, used in textbooks when teaching the theory
* a *directed acyclic graph* or DAG, which provides a visually friendly representation of the flow of
    operations
* *text obeying a particular grammar*, which is how neural nets are described in a programming language

For example, in PyTorch, a sample $ \mathit{f} $ satisfying the above constraints is represented like this:

```python
class Basic(nn.Module):
    def __init__(self):
        nn.Module.__init__(self)
        self.l1 = nn.Linear(in_features=784, # constraint 1
                            out_features=10, # constraint 2
                            bias=True)
        self.ac = nn.LogSoftmax(dim=1) # constraint 3

    def forward(self, x):
        # DAG represented in text as function calls.
        x = self.l1(x)
        x = self.ac(x)
        return x
```

How about a sample using 2D convolutions?

```python
class Conv2dReLU_12(nn.Module):
    def __init__(self):
        nn.Module.__init__(self)
        self.f0 = nn.Conv2d(in_channels=1, out_channels=62, kernel_size=(1, 1), bias=True)
        self.f1 = nn.ReLU()
        self.f2 = nn.Conv2d(in_channels=62, out_channels=18, kernel_size=(5, 5),)
        self.f3 = nn.Conv2d(in_channels=18, out_channels=22, kernel_size=(11, 11), bias=True)
        self.f4 = nn.ReLU()
        self.f5 = nn.Conv2d(in_channels=22, out_channels=10, kernel_size=(14, 14),)
        self.f6 = nn.LogSoftmax(dim=1) # constraint 3

    def forward(self, *inputs):
        x = inputs[0]
        x = x.view(x.shape[0], 1, 28, 28) # constraint 1
        # DAG represented in function calls.
        x = self.f0(x)
        x = self.f1(x)
        x = self.f2(x)
        x = self.f3(x)
        x = self.f4(x)
        x = self.f5(x)
        x = x.view(x.shape[0], 10) # constraint 2
        x = self.f6(x)
        return x
```

Now, a leap of faith generalization.  Every function $ \mathit{f} $ that satisfies the above constraints
will follow the below template:

```python
class Network(nn.Module):
    def __init__(self): # possibly some args, kwargs
        nn.Module.__init__(self)
        # a sequence of layer declarations
        self.activation = nn.LogSoftmax(dim=1)

    def forward(self, *inputs):
        x = inputs[0]
        # check constraint 1
        # represent DAG in function calls
        # check constraint 2
        x = self.activation(x) # check constraint 3
        return x
```

Having networks follow this template would save time when writing boilerplate code for train/validation/test
cycles. Let's add another simplifying constraint: if the neural network DAG is forced to be a straight line,
the function calls in the `forward` method can be in <u>the same order</u> as the declarations.  How do I
start designing such a template?

### `Jinja2`

From the `Jinja2` [website][Jinja2] (emphasis mine):

> Jinja is a modern and designer-friendly templating language for Python, modelled after Django's templates.
> [...] A Jinja template is simply a text file. Jinja can generate *any text-based format*.

Any text-based format, so the above Python code block also applies. The `Jinja2` templating language provides
mathematical operators, logical operators, `if-else`, and `for` statements. If I create a template similar to
the `Network` class above, ~~instantiating~~[^cpp] rendering that template with different parameters should
get the 1000 networks. Each network must have:

1. (**Constraint 1**): an `input_shape` member, which can be used to shape the input.[^shaping]
2. **a sequence of declarations**. Naming the layers is simple (a loop with `self.f1`, `self.f2` ...), 
    but generating the layer declaration on the RHS seems complicated.
3. **a sequence of function calls** (the simplified DAG) in the `forward` method. A loop with `x = self.f{{ i }}(x)`.
4. (**Constraint 2**): its output shape cast to `x,10` after the all the function calls.
5. (**Constraint 3**): a `LogSoftmax` layer after the template declarations, and call it last.

Is declaring a layer really that complex? Let's look at it again:

```python
   self.f5 = nn.Conv2d(in_channels=22, out_channels=10, kernel_size=(14, 14),)
```

Suppose I had an object `x` of type `Conv2d`, such that `str(x)` returned `"Conv2d(in_channels=22,
out_channels=10)"`?  Are there classes like this?

The Python standard library provides [`collections.namedtuple`][namedtuple], which has the right format for
stringified output.  But then I need to write `namedtuple` equivalents for so many classes! I wonder if there
is a way to examine (or inspect) the methods of a class to produce a `namedtuple`.

### `inspect`

From the [documentation][inspect], the `inspect` module in the Python standard library allows one to (emphasis
mine):

> [...] get information about live objects such as modules, classes, methods, functions, tracebacks, frame
> objects, and code objects [...] *examine the contents of a class*, retrieve the source code of a method,
> *extract and format the argument list for a function*, or get all the information needed to display a
> detailed traceback. 

For a class `A`, I'd like to get a `namedtuple` that has the same arguments and defaults as `A.__init__` , so
that I can generate a string `A(par1=val1, par2=val2)`. `inspect` is perfect for this.

```python
from collections import namedtuple
import inspect

def get_namedtuple(obj):
    klass = obj if inspect.isclass(obj) else type(obj)
    sig = inspect.signature(klass.__init__)
    params = {}
    for name, par in sig.parameters.items():
        if name in ("self", "*args", "**kwargs"):
            continue
        params[name] = ""
        if par.default != inspect.Parameter.empty:
            param[name] = par.default
    tmpl_string = namedtuple(klass.__name__, tuple(params.keys()))
    tmpl_string.__new__.__defaults__= tuple(params.values()))
    return tmpl_string

print(get_namedtuple(nn.ReLU)(inplace=True))
# ReLU(inplace=True)
print(get_namedtuple(nn.Linear)(in_features=2, out_features=3))
# Linear(in_features=2, out_features=3, bias=True)
```

Good enough; with the appropriate parameters, I can instantiate a `namedtuple` that prints the exact layer
declaration I want.[^meta] 

## Generating the 1000 Neural Networks

Though [AutoML][AutoML] has been around for [quite some time][Auto2], I didn't want to generate networks for
this exercise with any optimization in mind. The aim was to have 1000 networks obeying the 4 constraints; I
decided to use random parameters while instantiating each layer. 

* `bool` parameters are `True` with a probability in $ [0, 1] $.
* `int`/`float` parameters are randomly chosen from a given range with uniform probability.
* shape parameters like `kernel_size` are square i.e. only one random `int` selected.

Armed with the `inspect`/`Jinja2` combo, I wrote a [generation script][rando-driver] that would:

1. Select the number of layers in the network.
2. Select a computation layer (`Conv1d`, `Conv2d`, `Conv3d`, `Linear`, or `BasicBlock`).
3. Select an activation layer (None, `ReLU`, `SeLU`, `Sigmoid`, `Tanh`).
4. Generate each layers of the network one by one with random parameters, using a `namedtuple` template.

    1. Check that the generated layer can accept the input shape
    2. Precompute the output shape of the layer based on the input shape

5. Ensure the generated network satisfies all constraints.
6. Repeat to generate 1000 networks across different kinds of computation/activation combinations.

The majority of debugging the script was in step 4: the input tensor would pass through a particular layer,
change shape, and would then be incompatible as input for the next layer. This was particularly annoying with
the inputs for ResNet `BasicBlock` layers, which require an input of *larger than* a particular size, which is
not obvious from the declaration. 

On the same note, look at the `Conv2dReLU_12` code block again: Suppose it is known that the input is of shape `(1,
1, 28, 28)`, and the DAG of the neural network is provided in text. It should be possible to tell what the
shape of the output is at any point in the DAG *before* running the script to train/test the network. I know
the [`Conv2d` documentation][Conv2d] includes the calculation of output shape from input shape, but having an IDE
plugin to provide the shapes would save a lot of time. Alternatively, type-checking the shapes before running
the network could help as well.[^shapetype] 

## Closing notes

"Dumb" AutoML can be realized by generating neural nets following a flexible template, followed to selecting
one with the "best" performance characteristics. The [`randonet`][randonet] package (currently version 0.0.1)
contains the code involved to generate networks according to the ideas described above. I used it to generate
the networks for [`mnistk`][mnistk] (you can see the difference between [generated code][mnistk-gen] and the
[handwritten code][mnistk-manual]).

* Writing neural network programs involves boilerplate/scaffolding code, which can be offset with some
  templating (design patterns?) tailored to the specific problem at hand. I know wrapper packages exist, but
  when I last tried them I got lost between the abstractions and my customizations.

* The `inspect`/`Jinja2` combo has potential, especially for use cases involving the generation of contextual
    information from objects/function in a package: it can possibly be used for generating documentation or
    boilerplate code.

* Randomly generating a valid neural net program involves a lot of constraints, some of which are not
    apparent until the program is run. Removing some constraints would lead to wackier network architectures
    (an unconstrained DAG instead of a line graph, rectangular/cuboidal convolutions).



[^shaping]: 

    I added another simplification here: `Conv2d`/`ResNetStyle` networks have an input of shape `(N, 1, 28,
    28)`, `Conv1d` networks have `(N, 28, 28)` (yes, 28 channels), `Conv3d` networks have `(N, 16, 7, 7)`, and
    `Linear` networks have `(N, 784)`. I realized later that another layer of randomness could be added by
    listing all 2-factor and 3-factor combinations of 784, but by then I had gotten bored of debugging
    templated Python code.

[^meta]:
    
    Of course, there were too many classes in `torch.nn` to call this function class by class, so I wrote
    [another script][rando-factory] to generate the `namedtuple`s corresponding to each class, instantiate
    with the appropriate random values, and write the rendered templates to a file.  Debugging that was
    horrible: I had to typecheck the parameter defaults (PyTorch has an `int` as default for `kernel_size`
    instead of a `tuple`) generate the `namedtuple`s, check if I could generate text using them in the
    template, and *then* check if the generated text was valid Python code. 

[^shapetype]:
    
    Can the shape of input/output tensor be provided as a type annotation? How would that even work in Python?


[namedtuple]: https://docs.python.org/3.6/library/collections.html#collections.namedtuple
[inspect]: https://docs.python.org/3.6/library/inspect.html
[Conv2d]: https://pytorch.org/docs/stable/generated/torch.nn.Conv2d.html#torch.nn.Conv2d
[AutoML]: https://en.wikipedia.org/wiki/Automated_machine_learning
[Auto2]: https://github.com/hibayesian/awesome-automl-papers
[rando-driver]: https://github.com/ahgamut/randonet/blob/master/driver.py
[rando-factory]: https://github.com/ahgamut/randonet/blob/master/src/randonet/generator/factory_gen.py
[randonet]: https://github.com/ahgamut/randonet
[mnistk]: https://github.com/ahgamut/mnistk
[mnistk-gen]: https://github.com/ahgamut/mnistk/blob/master/src/mnistk/networks/conv1drelu_20.py
[mnistk-manual]: https://github.com/ahgamut/mnistk/blob/master/src/mnistk/run/trainer.py
[Jinja2]: https://jinja.palletsprojects.com/en/2.11.x/
[part1]: {{< relref "2020-11-20-netpicking-1.md" >}}
[^cpp]:
    
    Too much time around `C++` templates and the mountain of errors I generate using them.



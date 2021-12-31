---
categories: "machine-learning"
date: "2020-11-20T00:00:00Z"
title: 'Netpicking Part 1: Hello MNIST'
aliases: "/machine-learning/2020/11/20/netpicking-1"
---


The `Hello World!` introduction to neural network libraries has the user write a small network for the
[MNIST][MNIST] dataset, train it, test it, get 90% accuracy or more, and thereby get a feel for how the
library works.  When I started using [PyTorch][pytorch], I followed such a tutorial on their website. But I
wondered why a network with `Conv2d` and `ReLU` was picked in the tutorial. Why not a different convolution or
a `Linear` layer with a `Sigmoid` activation?

When someone designs a neural network, why do they pick a particular architecture?  Obviously, some
conventions have set in, but the primary reason is performance: Network *A* got a higher accuracy (or AUC)
than Network *B*, so use *A*. Is there more information I can use when making this decision?  Let's look at a
bunch of networks and find out.

## Measurements and Baselines

Comparing a bunch of networks seems similar to a programming contest:

| A regular program's | is somewhat like a neural net's |
|---------------------|---------------------------------|
| Accuracy on test cases | Accuracy on test set |
| Compilation Time  | Training Time | 
| Binary size | Number of parameters |
| Memory usage | Memory required to process a single input |
| Algorithm Complexity | number of `ops` in the computation |

I designed `Basic`, a simple[^basic] model with bias as the baseline for the metrics.  I trained it for `4`
epochs, with the resulting performance:

| network | weights | memory usage | training time | number of ops | accuracy |
|---------|---------|--------------|---------------|---------------|----------|
| `Basic` | 7850 | 1588 | 52.14s | 4 | 0.912 |

The next step is design a bunch of "similar" networks and obtain their performance metrics.

## Getting a bunch of networks

I quickly got bored writing different-yet-similar neural nets manually. [Yak shaving][yaks] to the rescue! I
ended up [generating][part2] 1000 neural networks following a sequential template. The networks are classified
along two axes: computation layer, and activation layer. The networks are distributed as per the below table:

|   ↓ Computation / Activation →  | `None` | `ReLU` | `SELU` | `Sigmoid` | `Tanh` |
|---------------------------------|--------|--------|--------|-----------|--------|
| `Linear` | 55 | 23 | 24 | 25 | 23 |
| `Conv1d` | 59 | 23 | 24 | 21 | 23 |
| `Conv2d` | 61 | 23 | 23 | 20 | 23 |
| `Conv3d` | 57 | 23 | 23 | 22 | 25 |
| `Conv1dThenLinear` | 34 | 17 | 17 | 17 | 15 |
| `Conv2dThenLinear` | 38 | 14 | 16 | 16 | 16 |
| `Conv3dThenLinear` | 33 | 18 | 16 | 18 | 15 |
| `ResNetStyle`[^resnet] | 0 | 100 | 0 | 0 | 0 |

For example, there are `23` networks where the computation layer is `Conv2d` and the activation layer is
`ReLU`.  

So 1001 networks. Each trained for 4 epochs. The MNIST test set contains 10000 samples. A prediction of each
input has 10 scores.  That gives a raw dataset of size `(1001, 4, 10000, 10)`, and a summary dataset of size
`(1001, 4, 20)`. Time to [crunch the numbers!][watert]

## Picking the "best" network

Let's use the below analogy:

> There are 1000 students writing the MNIST exam. The exam has 10000 questions, multiple choice.  The
> students use approved study material, which contains 60000 practice questions. Each student has taken the
> test 4 times. I have also written this exam, and I have a `Basic` idea of what a good score is.  I want to
> hire one or more students who perform well on this exam.

The following ten are just some of the queries that can be posed:

1. How did the students perform over 4 attempts?

    | Attempt\# | ≥ 80% | ≥ 90%|  ≥ 95% | ≥ 99% | ≥ 100% |
    |--|--|--|--|--|--|
    | 1 | 983 | 792 | 352 | 0 | 0 |
    | 2 | 990 | 879 | 461 | 0 | 0 |
    | 3 | 990 | 907 | 509 | 0 | 0 |
    | 4 | 993 | 924 | 527 | 5 | 0 |

   So the exam was easy, but very few got close to a perfect score.  Let's just consider the "good" students:
   those that got above 80% in all 4 test attempts. Of these, the "really good" students are those that got
   above 98% in their last test attempts, and they get a gold star.


2. I know the students studied together and developed common strategies. Which strategy led to more students
   scoring high marks?

   ![score1](/images/netpicking1/2.svg)

   Okay, `ResNetStyle` is first[^limit] (deeper networks better, skip connections are magic, blah blah), but
   what about everyone else? Unsurprisingly, `Conv2d` networks are second-best, but `Conv3d` networks seem to
   do an equally good job (lower maximum, but higher median and smaller spread). Adding `Linear` layers after
   convolution layers does not seem to be beneficial, perhaps the networks didn't have enough training epochs.

   ![score2](/images/netpicking1/2b.svg)

   * Argh! The move to consider networks without any activation was useless. Networks without
       activations are just linear functions; combining each network's weights would produce a matrix that is
       effectively equal to the one used in `Basic`.

   * As expected, networks that use the `ReLU` activation have a higher accuracy on average than any of the
       others.

   * Networks that use `SELU` activation are not as good as those with `ReLU`, but are more consistent.

   * `Sigmoid` and `Tanh` activations are both risky choices.

3. With the `Basic` strategy, I spent only `52` seconds studying for the test. How about the others?

   ![time](/images/netpicking1/3.svg)

   * The `Conv1d`, `Linear`, and `Conv1dThenLinear` networks take similar amounts of time to train. Does this
       mean that the `reshape` operation is slow? The other networks use 2D-convolutions or higher.

   * The gold stars are all across the board for `ResNetStyle` networks, and generally on the higher end for the
       others. However, the gold star in `Conv3dThenLinear` takes the *least* amount of training time in its
       class; are `Conv3d` networks slower to train?

4. With the `Basic` strategy, I had only `7850` keywords as part of my notes. How about the others?

   ![params](/images/netpicking1/4.svg)

   Again, the gold stars are on the higher end of the distributions. This could imply deeper or wider networks
   though.

5. With the `Basic` strategy, I used only `1588` pages for rough work. How about the others?

   ![mem](/images/netpicking1/5.svg)

   This plot is similar to the previous one. The memory required to hold the intermediate tensors is related
   to the layers that output these tensors, examining the gold stars individually may give some information.

6. With the `Basic` strategy, I needed only `4` steps to get an answer every time.
   How about the others?

   ![ops](/images/netpicking1/6.svg)

   Since all the networks are sequential, more operations means deeper networks. Now "deeper networks are
   better" can be seen: the higher ends have the gold stars, but not all of them.

7. Every student took the test 4 times. How did the scores change over each attempt?

   ![changes](/images/netpicking1/7.svg)

   This graph doesn't tell much. It makes a case for early stopping: in most cases, the first two
   epochs are sufficient to pick the best-trained network. There should be a better way to understand this
   data; are there any networks that were horrible in the first two epochs, and then suddenly found a
   wonderful local optimum?

8. How many questions were easy, weird, or confusing?

    Out of the 10000 samples in the test set,

    * **4247** samples were easy questions. All the networks predicted these correctly, so it is impossible to
        distinguish between the networks using any of these samples.

    * **8** samples were weird questions. More than 90% of networks predicted these *incorrectly*, but all of
        them agreed got the *same* incorrect answer.

        ![weird](/images/netpicking1/8.svg)

    * **5** samples were confusing questions. There was no clear agreement among the networks as to what the
        answer was.
        
        ![confusing](/images/netpicking1/8b.svg)

9. Let's take the student with the best score. Is this person the *best overall*?

    The network with the highest accuracy is `ResNetStyle_75`, with an accuracy of 99.1%. To be the best
    overall, it should have the highest accuracy for each class of inputs, so let's look at the *percentile*
    of `ResNetStyle_75` at predicting each digit correctly:

    | name | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 |
    |------|---|---|---|---|---|---|---|---|---|---|
    | `ResNetStyle_75` | 99.19 | 94.06 | 98.59 | 99.09 | 90.33 | 96.27 | 96.68 | 100.00 | 94.56 | 97.58 |

    So there are some networks that are more accurate than `ResNetStyle_75` at predicting individual classes.
    `ResNetStyle_75` has the worst percentile at predicting `4`s correctly. 


10. How does the best student compare to the `Basic` method?

    | network | weights | memory usage | training time | number of ops | accuracy |
    |---------|---------|--------------|---------------|---------------|----------|
    | `ResNetStyle_75` | 531605 | 254083 | 577.6s | 28 | 0.991 |
    | `Basic` | 7850 | 1588 | 52.14s | 4 | 0.912 |
    | Ratio | 67.7 | 160 | 11.07 | 7.0 | 1.08 |

    For an 8% increase in accuracy, `ResNetStyle_75` required 67x weights, 160x memory, and 11x training time.
    How many members should an *ensemble* of `Basic` networks have to get a similar accuracy combined? 11? 67?
    160?

## Closing Notes

Picking the best neural network, not surprisingly, depends on the definition of best (questions 3, 4, 5, and
6). Some use cases may value resource efficiency: a simple network with few parameters, that can be fast and
error-prone, would be easier to use in a constrained environment. Other cases may have heavy consequences
attached to a wrong prediction, and so will use a large, overparametrized network, located in a server with
multiple GPUs, to avoid error at all costs. Maybe the two extremes could work in tandem: the small network can
be provide a quick prediction, which can be checked by requesting the large network for a prediction if
needed.

* Comparing a bunch of neural networks could also reveal some features about the dataset (question 8) that is
    being used: if many networks are wrong for a given subset of the data, is the data labeled incorrectly?
    Is the training set not large/representative enough of the underlying distribution? Do all the networks
    suffer from a common issue?

* Designing a bunch of neural networks may give an idea towards the importance of a particular design
  (questions 1 and 2): do networks with similar designs get the same inputs wrong? It may also point to the
  relative benefit of training a network for more epochs (question 7).

* Designing a bunch of neural networks may also show the trade-offs involved in picking a particular network
  (questions 9 and 10). Is an ensemble of shallow networks "better" than a single deep network?

[^basic]:

    The network is just a `784x10` matrix multiplication, adding a bias vector, and a `Softmax` layer.

[^limit]:

    I realized after running all the networks that I could've modified the `BasicBlock` to use different
    activations instead of just `ReLU`, which would've given a nice square matrix of subplots, and info about
    how the `BasicBlock` architecture is affected by different activations.

[^resnet]: 
	
    The computation layer is a [ResNet `BasicBlock`][resblock].

[^mnistk]:
    
    The code for training/testing is in my Github repo [`mnistk`](https://github.com/ahgamut/mnistk). 


[MNIST]: https://en.wikipedia.org/wiki/MNIST_database
[watert]: https://www.youtube.com/watch?v=LPDn0PC6zFE
[resblock]: https://github.com/pytorch/vision/blob/master/torchvision/models/resnet.py
[pytorch]: https://pytorch.org/tutorials/beginner/deep_learning_60min_blitz.html
[yaks]: https://en.wiktionary.org/wiki/yak_shaving
[part2]: {{< relref "2020-11-24-netpicking-2.md" >}}


---
title: "Modeling Sales Pipeline using Monte Carlo Simulation"
date: 2023-09-23T12:44:58-04:00
draft: true
metathumbnail: "/sf_mc/hist.png"
description: "Monte Carlo Simulation can be used to model your historical sales pipeline data to account for some of the randomness and uncertainty in the sales process. In this post, Monte Carlo Simluation is introduced and then applied to
the same case introduced in a [previous post](/posts/modeling-a-sales-pipeline-as-a-markov-chain/)."
tags: ["python", "business development", "sales", "data modeling"]
categories: ["sales", "business development", "data modeling"]
keywords: ["python", "salesforce", "sales", "business development", "data modeling", "markov chain", "probability of opportunity close", "sales pipeline"]
---

TL;DR - Monte Carlo Simulation can be used to model your historical sales pipeline data to account for som of the randomness and uncertainty in the sales process. In this post, Monte Carlo Simluation is introduced and then applied to
the same case introduced in a [previous post](/posts/modeling-a-sales-pipeline-as-a-markov-chain/).

## What is Monte Carlo Simulation?
Monte Carlo Simulation is used to simulate a process using random sampling. It is a very general technique that can be used to model all kinds of processes in fields ranging from physics and chemistry, to economics and finance. In previous
posts, I used Monte Carlo Simulation to [Play 'Shut the Box'](/posts/shut-the-box/) and to [esimate the value of pi](/posts/computing-pi-by-throwing-darts/). Here, I use it to approach the sales pipeline
model introduced in a [previous post](/posts/modeling-a-sales-pipeline-as-a-markov-chain/) differently. Basically, that approach used a Markov Chain to model the sales pipeline. The probablities used in the Markov Chain were computed from the data, and understood to represent
the overall average probability of moving from one stage to another, or at least the best we could do to estimate that probability. Even if we were right
about those probabilities on average, a lot of different outcomes are possible for a given starting distribution of opportunties.

## Coin Example
As an example, the probability of flipping heads on a fair coin is 0.5, but if you flip a coin 10 times, you might get 7 heads and 3 tails, or 5 heads and 5 tails, or 10 heads and 0 tails, etc. Obviously some of those outcomes are more likely than others,
but they are all possible. Monte Carlo is a way to explore all the possibilities using random sampling. As an example, we can flip 10 coins 1000 times and keep track of how many heads we get each time. Here's some Python code to do that:

```python
import numpy as np

def flip_coin():
  return 1 if np.random.uniform() > 0.5 else 0

num_coins = 10
num_sims = 1000

results = {}
for i in range(num_sims):
  results[i] = np.sum([flip_coin() for _ in range(num_coins)])

vals = list(results.values())
print(f"Max: {np.max(vals)}, Min: {np.min(vals)}, Avg: {np.mean(vals)}")

```
The output will be different each run because it's random. You can see that even though we expect that on average we have 5 heads and 5 tails, we can get a lot of different results depending on our random sampling.

In some cases, you'll see that the maximum number of heads was 10, the minimum was 0, and the average was 5.03 - so on average we always get pretty close to the expected value of 5 heads, but at least once we flipped 10 coins and got no heads, and another time we got 10 heads!

The goal of applying Monte Carlo simulation to a process is often to explore
these different possible scenarios and evaluate their relative probabilities. Of course, this can be computed in the coin example, but is often hard or impossible to compute in other cases.

In our simple sales pipeline case, we can use Monte Carlo to explore the different possible outcomes of the sales pipeline based on a starting opportunity distribution and the average transition probabilities based on historical data. We already know what the we get using the average transition probabilities - but let's see what we get using Monte Carlo!

## Monte Carlo Simulation of the Sales Pipeline
In the previous post, we used a Markov Chain to model the sales pipeline. The probablities used in the Markov Chain were computed from the data, and understood to represent probablities of an opportunity in one stage to move to
another stage. We just need to make some updates to the script we used in the previous post to use Monte Carlo Simulation instead of updating the distribution using the transition matrix. Here's the [updated script](https://gist.github.com/heathhenley/7cc46f176c422a3c4817e958b9ab5b83):

This is the main function that runs the simulation (the full context is in the gist linked above)
```python
def perform_mc_sim(op_dist, transition_matrix):
  """ Run Monte Carlo simulation using estimate probabilities

  - Take each individiual 'opp' in the i'th stage
  - Move it to one of the possible j'th stages based on the transition matrix
      * This time, a random number is generated from a uniform distribution and
        we only move if the random number is less than the transition
        probability. So the average transition probability is the same as our
        transitition matrix, but we can incorporate randomness
  - Keep going until all the 'opps' are in "Closed Won" (index -2) or in
    "Closed Lost" (index -1)

  Returns the overall fraction of opportunities ending in "Closed Won"
  """
  op_types = len(op_dist)
  while np.sum(op_dist[-2:]) < np.sum(op_dist):
    # for each stage, take each op and decide where it moves randomly according
    # to the averages in the transition matrix
    for i in range(op_types):
      # for each actual opportunity in this stage, decide where to move it
      for _ in range(op_dist[i]):
        # generate random number (uniform)
        random = np.random.uniform()
        # check against each possible move in the transition matrix
        transition_prob = 0.0
        for j in range(op_types):
          if op_dist[i] <= 0:
            break
          prev = transition_prob
          # Note: we add here and then check that the random number is betwen
          # this probability and the last one we checked, eg if the they are:
          # [0.5 0.25 0.25 ... 0.0], if the random number is between 0 and 0.5 we
          # got to stage "0", if it's between 0.5 and 0.75 we go to stage "1",
          # etc
          transition_prob += transition_matrix[i][j]
          if prev < random < transition_prob:
            op_dist[i] -= 1
            op_dist[j] += 1
            frames_for_animation.append(op_dist.copy())
            break
        break
  return op_dist[-2]/np.sum(op_dist)
```
The script keeps going and move opportunities around until all of them end up in either "Closed Lost" or "Closed Won". It also creates an [an animation](https://www.youtube.com/watch?v=O8SJTLDscNg) of the evolution of the distribution of opportunities over time:

{{< youtube O8SJTLDscNg >}}

If we run it 1000 times, we get the following stats for the overall probability of an opportunity ending in "Closed Won", given the starting distribution of opportunities and the approximate transition probabilities:

| **Average** | **Standard Deviation** | **Max** | **Min** |
| --- | --- | --- | --- |
| 0.309 | 0.023 | 0.378 | 0.235 |

For comparison, in the Markov Chain post, we used just the average transition probabilities and got the following that 0.31 of the opportunities would end up in "Closed Won". So we can see that using Monte Carlo Simulation, we get a similar result on average, but there are some runs where we get a much higher or lower probability of ending in "Closed Won" (0.38 or 0.24, respectively).

We can even create a histogram of the results to see the approximate frequency of each value of "Closed Won":

![Histogram of Monte Carlo Results](/sf_mc/hist.png)

Using this approach, bounds can be attached the "value" of the pipeline based and other scenarios can be explored in addition to the average case.

## Conclusion
To go even further, it would be possible test the sensitivity to the estimates of the transiton matrix probabilities by simply perturbing them randomly by up a fixed amount and observing the changes to the results. Further the model could be updated as real world sales data is collected to improve the estimates of the transition matrix probabilities. Maybe I'll do that in a future post!

If you have any questions or comments, please reach out!
---
title: How to compute quantile effeciently
date: 2020-03-10
tags:
  - quantile
  - t-Digest
  - SLI Provider
---

use t-Digest to compute quantile in practise

<!--more-->

## Outline

* Background
* What is quantile
* How to calculate accurate quantile
* The problem of accurate quantile algorithm
* Histogram algorithm
* t-Digest algorithm

## Background

### SLI Provider

* SLI = Service level Indicator
* A measure of the service level provided by a service prociver to a customer
* Common metrics: latency, throughput, availability, and error rate

### Data aggregation

* average, sum...
* data ranked at 0%(min),0.1%, 1%, 50%(medium), 99%, 99.9%, 100%(max) => `quantile`

## What is quantile

> quantiles are `cut points` dividing the range of a probability distribution into continuous intervals with equal probabilities, 
> or dividing the observations in a sample in the same way.
> -- WikiPedia

### Speocialized quantiles

* 2-quantile is called the median
* 4-quantiles are called quartiles
* 100-quantiles are called percentiles

### Normalization

* kth q-quantiles => k/q quantile => p-quantile where 0 < p < 1
* 5th 100-quantiles = 0.05-quantile

## How to calculate accurate quantile

### The nearest-rank method

$$n= \lceil \frac{P}{100} \times N \rceil$$

The percentile values for the ordered list {15, 20, 35, 40, 50}

![percentile.png](percentile.png)

### The linear interpolation between closest ranks method

medium of {7, 8, 10, 15} is 8 or 9?

#### Mapping relation bewteen rank and percentile

1. the neearest-rank method

value|rank|percentile
-----|----|---------
7    |1   |25
8    |2   |50
10   |3   |75
15   |4   |100

2. Map the middle rank to 50th percentile

There are 3 variants. For large samples, the different methods agree closely

2.1. 

value|rank|percentile
-----|----|---------
7    |1   |12.5
8    |2   |37.5
10   |3   |62.5
15   |4   |87.5

$$ p = \frac{1}{N} (x-\frac{1}{2}) $$

2.2. 

value|rank|percentile
-----|----|---------
7    |1   |0
8    |2   |33.3
10   |3   |66.6
15   |4   |100

$$ p = \frac{x-1}{N-1} $$

2.3.

value|rank|percentile
-----|----|---------
7    |1   |20
8    |2   |40
10   |3   |60
15   |4   |80

$$ p = \frac{x}{N+1} $$

#### lineaer interpotation

![linearInterpolation.png](linearInterpolation.png)

#### The Apache Commons Mathematics Library

Let n be the length of the (sorted) array and 0 < p <= 100 be the desired percentile.

1. If n = 1 return the unique array element (regardless of the value of p); otherwise
2. Compute the estimated percentile position pos = p * (n + 1) / 100 and the difference, d between pos and floor(pos) (i.e. the fractional part of pos).
3. If pos < 1 return the smallest element in the array.
4. Else if pos >= n return the largest element in the array.
5. Else let lower be the element in position floor(pos) in the array and let upper be the next element in the array. Return lower + d * (upper - lower)

## The problem of accurate quantile algorithm

1. Retain all samples in memory
2. Sorting is infeasibility for very large datasets
3. Can not be used for distributed dataset

=> approximation algorithm

## Histogram algorithm

Prometheus use it for Histogram(metrics type)

![exampleHistogram.png](exampleHistogram.png)

steps:

1. Pick buckets.
2. Counts samples during the bucket.
3. Calculate quantile likes the Apache commons Mathematics library.

### Goodness

1. Only bucket(lower bound, upper bound, count) kept in memory
2. Fast and no sorting
3. Can be used for distributed dataset

### Badness

1. Artificial error introduced when picking buckets.

Can buckets be self-adaptation?

## t-Digest algorithm

### Data structure

1. Histogram: Bucket(lower bound, upper bound, count)
2. t-Digest: Centroid(mean, weight)

### Algorithm

1. aggregate
2. query

#### Aggregate: Buffer and Merge

Centroids: \[c1,c2,c3\]
Centroids left weight: \[w1,w2,w3\] //0<w1<w2<w3<=1

1. put values into Buffer: \[v1,v2,v3\] //vi = Centroid(vi,1)
2. sort Centroids & buffer: \[x1,x2,x3,x4,x5,x6\] //N = $\Sigma$xi.weight
3. centroid left weight: \[y1,y2,y3,y4,y5,y6\] //$yi = \Sigma_{j<i} x(j).weight$
4. merge from lelt to right: merge(x1,x2) , merge(x3,x4,x5) when \[y1,y2, w1, y3,y4,y5, w2, y6, w3\]

![merge.PNG](merge.PNG)

#### Scale function

how Centroids left weight: \[w1,w2,w3\] comes from?

$$ f(q)=\frac{\delta}{2\pi} sin^{-1}(2q-1)$$

![scaleFunction.PNG](scaleFunction.PNG)

#### Query

![query.PNG](query.PNG)

## Experiment

We introduce [Apache math percentile](https://commons.apache.org/proper/commons-math/javadocs/api-3.5/org/apache/commons/math3/stat/descriptive/rank/Percentile.html) as benchmark, it calculate percentile with no compression (not suitable for big data, but accurate)

![quantileCompare.PNG](quantileCompare.PNG)

In the winner column, t means t-Digest. latencyHistogram is an Object to calculate quantile using Histogram Algorithm, so l means latencyHistogram.

Only quantile 0.4 and quantile 0.9, the Bucket algorithm wins. Otherwise, the t-Digest is better.

Most t-Digest's error rate is less than 1%(except 1 case), some of them less than  0.1%.
Most latencyHistogram's error rate is larger than 1%(except 1 case), several times larger than t-Digest’s. 

So t-Digest is better.

## References

1. https://en.wikipedia.org/wiki/Service_level_indicator
2. https://en.wikipedia.org/wiki/Quantile
3. https://en.wikipedia.org/wiki/Percentile
4. https://en.wikipedia.org/wiki/Linear_interpolation
5. https://commons.apache.org/proper/commons-math/javadocs/api-3.5/org/apache/commons/math3/stat/descriptive/rank/Percentile.html
6. https://prometheus.io/docs/practices/histograms/#quantiles
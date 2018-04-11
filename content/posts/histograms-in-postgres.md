---
title: "Histograms in Postgres"
date: 2018-04-10T21:30:53-06:00
draft: false
---

Ever been searching through SQL tables and think, "Gee I wish I could just get a quick
look at a histogram of this data?" Yeah me too. Happens all the time but alas, you're
stuck in a SQL prompt somewhere and you're still 2-3 copy/pastes, an Excel array formula,
and 5-6 clicks way from reaching this nirvana.

Well fret no more because I'm about to show you how to calculate and "plot" a
histogram from _within Postgres_. That's right, you'll be making ascii art all
from within the confines of your `psql` shell.

![](/img/ascii-histogram.png)

### What is a histogram?
Histograms are way visual way of displaying the distribution of your data. It
helps answer the question, "What are the values of your data? and How often does
each value occur?".

Histograms shouldn't be confused with "frequency charts". "Frequency charts", while
they can be just as helpful, are only used for categorical (non-numerical) data.
For example, if you wanted to count how many countries name's started with each
letter of the alphabet, you'd see something like this:

![](/img/1st-letter-of-country-name.png)

So while it's a similar exercise, just know that when we're talking about histograms,
we're talking about numerical data. For example, a histogram showing the price
of a set of diamonds looks like this:

![](/img/diamonds.png)

So you can see how they're so useful! It's a great way to take a quick look at a
bunch of data without having to inspect each row individually or put everything
into a pivot table in Excel.

### Data
For this post I'm going to be using the [diamonds dataset](). It's a fairly common
sample dataset that's found in both industry and academia. It even ships with some
popular R packages like ggplot2.

The dataset consists of ~50K rows and 10 columns. In our example below we'll be
calculating a histogram for the the `price` column (just like the example above).

...insert first 5 rows of diamonds dataset...

### Methodology
Calculating histograms consists of a few steps:

- Calculate the ["bins"](https://en.wikipedia.org/wiki/Histogram)
- Counting the number of datapoints in each bin
- Making the visualization

### Calculating Your Bins
Determining the "right" number of buckets, or bins, for your histogram is a
hotly contested (or at least opinionated) topic. In short, there's no one
perfect way to calculate the number and size of your bins, you'll need to take
things on a case by case basis. There are a few common ways that have stayed
popular over the years:

- Square-root: take the sqare root of the number of data points (n)
- Sturges' formula: log-2 of n + 1
- Freedman–Diaconis': (2 * (75th percentile - 25th percentile)) / (n^(1/3)

For our case we're going to use a modified version of the square root method.
Since there's only so many rows we can display on our screen at once, we have
an upper bound on the number of bins that we can reasonably use. For practical
puposes, our output starts to break down after 50 bins, so the formula we'll
be using is:

```
min(50, sqrt(n))
```

Now we'll of course need to translate this to SQL. This is actually fairly easy
and intuitive to do:

```
select
    , min(x) as x_min
    , max(x) as x_max
    least(50, ceil(sqrt(count(x)))) as nbins
into temp
    bin_params
from
    rnorm;
```

There are a few things going on here:
1) Counting the # of instances of x we have (`n`)
2) Taking the `sqrt` of that value
3) Rounding that value up to the nearest whole number (`ceil`)
4) Taking the lower of the value of (3) and 50

As you can see we're also saving the min/max value for x. This will come in handy
later.

Next we'll create a `bin_range` table that contains the lower and upper bounds of
each bin. To do this we'll use the `generate_series` function in Postgres to
generate `n` values between `x_min` and `x_max`.

We'll then use the `lag` function to offset this data to give the lower and upper
bounds their own columns.

```
select
    generate_series(x_min::numeric, x_max::numeric, ((x_max - x_min) / nbins)::numeric) as bin
into temp bins
from
  bin_params;

select
    lag(bin) over (order by bin) as low_bin
    , b.bin as high_bin
into temp bin_range
from
    bins b;
```
<table border="1">
  <tr>
    <th align="center">low_bin</th>
    <th align="center">high_bin</th>
  </tr>
  <tr valign="top">
    <td align="right">&nbsp; </td>
    <td align="right">-3.92909625498578</td>
  </tr>
  <tr valign="top">
    <td align="right">-3.92909625498578</td>
    <td align="right">-3.673483410809675</td>
  </tr>
  <tr valign="top">
    <td align="right">-3.673483410809675</td>
    <td align="right">-3.417870566633570</td>
  </tr>
  <tr valign="top">
    <td align="right">-3.417870566633570</td>
    <td align="right">-3.162257722457465</td>
  </tr>
  <tr valign="top">
    <td align="right">...</td>
    <td align="right">...</td>
  </tr>
</table>

### Binning and Counting
Conceptually this part is quite simple: we need to create a frequency table
that gives us the # of points that occurr in each bin. However in SQL terms
this is a little tricky. We're going to be doing something that will make your
Data Warehouse Engineer cringe...we'll be doing a cartesian join!

We'll start by joining our `bins` table to our `diamonds` table. We're going
to be doing an inner join because each bin should at least have *SOME* data in
it (we calculated them with the same data after all). Now the next step is very
important. We're going to join 2 tables where the value of `price` is within the
range of one of the bins. It looks like this:

```
select
    b.low_bin
    , b.high_bin
    , r.x
from
    bin_range b
left join
    rnorm r
        on r.x <= b.high_bin and
        r.x > b.low_bin
where
    b.low_bin is not null and
    b.high_bin is not null
order by
    b.low_bin;
```
<table border="1">
  <tr>
    <th align="center">low_bin</th>
    <th align="center">high_bin</th>
    <th align="center">x</th>
  </tr>
  <tr valign="top">
    <td align="right">-3.60916920471936</td>
    <td align="right">-3.375286320204154</td>
    <td align="right">&nbsp; </td>
  </tr>
  <tr valign="top">
    <td align="right">-3.375286320204154</td>
    <td align="right">-3.141403435688948</td>
    <td align="right">-3.14852622197941</td>
  </tr>
  <tr valign="top">
    <td align="right">-3.375286320204154</td>
    <td align="right">-3.141403435688948</td>
    <td align="right">-3.29226258210838</td>
  </tr>
  <tr valign="top">
    <td align="right">-3.375286320204154</td>
    <td align="right">-3.141403435688948</td>
    <td align="right">-3.27717558760196</td>
  </tr>
  <tr valign="top">
    <td align="right">-3.141403435688948</td>
    <td align="right">-2.907520551173742</td>
    <td align="right">-3.06108768424019</td>
  </tr>
  <tr valign="top">
    <td align="right">-3.141403435688948</td>
    <td align="right">-2.907520551173742</td>
    <td align="right">-2.96137290727347</td>
  </tr>
  <tr valign="top">
    <td align="right">-3.141403435688948</td>
    <td align="right">-2.907520551173742</td>
    <td align="right">-2.96593475434929</td>
  </tr>
  <tr valign="top">
    <td align="right">...</td>
    <td align="right">...</td>
    <td align="right">...</td>
  </tr>
</table>

This gets us almost all the way there. Last thing we need to do is aggregate the
bins and count the # of times each set of bins has a particular value.

```
select
    b.low_bin
    , b.high_bin
    , count(*) as freq
from
    bin_range b
left join
    rnorm r
        on r.x <= b.high_bin and
        r.x > b.low_bin
where
    b.low_bin is not null and
    b.high_bin is not null
group by
    b.low_bin
    , b.high_bin
order by
    b.low_bin;
```
<table border="1">
  <tr>
    <th align="center">low_bin</th>
    <th align="center">high_bin</th>
    <th align="center">freq</th>
  </tr>
  <tr valign="top">
    <td align="right">...</td>
    <td align="right">...</td>
    <td align="right">...</td>
  </tr>
  <tr valign="top">
    <td align="right">-2.547947306347932</td>
    <td align="right">-2.282606475399404</td>
    <td align="right">25</td>
  </tr>
  <tr valign="top">
    <td align="right">-2.282606475399404</td>
    <td align="right">-2.017265644450876</td>
    <td align="right">28</td>
  </tr>
  <tr valign="top">
    <td align="right">-2.017265644450876</td>
    <td align="right">-1.751924813502348</td>
    <td align="right">41</td>
  </tr>
  <tr valign="top">
    <td align="right">-1.751924813502348</td>
    <td align="right">-1.486583982553820</td>
    <td align="right">35</td>
  </tr>
  <tr valign="top">
    <td align="right">-1.486583982553820</td>
    <td align="right">-1.221243151605292</td>
    <td align="right">50</td>
  </tr>
  <tr valign="top">
    <td align="right">-1.221243151605292</td>
    <td align="right">-0.955902320656764</td>
    <td align="right">59</td>
  </tr>
  <tr valign="top">
    <td align="right">-0.955902320656764</td>
    <td align="right">-0.690561489708236</td>
    <td align="right">43</td>
  </tr>
  <tr valign="top">
    <td align="right">-0.690561489708236</td>
    <td align="right">-0.425220658759708</td>
    <td align="right">43</td>
  </tr>
  <tr valign="top">
    <td align="right">-0.425220658759708</td>
    <td align="right">-0.159879827811180</td>
    <td align="right">40</td>
  </tr>
  <tr valign="top">
    <td align="right">...</td>
    <td align="right">...</td>
    <td align="right">...</td>
  </tr>
</table>


### Our Psuedo-chart
Alright we're almost there! Time for the fun part. We're going to make a pseudo-visualization right here from our SQL table. Instead of displaying the frequency for each bin, we're going to make a bar chart. To do that we'll be using the `repeat(s char, n int)` function from Postgres. This function will repeat a string (`s`), `n` number of times.

So in our case we'll be repeat the `'|'` character the by the count for each bin. To scale our chart, we'll actually apply normalize so that the largest frequency has 50 `'|'` marks and scale accordingly for the other buckets.

```
select
  f.low_bin
  , f.high_bin
  , repeat('|', ceil(50 * f.freq / (select max(freq)::numeric from frequency_table))::int)
from
  frequency_table f
;
low_bin       |      high_bin      |                       count
--------------------+--------------------+----------------------------------------------------
-3.70145826274529 | -3.474553561931394 | ||||
-3.474553561931394 | -3.247648861117498 | |
-3.247648861117498 | -3.020744160303602 | |||||
-3.020744160303602 | -2.793839459489706 | ||||||||
-2.793839459489706 | -2.566934758675810 | |||||
-2.566934758675810 | -2.340030057861914 | |||||||||
-2.340030057861914 | -2.113125357048018 | ||||||||||||||||||||||||
-2.113125357048018 | -1.886220656234122 | ||||||||||||||||||||||||||||||
-1.886220656234122 | -1.659315955420226 | ||||||||||||||||||||||||||||||||||||
-1.659315955420226 | -1.432411254606330 | ||||||||||||||||||||||||||||||||||||
-1.432411254606330 | -1.205506553792434 | |||||||||||||||||||||||||||||||||||||||||
-1.205506553792434 | -0.978601852978538 | ||||||||||||||||||||||||||||||||||||||||||||||||||
-0.978601852978538 | -0.751697152164642 | ||||||||||||||||||||||||||||||||||||||||||
-0.751697152164642 | -0.524792451350746 | ||||||||||||||||||||||||||||||||||||||||||||||||||
-0.524792451350746 | -0.297887750536850 | ||||||||||||||||||||||||||||||||||||
-0.297887750536850 | -0.070983049722954 | ||||||||||||||||||||||||||||
-0.070983049722954 |  0.155921651090942 | |||||||||||||||||
0.155921651090942 |  0.382826351904838 | ||||||||||||
0.382826351904838 |  0.609731052718734 | ||||||||||||||||
0.609731052718734 |  0.836635753532630 | ||||||||||
0.836635753532630 |  1.063540454346526 | ||||||||||||
1.063540454346526 |  1.290445155160422 | ||||||
1.290445155160422 |  1.517349855974318 | ||
```

There you have it! You can simplify this script a lot by removing some of the more detailed bits regarding the bin selecting and scaling. You can find [simplified versions in this gist]().

![](//img/ascii-histogram.png)

### Before You Go
Some other helpful resources for histograms and all things SQL:

- [Khan Academy: Creating a Histogram](https://www.khanacademy.org/math/probability/data-distributions-a1/displays-of-distributions/v/histograms-intro)
- [Histograms in R](https://www.datacamp.com/community/tutorials/make-histogram-basic-r)
- [Density Plots in R](http://www.statmethods.net/graphs/density.html)
- [Histograms in Excel](https://support.office.com/en-us/article/Create-a-histogram-b6814e9e-5860-4113-ba51-e3a1b9ee1bbe)

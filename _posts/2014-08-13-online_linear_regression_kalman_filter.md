---
layout: post
title: Online Linear Regression using a Kalman Filter
---

[Linear regression](https://en.wikipedia.org/wiki/Linear_regression) is useful for many financial applications such as finding the hedge ratio between two assests in a pair trade. In a perfect world, the realtionship between assests would remain constant along with the slope and intercet of a linear regression. Unfortutanely this is usually the exception rather than the rule. In this post, I'm going to show you how to use a [Kalman filter](https://en.wikipedia.org/wiki/Kalman_filter) for online linear regression that calculates the time-varying slope and intercept.{{ more }} The Python module, [pykalman](http://pykalman.github.io/), is used to easily construct a Kalman filter. The complete iPython notebook used to do the analysis below is available [here](http://nbviewer.ipython.org/github/aidoom/aidoom.github.io/blob/master/notebooks/2014-08-13-online_linear_regression_kalman_filter.ipynb).

For this example, I'm going to use two related ETF's, the iShares MSCI Australia (EWA) and iShares MSCI Canada (EWC). We can use the `DataReader` function from [pandas](http://pandas.pydata.org/) to download the daily adjusted closing prices for the EWA and EWC ETF's from Yahoo.
{% highlight python %}
from pandas.io.data import DataReader
secs = ['EWA', 'EWC']
data = DataReader(secs, 'yahoo', '2010-1-1', '2014-8-1')['Adj Close']
{% endhighlight %}

The correlation between the two assests adjusted closing prices can be visualized using a scatter plot with each point colored by date. Clearly, the relationship between the ETF's changes between 2010 and 2014 and can't be described accurately by a simple linear regression with constant slope and intercept.

![price_corr](/img/2014-08-13-online_linear_regression_kalman_filter/price_corr.png)

Before we get started on the Kalman filter, recall the equation for a linear regression

$$
\begin{gather*}
a_{k} = \beta b_{k} + \alpha
\end{gather*}
$$

where $$ a_{k} $$ and $$ b_{k} $$ is the adjusted closing price of EWC and EWA respectively and $$ \beta $$ and $$ \alpha $$ is the slope and intercept. Rewriting in in vector form gives

$$
\begin{gather*}
a_{k} = \boldsymbol{\beta} \bf{b}_{k} \\
\boldsymbol{\beta} = \begin{bmatrix} \beta \ \alpha \end{bmatrix} \\
{\bf{b}}_{k} = \begin{bmatrix} b_{k} \\ 1 \end{bmatrix}
\end{gather*}
$$

The Kalman filter is a linear state-space model that operates recursively on streams of noisy input data to produce a statistically optimal estimate of the underlying system state. The general form of the Kalman filter state-space model consits of a transition and observation equation

$$
\begin{gather*}
  {\bf x}_{k+1} = {\bf A}_{k} {\bf x}_{k} + {\bf w}_{k} \\
  {\bf z}_{k} = {\bf H}_{k} {\bf x}_{k} + {\bf v}_{k}
\end{gather*}
$$

where $$ {\bf x}_{k} $$ and $$ {\bf z}_{k} $$ are the hidden state and observation vectors at time $$ k $$. $$ {\bf A}_{k} $$ and $$ {\bf H}_{k} $$ are the trasition and observation matrices. $$ {\bf w}_{k} $$ and $$ {\bf v}_{k} $$ are Guassian noise with zero mean.

For our application, we assume that the hidden state variable, $$ {\bf x}_{k} $$, is the slope and intercept of the linear regression denoted by the vector $$ \boldsymbol{\beta} $$ above. We also assume the slope and intercept follow a random walk by setting $$ {\bf A}_{k} $$ equal to the identity matrix. Our transition equation now looks like

$$ {\boldsymbol \beta}_{k+1} = {\bf I} {\boldsymbol \beta}_{k} + {\bf w}_{k} $$

This simply says that $$ \boldsymbol{\beta} $$ for the next timestep is the current $$ \boldsymbol{\beta} $$ plus some noise.

The next step is to fit our model to the observation equation of the Kalman filter. To do this, we make the EWC adjusted closing prices the observations, $$ {\bf z}_{k} $$, and the observation martix, $$ {\bf H}_{k} $$, is a 1x2 vector consisting of the EWA adjusted closing price in the first column and ones in the second column as in the $$ {\bf{b}}_{k} $$ vector above. This is simply a linear regression between the two assests. For [pykalman](http://pykalman.github.io/), the observation matrix `obs_mat` is constructed using

{% highlight python %}
obs_mat = np.vstack([data.EWA, np.ones(data.EWA.shape)]).T[:, np.newaxis]
{% endhighlight %}

and looks like

{% highlight python %}
array([[[ 19.36,   1.  ]],
       [[ 19.42,   1.  ]],
       [[ 19.49,   1.  ]],
       ..., 
       [[ 26.02,   1.  ]],
       [[ 26.24,   1.  ]],
       [[ 26.42,   1.  ]]])
{% endhighlight %}

The last thing we need to specify is the noise terms $$ {\bf{w}}_{k} $$ and $$ {\bf{v}}_{k} $$. We set the observation covariance, $$ {\bf{v}}_{k} $$, to unity. We treat the transition covariance, $$ {\bf{w}}_{k} $$, as a parameter that can be adjusted to control how quickly the slope and intercept change.
{% highlight python %}
delta = 1e-5
trans_cov = delta / (1 - delta) * np.eye(2)
{% endhighlight %}

Now, we can instantiate the `KalmanFilter` class from the pykalman module
{% highlight python %}
kf = KalmanFilter(n_dim_obs=1, n_dim_state=2,
                  initial_state_mean=np.zeros(2),
                  initial_state_covariance=np.ones((2, 2)),
                  transition_matrices=np.eye(2),
                  observation_matrices=obs_mat,
                  observation_covariance=1.0,
                  transition_covariance=trans_cov)
{% endhighlight %}
and calculate the filtered state means and covariances
{% highlight python %}
state_means, state_covs = kf.filter(data.EWC.values)
{% endhighlight %}

Finally, we can plot the slope and intercept to see how they change over time
![slope_intercept](/img/2014-08-13-online_linear_regression_kalman_filter/slope_intercept.png)

A more interesting way to visualize this is to overlay every fifth regression line on the EWA vs EWC scatter plot so we can clearly see the how the regression line adjusts over time
![price_corr_regress](/img/2014-08-13-online_linear_regression_kalman_filter/price_corr_regress.png)

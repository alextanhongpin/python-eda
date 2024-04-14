# A/B Testing

In this notebook, we will explore the critical steps in an A/B test. From the scenario presented [^1], the goal is to _improve user retention_.

The current game difficulty setting is _hard_ (control), and the assumption is lowering the difficulty to _medium_ (treatment) will improve user retention from 70% to 75%.

| Type | Notes | Retention (%) |
| - | - | - |
| control | difficulty setting _hard_ | 70% |
| treatment | difficulty setting _medium_ | 75% (desired) |


There are other calculators available, such as [^2] or [^3]

References:

[^1]: https://datastud.dev/posts/ab-testing
[^2]: https://stats.stackexchange.com/questions/605466/how-to-get-python-statsmodels-to-match-evan-millers-famous-ab-test-sample-size
[^3]: https://abtestguide.com/abtestsize/
- https://blog.marvik.ai/2023/07/31/making-the-right-changes-using-a-b-testing-to-optimize-your-product/

## Minimum Detectable Effect

In order for us to calculate the _sample size_ that we need to collect for the experiment, we first need to calculate the _minimum detectable effect_. The increase from 70% to 75% gives us the information required to calculate the MDE.


```python
import statsmodels.api as sm

init_prop = 0.7
mde_prop = 0.75
effect_size = sm.stats.proportion_effectsize(init_prop, mde_prop)
print(
    "From a change from {:.2f} to {:.2f}, the effect size is {:.3f}".format(
        init_prop, mde_prop, effect_size
    )
)
```

    From a change from 0.70 to 0.75, the effect size is -0.112


## Sample Size

Once we have our MDE, we can calculate the sample size required.

We set `nobs1` (number of observations of sample 1) parameter to `None`, which signifies this is the value we are looking to solve.

A **significance level of 0.05** and **power of 0.8** are commonly chosen default values, but can be adjusted based on the scenario and the desired false positive vs false negative sensitivity (?)

NOTE:

`tt_ind_solve_power` and `zt_ind_solve_power` are shortcut functions in the statsmodels power module that can solve for any parameter in the power equations. The main difference between the two is how they treat the internal energy change. 
- tt_ind_solve_power: Calculates the sample size for a two independent sample t-test. It requires the following parameters:
    - effect_size: The standardized effect size, which is the difference between the two means divided by the standard deviation. This value must be positive.
- zt_ind_solve_power: Calculates the desired sample size for a z-test scenario. It requires the following parameters:
    - Minimum size of the effect



```python
from math import ceil

# sm.stats.tt_ind_solve_power
sample_size = sm.stats.zt_ind_solve_power(
    effect_size=effect_size, nobs1=None, alpha=0.05, power=0.8
)
print(
    "{} sample size required given power analysis and input parameters".format(
        ceil(sample_size)
    )
)
```

    1250 sample size required given power analysis and input parameters



```python
import statsmodels.stats.api as sms

# Alternative
required_n = sms.NormalIndPower().solve_power(effect_size, power=0.8, alpha=0.05)
required_n
```




    1249.5838647379314



## Experiment Data Overview

We need to collect a random sample of 1,250 players that played the _medium_ level and the same amount of players that played the _hard_ level.

For simulation, we assumed 980 users (of 1,250) continued playing after reaching the _medium_ difficulty level while 881 (of 1,250) continued playing after reaching the _hard_ difficulty level.

# Performing the z-test


```python
# Get the z test requirements.
medium_successes = 980  # treatment, (roughly 78%, 980/1250)
hard_successes = 881  # control, (roughly 70%, 881/1250)
trials = 1250
```


```python
from statsmodels.stats.proportion import proportions_ztest

z_stat, p_value = proportions_ztest(
    count=[hard_successes, medium_successes], nobs=trials  # [control, treatment]
)
print("z-stat of {:.2f} and p-value of {:.2f}".format(z_stat, p_value))
```

    z-stat of -4.54 and p-value of 0.00


A **p-value below 0.05** meets our criteria. Reducing the level difficulty to medium is the way to go.


```python
from statsmodels.stats.proportion import proportion_confint

(lower_con, lower_treat), (upper_con, upper_treat) = proportion_confint(
    count=[hard_successes, medium_successes],
    nobs=trials,
)
print(
    "confident interval for control group: [{:.3f}, {:.3f}] ({} - {})".format(
        lower_con, upper_con, ceil(lower_con * trials), ceil(upper_con * trials)
    )
)
print(
    "confident interval for treatment group: [{:.3f}, {:.3f}] ({} - {})".format(
        lower_treat, upper_treat, ceil(lower_treat * trials), ceil(upper_treat * trials)
    )
)
```

    confident interval for control group: [0.680, 0.730] (850 - 913)
    confident interval for treatment group: [0.761, 0.807] (952 - 1009)



```python
z_stat, p_value = proportions_ztest(count=[881, 923], nobs=1250)

print("z-stat of {:.2f} and p-value of {:.2f}".format(z_stat, p_value))
```

    z-stat of -1.87 and p-value of 0.06



```python
75 / 100 * 1250
```




    937.5




```python
from statsmodels.stats.weightstats import ztest

control = [1 if i < 881 else 0 for i in range(1250)]
treatment = [1 if i < 980 else 0 for i in range(1250)]
tstat, pvalue = ztest(control, treatment)
print("t-stat: {:.2f}, p-value: {:.2f}".format(tstat, pvalue))
```

    t-stat: -4.56, p-value: 0.00


p-value is less than 0.05, so we can reject the null hypothesis.


```python
control = [1 if i < 881 else 0 for i in range(1250)]
treatment = [1 if i < 923 else 0 for i in range(1250)]
tstat, pvalue = ztest(control, treatment)
print("t-stat: {:.2f}, p-value: {:.2f}".format(tstat, pvalue))
```

    t-stat: -1.87, p-value: 0.06


p-value is more than 0.05, so we cannot reject the null hypothesis.


```python

```




    985.0710001811176




```python

```

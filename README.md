# Formation_2020_RBouet_Stat_GLMM


There are situations under which the analysis of variance and maximum likelihood, the major tool for mixed models, produce the same result. It is therefore tempting to start with Anova and then move on. 


# Part 1 - Why Mixed Effects Modeling?

## What is ANOVA? When can I use it?

For experimentalists, the go-to statistical tool is the ANOVA. The way we learn ANOVA often feels like cooking or DIY: take means, sum squares in a specific way, split them across cells of a table, introduce some degrees-of-freedom, and if you have followed the recipe closely, get the magical p-values.

<center>

![Typical ANOVA table](../Pictures/ANOVA_table.jpeg)

</center> </br>

With time, we forget that the **AN**alysis **O**f **VA**riance is first and foremost based on a linear model: 

$$Y = \beta_0 + \beta_1 X + \epsilon$$

where $Y$ is a *dependent variable*, $\beta_1$ is the *slope* (aka *coefficient*) for the *independent variable* (aka *regressor* or *factor*) $X$, $\beta_0$ is the *intercept* (i.e. the value $Y$ when $X=0$). The equation simply formalizes how $Y$ changes as a function of $X$, with $X$ representing experimental conditions, groups of subjects, covariables, etc.

<!-- The only difference is that its reference point is the mean of the dataset. When we described the equations above we said that to interpret the results of the linear model we would look at the slope term; this indicates the rate of changes in $Y$ if we change one variable and keep the rest constant. The ANOVA calculates the effects of each treatment based on the grand mean, which is the mean of the variable of interest.  -->


Classical ANOVA uses a least squares method to solve the equation and find approximate values of the $\beta$ ; it uses $F$-tests for significance. Because of that, it requires a number of assumptions that are rarely all fulfilled:

- **Balanced design** <br>
  = same number of observations in each level of a factor. For example, same number of subjects in all groups of a between-group design.
  This is rarely exactly true and sometimes far from it (e.g. a group of hard-to-find patients vs. a large group of controls) <br>
  
- **No missing datas** <br>
  If data is missing in one of the cells (e.g. corrupted data for one condition in one subject), degrees-of-freedom are no longer valid.
  Often the data of the corrupted subject needs to be dropped entirely. <br>

- **Normal distribution** <br>
  This assumption is required by the $F$-test. It is possible to circumvent it using *generalized linear models*, which can handle various common distributions, bit it gets us far from the classical ANOVA. <br>
  
- **Homoskedasticity** <br>
  This assumptions is also required by $F$-tests. Homoscedasticity is the *equality of variance* across values of the regressors (e.g. groups, levels of a factor, etc.). Homoscedasticity describes a situation in which the error term (that is, the ???noise??? or random disturbance in the relationship between the independent variables and the dependent variable) are drawn from the same normal distribution across all values of the independent variables.

```{r, warning=FALSE, fig.align='center', echo=FALSE}
n = 20
df <- data.frame(group = c(rep("A",n),rep("B",n),rep("A",n),rep("B",n)),
                 skedasticity = c(rep("homoskedastic",2*n), rep("heteroskedastic",2*n)),
                 y = c(rnorm(n,5,2),rnorm(n,1,2), rnorm(n,5,2),rnorm(n,1,7))) 
                 
ggplot(df, aes(x = group, color = group, y = y)) +
  facet_grid(. ~ skedasticity) +
  geom_point(position = position_jitter(.1))
```

  The violation of homoskedasticity often has a dramatic effect on Type I error rate while decreasing power. <br>
  
- **Sphericity** <br>
  This is the equivalent of homoskedasticity for repeated measures Anova (for an intuition of this, see https://stats.stackexchange.com/a/213433). Its violation has the same consequences.
  Sphericity states that variances of each of the pairwise contrasts should be the same.
  Example: consider the data below, obtained from a within-subject design. 
  
```{r, warning=FALSE, fig.align='center', echo=FALSE}
df = data.frame(id = seq(1,5), A = c(30,35,25,15,9), B = c(27,30,30,15,12), C = c(20,28,20,12,7)) %>%
     gather(key = treatment, value = outcome, A,B,C)

ggplot(df, aes(x = treatment, y = outcome, group = id, color = as.factor(id))) +
  geom_point() + geom_line() + labs(color = "Patient")
```

We can see that the difference between treatments B and C varies across patients, but it goes in the same direction ; similarly for the difference between treatments A and C. However, the difference between treatments A and B varies more across subjects, it even goes in opposite directions. Actually, its variance is higher than the two other contrasts.

```{r, warning=FALSE, fig.align='center', echo=FALSE}
df = data.frame(id = seq(1,5), A_B = c(30-27,35-30,25-30,15-15,9-12), B_C = c(27-20,30-28,30-20,15-12,12-7), A_C = c(30-20,35-28,25-20,15-12,9-7)) %>%
     gather(key = contrasts, value = value, A_B, B_C, A_C) %>%
     group_by(contrasts) %>% mutate(var = var(value))

ggplot(df, aes(x = contrasts, y = value, group = id, color = as.factor(id))) +
  geom_point() + geom_text(aes(label = var), y=12, color="black") + expand_limits(y=12) +
  labs(color = "Patient")
```

  The sphericity assumption is rarely met in real data, hence the development of corrections procedures (Greenhouse-Geisser, Huynd-Feldt...).
  

<!-- But a more direct way to think about sphericity is to say that it requires that (for exemple) all subjects in each group change in the same way over trials. In other words the slopes of the lines regressing the dependent variable are the same for all subjects. Put that way it is easy to see that sphericity can really be an unrealistic assumption. -->
<!-- => J'ai un doute sur ce paragraphe. De ce que j'ai compris la sph??ricit?? n'exige pas que les slopes de tous les sujets soient les m??mes, seulement que les variances des diff??rents contrasts soient les m??mes. Du coup je ne suis pas s??r que l'on puisse parler de sph??ricit?? pour des r??gresseurs continus. -->



## ANOVA vs. linear regression

Most people either use ANOVA for categorical predictor variables, or regression for continuous predictor variables. But we have seen that ANOVA is based on a linear regression, so they are actually the same thing! <br>

<!-- ANCOVA: continuous and categorical predictors. <br> -->
<!-- Regression: categorical and continuous predictors. -->

<!--  Why use regression as opposed to ANOVA ? <br> -->
<!-- 	No temptation to dichotomize continuous predictors. <br> -->
<!-- 	Intuitive interpretation. -->
<!-- => j'ai pas compris ce que tu voulais dire l?? -->
	
When you use Anova, you implicitly describe your data with linear model. <br>
Let's check this with real data:

Anova:
```{r, warning=FALSE, fig.align='center', echo=TRUE}
load(file = file.path(getwd(), 'Datas/Raw_Datas.RData')) 

anova.RT <- aov(RT ~ Time, data = Raw.Datas)
anova.RT$coefficients
summary(anova.RT)
```

Linear Model:
```{r, warning=FALSE, fig.align='center', echo=TRUE}
lm.RT <- lm(RT ~ Time, data = Raw.Datas)
lm.RT$coefficients
summary(lm.RT)
```

Coefficients, degrees-of-freedom, F-statistics and p-values are all exactly the same!




## Beyond the simple linear model

Let's imagine that you're trying to estimate RT along the Time. 
A simple linear model could be used to estimate this relationship:
$$RT = \beta_0 + \beta_1 * Time + \epsilon$$ <br>

```{r, warning=FALSE, fig.align='center', echo=FALSE}

Raw.Datas %>%
  ggplot(aes(x = Time, y = RT)) +
  geom_point() +
  geom_smooth(method = "lm", color = "black", se = FALSE) + 
  xlab("Time (day)") + scale_x_continuous(breaks=seq(0,10)) +
  ylab("RT (ms)")
```

```{r, warning=FALSE, fig.align='center'}
coef.lm <- round(coef(lm(RT ~ Time, data = Raw.Datas))[["Time"]])
```

We find a RT increase of `r coef.lm`ms/Day .

But, there are different Subjects. And if we color in terms of Subject :

```{r, warning=FALSE, fig.align='center', echo=FALSE}
Raw.Datas %>%
  ggplot(aes(x = Time, y = RT)) +
  geom_point(aes(color = Subject)) +
  geom_smooth(method = "lm", color = "black", se = FALSE) + 
  xlab("Time (day)") +
  ylab("RT (ms)")
```
It is clear that there are great variations in RT across Subjects. Kevin for example is faster than Sam. 

```{r, warning=FALSE, fig.align='center', echo=FALSE}
Raw.Datas %>%
  ggplot(aes(x = Subject, y = RT, color = Subject, fill = Subject)) +
  geom_violin(alpha = 0.4, position = position_dodge(width = 0.9)) +
  geom_boxplot(width=.2, alpha = 0.5, outlier.alpha = 0, position = position_dodge(width = 0.9)) +
  stat_summary(fun.y = "mean", geom = "point", size = 2, position = position_dodge(width=0.9), color = "white") +
  theme(axis.text.x = element_text(angle = 45, hjust = 1)) + 
  ylab("RT (ms)")
```

It may be the case that each Subject has a different starting RT, while the day rate at which RT increase is consistent across Subjects. If we believe this to be the case, we would want to allow the **intercept** to vary by Subject:

$$RT = \beta_{0,i} + \beta_1 * Time + \epsilon$$ <br>

where $i$ indexes the Subjects.


```{r, warning=FALSE, fig.align='center', echo=FALSE}
df.coeffs <- as.data.frame(coef(lmer(RT ~ Time + (1|Subject), data = Raw.Datas))$Subject) %>%
             rownames_to_column("Subject") 

Raw.Datas %>%
  ggplot(aes(x = Time, y = RT, color = Subject)) +
  geom_point() +
  geom_abline(data = df.coeffs, aes(slope = Time, intercept = `(Intercept)`, color = Subject)) +
  # geom_line(aes(x = Time, y = random.intercpet.preds, colour = Subject)) + 
  xlab("Time (day)") +
  ylab("RT (ms)")
```

This strategy allows us to capture variation in initial RT of our subjects. The slope over time is still estimated globally, i.e. it is assumed to be the same for all subjects. Because some subjects increase while others decrease, it appears flat on the whole.

Let's try to incorporate additional information into our model. It's reasonable to imagine that the most realistic situation is a different start RT **and** increase (or decrease) at different rates depending on the Subject:

$$RT = \beta_{0,i} + \beta_{1,i} * Time + \epsilon$$ <br>

where $i$ indexes the Subjects.



```{r, warning=FALSE, fig.align='center', echo=FALSE}

Raw.Datas %>%
  ggplot(aes(x = Time, y = RT, color = Subject)) +
  geom_point() +
  geom_smooth(method = "lm", se = FALSE) + 
  xlab("Time (day)") +
  ylab("RT (ms)")

coef(lmer(RT ~ Time + (Time||Subject), data = Raw.Datas ))$Subject

```

Can I use ANOVA ? <br>

A balanced design ? <p style="color:green">OK</p>    
No missing datas ? <p style="color:red">KO</p>    
Normally distribution ? <p style="color:red">KO</p>    
Homoskedasticity ? <p style="color:red">KO</p>    
sphericity ?   <p style="color:red">KO</p>    





# Part 2 - Mixed Models

## The spirit
In Part 1 we have seen that to take into account variability across subjects and non-independence of data within each subject, we need to model intercepts and slopes for each subject.

Linear mixed models (LMMs) do exactly that. In addition, LMMs take into account that our subjects are a random sample drawn from a more general population. Therefore, LMMs model subjects' intercepts and slopes as drawn from a normal distribution centered on the (estimated) population average:

$$\beta \sim \mathcal{N}(\beta_{mean},g)$$
We can rewrite this by splitting it into a *fixed* term (a constant) and a *random* term:

$$\beta = \beta_{mean} + u$$
$$u \sim \mathcal{N}(0,g)$$

LMMs are said to be mixed because they contain both fixed and random terms (aka *effects*).



## Specifying the model

### Fixed and random effects

The first thing to do when we define a LMM is to decide what are the fixed and random effects:

- **fixed**: any variable whose impact on the dependent variable you care about. This includes experimental conditions, groups, covariates such as gender, age, performance, etc. + interactions of interest. In general, levels of fixed effects are all known. For instance, *Gender* has male and female and that's it.
<!-- is always a fixed variable because we if we are comparing genders we only have two genders to work with. -->

- **random**: any variable that may introduce non-independence in the data, but whose levels are drawn randomly from a wider population. In our field it is tyically *Subject* (or *Animal*).  
In education research, if a test is carried out in multiple classes or schools, then *Class* and *School* are also random effects (in addition to *Student*).  
<!-- For instance, we might randomly choose 2 schools from all possible schools in France.  -->
In emotions or linguistic research, it is common to have a number of stimuli (e.g. 10 happy faces vs. 10 sad faces, or 20 words vs. 20 non-words) that do not exhaust the entire set of possible stimuli. Therefore, *Stimulus* (or *Item*) should also be included as a random factor.


Note that in some cases, an effect can be entered **both as a fixed and a random effect**. In education research for example, we might be interested in both generalizing over all schools (random effect) *and* in the differences between the schools that participated to the study (fixed effect).

<!-- Our primary interest is not in the differences between school-A and school-B, for example, but in the variance of school performance.  -->

<!-- As we saw, **sphericity** hypothesis means that the pattern of covariances or correlations is constant across trials. Mixed model, allow you to specify some other pattern for those covariances. -->



## Formula

Go to [this page](http://conjugateprior.org/2013/01/formulae-in-r-anova/ "Formulae in R: ANOVA and other models, mixed and fixed") for a complete description of linear model syntax in R.

### Fixed Effects

<center>

![Formula for fixed effects](../Pictures/Formula_Fixed.png)

</center> </br>



### Random Effects

<center>

![Formula for random effects](../Pictures/Formula_Random.png) 

</center> </br>


#### Intercept

Let's go back to our sample data. RT are continuous and non-negative, so we will use the Gamma family.

The variation between Subject is big, we account for them in our model!

```{r, warning=FALSE, fig.align='center', echo=FALSE}
Raw.Datas %>%
  ggplot(aes(x = Subject, y = RT, color = Subject, fill = Subject)) +
  geom_boxplot(width=.2, alpha = 0.5, outlier.alpha = 0, position = position_dodge(width = 0.9)) +
  theme(axis.text.x = element_text(angle = 45, hjust = 1)) + 
  ylab("RT (ms)")
```

The levels of subject variations are infinite, we have here only 5 levels, so we must take into account to generalize our analysis.

Including `(1|Subject)` in the model means "*assume an intercept that???s different for each subject"*. `1` stands for the intercept here.

Like this, you specifically say that there are several RT per subject (repeated measurement). And these responses will depend on each subject???s baseline level. This effectively resolves the non-independence that stems from having multiple responses by the same subject.

```{r, warning=FALSE, fig.align='center', message=FALSE}
model.01 <- lmer(RT ~ Time + (1|Subject), data = Raw.Datas)
summary(model.01)
```

And we can ge the values of intercept by Subject:
```{r, warning=FALSE, fig.align='center', message=FALSE}
ranef(model.01)
```

We???re not done yet. One of the coolest things about mixed models is coming up now, so hang on!

#### Slopes

Let???s have a look at the coefficients of the model by subject.

The `coef()` function in lme4 returns the posterior modes of the $\beta_i$. That is for a given $i$, knowledge of the overall distribution of $\beta_i$.

```{r, warning=FALSE, fig.align='center', message=FALSE}
coef(model.01)$Subject["Time"]
```
You see that each subject is assigned a different intercept
<!-- (Remember, values provide from a Gamma model. exp(x) transformation to go back RT). -->
That???s what we would expect, given that we've told the model with `(1|subject)` to take by-subject variability into account. <br>
In this model, we account for baseline-differences in RT, but we assume that whatever the effect of Time is, it???s going to be the same for all subjects (i.e. `coef(model.01)$Subject[1,'Time']`).

We saw previously there is a subject-specific Time effect.

```{r, warning=FALSE, fig.align='center', message=FALSE, echo=FALSE}
Raw.Datas %>%
  ggplot(aes(x = Time, y = RT, color = Subject)) +
  geom_point() +
  geom_smooth(method = "lm", se = FALSE) + 
  xlab("Time (day)") +
  ylab("RT (ms)")
```

Including `(Time|Subject)` in the model means "*assume an intercept that???s different for each subject*" AND "*assume a slope (Time effect) different for each subject*".

```{r, warning=FALSE, fig.align='center', message=FALSE}
model.02 <- lmer(RT ~ Time + (Time|Subject), data = Raw.Datas)
summary(model.02)
coef(model.02)$Subject["Time"]
```

```{r, warning=FALSE, fig.align='center', message=FALSE, echo=FALSE}
lattice::dotplot(ranef(model.02), scales = list(x =list(relation = 'free')))
```

Here is the conditional estimated mean and standard deviation (95% prediction interval) for each random effect.


As we can see, effect Time variation are smaller than Subject variation. All values overlap so we can ask the validity of random slope.

But is that a valid assumption? <br>
Now you have two models to compare with each other. One with the effect in question, one without the effect in question. We perform the likelihood ratio test using the `anova()` function:

```{r, warning=FALSE, fig.align='center', message=FALSE}
anova(model.01, model.02)
```

You find a Chi-Square value, the associated degrees of freedom and the p-value.
 
Contrary to what we saw in the plots, the random-effects term for Time is significant even in the presence of Subject term.

The p-value tests the hypothesis H0 = *no difference between the ability the two models to explain data*. Here, p <0.05, there is a difference and we can deduce that `model.02` is better than `model.01` because of its complexity and residual improvement (0.07629 -> 0.07422).
AIC, BIC and LogLikelihood decrease say "no over fitting".

So we keep `model.02`.


**Which random slopes should I specify ?**  
You can almost always expect that people differ with how they react to an experimental manipulation. And likewise, you can almost always expect that the effect of an experimental manipulation is not going to be the same for all items. Therefore, not including the corresponding slopes would amount to ignore existing non-independence, something that is known to be a major issue in linear models (see part 1 for an example of incorrect inference drawn from such negligence). In particular, mixed models without random slopes have been shown through simulations in ecology ([Schielzeth & Forstmeier, 2009](https://academic.oup.com/beheco/article/20/2/416/218997 "Conclusions beyond support: overconfident estimates in mixed models")) and psycholinguistics ([Barr et al, 2013](http://www.sciencedirect.com/science/article/pii/S0749596X12001180 "Random effects structure for confirmatory hypothesis testing: Keep it maximal")) to be anti-conservative, i.e. they have a relatively high Type I error rate.  

Unfortunately there are situations where including all the relevant random slopes will lead to model failure. See [Debugging](#debugging) to learn how to identify and address this issue.


<!-- ### Nesting and crossing -->

<!-- Another advantage : -->
<!-- In longitudinal studies of subjects in social contexts (students in classrooms) we almost always have partial crossing of the subject and the context factors, meaning that, over the course of the study, a particular student may be observed in more than one class but not all students are observed in all classes. The student and class factors are neither fully crossed nor strictly nested. -->
<!-- The same model specification can be used for data with nested or crossed or partially crossed factors.  Nesting or crossing is determined from the structure of the factors in the data, not the model specification. -->


## Diagnostics

"Essentially, all models are wrong, but some are useful" <br>
Most of the time a decent model is better than none at all. So take your model, try to improve on it, and then decide whether the accuracy is good enough to be useful for your purposes.

Plots empirical quantiles of a variable against theoretical quantiles of a comparison distribution (only for normal distributions):

```{r, warning=FALSE, fig.align='center', message=FALSE}
qqPlot(residuals(model.02))
```

<!-- Here is a strong correlation between the model???s predictions and its actual results. -->
Below, an example of residuals nicely random across fitted values:

```{r, warning=FALSE, fig.align='center', message=FALSE}
plot(model.02)
```

If you can detect a clear pattern, shape or trend, or heterogeneity along the X-axis, then your model has room for improvement. Examples:

<!-- <center> -->

<!-- ![Formula Fixed Effects.](../Pictures/Diagnos_good.png) -->

<!-- </center> </br> -->

<center>

![Formula Fixed Effects.](../Pictures/Diagnos_bad_01.png) ![Formula Fixed Effects.](../Pictures/Diagnos_bad_02.png) ![Formula Fixed Effects.](../Pictures/Diagnos_bad_03.png)

</center> </br>

You may improve on your model by:
- including additional predictors or higher polynomial orders to correct non-linearities
- transforingm some predictors (log, square root...)
- assessing the impact of an outlier
- using generalized LMMs and choose an adequate family distribution


Diagnostics of GLMMs are a bit more difficult. The `DHARMa` library is dedicated to that: https://cran.r-project.org/web/packages/DHARMa/vignettes/DHARMa.html

<!-- is used a simulation-based approach. <br> -->



```{r, warning=FALSE, fig.align='center', message=FALSE, echo=FALSE}
# simulationOutput <- simulateResiduals(fittedModel = model.02, n = 2000)
# plot(simulationOutput)
```




## Significance

If you want to publish, you will most likely need to report some kind of $p$-value.

`lmer()` provides estimates, standard errors, residuals and $t$-values, but no $p$-values because there is no universally valid method to calculate the required degrees-of-freedom for LMMs. The correct anwer depends on both the structure of data (balanced or not, full factorial or not, etc.) and the structure of the random effects. In many situations, there is not even a clear answer. For all these reasons, the main author of `lme4` has [decided](https://stat.ethz.ch/pipermail/r-help/2006-May/094765.html "Douglas Bates on lmer and p-values") not to provide any $p$-values.

However, other package authors have provided various methods. So-called **conditional F-tests with approximated degrees of freedom** are universally recommended. They are simple and have the best control of Type I error rate (i.e. false positives, see [Luke 2017](https://link.springer.com/article/10.3758/s13428-016-0809-y "Evaluating significance in linear mixed-effects models in R")). Unfortunately they do not apply to generalized models, for which we need to fall back on other methods (see [Significance/GLMMs](#glmm:significance)). 

There are 2 possible approximations of df: **Satterthwaite** and **Kenward-Roger**. Kenward-Roger is the most reliable but is exponentially time and RAM consuming with increasing number of data points and model complexity. Satterthwaite is extremely fast and is virtually as good as KR, except in extreme situations of scarce data (such as: a handful of participants, two 3-level factors and only one observation per cell, see [Schaalje et al. 2002](https://doi.org/10.1198/108571102726 "Adequacy of approximations to distributions of test statistics in complex mixed linear models")).

You will notice that the degrees of freedom are fractional: that is due to the fact that whole-plot and subplot variations are combined when standard errors are estimated. 

Here's the code in R:
```r
library(lmerTest)
options(contrasts = c("contr.sum","contr.poly")) # absolutely necessary for interpretable type III ANOVA 
anova(model, ddf = "Satterthwaite", type="3")
```

or (same results but richer output):

```r
summary(model)
```
Note that the `anova()` and `summary()` functions above are not base R, they have been overloaded by `lmerTest` for linear mixed models.




# Part 3 - Generalized mixed models

## Family

lmer is used to fit linear mixed-effect models, so it assumes that the residual error has a Gaussian distribution. But if your dependent variable is a binary outcome, then the error distribution is binomial and not Gaussian. In this case you have to use glmer.


LMMs are no more robust to such violations than any other linear model, but they can be easily extended to *generalized* linear mixed models (GLMMs) where a non-normal distribution can be specified.

You can test the distribution with several tools provide by R but all are strongly limited.
Our advise is to plot the density and folow the definition.
Examples of classic situations where GLMMs are needed:

- **reaction times**: they usually follow a [**Gamma**](https://en.wikipedia.org/wiki/Gamma_distribution "Gamma distribution on Wikipedia") or [**Inverse Gaussian**](https://en.wikipedia.org/wiki/Inverse_Gaussian_distribution "Inverse Gaussian distribution on Wikipedia") distribution. More generally, whenever your outcome is **continuous** and **non-negative**, then you should consider a Gamma distribution.  
The most common alternative is to transform the data to render it more normally distributed. Indeed, log or Box-Cox transforms work very well on reaction times. However, the transformation approach makes interpretation in terms of mental chronometry more difficult: what does it mean that 2 conditions differ by 3.5 units of log(RT)? See also [Lo & Andrews 2015](http://journal.frontiersin.org/article/10.3389/fpsyg.2015.01171/full "To transform or not to transform: using generalized linear mixed models to analyse reaction time data [Open access article]").

```{r, echo=FALSE, fig.height=3, fig.width=4}
df.gamma = data.frame(y = 150*rgamma(1000,3,1))
ggplot(df.gamma, aes(x = y)) +
geom_histogram(binwidth = 50, color="black", fill="grey50") +
labs(title="Gamma distribution", x = "Reaction time (ms)")
```


- **accuracy** obviously follows a **binomial** distribution. More generally, whenever the dependent variable takes on 2 values, you may consider the binomial distribution. The use of the binomial distribution can also be adapted to proportions, see this [post](https://stats.stackexchange.com/a/189117) by one of the authors of `lme4`.

```{r, echo=FALSE, fig.height=3, fig.width=4}
df.bern <- data.frame(y = sample(c("incorrect","correct"), 1000, replace=TRUE, prob=c(.15,.85)))
ggplot(df.bern, aes(x = as.factor(y), fill = y)) +
stat_count(color="black") + scale_fill_discrete(h.start = 180) + guides(fill=FALSE) + 
labs(title="Binomial distribution", x = "Accuracy") 
```

- **counts** of an event in a given time interval (e.g. a specific behavior in an animal), or of an object in a given area (e.g. a specific cell in a sample of tissue) commonly follow a [**Poisson distribution**](https://en.wikipedia.org/wiki/Poisson_distribution "Poisson distribution on Wikipedia").

```{r, echo=FALSE, fig.height=3, fig.width=4}
df.pois = data.frame(y = rpois(1000,3))
ggplot(df.pois, aes(x = as.factor(y))) +
stat_count(color="black", fill="grey50") +
labs(title="Poisson distribution", x = "Number of children") 
```

- some data, such as **responses to questionnaire items**, are **ordinal**. GLMMs can be fitted on ordinal data using the package `ordinal`. The approach for this kind of model is very particular and beyond the scope of this presentation.

```{r, echo=FALSE, fig.height=3, fig.width=4}
df.ord = data.frame(y = c(rep("1x/month",118), rep("1x/week",322), rep("2-3x/week", 385), rep("Every night",175))) %>%
  mutate(y = factor(y, levels = c("1x/month", "1x/week", "2-3x/week", "Every night")))
ggplot(df.ord, aes(x = as.factor(y), fill=y)) +
stat_count(color="black") + guides(fill=FALSE) + 
labs(title="Ordinal data", x = "Dreams frequency") 
```

For instance, in the previous data:

```{r, warning=FALSE, fig.align='center', echo=FALSE}
Raw.Datas %>%
  ggplot(aes(x=RT)) +
  geom_density()
```

It's easy to define a glmer, exactly the same as lmer with family argument.

```r
model.03 <- glmer(RT ~ Time + (1|Subject), 
                 data = Raw.Datas,
                 family = Gamma(link = "identity"))
summary(model.03)  
```


The link function provides the relationship between the linear predictor and the mean of the distribution function. It could be (options depend on the family):

- Identity
- Inverse
- log
- logit, probit

To better understand the link function and what???s the difference between a log link and log transforming your data ? [**Link**](https://www.r-bloggers.com/2018/10/generalized-linear-models-understanding-the-link-function/)

Residual interpretation for generalized linear mixed models (GLMMs) is often problematic. 
[**DHARMa**](https://cran.r-project.org/web/packages/DHARMa/vignettes/DHARMa.html) package aims at solving these problems by creating readily interpretable residuals for generalized linear (mixed) models that are standardized. This is achieved by a simulation-based approach.

![DHARMa Residuals plot](../Pictures/Dharma.png)


## Significance {#glmm:significance}

For GLMMs, approximations of df are possible in theory but not implemented in $R$. Thus, we need to fall back on alternatives (the following are also valid for simple LMMs but are not as good as the approximations of df):

- the **$t$-as-$z$** aka **Wald chi-square** method: the logic is that, if we have many observations, the df will be quite large and the distribution of our $t$-values will approach the $z$-distribution (= normal distribution). And because the squares of a normally distributed variable follow a chi-square distribution with 1 df (noted $\chi^2(1)$), we can do a Wald test.
This method is virtually instantaneous but the most anti-conservative: it can increase the false positive rate quite a lot, especially with small datasets (<50 subjects). Code:

```r
library(car)
Anova(model, test.statistic = "Chisq") # "Chisq" is the default value
```


- the **likelikood ratio test (LRT)** approach : the logic is to take the full model, drop one factor, and compare the two models to estimate the significance of that dropped model.
It is slightly less conservative than the Wald $\chi^2$. It has been implemented in different base $R$ and `lme4` functions. However, the implementation in `afex` is the easiest to use:

```r
library(afex)
mixed(formula, data, method="LRT")
```

Parametric bootstrap is a variant of LRT that offers a better control of Type I error rate, but is more time consuming.


```r
mixed(formula, data, method = "PB")
```





# Part 4 - Debugging models {#debugging}

## Model fitting
  
The tricky part in any statistical method is **model fitting**. Fitting a model means *estimating its parameters/coefficients given the data*.

Linear least squares (the procedure used for classic linear regressions) provides a closed-form solution to the fitting problem. This means that we can write down mathematical expressions for the model parameters. The only difficulty is to evaluate (= calculate) those expressions.

<center>

![Ordinary Least Squares (OLS)](../Pictures/OLS.png)

</center> </br>

*Linear mixed models do not have closed-form solutions*. Fitting LMMs is still an active area of research. Many approaches have been proposed and implemented in various R packages, `lme4` is only one of them (see Ben Bolker's [table of implemented methods](https://bbolker.github.io/mixedmodels-misc/glmmFAQ.html#what-methods-are-available-to-fit-estimate-glmms "What methods are available to fit (estimate) GLMMs?")). They all draw on probability theory and numerical algebra, which means it has to approximate intractable integrals and do crazy stuff on big matrices. With the complexity of the task come many opportunities for things to go a bit weird or even plain wrong.

<center>

![The shortname for all this gibberish is "lmer()"](../Pictures/lme4-maths.png)
</center> </br>

## Preventive measures {#debugging:preventive}

While there are many ways to address the warnings frequently issued by `lme4`, even professional statisticians can not (yet) provide a clear, systematic strategy. However, three things are recommended to reduce the risk of warnings:

- **collect enough data** to support the complexity of your model

- **scaling:** if predictors differ widely in their range (e.g. mouse weight in kilograms and age in months), scale them using the `scale()` function so that they all have mean of 0 and standard deviation of 1  
  
- **use the BOBYQA optimizer:** it is the default for LMMs but not for GLMMs.
```{r, eval=FALSE}
glmer(..., control = glmerControl(optimizer="bobyqa"))
```




## Convergence issues

All procedures to fit LMMs are iterative. This means that there is an algorithm, called the **optimizer**, that repeats the same operations over and over until it is "satisfied", i.e. it has **converged**.

Optimizers can be conceived as hikers who evolve in a landscape, looking for the lowest point. Their field of vision is very limited, they can only see one step ahead around them. At each iteration, they take the step that allow them to go *down*. They stop when they don't know where to go next.

<center>

![An optimizer walking down the parameter space](../Pictures/optimizer-landscape.jpg)

</center> </br>

When an optimizer stops, it checks whether it has reached destination (= converged) by evaluating the local slope, or **gradient**. It is satisfied only if the gradient is close to 0 (think of it as a flat lake or sea level). Practically, it checks that it is smaller than a certain **tolerance**. If it is not, it will throw out a warning like this:

```{r, echo=FALSE}
warning('Model failed to converge with max|grad| = 0.7741 (tol = 0.001)')
```
This warning is extremely common. The authors of `lme4` acknowledge that the tolerance they chose for the gradient is too strict. So there is a high chance of this warning being a false alarm. The following fixes can be tried sequentially until one works:

1) if `max|grad| < 0.01`, you can safely ignore it. If you're a risk-taker, you can even ignore it up to 0.1 or 1. Those numbers are arbitrary anyway.
2) if applicable: scale predictors, use BOBYQA optimizer (see [Debugging - preventive measures](#debugging:preventive))
3) try multiple optimizers. If they all agree and provide the same results, you're safe. Potentially a bit long if the model is complex. Code:  
```{r, eval=FALSE}
modelfit.all <- allFit(model)
summary(modelfit.all)
```

4) restart the fit from the last value (= force the optimizer to resume walking)
```{r, eval=FALSE}
if (isLMM(fm1)) {
  pars <- getME(fm1,"theta")
} else {
  # GLMM: requires both random and fixed parameters
  pars <- getME(fm1, c("theta","fixef")) 
}
m.restart <- update(m, start = pars)
```

5) restart the fit from a slightly perturbed value (= nudge the optimizer to help it out of the crack where it was stuck)
```{r, eval=FALSE}
# random sampling of new values within a 1% margin
pars2 <- runif(length(pars),pars/1.01,pars*1.01) 
m.restart2 <- update(m, start = pars2)
```


Sometimes the optimizer is in such a difficult position that it can not even evaluate the gradient. When that happens, it will throw out one (or several) of these messages:

```{r, echo=FALSE}
warning('Model failed to converge: degenerate Hessian with 1 negative eigenvalues')
```
```{r, echo=FALSE}
warning('In checkConv(attr(opt, "derivs"), opt$par, ctrl = control$checkConv,  ... :
  unable to evaluate scaled gradient')
```
```{r, echo=FALSE}
warning('In checkConv(attr(opt, "derivs"), opt$par, ctrl = control$checkConv,  ... :
   Hessian is numerically singular: parameters are not uniquely determined')
```

There is nothing specific you can do to address this issue (but you can try any of the things described above and [below](#simplify-models)).

Remember that convergence warnings do not mean something is wrong, only that something *might* be wrong. Always double-check your data and model, and have a look at the estimates with `summary()` to see if anything looks weird.

For more explanations on optimizers convergence issue, see:  

- `?convergence` in `lme4` manual
- this [post](https://rpubs.com/bbolker/lme4_convergence "Exploring convergence issues") and [that one](https://rpubs.com/bbolker/lme4trouble1 "lme4 convergence warnings: troubleshooting") by Ben Bolker
- this [post](https://stats.stackexchange.com/a/111083) on StackExchange


## Boundary (singular) fit

Linear least squares methods throw out a result no matter what. It does not care whether the model is appropriate to the data. In contrast, LMMs can be sensitive to whether the model makes sense at all given the data, and in particular to how complex the model is compared to the data. If data is insufficiently informative to properly fit the model, estimates may behave badly. Most often this will affect the variance-covariance matrix of random effects, which might converge to extreme values (0 for variance, 1 or -1 for covariances). This is called by `lme4` a **boundary (or singular) fit**. 

> Note: when there are more than 2 variance-covariance parameters, boundary (singular) fits do not necessarily manifest as extreme values in the random effects' parameters. 

```{r, echo=FALSE}
warning('boundary (singular) fit: see ?isSingular')
```

This warning may mean one of two things:  :
there is **no intercept or slope variance in the data**. For example, maybe the treatment effect is very homogeneous across your sample of subjects. In which case, you can drop the corresponding parameter from your model.
- there might be variance in your data, but not **enough data** to describe it accurately. This interpretation is more likely. It points to the fact that your design is somehow underpowered and that you are being too ambitious, something an ANOVA would never tell you.

<center>

![According to this inspirational card, LMM is a better friend than ANOVA.](../Pictures/true-friends.jpg)

</center> </br>

You can address the issue in 3 different ways:

1) **collect more data**
2) **simplify the model**
3) go **Bayesian**: this is beyond this introduction, sorry!


For more explanations on boundary (singular) fit, see:  

- `?isSingular` in `lme4` manual
- Ben Bolker comments on his [GLMM FAQ](https://bbolker.github.io/mixedmodels-misc/glmmFAQ.html#singular-models-random-effect-variances-estimated-as-zero-or-correlations-estimated-as---1 "Singular models: random effect variances estimated as zero, or correlations estimated as +/- 1")



## Finding the right structure of random effects {#simplify-models}

Sound theoretical considerations and [expert opinion](http://www.sciencedirect.com/science/article/pii/S0749596X12001180 "Random effects structure for confirmatory hypothesis testing: Keep it maximal (Barr et al. 2013)") tell us that we should include all within-subject experimental manipulations and their interactions as slopes in the by-subject random effects.

Experience with real life data shows that such models often fail to fit properly with our limited datasets. Fortunately, simulations indicate that in such situation a model can be simplified without increasing the Type I error rate ([Matuschek et al. 2017](http://www.sciencedirect.com/science/article/pii/S0749596X17300013 "Balancing Type I error and power in linear mixed models")).

[Bates et al. 2015](http://arxiv.org/abs/1506.04967 "Parsimonious Mixed Models") discuss extensively how to estimate model complexity and find the sweet spot. The logic is to proceed iteratively, following one of two strategies:  

1) starting **from a basic model** (e.g., just a random intercept) and enriching up to the point where it becomes too complex
2) starting **from a complex model** and pruning until you lose fitting

At each step, you can make a decision based on two diagnostic tools:

- likelihood ratio test (LRT) using `anova(model1,model2)`, where `model2` is a model that has one more random parameter than `model1`


- `summary(rePCA(model))` This function does a principal component analysis of the random effects variance-covariance matrix, along with the variance explained. If, say, 100% of the variance is explained with only 5 components out of 8, it suggests that you can safely remove 3 parameters. You just need to find which. Start with the most complex!





# Part 5 - Post-Hoc


**emmeans** offers great functionality for both plotting and contrasts. 
Compute estimated marginal means (???least-squares means??? or ???predicted marginal means???), standard errors, confidence intervals and df estimated for specified factors or factor combinations in a (generalized) (mixed) linear model.

Huge list of model classes are supported by emmeans ([to see more](https://cran.r-project.org/web/packages/emmeans/vignettes/models.html)). <br>
* anova <br>
* linear model <br>
* linear mixed models <br>
* Generalized models <br>
* bayesian models <br>
* Manova <br>
* GAM <br>
* ...


For LMM, the possible values are "kenward-roger", "satterthwaite".
The computation time required depends roughly on the number of observations (inverting N x N matrix). Also, to avoid a message `get_emm_option("pbkrtest.limit")`
You should opt for the Satterthwaite method.

For GLMM, tests and confidence intervals are asymptotic. This just sets all the degrees of freedom to Inf. That???s emmeans???s way of using z statistics rather than t statistics. The asymptotic methods tend to make confidence intervals a bit too narrow and P values a bit too low; but they involve much, much less computation. 


## Categorical Variable

### Main Effects Only

See `emmeans` vignette: [*Confidence intervals and tests*](https://cran.r-project.org/web/packages/emmeans/vignettes/confidence-intervals.html)

```{r, warning=FALSE, fig.align='center', message=FALSE}
model.03.fact <- lmer(RT ~ as.factor(as.character(Time)) + (1|Subject), data = Raw.Datas)
emm.03.fact <- emmeans(model.03.fact, ~ Time, lmer.df = "satterthwaite")
emm.03.fact
```

Emmergence test (i.e. different from 0).

```{r, warning=FALSE, fig.align='center', message=FALSE}
summary(emm.03.fact, infer=c(T,T)) # p-values and confidence intervals
test(emm.03.fact) # only p-values = shortcut for infer=c(T,F)
confint(emm.03.fact) # only confidence intervals = shortcut for infer=c(F,T)
```


Customize your test.
For instance, to see if RT is significantly higher than 440 with Bonferroni correction.

**side** argument specifying whether the test is left-tailed (-1); right-tailed (1) or two-sided (2)

```{r, warning=FALSE, fig.align='center', message=FALSE}
test(emm.03.fact, null = 440, side = 1, adjust = "bonferroni")
```

Naming the method used to adjust pvalues or confidence limits. <br>
("holm", "hochberg", "hommel", "bonferroni", "BH", "BY", "fdr", and "none")
The adjust argument specifies a multiplicity adjustment for tests or confidence intervals. This adjustment always is applied separately to each table or sub-table that you see in the printed output.

This object can now also be used to compare whether or not there are differences between the levels of the factor. By default Tuckey correction is used.

adjuste and side arguments are still available.

```{r, warning=FALSE, fig.align='center', message=FALSE}
pairs(emm.03.fact)
```

```{r, warning=FALSE, fig.align='center', message=FALSE}
# We can also get estimated means and pairwise contrasts simultaneously:
emmeans(model.03.fact, pairwise ~ Time, lmer.df = "satterthwaite")
```


Or a slightly more powerful method.
Dealing with General Linear Hypothesis, we use the model error (from the variance-covariance matrix of the main parameters of a fitted model), not individual parts of the dataset (as.glht function).
Also, method free from package **multcomp**, which takes the correlation of the model parameters into account.

```{r, warning=FALSE, fig.align='center', message=FALSE}
summary(as.glht(pairs(emm.03.fact)), test=adjusted("free"))
```



### Interaction

We don't report that before, RT deal with Time factor AND Training factor (yes/no). Let's imagine a significant interaction. 

```{r, echo=FALSE, fig.height=7, fig.width=10}
Raw.Datas %>%
  ggplot(aes(x = Time, y = RT, color = Training)) +
  geom_point() +
  geom_smooth(method = "lm")
```

As usual, a quick, easy and powerful formula syntax.
It should be of the form `~ specs | by `

specs : names of the preedictor over wich marginals are desired. <br>
by    : names of predictors to condition on.

![Formula for fixed effects](../Pictures/emmeans_formula.png)



```{r, warning=FALSE, fig.align='center', message=FALSE}
model.04.fact <- lmer(RT ~ as.factor(as.character(Time))*Training + (1|Subject), data = Raw.Datas)
emm.04.fact <- emmeans(model.04.fact, specs = "Time", by = "Training")
emm.04.fact
```



### Custom Contrasts

The user may write a custom contrast function for use in contrast.

In our exemple, pairwise method give too much tests for all 10 Time levels.

Just baseline comparison is needed.

```{r, warning=FALSE, fig.align='center', message=FALSE}
des_cont <- list("0_1" = c(1, -1, 0, 0, 0, 0, 0, 0, 0, 0),
  	             "0_2" = c(1, 0, -1, 0, 0, 0, 0, 0, 0, 0),
  	             "0_3" = c(1, 0, 0, -1, 0, 0, 0, 0, 0, 0),
  	             "0_4" = c(1, 0, 0, 0, -1, 0, 0, 0, 0, 0),
  	             "0_5" = c(1, 0, 0, 0, 0, -1, 0, 0, 0, 0),
  	             "0_6" = c(1, 0, 0, 0, 0, 0, -1, 0, 0, 0),
  	             "0_7" = c(1, 0, 0, 0, 0, 0, 0, -1, 0, 0),
  	             "0_8" = c(1, 0, 0, 0, 0, 0, 0, 0, -1, 0),
  	             "0_9" = c(1, 0, 0, 0, 0, 0, 0, 0, 0, -1))

test(contrast(emm.04.fact, des_cont, adjust = "FDR"), side = ">")

```
Here we use only the contrast (des_cont) with FDR correction and only for > side. <br>

### Display

**afex_plot()** visualizes results from factorial experiments combining estimated marginal means and uncertainties associated with the estimated means in the foreground.


```{r, warning=FALSE, fig.align='center', message=FALSE}
plot(emm.04.fact, 
     comparison = TRUE,
     CIs = TRUE,
     horizontal = FALSE)

afex_plot(model.04.fact,
          "Time", "Training",
          error = "model",
          error_ci = FALSE,
          data_geom = ggplot2::geom_boxplot, 
          mapping = c("linetype", "shape", "fill")) +
  facet_wrap(~Training)

```

Error bars provide a grahical representation of the variability of the estimated means. In the case of a mixed-design, no single type of error bar is possible that allows comparison across all crossed random conditions. 
There exist several possibilities which particular measure of variability to use:
- confidence intervals (model-based)
- standard errors (model-based)


In the case of designs involving repeated-measures factors these mesures cannot be used as significant differences as this requires knowledge about the correlation between measures. ` error = "within"" ` allow judging differences between repeated-measures condition.


```{r, warning=FALSE, fig.align='center', message=FALSE}
model.05.fact <- lmer(RT ~ as.factor(as.character(Time))*Training + (as.factor(as.character(Time))|Subject), data = Raw.Datas)

afex_plot(model.05.fact,
          "Time", "Training",
          error = "within",
          error_ci = FALSE,
          data_geom = ggplot2::geom_boxplot, 
          mapping = c("linetype", "shape", "fill")) +
  facet_wrap(~Training)

```



## Continuous numerical variable

The `emtrends` function is useful when a fitted model involves a continuous numerical predictor and an interacting with another predictor.

### Linear model

```{r, warning=FALSE, fig.align='center', message=FALSE}
model.04.num <- glmer(RT ~ Training*Time + (Time|Subject), data = Raw.Datas)
emm.01 <- emtrends(model.04.num, 
                   specs = "Training", var = "Time", 
                  options = list())
emm.01
```

Emergence test: slope significance.

```{r, warning=FALSE, fig.align='center', message=FALSE}
summary(emm.01, infer=c(T,T))
```

To display these slopes. We have to plot the predicted values.
```{r, warning=FALSE, fig.align='center', message=FALSE}
plot(ggpredict(model.04.num, terms = c("Time", "Training")))
```

Or we could use the **emmip()** function.

```{r, warning=FALSE, fig.align='center', message=FALSE}
emmip(model.04.num, 
      Training ~ Time, 
      cov.reduce = range) 
```      
Note the **cov.reduce = range** argument is passed because by default, each covariate is reduced to only one value (the mean). Instead, we call the **range()** function to obtain the minimum and maximum diameter.



Test difference between slopes:
```{r, warning=FALSE, fig.align='center', message=FALSE}
emtrends(model.04.num, pairwise ~ Training, var="Time")
```

Test difference between factor levels at different points of the continuous variable:

```{r, warning=FALSE, fig.align='center', message=FALSE}
emm.02 <- emmeans(model.04.num, ~ Time | Training, 
                  at = list(Time = c(0, 5, 9)))
pairs(emm.02)
```

All arguments are still available.

```{r, warning=FALSE, fig.align='center', message=FALSE}
des_cont <- c(list("0-5" = c(1, -1,  0),
                   "0-9" = c(1,  0, -1)))
test(contrast(regrid(emm.02), des_cont, adjust = "FDR"), side = ">")
```


### Non-Linear model


```{r, warning=FALSE, fig.align='center', message=FALSE}
library(splines)
model.05.num <- glmer(RT ~ Training*ns(Time,2) + (ns(Time,2)|Subject), data = Raw.Datas)
```

There are many functions to generate a non-linear behaviour. Here we use a **ns()** function where spline was generate. **Knots** argument is the number of breakpoints taht define the spline. Here is close to plynomial degree 2.

Then we can't test the slope differences. We have to test difference between factor levels at different points of the continuous variable.



## Ordinal variable

              


# Part 6 - FAQ Reviewers


## Effect size

## Why you used a GLM ? 

For instance:
"In most previous studies of the attributional model, ANOVAs were used to analyze the effects of experimentally manipulated factors (...). Therefore, the use of these methods needs to be better justified and the methods need to be explained in more detail. In addition, the traditional analyses should also be reported to judge whether results comparable to those found in previous studies are obtained with these methods."


## Power

Only the simulation could provide a power estimation. <br>
Bates, Kliegl, Vasishth, and Baayen (2015) said that GLM is a best way to deal with heterogeneity :"as it results in the highest power without an increase in Type I errors". <br>

### R2

Here, a paper to talk about **R2* and twelve properties of "traditional"" R2 for regression models: 
https://besjournals.onlinelibrary.wiley.com/doi/full/10.1111/j.2041-210x.2012.00261.x
<br>
<br>
With **r.squaredGLMM()** (package MuMIn), we calculate conditional and marginal coefficient of determination for GLM. <br>
Marginal represents the variance explained by the fixed effects. <br>
Conditional is interpreted as a variance explained by the entire model, including both fixed and random effects. <br>

### Bayesian

I'm always looking for a way to talk about the power of analysis with GLM. I wonder if the wonderful world of Bayesian can help us. 
Here is the theory (if there are experts to correct me, I am waiting for your feedback) : <br>

In frequentist world, power analysis can help disentangle these alternatives : <br>
evidence of absence <br>
evidence of presence  <br>
absence of evidence <br>

In our analysis (GLM), power is a tricky question. But Bayesian approach could be helpful (https://www.nature.com/articles/s41593-020-0660-4).  <br>

The Bayesian Factor (BF) should primarily be seen as a continuous measure of evidence. <br>
BF quantifies the degree to which the data warrant a change in beliefs, and it therefore represents the strength of evidence that the data provide for H0 vs H1. <br>

The advantage of retaining a continuous representation of evidence was stressed by Rozeboom <br> (http://stats.org.uk/statistical-inference/Rozeboom1960.pdf):  <br>
???The null-hypothesis significance test treats ???acceptance??? or ???rejection??? of a hypothesis as though these were decisions one makes. But a hypothesis is not something, like a piece of pie offered for dessert, which can be accepted or rejected by a voluntary physical action. Acceptance or rejection of a hypothesis is a cognitive process, a degree of believing or disbelieving which, if rational, is not a matter of choice but determined solely by how likely it is, given the evidence, that the hypothesis is true.???  <br>

Userers can judge the strength of the evidence directly from the numerical value of BF, with a BF twice as high providing evidence twice as strong. In contrast, it can be difficult to interpret an actual P value as strength of evidence, as P = 0.01 does not provide five times as much evidence as P = 0.05. <br>

Jeffreys proposed reference values to guide the interpretation of the strength of the evidence9. These values were spaced out in exponential half steps of 10, 100.5 ??? 3, 101 = 10, 101.5 ??? 30, etc., to be equidistant on a log scale. <br>

Bayesian approach provide several metrics.  <br>
The Probability of Direction (**pd**) is an index of effect existence representing the certainty with which an effect goes in a particular direction (i.e., is positive or negative). Beyond its simplicity of interpretation, understanding and computation, this index also presents other interesting properties. It is independent from the model, i.e., it is solely based on the posterior distributions and does not require any additional information from the data or the model. The pd is not relevant for assessing the size or importance of an effect and is not able to provide information in favor of the null hypothesis. In other words, a high pd suggests the presence of an effect but a small pd does not give us any information about how plausible the null hypothesis is, suggesting that this index can only be used to eventually reject the null hypothesis (which is consistent with the interpretation of the frequentist p-value). In contrast, **BF**s increase or decrease as the evidence becomes stronger. <br>



# Conclusion

Where classical ANOVA uses a least squares solution, LMMs uses a maximum likelihood solution([details here](https://www.seascapemodels.org/rstats/2018/04/13/how-to-use-the-AIC.html)). As a consequence, LMMs do not need balanced and complete data. A tremendous advantage of this is that we no longer need to pool and average subject data as we do in ANOVAs: with LMMs we can **model all trials!**

LMMs are also very robust to violations of homoskedasticity and sphericity (e.g. [Arnau et al. 2013](https://link.springer.com/article/10.3758/s13428-012-0306-x "The effect of skewness and kurtosis on the robustness of linear mixed models [Open access article]")). With balanced design, no missing datas and sphericity, ANOVA and LMMs yield the same results.

The default parameter estimation criterion for linear mixed models is restricted (or ???residual???) maximum likelihood (REML). The likelihood of the parameters, given the observed data. You can read more about LMMs fitting procedure [here](http://lme4.r-forge.r-project.org/slides/2011-01-11-Madison/4Theory.pdf "Douglas Bates - Theory of linear mixed models"). 


LMM have many advantages over ANOVA, including (but not limited to):

  - Analysis of single trial data (as opposed to aggregated means per condition).
  - Specifying more than one random factor.
  - The use of continuous variables as predictors.
  - Making you look like you know what you???re doing.
  - Defeating the reviewer.
  - The ability to specify custom models


*Does using lmer() make one a bayesian ?*
No. But with model parameters that are random, the use of posterior distributions, shrinkage of estimates towards a more stable target, life suddenly feels more bayesian. 



# References

[Overview](https://www.r-bloggers.com/linear-models-anova-glms-and-mixed-effects-models-in-r/)

[LME4](http://lme4.r-forge.r-project.org/slides/2011-01-11-Madison/4Theory.pdf "Douglas Bates - Theory of linear mixed models")

[Sphericity](https://www.uvm.edu/~dhowell/StatPages/Mixed-Models-Repeated/Mixed-Models-for-Repeated-Measures1.html "Dans le paragraphe ?? Covariance Structures ??")

[Great resource for the formula syntax](http://conjugateprior.org/2013/01/formulae-in-r-anova/)

[Diagnostics LMER](http://docs.statwing.com/interpreting-residual-plots-to-improve-your-regression/#x-unbalanced-header)

[Diagnostics GLMER](https://cran.r-project.org/web/packages/DHARMa/vignettes/DHARMa.html)
[GAM](http://www.martijnwieling.nl/statistics.php)

[Many many things](https://joelemartinez.com/2015/07/14/mixed-effect-models/)

[Maximal random effects structure](https://www.sciencedirect.com/science/article/pii/S0749596X12001180)

[Parsimonious Random effect structure](https://www.sciencedirect.com/science/article/pii/S0749596X17300013#s0005)



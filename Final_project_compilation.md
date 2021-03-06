---
title: "STAT 253 Final Project"
author: "Ethan Hyslop, Leah Robotham, Jacqueline Ong, Emma Iverson"
date: "5/5/2022"
output: 
  html_document:
    keep_md: TRUE
    toc: TRUE
    toc_float: TRUE
    df_print: paged
    code_download: true
---





```r
# library statements 
library(dplyr)
library(readr)
library(broom)
library(ggplot2)
library(tidymodels)
library(lubridate)
library(gridExtra)
tidymodels_prefer() # Resolves conflicts, prefers tidymodel functions
```

  In this analysis, we utilize statistical machine learning techniques to study the spread of COVID-19 cases between August 13th, 2021 and January 27th, 2022, when we downloaded the source data. We selected this date range to focus on the prevalence of COVID-19 Delta and Omicron variants as disruptors in previously established COVID trend data, and hope that with this research we may contribute to a growing understanding of what factors lead to COVID spread. Additionally, a later analysis seeks to illuminate how case trends cluster across different locations, further developing COVID-spread as a geographically heterogeneous phenomenon. 
  
  Our analysis in this project centers around three research questions. Our first line of query utilizes a LASSO regression model to predict the quantity of new cases within the United States. Here, units represent individual predicted cases which may be explained by variation in the predictor variables detailed above. This first question seeks to understand what impacts the spread of COVID variants, addressing the core motivation of our research to expand this knowledge. Next, we used a random forest decision tree to classify global new case rate by continent. This was performed to understand how variation in new case rates in different regions of the world may vary in causal factors, seen here in how the model classifies cases. Ultimately, this section seeks to empower a better geographically-specific response. Finally, we built on this classification analysis by using K-means clustering to group observed international vaccination rates and total cases by continent. This was done in an effort to understand natural groups within the data, associated with case trends seen globally.



```r
# read in data
covid <- read_csv("owid-covid-data.csv")
```
  A case in our dataset represents the covid data from the United States on a specific date within our date range. We chose to use only cases from the United States within this specific date range to minimize the number of NULL values in our dataset. The quantitative outcome variable for regression we are trying to predict is the number of new cases on any given day. We used 10 predictor variables: total_cases, new_cases, date, total_deaths, reproduction_rate, people_vaccinated_per_hundred, people_fully_vaccinated_per_hundred, total_boosters_per_hundred, tests_per_case, icu_patients and positive_rate. The data was collected by Our World in Data project from the official numbers from governments and health ministries worldwide in an effort to document and understand covid trends. 

```r
# data cleaning
covid <- covid %>% 
  filter(location == 'United States') %>%
  select(total_cases, new_cases, date, total_deaths, reproduction_rate, people_vaccinated_per_hundred, people_fully_vaccinated_per_hundred, total_boosters_per_hundred, tests_per_case, icu_patients, positive_rate) %>%
  filter(date >= as.Date("2021-08-13"))%>%
  na.omit(covid)
```
  Coming into this project, we suspect that there will be a strong negative correlation between vaccination metrics and the number of total cases that are reported in any location.


```r
# creation of cv folds
set.seed(123)
covid_cv10 <- vfold_cv(covid, v = 10)
```

WRITE SOMETHING HERE ABOUT FOLLOWING CHUNK



```r
# model spec
# OLS
lm_spec <-
    linear_reg() %>% 
    set_engine(engine = 'lm') %>% 
    set_mode('regression')

mod1 <- fit(lm_spec,
            new_cases ~ ., 
            data = covid)

mod2 <- fit(lm_spec,
            new_cases ~ reproduction_rate, 
            data = covid)

mod3 <- fit(lm_spec,
            new_cases ~ reproduction_rate+people_vaccinated_per_hundred, 
            data = covid)

mod4 <- fit(lm_spec,
            new_cases ~ reproduction_rate+people_vaccinated_per_hundred+people_fully_vaccinated_per_hundred, 
            data = covid)

mod5 <- fit(lm_spec,
            new_cases ~ reproduction_rate+people_vaccinated_per_hundred+people_fully_vaccinated_per_hundred+total_boosters_per_hundred, 
            data = covid)


covid_rec_ols <- recipe(new_cases ~ .,  data = covid) %>%
  step_normalize(all_numeric_predictors()) 

#mod1
cov_wf_ols <- workflow() %>%
  add_recipe(covid_rec_ols) %>%
  add_model(lm_spec)%>%
  step_normalize(all_numeric_predictors()) %>%
  step_dummy(all_nominal_predictors()) %>%
  step_nzv(all_predictors())



mod1 <- fit(cov_wf_ols,
            data = covid) 

#mod2
cov_wf_ols2 <- workflow() %>%
  add_formula(new_cases ~ reproduction_rate)%>%
  add_model(lm_spec)%>%
  step_normalize(all_numeric_predictors()) %>%
  step_dummy(all_nominal_predictors()) %>%
  step_nzv(all_predictors())

cov_wf_ols3 <- workflow() %>%
  add_formula(new_cases ~ reproduction_rate+people_vaccinated_per_hundred)%>%
  add_model(lm_spec)%>%
  step_normalize(all_numeric_predictors()) %>%
  step_dummy(all_nominal_predictors()) %>%
  step_nzv(all_predictors())

cov_wf_ols4 <- workflow() %>%
  add_formula(new_cases ~ reproduction_rate+people_vaccinated_per_hundred+people_fully_vaccinated_per_hundred)%>%
  add_model(lm_spec)%>%
  step_normalize(all_numeric_predictors()) %>%
  step_dummy(all_nominal_predictors()) %>%
  step_nzv(all_predictors())

cov_wf_ols5 <- workflow() %>%
  add_formula(new_cases ~ reproduction_rate+people_vaccinated_per_hundred+people_fully_vaccinated_per_hundred+total_boosters_per_hundred)%>%
  add_model(lm_spec)%>%
  step_normalize(all_numeric_predictors()) %>%
  step_dummy(all_nominal_predictors()) %>%
  step_nzv(all_predictors())
```


```r
mod1 %>% 
  tidy() %>% 
  slice(-1) %>% 
  mutate(lower = estimate - 1.96*std.error, upper = estimate + 1.96*std.error) %>% 
  ggplot() + 
    geom_vline(xintercept=0, linetype=4) + 
    geom_point(aes(x=estimate, y=term)) + 
    geom_segment(aes(y=term, yend=term, x=lower, xend=upper), arrow = arrow(angle=90, ends='both', length = unit(0.1, 'cm'))) + 
    labs(x = 'Coefficient estimate (95% CI)', y = 'Feature') +  
    theme_classic() 

mod1 %>%
  tidy()
```

```r
mod1_output <- mod1 %>% 
    predict(new_data = covid) %>%
    bind_cols(covid) %>%
    mutate(resid = new_cases - .pred)

mod2_output <- mod2 %>% 
    predict(new_data = covid) %>%
    bind_cols(covid) %>%
    mutate(resid = new_cases - .pred)

mod3_output <- mod3 %>% 
    predict(new_data = covid) %>%
    bind_cols(covid) %>%
    mutate(resid = new_cases - .pred)

mod4_output <- mod4 %>% 
    predict(new_data = covid) %>%
    bind_cols(covid) %>%
    mutate(resid = new_cases - .pred)

mod5_output <- mod5 %>% 
    predict(new_data = covid) %>%
    bind_cols(covid) %>%
    mutate(resid = new_cases - .pred)
```


```r
mod1_output %>%
    mae(truth = new_cases, estimate = .pred)

mod2_output %>%
    mae(truth = new_cases, estimate = .pred)

mod3_output %>%
    mae(truth = new_cases, estimate = .pred)

mod4_output %>%
    mae(truth = new_cases, estimate = .pred)

mod5_output %>%
    mae(truth = new_cases, estimate = .pred)
```


```r
#mod1

mod1_tc <- ggplot(mod1_output, aes(y=resid, x=total_cases)) +
    geom_point() +
    geom_smooth() +
    geom_hline(yintercept = 0, color = "red") + 
    theme_classic()

mod1_td <- ggplot(mod1_output, aes(y=resid, x=total_deaths)) +
    geom_point() +
    geom_smooth() +
    geom_hline(yintercept = 0, color = "red") + 
    theme_classic()

mod1_rr <- ggplot(mod1_output, aes(y=resid, x=reproduction_rate)) +
    geom_point() +
    geom_smooth() +
    geom_hline(yintercept = 0, color = "red") + 
    theme_classic()

mod1_pvph <- ggplot(mod1_output, aes(y=resid, x=people_vaccinated_per_hundred)) +
    geom_point() +
    geom_smooth() +
    geom_hline(yintercept = 0, color = "red") + 
    theme_classic()

mod1_pfvph <- ggplot(mod1_output, aes(y=resid, x=people_fully_vaccinated_per_hundred)) +
    geom_point() +
    geom_smooth() +
    geom_hline(yintercept = 0, color = "red") + 
    theme_classic()

mod1_tbph <- ggplot(mod1_output, aes(y=resid, x=total_boosters_per_hundred)) +
    geom_point() +
    geom_smooth() +
    geom_hline(yintercept = 0, color = "red") + 
    theme_classic()

mod1_tpc <- ggplot(mod1_output, aes(y=resid, x=tests_per_case)) +
    geom_point() +
    geom_smooth() +
    geom_hline(yintercept = 0, color = "red") + 
    theme_classic()

mod1_date <- ggplot(mod1_output, aes(y=resid, x=date)) +
    geom_point() +
    geom_smooth() +
    geom_hline(yintercept = 0, color = "red") + 
    theme_classic()

mod1_icu <- ggplot(mod1_output, aes(y=resid, x=icu_patients)) +
    geom_point() +
    geom_smooth() +
    geom_hline(yintercept = 0, color = "red") + 
    theme_classic()

grid.arrange(mod1_tc, mod1_td, mod1_rr, mod1_pvph, nrow =2, ncol = 2)

grid.arrange(mod1_pfvph, mod1_tbph, mod1_tpc, mod1_date, nrow=2, ncol=2)
```


```r
#mod2
ggplot(mod2_output, aes(y=resid, x=reproduction_rate)) +
    geom_point() +
    geom_smooth() +
    geom_hline(yintercept = 0, color = "red") + 
    theme_classic()
```


```r
#mod3
ggplot(mod3_output, aes(y=resid, x=reproduction_rate)) +
    geom_point() +
    geom_smooth() +
    geom_hline(yintercept = 0, color = "red") + 
    theme_classic()

ggplot(mod3_output, aes(y=resid, x=people_vaccinated_per_hundred)) +
    geom_point() +
    geom_smooth() +
    geom_hline(yintercept = 0, color = "red") + 
    theme_classic()
```


```r
#mod4

ggplot(mod4_output, aes(y=resid, x=reproduction_rate)) +
    geom_point() +
    geom_smooth() +
    geom_hline(yintercept = 0, color = "red") + 
    theme_classic()

ggplot(mod4_output, aes(y=resid, x=people_vaccinated_per_hundred)) +
    geom_point() +
    geom_smooth() +
    geom_hline(yintercept = 0, color = "red") + 
    theme_classic()

ggplot(mod4_output, aes(y=resid, x=people_fully_vaccinated_per_hundred)) +
    geom_point() +
    geom_smooth() +
    geom_hline(yintercept = 0, color = "red") + 
    theme_classic()
```


```r
#mod5
ggplot(mod5_output, aes(y=resid, x=reproduction_rate)) +
    geom_point() +
    geom_smooth() +
    geom_hline(yintercept = 0, color = "red") + 
    theme_classic()

ggplot(mod5_output, aes(y=resid, x=people_vaccinated_per_hundred)) +
    geom_point() +
    geom_smooth() +
    geom_hline(yintercept = 0, color = "red") + 
    theme_classic()

ggplot(mod5_output, aes(y=resid, x=people_fully_vaccinated_per_hundred)) +
    geom_point() +
    geom_smooth() +
    geom_hline(yintercept = 0, color = "red") + 
    theme_classic()

ggplot(mod5_output, aes(y=resid, x=total_boosters_per_hundred)) +
    geom_point() +
    geom_smooth() +
    geom_hline(yintercept = 0, color = "red") + 
    theme_classic()
```

From the residual plots, it seems that `people_fully_vaccinated_per_hundred	`, `people_vaccinated_per_hundred` and	
`total_boosters_per_hundred` are variables that look like they would be better modeled with nonlinear relationships.  


```r
mod1_cv <- fit_resamples(cov_wf_ols,
  resamples = covid_cv10, 
  metrics = metric_set(rmse, rsq, mae))
  
  
mod2_cv <- fit_resamples(cov_wf_ols2,
  resamples = covid_cv10, 
  metrics = metric_set(rmse, rsq, mae))
  
  
mod3_cv <- fit_resamples(cov_wf_ols3,
  resamples = covid_cv10, 
  metrics = metric_set(rmse, rsq, mae))
  
  
mod4_cv <- fit_resamples(cov_wf_ols4,
  resamples = covid_cv10, 
  metrics = metric_set(rmse, rsq, mae))
  
  
mod5_cv <- fit_resamples(cov_wf_ols5,
  resamples = covid_cv10, 
  metrics = metric_set(rmse, rsq, mae))
```


```r
mod1_cv %>% unnest(.metrics) %>%
  filter(.metric == 'rmse') %>%
  summarize(RMSE_CV = mean(.estimate))

mod2_cv %>% unnest(.metrics) %>%
  filter(.metric == 'rmse') %>%
  summarize(RMSE_CV = mean(.estimate))

mod3_cv %>% unnest(.metrics) %>%
  filter(.metric == 'rmse') %>%
  summarize(RMSE_CV = mean(.estimate))

mod4_cv %>% unnest(.metrics) %>%
  filter(.metric == 'rmse') %>%
  summarize(RMSE_CV = mean(.estimate))

mod5_cv %>% unnest(.metrics) %>%
    filter(.metric == 'rmse') %>%
    summarize(RMSE_CV = mean(.estimate))

mod1_cv %>% collect_metrics()

mod2_cv %>% collect_metrics()

mod3_cv %>% collect_metrics()

mod4_cv %>% collect_metrics()

mod5_cv %>% collect_metrics()
```



```r
# LASSO
lm_lasso_spec <- 
  linear_reg() %>%
  set_args(mixture = 1, penalty = tune()) %>% 
  set_engine(engine = 'glmnet') %>% 
  set_mode('regression') 
```


```r
# recipes & workflows
# specify Recipe (if you have preprocessing steps)

covid <- covid %>% select(-date)

covid_recipe <- recipe(new_cases ~ ., data = covid) %>%
  step_nzv(all_predictors()) %>%
  step_novel(all_nominal_predictors()) %>%
  step_normalize(all_numeric_predictors()) %>%
  step_dummy(all_nominal_predictors()) %>%
  step_naomit(all_predictors()) #skip = TRUE


covid_wf <- workflow() %>%
  add_recipe(covid_recipe) %>%
  add_model(lm_spec)

covid_wf_lasso <- workflow() %>%
  add_recipe(covid_recipe) %>%
  add_model(lm_lasso_spec)
```


```r
# fit & tune models

mod_lasso_fit <- fit(covid_wf_lasso, data = covid)

plot(mod_lasso_fit %>% extract_fit_parsnip() %>% pluck('fit'), # way to get the original glmnet output
     xvar = "lambda") # glmnet fits the model with a variety of lambda penalty values

# tune model (try a variety of lambda penalty values)
penalty_grid <- grid_regular(penalty(range = c(-5,3)), levels = 30)
```

c.


```r
tune_res <- tune_grid( # new function for tuning parameters
  covid_wf_lasso, # workflow
  resamples = covid_cv10, # cv folds
  metrics = metric_set(rmse, mae),
  grid = penalty_grid # penalty grid defined above
)

#  calculate/collect CV metrics
collect_metrics(tune_res) %>%
  filter(.metric == 'mae') %>% # or choose mae
  select(penalty, mae = mean, std_err) 

# best penalty
best_penalty <- select_best(tune_res, metric = 'mae') # choose penalty value based on lowest mae or rmse
best_se_penalty <- select_by_one_std_err(tune_res, metric = 'mae', desc(penalty)) # choose penalty value based on the largest penalty within 1 se of the lowest CV MAE
```


```r
#Fit Final Model
final_wf <- finalize_workflow(covid_wf_lasso, best_penalty) # incorporates penalty value to workflow
final_wf_se <- finalize_workflow(covid_wf_lasso, best_se_penalty) # incorporates penalty value to workflow

final_fit <- fit(final_wf, data = covid)
final_fit_se <- fit(final_wf_se, data = covid)

tidy(final_fit)
tidy(final_fit_se)
```

 
d.


```r
# visual residuals

tune_res %>% collect_metrics() %>% filter(penalty == (best_se_penalty %>% pull(penalty)))
```

### Residual Plot of LASSO Model Prediction


```r
lasso_mod_out <- final_fit_se %>%
    predict(new_data = covid) %>%
    bind_cols(covid) %>%
    mutate(resid = new_cases - .pred)

lasso_mod_out %>% 
  ggplot(aes(x = .pred, y = resid)) + 
  geom_point() +
  geom_smooth(se = FALSE) + 
  geom_hline(yintercept = 0) + 
  theme_classic()
```

### Residuals from LASSO Model

Our group determined from homework 1 that the LASSO model was the best model for our data so far, because it had the lowest MAE compared to the other models we tried.


```r
pl1 <- ggplot(lasso_mod_out, aes(y=resid, x=total_cases)) +
    geom_point() +
    geom_smooth() +
    geom_hline(yintercept = 0, color = "red") + 
    theme_classic()

pl2 <- ggplot(lasso_mod_out, aes(y=resid, x=total_deaths)) +
    geom_point() +
    geom_smooth() +
    geom_hline(yintercept = 0, color = "red") + 
    theme_classic()

pl3 <- ggplot(lasso_mod_out, aes(y=resid, x=reproduction_rate)) +
    geom_point() +
    geom_smooth() +
    geom_hline(yintercept = 0, color = "red") + 
    theme_classic()

pl4<- ggplot(lasso_mod_out, aes(y=resid, x=people_vaccinated_per_hundred)) +
    geom_point() +
    geom_smooth() +
    geom_hline(yintercept = 0, color = "red") + 
    theme_classic()

pl5 <- ggplot(lasso_mod_out, aes(y=resid, x=people_fully_vaccinated_per_hundred)) +
    geom_point() +
    geom_smooth() +
    geom_hline(yintercept = 0, color = "red") + 
    theme_classic()

pl6 <- ggplot(lasso_mod_out, aes(y=resid, x=total_boosters_per_hundred)) +
    geom_point() +
    geom_smooth() +
    geom_hline(yintercept = 0, color = "red") + 
    theme_classic()

pl7 <- ggplot(lasso_mod_out, aes(y=resid, x=tests_per_case)) +
    geom_point() +
    geom_smooth() +
    geom_hline(yintercept = 0, color = "red") + 
    theme_classic()

pl8 <- ggplot(lasso_mod_out, aes(y=resid, x=icu_patients)) +
    geom_point() +
    geom_smooth() +
    geom_hline(yintercept = 0, color = "red") + 
    theme_classic()

pl9 <- ggplot(lasso_mod_out, aes(y=resid, x=positive_rate)) +
    geom_point() +
    geom_smooth() +
    geom_hline(yintercept = 0, color = "red") + 
    theme_classic()

pl1
pl2
pl3
pl4
pl5
pl6
pl7
pl8
pl9
```

From the residual plots of the LASSO model above, it looks like we can try to model all of these predictors with nonlinear relationships. This is because the plots show heteroskedasticity. 

e.

We can look at the LASSO regression results to determine variable importance. LASSO is a shrinking algorithm which will eliminate weak predictors from the model by setting their coefficients to 0. From the estimate values of the coefficients, we can see that `total_deaths`, `people_vaccinated_per_hundred` and `total_boosters_per_hundred` have been shrunk to 0 and are not considered important variables in the model. 

However, the magnitude of the other coefficients produced by the LASSO algorithm are quite large, so this indicates that they are all important variables.


HOMEWORK 3

<br>

2. **Non linearity**

```r
# GAMS

gam_spec <- 
  gen_additive_mod() %>%
  set_engine(engine = 'mgcv') %>%
  set_mode('regression') 

gam_mod <- fit(gam_spec,
    new_cases ~ s(total_cases) + s(total_deaths) + s(total_deaths) + s(reproduction_rate) + s(people_vaccinated_per_hundred) + s(people_fully_vaccinated_per_hundred) + s(tests_per_case) + s(icu_patients) + s(positive_rate) + s(total_boosters_per_hundred),
    data = covid
)

# Diagnostics: Check to see if the number of knots is large enough (if p-value is low, increase number of knots)
par(mfrow=c(2,2))
gam_mod %>% pluck('fit') %>% mgcv::gam.check() 
```

```r
gam_mod %>% pluck('fit') %>% summary() 
gam_mod %>% pluck('fit') %>% plot()
gam_mod %>% pluck('fit') %>% plot( all.terms = TRUE, pages = 1)
```


```r
#fitting a more simple GAM model by only allowing for a non-linear function if the edf is greater than 3.
gam_mod2 <- fit(gam_spec,
    new_cases ~ s(total_cases) + s(total_deaths) + s(total_deaths) + s(reproduction_rate) + s(people_vaccinated_per_hundred) + people_fully_vaccinated_per_hundred + tests_per_case + icu_patients + s(positive_rate) + s(total_boosters_per_hundred),
    data = covid
)

gam_mod2 %>% pluck('fit') %>% summary() 
gam_mod2 %>% pluck('fit') %>% plot()
```



```r
lm_spec <-
  linear_reg() %>%
  set_engine(engine = 'lm') %>%
  set_mode('regression')

data_cv8 <- covid %>% 
    vfold_cv(v = 8)

covid_rec_spline <- recipe(new_cases ~ total_cases + total_deaths + reproduction_rate + people_vaccinated_per_hundred + people_fully_vaccinated_per_hundred+ total_boosters_per_hundred+ tests_per_case + icu_patients+ positive_rate, data = covid)
  
spline_rec <- covid_rec_spline %>% 
  step_ns(total_cases, deg_free = 9) %>%
  step_ns(total_deaths, deg_free = 5) %>% 
  step_ns(reproduction_rate, deg_free = 9) %>% 
  step_ns(people_vaccinated_per_hundred, deg_free = 2) %>% 
  step_ns(total_boosters_per_hundred, deg_free = 8) %>%
  step_ns(positive_rate, deg_free = 1)

# Check the pre-processed data
spline_rec %>% prep(covid) %>% juice()

covid_spline_wf <- workflow() %>%
    add_model(lm_spec) %>%
    add_recipe(covid_rec_spline)
  
spline_wf <- workflow() %>% 
  add_recipe(spline_rec) %>%
  add_model(lm_spec) 

fit_resamples(
    covid_spline_wf,
    resamples = data_cv8, # cv folds
    metrics = metric_set(mae,rmse,rsq)                     
) %>% collect_metrics()
```


```r
fit_resamples(
    spline_wf,
    resamples = data_cv8, # cv folds
    metrics = metric_set(mae,rmse,rsq)                     
) %>% collect_metrics()
```


```r
ns_mod <- spline_wf %>%
  fit(data = covid) 

ns_mod %>%
  tidy()

spline_mod_output <- covid %>%
  bind_cols(predict(ns_mod, new_data = covid)) %>%
    mutate(resid = new_cases - .pred)

p4 <- ggplot(spline_mod_output, aes(x = total_cases, y = resid)) +
    geom_point() +
    geom_smooth() +
    geom_hline(yintercept = 0, color = "red") +
    theme_classic()
p5 <- ggplot(spline_mod_output, aes(x = total_deaths, y = resid)) +
    geom_point() +
    geom_smooth() +
    geom_hline(yintercept = 0, color = "red") +
    theme_classic()
p6 <- ggplot(spline_mod_output, aes(x = reproduction_rate, y = resid)) +
    geom_point() +
    geom_smooth() +
    geom_hline(yintercept = 0, color = "red") +
    theme_classic()
p7 <- ggplot(spline_mod_output, aes(x = people_vaccinated_per_hundred, y = resid)) +
    geom_point() +
    geom_smooth() +
    geom_hline(yintercept = 0, color = "red") +
    theme_classic()
p8 <- ggplot(spline_mod_output, aes(x = total_boosters_per_hundred, y = resid)) +
    geom_point() +
    geom_smooth() +
    geom_hline(yintercept = 0, color = "red") +
    theme_classic()
p9 <- ggplot(spline_mod_output, aes(x = positive_rate, y = resid)) +
    geom_point() +
    geom_smooth() +
    geom_hline(yintercept = 0, color = "red") +
    theme_classic()
p4
p5
p6
p7
p8
p9
```


```r
### RELABEL PLOTS to include lasso and splines for distinction
grid.arrange(pl1, p4, nrow = 1, ncol = 2)
grid.arrange(pl2, p5, nrow = 1, ncol = 2)
grid.arrange(pl3, p6, nrow = 1, ncol = 2)
grid.arrange(pl4, p7, nrow = 1, ncol = 2)
grid.arrange(pl6, p8, nrow = 1, ncol = 2)
grid.arrange(pl9, p9, nrow = 1, ncol = 2)
```

* Compare insights from variable importance analyses here and the corresponding results from the Investigation 1. Now after having accounted for nonlinearity, have the most relevant predictors changed?

Comparing between our LASSO model and nonlinear splines model, we see the inclusion of previously zeroed out variables including total deaths, people vaccinated per hundred, and people boosted per hundred. However, our most predictive variables are still the greatest in the model.  



* Do you gain any insights from the GAM output plots (easily obtained from fitting smoothing splines) for each  predictor?

The GAM output plots reveal a handful of non-linear variables, especially total vaccinations per hundred, and total boosters per hundred. This demonstrates the value of fitting models allowing for non-linearity, as forcing linearity upon these would have resulted in a poor estimate. 
  

* Compare model performance between your GAM models that the models that assuming linearity.
How does test performance of the GAMs compare to other models you explored?

Across all metrics, our GAM models perform somewhat better than those which assume non-linearity. We can see this in the lower MAE, lower RMSE, and slightly higher r-squared.

3. **Summarize investigations**
    - Decide on an overall best model based on your investigations so far. To do this, make clear your analysis goals. Predictive accuracy? Interpretability? A combination of both?

Our analysis goal is to predict the quantity of new COVID cases. Given the universal salience of this subject, it is important to have both highly interpretable results as well as a quality predictive model. To attempt to explain the rate of new COVID cases, we have fit a series of least-squares linear regression models, as well as a series of models subjected to LASSO linear regression. Of our fit models, we find that the best-fit LASSO model has the lowest mean absolute error (MAE), therefore producing the most accurate prediction of our fit models. However, even this model results in a sizable MAE. To correct this, we must consider the non linearity of some input variables, and undertake steps to model them using polynomial regression. 

HW3 Update:

We attempted to correct for nonlinearity through the implementation in splines, however these models had an even larger MAE. Therefore, we will continue using the last LASSO model. 


<br>

4. **Societal impact**
    - Are there any harms that may come from your analyses and/or how the data were collected?
    
    These analyses focus on the COVID-19 pandemic which has resulted in widespread illness and death. The data reflects the very real tragedies of people around the world, and as such should be considered with gravity. These data have been collected in an anonymous way to respect the privacy of those they represent, and can be used in the important effort to understand better the spread of the disease. We do not anticipate incurring any societal harm through the use and analysis of this data. 
    
    - What cautions do you want to keep in mind when communicating your work?
    
It is important to remember that the reporting of these data has been done individually by each country, and therefore is incomplete and prone to some inconsistencies. Additionally, it is important to remember the people behind these data, both for context and in recognition of the lives lost. 




<br><br><br>



# Homework 4

## Required Analysis

For this homework,

1. **Specify the research question for a classification task.**

Can we predict the continent the observation comes from?

2. **Try to implement at least 2 different classification methods to answer your research question.**

1. Logistic regression
2. Decision tree

3. **Reflect on the information gained from these two methods and how you might justify this method to others.**

#### Your Work {-}


```r
# library statements 
library(dplyr)
library(readr)
library(broom)
library(ggplot2)
library(tidymodels)
library(lubridate)
library(gridExtra)
library(probably)
tidymodels_prefer() # Resolves conflicts, prefers tidymodel functions

# read in data
covid <- read_csv("owid-covid-data.csv")
```


```r
# data cleaning
covid <- covid %>%
  select(total_cases, new_cases, date, total_deaths, reproduction_rate, people_vaccinated_per_hundred, people_fully_vaccinated_per_hundred, total_boosters_per_hundred, tests_per_case, icu_patients, positive_rate, life_expectancy, population_density, median_age, gdp_per_capita, continent) %>%
  filter(date >= as.Date("2021-08-13"))%>%
  na.omit(covid)
```

- We're considering COVID cases occurring after the first recorded booster vaccines (13th August, 2021) to best capture today's landscape.  
- This is when vaccinations started to become widespread (at least in the United States).
- In addition to our existing variables, we also added the following to our model to predict the continent:
    - `life_expectancy`
    - `population_density`
    - `median_age`
    - `gdp_per_capita`

## Classification - Methods

### LASSO Logistic Regression 

We wil fit a LASSO logistic regression model for the `continent` outcome, and allow all possible predictors to be considered.

We initially try a sequence of 100 $\lambda$'s from 1 to 10, which we then adjust based on the plot of test AUC versus $\lambda$.

Since we can only perform binominal logistic regression, we had to create a new variable, `NorthAmerica_or_Not`, categorizing the data based on whether its `continent` was "North America" or not. This new variable will be the outcome we try to predict using this method, setting the reference level as not North America.

To fit the model, we had to remove `date` and `continent` as predictors, since their formats did not work with the engine.


```r
# Create new binary variable
covid <-covid %>% 
  mutate(NorthAmerica_or_Not = ifelse(continent == 'North America', 'North_America', 'Not_North_America'))

# Set reference level (to the outcome you are NOT interested in)
covid <- covid %>%
  mutate(NorthAmerica_or_Not = relevel(factor(NorthAmerica_or_Not), ref='Not_North_America')) #set reference level

# creation of cv folds
set.seed(123)
covid_cv10 <- vfold_cv(covid, v = 10)

# Logistic LASSO Regression Model Spec
logistic_lasso_spec_tune <- logistic_reg() %>%
    set_engine('glmnet') %>%
    set_args(mixture = 1, penalty = tune()) %>%
    set_mode('classification')

# Recipe
logistic_rec <- recipe(NorthAmerica_or_Not ~ total_cases + new_cases + total_deaths + reproduction_rate + people_vaccinated_per_hundred + people_fully_vaccinated_per_hundred + total_boosters_per_hundred + tests_per_case + icu_patients + positive_rate + life_expectancy + population_density + median_age + gdp_per_capita, data = covid) %>%
    step_normalize(all_numeric_predictors()) %>% 
    step_dummy(all_nominal_predictors())

# Workflow (Recipe + Model)
log_lasso_wf <- workflow() %>% 
    add_recipe(logistic_rec) %>%
    add_model(logistic_lasso_spec_tune) 

# Tune Model (trying a variety of values of Lambda penalty)
penalty_grid <- grid_regular(
  penalty(range = c(-5, 1)), #log10 transformed  (kept moving min down from 0)
  levels = 100)

tune_output <- tune_grid( 
  log_lasso_wf, # workflow
  resamples = covid_cv10, # cv folds
  metrics = metric_set(roc_auc,accuracy),
  control = control_resamples(save_pred = TRUE, event_level = 'second'),
  grid = penalty_grid # penalty grid defined above
)

# Visualize Model Evaluation Metrics from Tuning
autoplot(tune_output) + theme_classic()
```


#### Inspecting the model 

We will choose the final model whose CV AUC is within one standard error of the overall best metric. 


```r
# Select Penalty
best_se_penalty <- select_by_one_std_err(tune_output, metric = 'roc_auc', desc(penalty)) # choose penalty value based on the largest penalty within 1 se of the highest CV roc_auc
best_se_penalty

# Fit Final Model
final_fit_se <- finalize_workflow(log_lasso_wf, best_se_penalty) %>% # incorporates penalty value to workflow 
    fit(data = covid)

final_fit_se %>% tidy() %>%
  filter(estimate == 0)
```
With the set of penalty values that we used, the following variables had their coefficients set to 0:
- `total_cases`
- `people_fully_vaccinated_per_hundred`

Apparently, the total number of cases and number of people fully vaccinated per hundred were not that important in predicting whether or not an observation originated from North America or not, after accounting for the other features in the model.

Here we will look at how long a variable stayed in the model as a measure of variable importance:


```r
glmnet_output <- final_fit_se %>% extract_fit_engine()
    
# Create a boolean matrix (predictors x lambdas) of variable exclusion
bool_predictor_exclude <- glmnet_output$beta==0

# Loop over each variable
var_imp <- sapply(seq_len(nrow(bool_predictor_exclude)), function(row) {
    # Extract coefficient path (sorted from highest to lowest lambda)
    this_coeff_path <- bool_predictor_exclude[row,]
    # Compute and return the # of lambdas until this variable is out forever
    ncol(bool_predictor_exclude) - which.min(this_coeff_path) + 1
})

# Create a dataset of this information and sort
var_imp_data <- tibble(
    var_name = rownames(bool_predictor_exclude),
    var_imp = var_imp
)
var_imp_data %>% arrange(desc(var_imp))
```

Based on the output above, the variables deemed the most important in predicting the continent based on how long they persisted in the LASSO algorithm as the penalty increased. 

Surprisingly, `total_cases` and `people_fully_vaccinated_per_hundred` *are* important!

#### Interpreting evaluation metrics

We inspect the overall CV results for the "best" $\lambda$, and compute the no-information (NIR). 


```r
# CV results for "best lambda"
tune_output %>%
    collect_metrics() %>%
    filter(penalty == best_se_penalty %>% pull(penalty))
           
# Count up the continent outcomes
covid %>% count(NorthAmerica_or_Not) # Name of the outcome variable goes inside count()

# Compute the NIR
2596/(2596+325)
```

The relative frequency of the majority class is 0.89, meaning 89% of our observations originate from outside of North America. 

The AUC for our model is 1.0! Apparently, our model is a perfect classifier of whether or not a case originated from North America!

#### Choosing a threshold and final model

After using LASSO and balancing bias and variance, we use the final model to make predictions: 


```r
# Soft Predictions on Training Data
final_output <- final_fit_se %>% predict(new_data = covid, type='prob') %>% bind_cols(covid)

# FIX THIS
final_output %>%
  ggplot(aes(x = NorthAmerica_or_Not, y = .pred_North_America)) +
  geom_boxplot()

# ROC curve of sensitivity and specificity
final_output %>%
    roc_curve(NorthAmerica_or_Not,.pred_North_America,event_level = 'second') %>%
    autoplot()
```

This ROC curve is 1.0! Apparently, our model is a perfect classifier with sensitivity of 1. 

Sensitivity is the measure of how North-American continents are correctly predicted as North-American by our model. 

Specificity is measure of how often non-North-American continents are correctly predicted to be not North-America by our model. 

We can also calculate the J index:


```r
# thresholds in terms of reference level
threshold_output <- final_output %>%
    threshold_perf(truth = NorthAmerica_or_Not, estimate = .pred_North_America, thresholds = seq(0,1,by=.01)) 

# J-index v. threshold for not_spam
threshold_output %>%
    filter(.metric == 'j_index') %>%
    ggplot(aes(x = .threshold, y = .estimate)) +
    geom_line() +
    labs(y = 'J-index', x = 'threshold') +
    theme_classic()
```


```r
threshold_output %>%
    filter(.metric == 'j_index') %>%
    arrange(desc(.estimate))
```


```r
# Distance v. threshold for not_North_America

threshold_output %>%
    filter(.metric == 'distance') %>%
    ggplot(aes(x = .threshold, y = .estimate)) +
    geom_line() +
    labs(y = 'Distance', x = 'threshold') +
    theme_classic()
```


```r
threshold_output %>%
    filter(.metric == 'distance') %>%
    arrange(.estimate)
```


```r
log_metrics <- metric_set(accuracy,sens,yardstick::spec)

final_output %>%
    mutate(.pred_class = make_two_class_pred(.pred_Not_North_America, levels(NorthAmerica_or_Not), threshold = .98)) %>%
    log_metrics(truth = NorthAmerica_or_Not, estimate = .pred_class, event_level = 'second')
```

95% of COVID case classifications into North America or not are expected to be correct under this model - wow!

Our NIR is 88%, so our model performs better and is actually almost 100% accurate...! However, most of our cases do not actually originate from North America...
# Decision tree
2. 

3. **Reflect on the information gained from these two methods and how you might justify this method to others.**

#### Your Work {-}


```r
# library statements 
library(dplyr)
library(readr)
library(broom)
library(ggplot2)
library(tidymodels)
library(lubridate)
library(gridExtra)
library(vip)
library(ranger)
tidymodels_prefer() # Resolves conflicts, prefers tidymodel functions
conflicted::conflict_prefer("vi", "vip")

# read in data
covid <- read_csv("owid-covid-data.csv")
```

```r
nafun <- function(x) {
  
}
```



```r
# data cleaning
covid <- covid %>%
  filter(date >= as.Date("2021-08-13"))%>%
  select(-icu_patients, -icu_patients_per_million, -hosp_patients, -hosp_patients_per_million, -weekly_icu_admissions, -weekly_icu_admissions_per_million, -weekly_hosp_admissions, -weekly_hosp_admissions_per_million, -total_tests, -total_tests_per_thousand, -new_tests_smoothed, -new_tests_smoothed_per_thousand, -handwashing_facilities, -excess_mortality_cumulative_absolute, -excess_mortality_cumulative, -excess_mortality, -excess_mortality_cumulative_per_million, -male_smokers, -female_smokers, -hospital_beds_per_thousand, -extreme_poverty, -location)

# SHOULD WE REMOVE LOCATION?
```


```r
covid <- covid %>%
  na.omit(covid) %>%
  filter(continent %in% c("North America", "Asia", "South America", "Africa", "Europe", "Oceania")) %>%
  mutate(continent = factor(continent))
```

- We're considering COVID cases occurring after the first recorded booster vaccines (13th August, 2021) to best capture today's landscape.  
- This is when vaccinations started to become widespread (at least in the United States).
- In addition to our existing variables, we also added the following to our model to predict the continent:
    - `life_expectancy`
    - `population_density`
    - `median_age`
    - `gdp_per_capita`
    




```r
# Bagging and Random Forest

# Model Specification
rf_spec <- rand_forest() %>%
  set_engine(engine = 'ranger') %>% 
  set_args(mtry = NULL, # size of random subset of variables; default is floor(sqrt(ncol(x)))
           trees = 1000, # Number of trees
           min_n = 2,
           probability = FALSE, # FALSE: hard predictions
           importance = 'impurity') %>% 
  set_mode('classification') # change this for regression tree

# Recipe

rf_rec <- recipe(continent ~ ., data = covid)

# Workflows
data_wf_mtry2 <- workflow() %>%
  add_model(rf_spec %>% set_args(mtry = 2)) %>%
  add_recipe(rf_rec)

# Create workflows for mtry = 7, 22, and 44
data_wf_mtry7 <- workflow() %>%
  add_model(rf_spec %>% set_args(mtry = 7)) %>%
  add_recipe(rf_rec)

data_wf_mtry22 <- workflow() %>%
  add_model(rf_spec %>% set_args(mtry = 22)) %>%
  add_recipe(rf_rec)

data_wf_mtry44 <- workflow() %>%
  add_model(rf_spec %>% set_args(mtry = 44)) %>%
  add_recipe(rf_rec)
```


```r
# Fit Models

set.seed(123) # make sure to run this before each fit so that you have the same 1000 trees
data_fit_mtry2 <- fit(data_wf_mtry2, data = covid)

set.seed(123)
data_fit_mtry7 <- fit(data_wf_mtry7, data = covid)

set.seed(123) 
data_fit_mtry22 <- fit(data_wf_mtry22, data = covid)

set.seed(123)
data_fit_mtry44 <- fit(data_wf_mtry44, data = covid)
```


```r
# Custom Function to get OOB predictions, true observed outcomes and add a model label
rf_OOB_output <- function(fit_model, model_label, truth){
    tibble(
          .pred_continent = fit_model %>% extract_fit_engine() %>% pluck('predictions'), #OOB predictions
          continent = truth,
          model = model_label
      )
}

#check out the function output
rf_OOB_output(data_fit_mtry2,'mtry2', covid %>% pull(continent))
```


```r
# Evaluate OOB Metrics

data_rf_OOB_output <- bind_rows(
    rf_OOB_output(data_fit_mtry2,'mtry2', covid %>% pull(continent)),
    rf_OOB_output(data_fit_mtry7,'mtry7', covid %>% pull(continent)),
    rf_OOB_output(data_fit_mtry22,'mtry22', covid %>% pull(continent)),
    rf_OOB_output(data_fit_mtry44,'mtry44', covid %>% pull(continent))
)


data_rf_OOB_output %>% 
    group_by(model) %>%
    accuracy(truth = continent, estimate = .pred_continent)
```

```r
# Preliminary Interpretation

data_rf_OOB_output %>% 
    group_by(model) %>%
    accuracy(truth = continent, estimate = .pred_continent) %>%
  mutate(mtry = as.numeric(stringr::str_replace(model,'mtry',''))) %>%
  ggplot(aes(x = mtry, y = .estimate )) + 
  geom_point() +
  geom_line() +
  theme_classic()
```


```r
# Evaluating the forest

data_fit_mtry7
```


```r
rf_OOB_output(data_fit_mtry7,'mtry7', covid %>% pull(continent)) %>%
    conf_mat(truth = continent, estimate= .pred_continent)
```


```r
# Variable importance measures

data_fit_mtry7 %>% 
    extract_fit_engine() %>% 
    vip(num_features = 30) + theme_classic() #based on impurity
```



```r
data_wf_mtry7 %>% 
  update_model(rf_spec %>% set_args(importance = "permutation")) %>% #based on permutation
  fit(data = covid) %>% 
    extract_fit_engine() %>% 
    vip(num_features = 30) + theme_classic()
```


```r
## pick important variables to visualize

ggplot(covid, aes(x = continent, y = aged_70_older)) +
    geom_violin() + theme_classic()

ggplot(covid, aes(x = continent, y = total_vaccinations_per_hundred)) +
    geom_violin() + theme_classic()

ggplot(covid, aes(x = continent, y = total_deaths_per_million)) +
    geom_violin() + theme_classic()

ggplot(covid, aes(x = continent, y = total_cases)) +
    geom_violin() + theme_classic()

ggplot(covid, aes(x = continent, y = new_deaths_smoothed)) +
    geom_violin() + theme_classic()
```



## Classification - Methods

1. **Indicate at least 2 different methods used to answer your classification research question.**

Used logistic regression and random forests.

2. **Describe what you did to evaluate the models explored.**
3. **Indicate how you estimated quantitative evaluation metrics.**

For the logistic regression, we inspected the overall cross-validation results to determine the best lambda. We also determined the no-information-rate (NIR), which was 89%, meaning in this case that 89% of our cases represented "false (non-North America)" values. We also used the ROC curve to plot sensitivity (rate of true positive, predicting North American cases and North American) and specificity (true negative rate, predicting non-North-American cases as non-North American) of the overall model. It returned an ROC of 1 with a stardard deviation of 0, meaning it has essentially perfect prediction. Finally, we evaluated the model accuracy measure from our logistic quantitative output, which equaled .95, with a one standard deviation estimate of +-0.00137, suggesting strong 95% predictive accuracy. 

For the decision tree, we used out-of-bag error (OOB) and the test confusion matrix. Our OOB returned as .02%, meaning the our model makes the wrong hard classification .02% of the time, which in our test confusion matrix is shown to be one wrong prediction. This suggests that the tree model has a 99.98% accuracy, making it slightly more accurate than the logistic model. However, this difference is within 5% accuracy, suggesting that either model could offer high predictive power in this research context. 


4. **Describe the goals / purpose of the methods used in the overall context of your research investigations.**

These methods were employed to predict the origin continent of each case.

## Classification - Results

1. **Summarize your final model and justify your model choice (see below for ways to justify your choice)**.
2. **Compare the different classification models tried in light of evaluation metrics, variable importance, and data context.**


These models also differ by metrics and variable importance. In the logistic regression, total cases, people vaccinated per hundred, and ICU patients proved to be among the most important variables. This is in contrast to the random forest model, which finds population age to generally be the most important predictor of new COVID cases.

We have selected the random forest model. This operates as a bagged decision tree, and can predict which continent each case is from. We selected this model for two reasons. First, and most importantly, this model can predict each content specifically, whereas the logistic model is restricted to a binary outcome of North America/Not North America. This a critical difference, as it allows for much more specific predictions within this data context. Secondly, this model has a slightly higher accuracy rating of 4.98%.





3. **Display evaluation metrics for different models in a clean, organized way. This display should include both the estimated metric as well as its standard deviation. (This won???t be available from OOB error estimation. If using OOB, don???t worry about reporting the SD.)**

all are listed above except standard deviation (ASK BRYAN DO NOT LEAVE THIS WHEN SUBMITTING)
4. **Broadly summarize conclusions from looking at these evaluation metrics and their measures of uncertainty.**

Overall the random forest model produces a more accurate prediction with fewer errors, and (ADD STANDARD DEVIATION STUFF HERE). The variables used are also different, focusing more on population demographic data than COVID data specifically. Finally, within the study context, this model also makes more sense due to the specificity of outcome it can offer via multiple-continent outcomes.  

## Classification - Conclusions

1. **Interpret evaluation metric(s) for the final model in context. Does the model show an acceptable amount of error?**

The model has an extremely low level of error, predicting results with 99.98% accuracy. This is far above any degree of predictive ability that could be expected, and is certainly an acceptable amount of error, especially considering the relatively low-stakes context of this study. 

2. **If using OOB error estimation, display the test (OOB) confusion matrix, and use it to interpret the strengths and weaknesses of the final model.**


```r
rf_OOB_output(data_fit_mtry7,'mtry7', covid %>% pull(continent)) %>%
    conf_mat(truth = continent, estimate= .pred_continent)
```

Here, we see that only one case has been incorrectly predicted in the training data, demonstrating the extreme predictive ability of this random forest model. 
3. **Summarization should show evidence of acknowledging the data context in thinking about the sensibility of these results.**

Overall, this model predicts with almost flawless accuracy the continent that each case of COVID data was collected in. In the context of this data, as well as the type of work which could draw on these findings, this model has successfully achieved its desired outcome. For medical professionals hoping to understand explanatory factors of COVID spread, or governments seeking to determine regional variation in explanatory factors, this model can provide direction with confidence.

# K-means Clustering

In the code below, we construct a dendrogram using the quantitative variables in our dataset to predict whether or not a case originates from North America. This approach is a visual way to explore clustering and how clusters form from grouping on the quantitative characteristics of the countries where cases were recorded, e.g. the tests per case. If the clustering didn't 


```r
# Random subsample of 50 cases
set.seed(123)

covid <- covid %>%
  slice_sample(n = 100)

# select the variables to be used in clustering

covid_sub <- covid %>% 
  select(total_cases, people_fully_vaccinated_per_hundred)
```

### Picking k

The **Elbow Method** is based on the visualization below. We pick the $k$ that corresponds to the bend in the curve (the "elbow") such that you are minimizing total within-cluster sum of squares while keeping the number of clusters small (simpler, more interpretable).

This enables us to pick $k$ using a data-driven approach. We would be comparing the **total squared distance of each case from its assigned centroid** for different values of $k$. (This measure is available within the `$tot.withinss` component of objects resulting from `kmeans()`.)

Based on the visualization below, we pick $k = 6$. After this, there is not as much meaningful decrease in heterogeneity (dissimilarity).

This makes a lot of sense because there are 6 continents where cases are observed in the data set.

```r
# Data-specific function to cluster and calculate total within-cluster SS
covid_cluster_ss <- function(k){
    # Perform clustering
    kclust <- kmeans(scale(covid_sub), centers = k)

    # Return the total within-cluster sum of squares
    return(kclust$tot.withinss)
}

tibble(
    k = 1:15,
    tot_wc_ss = purrr::map_dbl(1:15, covid_cluster_ss)
) %>% 
    ggplot(aes(x = k, y = tot_wc_ss)) +
    geom_point() + 
    labs(x = "Number of clusters",y = 'Total within-cluster sum of squares') + 
    theme_classic()
```

### Cluster interpretation


```r
# Perform clustering: should you use scale()?
set.seed(123)
kclust_k6_2vars <- kmeans(scale(covid_sub), centers = 6)

covid <- covid %>%
    mutate(kclust_6_2vars = factor(kclust_k6_2vars$cluster))


covid %>%
  count(continent,kclust_6_2vars)

covid %>%
    group_by(kclust_6_2vars) %>%
    summarize(across(c(total_cases, people_fully_vaccinated_per_hundred), mean))
```

Group 3 has the highest mean total cases and the second lowest mean total vaccination rate. Group 5 could be named low case rate and low vaccination rate as it has the lowest number of total cases and the lowest rate of people fully vaccinated. Group 6 has the highest number of people fully vaccinated per hundred but also has the third lowest total cases which is surprising to us but this is indicative that other factors can affect clustering. 

### Clustering Visualization


```r
# Run k-means for k = centers = 6
kclust_k6 <- kmeans(scale(covid_sub), centers = 6)

# Add a variable (kclust_k3) to the original dataset 
# containing the cluster assignments
covid_sub <- covid_sub %>%
    mutate(kclust_6 = factor(kclust_k6$cluster))

# Visualize the cluster assignments on the original scatterplot
ggplot(covid_sub, aes(x = total_cases, y = people_fully_vaccinated_per_hundred, color = kclust_6)) +
    geom_point() + 
    theme_classic()
```

### Clustering Evalution

- The two features we chose in the clustering were `total_cases` and `people_fully_vaccinated_per_hundred`. We chose these variables because the variable importance calculation from the LASSO logistic regression done previously picked these two variables as the most important. Aditionally, the clustering can only be done for quantitative variables since we are able to measure the distance between them.
- It looks like there are 6 clusters with varying vaccination rate per hundred people and total number of cases.

## Homework Prompts

1. **Indicate at least 2 different methods used to answer your classification research question.**
2. **Describe what you did to evaluate the models explored.**
3. **Indicate how you estimated quantitative evaluation metrics.**
4. **Describe the goals / purpose of the methods used in the overall context of your research investigations.**

## Classification - Results

1. **Summarize your final model and justify your model choice (see below for ways to justify your choice)**.
2. **Compare the different classification models tried in light of evaluation metrics, variable importance, and data context.**
3. **Display evaluation metrics for different models in a clean, organized way. This display should include both the estimated metric as well as its standard deviation. (This won???t be available from OOB error estimation. If using OOB, don???t worry about reporting the SD.)**
4. **Broadly summarize conclusions from looking at these evaluation metrics and their measures of uncertainty.**

## Classification - Conclusions

1. **Interpret evaluation metric(s) for the final model in context. Does the model show an acceptable amount of error?**
2. **If using OOB error estimation, display the test (OOB) confusion matrix, and use it to interpret the strengths and weaknesses of the final model.**
3. **Summarization should show evidence of acknowledging the data context in thinking about the sensibility of these results.**

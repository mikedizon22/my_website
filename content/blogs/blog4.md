---
categories:
- ""
- ""
date: "2017-10-31T22:42:51-05:00"
description: ""
draft: false
image: machinelearning.jpg
keywords: ""
slug: hw4
title: Machine Learning
---

```{r}
#| label: load-libraries
#| echo: false # This option disables the printing of code (only output is displayed).
#| message: false
#| warning: false
options(scipen = 999) #disable scientific notation
library(tidyverse)
library(tidymodels)
library(GGally)
library(sf)
library(leaflet)
library(janitor)
library(rpart.plot)
library(here)
library(scales)
library(vip)
```


# The Bechdel Test

```{r read_data}

bechdel <- read_csv(here::here("data", "bechdel.csv")) %>% 
  mutate(test = factor(test)) 
glimpse(bechdel)

```
How many films fail/pass the test, both as a number and as a %?

```{r}

#Compute for the number and % of films that fail/pass the test and store into a variable
summary_data <- bechdel %>%
  
#Count the number of films that pass or fail the test
  summarize(
    pass_count = sum(test == "Pass"),
    fail_count = sum(test == "Fail"),
  ) %>% 
  
#Add a new column to determine the proportion
  mutate(
    pass_percentage = pass_count / nrow(bechdel),
    fail_percentage = fail_count / nrow(bechdel)
    )

```

**Answer:** There were 622 (45%) films that passed and 772 (55%) films that failed.

## Movie scores
```{r}
ggplot(data = bechdel, aes(
  x = metascore,
  y = imdb_rating,
  colour = test
)) +
  geom_point(alpha = .3, size = 3) +
  scale_colour_manual(values = c("tomato", "olivedrab")) +
  labs(
    x = "Metacritic score",
    y = "IMDB rating",
    colour = "Bechdel test"
  ) +
 theme_light()
```


# Split the data
```{r}
# **Split the data**

set.seed(123)

data_split <- initial_split(bechdel, # updated data
                           prop = 0.8, 
                           strata = test)

bechdel_train <- training(data_split) 
bechdel_test <- testing(data_split)
```

Check the counts and % (proportions) of the `test` variable in each set.
```{r}

#Calculate the number of films that pass/fail using the training data and store into a variable
train_summary <- bechdel_train %>% 
  
#Count the number of films that pass/fail the test
  summarize(
    pass_count = sum(test == "Pass"),
    fail_count = sum(test == "Fail")
  ) %>% 
  
#Add a new column to compute for the proportion
  mutate(
    pass_percentage = pass_count / nrow(bechdel_train),
    fail_percentage = fail_count / nrow(bechdel_train)
    )

#Calculate the number of films that pass/fail using the testing data and store into a variable
test_summary <- bechdel_test %>% 
  
#Count the number of films that pass/fail the test
  summarize(
    pass_count = sum(test == "Pass"),
    fail_count = sum(test == "Fail")
  ) %>% 
  
#Add a new column to compute for the proportion
  mutate(
    pass_percentage = pass_count / nrow(bechdel_test),
    fail_percentage = fail_count / nrow(bechdel_test)
  )

```

**Answer:** In the training data, there were 497 (45%) films that passed and 617 (55%) films that failed. On the other hand, in the test data, there were 125 (45%) films that passed and 155 (55%) films that failed.

## Feature exploration

## Any outliers? 

```{r}

bechdel %>% 
  select(test, budget_2013, domgross_2013, intgross_2013, imdb_rating, metascore) %>% 

    pivot_longer(cols = 2:6,
               names_to = "feature",
               values_to = "value") %>% 
  ggplot()+
  aes(x=test, y = value, fill = test)+
  coord_flip()+
  geom_boxplot()+
  facet_wrap(~feature, scales = "free")+
  theme_bw()+
  theme(legend.position = "none")+
  labs(x=NULL,y = NULL)

```

**Answer:** The outlier for budget_2013 was Avatar with a budget of 46.14, which failed the Bechdel test. The outlier for domgross_2013 was Jaws with 112.53, which failed the Bechdel test. Another outlier for this category was The Exorcist with 107.43, which passed the Bechdel test. The clear outlier for the imdb_rating was the Son of the Mask with 2.1, which failed the Bechdel test. For intgross_2013, Titanic was the outlier for pass with 317.19 while Avatar was the outlier for fail with 302.26. Lastly, for the only outlier for metascore was Mortal Kombat:Annihilation with 11, which passed the Bechdel test.

## Scatterplot - Correlation Matrix

Write a paragraph discussing the output of the following 
```{r, warning=FALSE, message=FALSE}
bechdel %>% 
  select(test, budget_2013, domgross_2013, intgross_2013, imdb_rating, metascore)%>% 
  ggpairs(aes(colour=test), alpha=0.2)+
  theme_bw()
```

**Answer:** From the ggpairs output, we can observe that the distribution of budget_2013, domgross_2013, and int_gross2013 are skewed to the right, which means that there are more instances wherein these amounts are less than the average values. This probably means that a film typically uses less budget for the production of the film, but also tends to take in less international and domestic gross revenues, as also confirmed by the boxplots. However, both the distributions and boxplots also show instances where a film is highly profitable and well-funded in terms of budget. On the other hand, the imdb rating is skewed to the left and the metascore is more or less normally distributed, as also shown by the boxplots. From the pairwise correlations and scatterplots, we can observe that there is a strong correlation between int_gross2013 and dom_gross2013 as well as metascore and imdb_rating.  

## Categorical variables

Write a paragraph discussing the output of the following 
```{r}
bechdel %>% 
  group_by(genre, test) %>%
  summarise(n = n()) %>% 
  mutate(prop = n/sum(n))
  
 
bechdel %>% 
  group_by(rated, test) %>%
  summarise(n = n()) %>% 
  mutate(prop = n/sum(n))
```

**Answer:** With the exception of the one film under the musical genre, the table shows that for all genres, there is a higher proportion of films that fail the Bechdel test than those that pass. The same conclusion can also be observed when the films are grouped by rating - there is a higher proportion of films that fail the Bechdel test than those that pass.

# Train first models. `test ~ metascore + imdb_rating`

```{r}
lr_mod <- logistic_reg() %>% 
  set_engine(engine = "glm") %>% 
  set_mode("classification")

lr_mod


tree_mod <- decision_tree() %>% 
  set_engine(engine = "C5.0") %>% 
  set_mode("classification")

tree_mod 
```

```{r}


lr_fit <- lr_mod %>% # parsnip model
  fit(test ~ metascore + imdb_rating, # a formula
    data = bechdel_train # dataframe
  )

tree_fit <- tree_mod %>% # parsnip model
  fit(test ~ metascore + imdb_rating, # a formula
    data = bechdel_train # dataframe
  )
```

## Logistic regression

```{r}
lr_fit %>%
  broom::tidy()

lr_preds <- lr_fit %>%
  augment(new_data = bechdel_train) %>%
  mutate(.pred_match = if_else(test == .pred_class, 1, 0))

```

### Confusion matrix

```{r}
lr_preds %>% 
  conf_mat(truth = test, estimate = .pred_class) %>% 
  autoplot(type = "heatmap")


```


## Decision Tree
```{r}
tree_preds <- tree_fit %>%
  augment(new_data = bechdel) %>%
  mutate(.pred_match = if_else(test == .pred_class, 1, 0)) 


```

```{r}
tree_preds %>% 
  conf_mat(truth = test, estimate = .pred_class) %>% 
  autoplot(type = "heatmap")
```

## Draw the decision tree

```{r}
draw_tree <- 
    rpart::rpart(
        test ~ metascore + imdb_rating,
        data = bechdel_train, # uses data that contains both birth weight and `low`
        control = rpart::rpart.control(maxdepth = 5, cp = 0, minsplit = 10)
    ) %>% 
    partykit::as.party()
plot(draw_tree)

```

# Cross Validation

Run the code below. What does it return?

```{r}
set.seed(123)
bechdel_folds <- vfold_cv(data = bechdel_train, 
                          v = 10, 
                          strata = test)
bechdel_folds
```

**Answer:** The code returns a set of ten stratified folds for cross-validation purposes. The vfold_cv function randomly splits the training data in ten groups of roughly equal size.

## `fit_resamples()`

Trains and tests a resampled model.

```{r}
lr_fit <- lr_mod %>%
  fit_resamples(
    test ~ metascore + imdb_rating,
    resamples = bechdel_folds
  )


tree_fit <- tree_mod %>%
  fit_resamples(
    test ~ metascore + imdb_rating,
    resamples = bechdel_folds
  )
```


## `collect_metrics()`

Unnest the metrics column from a tidymodels `fit_resamples()`
```{r}

collect_metrics(lr_fit)
collect_metrics(tree_fit)


```


```{r}
tree_preds <- tree_mod %>% 
  fit_resamples(
    test ~ metascore + imdb_rating, 
    resamples = bechdel_folds,
    control = control_resamples(save_pred = TRUE) #<<
  )

# What does the data for ROC look like?
tree_preds %>% 
  collect_predictions() %>% 
  roc_curve(truth = test, .pred_Fail)  

# Draw the ROC
tree_preds %>% 
  collect_predictions() %>% 
  roc_curve(truth = test, .pred_Fail) %>% 
  autoplot()

```


# Build a better training set with `recipes`

## Preprocessing options

- Encode categorical predictors
- Center and scale variables
- Handle class imbalance
- Impute missing data
- Perform dimensionality reduction 
- ... ...

## To build a recipe

1. Start the `recipe()`
1. Define the variables involved
1. Describe **prep**rocessing [step-by-step]

## Collapse Some Categorical Levels

Do we have any `genre` with few observations?  Assign genres that have less than 3% to a new category 'Other'


```{r}
#| echo = FALSE
bechdel %>% 
  count(genre) %>% 
  mutate(genre = fct_reorder(genre, n)) %>% 
  ggplot(aes(x = genre, 
             y = n)) +
  geom_col(alpha = .8) +
  coord_flip() +
  labs(x = NULL) +
  geom_hline(yintercept = (nrow(bechdel_train)*.03), lty = 3)+
  theme_light()
```


```{r}
movie_rec <-
  recipe(test ~ .,
         data = bechdel_train) %>%
  
  # Genres with less than 5% will be in a catewgory 'Other'
    step_other(genre, threshold = .03) 
```
  

## Before recipe

```{r}
#| echo = FALSE
bechdel_train %>% 
  count(genre, sort = TRUE)
```


## After recipe

```{r}
movie_rec %>% 
  prep() %>% 
  bake(new_data = bechdel_train) %>% 
  count(genre, sort = TRUE)
```

## `step_dummy()`

Converts nominal data into numeric dummy variables

```{r}
#| results = "hide"
movie_rec <- recipe(test ~ ., data = bechdel) %>%
  step_other(genre, threshold = .03) %>% 
  step_dummy(all_nominal_predictors()) 

movie_rec 
```

## Let's think about the modelling 

What if there were no films with `rated` NC-17 in the training data?

-  Will the model have a coefficient for `rated` NC-17?
-  What will happen if the test data includes a film with `rated` NC-17?

**Answer:** The model will not have a coefficient for `rated` NC-17 as the model did not encounter any examples of films with that rating in the training data. If the test data includes films with that rating, the model will still make predictions using other available features although it may not be as accurate as other films with other ratings.

## `step_novel()`

Adds a catch-all level to a factor for any new values not encountered in model training, which lets R intelligently predict new levels in the test set.

```{r}

movie_rec <- recipe(test ~ ., data = bechdel) %>%
  step_other(genre, threshold = .03) %>% 
  step_novel(all_nominal_predictors) %>% # Use *before* `step_dummy()` so new level is dummified
  step_dummy(all_nominal_predictors()) 

```


## `step_zv()`

Intelligently handles zero variance variables (variables that contain only a single value)

```{r}
movie_rec <- recipe(test ~ ., data = bechdel) %>%
  step_other(genre, threshold = .03) %>% 
  step_novel(all_nominal(), -all_outcomes()) %>% # Use *before* `step_dummy()` so new level is dummified
  step_dummy(all_nominal(), -all_outcomes()) %>% 
  step_zv(all_numeric(), -all_outcomes()) 
  
```


## `step_normalize()`

Centers then scales numeric variable (mean = 0, sd = 1)

```{r}
movie_rec <- recipe(test ~ ., data = bechdel) %>%
  step_other(genre, threshold = .03) %>% 
  step_novel(all_nominal(), -all_outcomes()) %>% # Use *before* `step_dummy()` so new level is dummified
  step_dummy(all_nominal(), -all_outcomes()) %>% 
  step_zv(all_numeric(), -all_outcomes())  %>% 
  step_normalize(all_numeric()) 

```


## `step_corr()`

Removes highly correlated variables

```{r}
movie_rec <- recipe(test ~ ., data = bechdel) %>%
  step_other(genre, threshold = .03) %>% 
  step_novel(all_nominal(), -all_outcomes()) %>% # Use *before* `step_dummy()` so new level is dummified
  step_dummy(all_nominal(), -all_outcomes()) %>% 
  step_zv(all_numeric(), -all_outcomes())  %>% 
  step_normalize(all_numeric())



movie_rec
```


# Define different models to fit

```{r}
## Model Building

# 1. Pick a `model type`
# 2. set the `engine`
# 3. Set the `mode`: regression or classification

# Logistic regression
log_spec <-  logistic_reg() %>%  # model type
  set_engine(engine = "glm") %>%  # model engine
  set_mode("classification") # model mode

# Show your model specification
log_spec

# Decision Tree
tree_spec <- decision_tree() %>%
  set_engine(engine = "C5.0") %>%
  set_mode("classification")

tree_spec

# Random Forest
library(ranger)

rf_spec <- 
  rand_forest() %>% 
  set_engine("ranger", importance = "impurity") %>% 
  set_mode("classification")


# Boosted tree (XGBoost)
library(xgboost)

xgb_spec <- 
  boost_tree() %>% 
  set_engine("xgboost") %>% 
  set_mode("classification") 

# K-nearest neighbour (k-NN)
knn_spec <- 
  nearest_neighbor(neighbors = 4) %>% # we can adjust the number of neighbors 
  set_engine("kknn") %>% 
  set_mode("classification") 
```


# Bundle recipe and model with `workflows`


```{r}
log_wflow <- # new workflow object
 workflow() %>% # use workflow function
 add_recipe(movie_rec) %>%   # use the new recipe
 add_model(log_spec)   # add your model spec

# show object
log_wflow


## A few more workflows

tree_wflow <-
 workflow() %>%
 add_recipe(movie_rec) %>% 
 add_model(tree_spec) 

rf_wflow <-
 workflow() %>%
 add_recipe(movie_rec) %>% 
 add_model(rf_spec) 

xgb_wflow <-
 workflow() %>%
 add_recipe(movie_rec) %>% 
 add_model(xgb_spec)

knn_wflow <-
 workflow() %>%
 add_recipe(movie_rec) %>% 
 add_model(knn_spec)

```

HEADS UP

1. How many models have you specified?

**Answer:** There were five models that were specified namely Logistic Regression, Decision Tree, Random Forest, Boosted Tree (XGBoost), and K-Nearest Neighbour (k-NN).

2. What's the difference between a model specification and a workflow?

**Answer:** A model specification provides the definition of details and configurations of a specific machine learning algorithm, particularly the type of model to be used (such as the ones mentioned above) and sets parameters specific to the model. On the other hand, a workflow is more comprehensive as it encompasses the entire modelling process, such as data preprocessing, model specification, model fitting, and model evaluation.

3. Do you need to add a formula (e.g., `test ~ .`)  if you have a recipe?

It is not necessary to add a formula if you have a recipe as it already defines the data preprocessing steps, including variable selection, transformation, and encoding.

# Model Comparison

You now have all your models. Adapt the code from slides `code-from-slides-CA-housing.R`, line 400 onwards to assess which model gives you the best classification. 


```{r}

log_res <- log_wflow %>% 
  fit_resamples(
    resamples = bechdel_folds, 
    metrics = metric_set(
      recall, precision, f_meas, accuracy,
      kap, roc_auc, sens, spec),
    control = control_resamples(save_pred = TRUE)) 

# Show average performance over all folds (note that we use log_res):
log_res %>%  collect_metrics(summarize = TRUE)

# Show performance for every single fold:
log_res %>%  collect_metrics(summarize = FALSE)



## `collect_predictions()` and get confusion matrix{.smaller}

log_pred <- log_res %>% collect_predictions()

log_pred %>%  conf_mat(test, .pred_class) 

log_pred %>% 
  conf_mat(test, .pred_class) %>% 
  autoplot(type = "mosaic") +
  geom_label(aes(
      x = (xmax + xmin) / 2, 
      y = (ymax + ymin) / 2, 
      label = c("TP", "FN", "FP", "TN")))


log_pred %>% 
  conf_mat(test, .pred_class) %>% 
  autoplot(type = "heatmap")


## ROC Curve

log_pred %>% 
  group_by(id) %>% # id contains our folds
  roc_curve(test, .pred_Pass) %>% 
  autoplot()


## Decision Tree results

tree_res <-
  tree_wflow %>% 
  fit_resamples(
    resamples = bechdel_folds, 
    metrics = metric_set(
      recall, precision, f_meas, 
      accuracy, kap,
      roc_auc, sens, spec),
    control = control_resamples(save_pred = TRUE)
    ) 

tree_res %>%  collect_metrics(summarize = TRUE)


## Random Forest

rf_res <-
  rf_wflow %>% 
  fit_resamples(
    resamples = bechdel_folds, 
    metrics = metric_set(
      recall, precision, f_meas, 
      accuracy, kap,
      roc_auc, sens, spec),
    control = control_resamples(save_pred = TRUE)
    ) 

rf_res %>%  collect_metrics(summarize = TRUE)

## Boosted tree - XGBoost

xgb_res <- 
  xgb_wflow %>% 
  fit_resamples(
    resamples = bechdel_folds, 
    metrics = metric_set(
      recall, precision, f_meas, 
      accuracy, kap,
      roc_auc, sens, spec),
    control = control_resamples(save_pred = TRUE)
    ) 

xgb_res %>% collect_metrics(summarize = TRUE)

## K-nearest neighbour

knn_res <- 
  knn_wflow %>% 
  fit_resamples(
    resamples = bechdel_folds, 
    metrics = metric_set(
      recall, precision, f_meas, 
      accuracy, kap,
      roc_auc, sens, spec),
    control = control_resamples(save_pred = TRUE)
    ) 

knn_res %>% collect_metrics(summarize = TRUE)


## Model Comparison

log_metrics <- 
  log_res %>% 
  collect_metrics(summarise = TRUE) %>%
  # add the name of the model to every row
  mutate(model = "Logistic Regression") 

tree_metrics <- 
  tree_res %>% 
  collect_metrics(summarise = TRUE) %>%
  mutate(model = "Decision Tree")

rf_metrics <- 
  rf_res %>% 
  collect_metrics(summarise = TRUE) %>%
  mutate(model = "Random Forest")

xgb_metrics <- 
  xgb_res %>% 
  collect_metrics(summarise = TRUE) %>%
  mutate(model = "XGBoost")

knn_metrics <- 
  knn_res %>% 
  collect_metrics(summarise = TRUE) %>%
  mutate(model = "Knn")

# create dataframe with all models
model_compare <- bind_rows(log_metrics,
                           tree_metrics,
                           rf_metrics,
                           xgb_metrics,
                           knn_metrics) 

#Pivot wider to create barplot
  model_comp <- model_compare %>% 
  select(model, .metric, mean, std_err) %>% 
  pivot_wider(names_from = .metric, values_from = c(mean, std_err)) 

# show mean are under the curve (ROC-AUC) for every model
model_comp %>% 
  arrange(mean_roc_auc) %>% 
  mutate(model = fct_reorder(model, mean_roc_auc)) %>% # order results
  ggplot(aes(model, mean_roc_auc, fill=model)) +
  geom_col() +
  coord_flip() +
  scale_fill_brewer(palette = "Blues") +
   geom_text(
     size = 3,
     aes(label = round(mean_roc_auc, 2), 
         y = mean_roc_auc + 0.08),
     vjust = 1
  )+
  theme_light()+
  theme(legend.position = "none")+
  labs(y = NULL)

## `last_fit()` on test set

# - `last_fit()`  fits a model to the whole training data and evaluates it on the test set. 
# - provide the workflow object of the best model as well as the data split object (not the training data). 
 
last_fit_xgb <- last_fit(xgb_wflow, 
                        split = data_split,
                        metrics = metric_set(
                          accuracy, f_meas, kap, precision,
                          recall, roc_auc, sens, spec))

last_fit_xgb %>% collect_metrics(summarize = TRUE)

#Compare to training
xgb_res %>% collect_metrics(summarize = TRUE)


## Variable importance using `{vip}` package

library(vip)

last_fit_xgb %>% 
  pluck(".workflow", 1) %>%   
  pull_workflow_fit() %>% 
  vip(num_features = 10) +
  theme_light()


## Final Confusion Matrix

last_fit_xgb %>%
  collect_predictions() %>% 
  conf_mat(test, .pred_class) %>% 
  autoplot(type = "heatmap")


## Final ROC curve
last_fit_xgb %>% 
  collect_predictions() %>% 
  roc_curve(test, .pred_Pass) %>% 
  autoplot()

```

**Answer:** The random forest model provided the best classification with a mean_roc_auc of 0.66. The logistic regression model provided the worst classification with a mean_roc_auc of 0.47.
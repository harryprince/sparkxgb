
<!-- README.md is generated from README.Rmd. Please edit that file -->

# sparkxgb

**sparkxgb** is a [sparklyr](https://spark.rstudio.com/) extension that
provides an interface to [XGBoost](https://github.com/dmlc/xgboost) on
Spark.

## Installation

You can install the development version of sparkxgb with:

``` r
# sparkxgb requires the development version of sparklyr
devtools::install_github("rstudio/sparklyr")
devtools::install_github("kevinykuo/sparkxgb")
```

## Example

**sparkxgb** supports the familiar formula interface for specifying
models:

``` r
library(sparkxgb)
library(sparklyr)
library(dplyr)

sc <- spark_connect(master = "local")
iris_tbl <- sdf_copy_to(sc, iris)

xgb_model <- xgboost_classifier(
  iris_tbl, 
  Species ~ .,
  objective = "multi:softprob",
  num_class = 3,
  num_round = 50, 
  max_depth = 4
)

xgb_model %>%
  ml_predict(iris_tbl) %>%
  select(Species, predicted_label, starts_with("probability_")) %>%
  glimpse()
#> Observations: ??
#> Variables: 5
#> $ Species                <chr> "setosa", "setosa", "setosa", "setosa",...
#> $ predicted_label        <chr> "setosa", "setosa", "setosa", "setosa",...
#> $ probability_versicolor <dbl> 0.003566429, 0.003564076, 0.003566429, ...
#> $ probability_virginica  <dbl> 0.001423170, 0.002082058, 0.001423170, ...
#> $ probability_setosa     <dbl> 0.9950104, 0.9943539, 0.9950104, 0.9950...
```

It also provides a Pipelines API, which means you can use a
`xgboost_classifier` or `xgboost_regressor` in a pipeline as any
`Estimator`, and do things like hyperparameter tuning:

``` r
pipeline <- ml_pipeline(sc) %>%
  ft_r_formula(Species ~ .) %>%
  xgboost_classifier(
    objective = "multi:softprob",
    num_class = 3
  )

param_grid <- list(
  xgboost = list(
    max_depth = c(1, 5),
    num_round = c(10, 50)
  )
)

cv <- ml_cross_validator(
  sc,
  estimator = pipeline,
  evaluator = ml_multiclass_classification_evaluator(
    sc, 
    label_col = "label",
    raw_prediction_col = "rawPrediction"
  ),
  estimator_param_maps = param_grid
)

cv_model <- cv %>%
  ml_fit(iris_tbl)

summary(cv_model)
#> Summary for CrossValidatorModel 
#>             <cross_validator_eb6a75829229> 
#> 
#> Tuned Pipeline
#>   with metric f1
#>   over 4 hyperparameter sets 
#>   via 3-fold cross validation
#> 
#> Estimator: Pipeline
#>            <pipeline_eb6af490ce5> 
#> Evaluator: MulticlassClassificationEvaluator
#>            <multiclass_classification_evaluator_eb6a2cd582dd> 
#> 
#> Results Summary: 
#>          f1 num_round_1 max_depth_1
#> 1 0.9549670          10           1
#> 2 0.9674460          10           5
#> 3 0.9488665          50           1
#> 4 0.9613854          50           5
```

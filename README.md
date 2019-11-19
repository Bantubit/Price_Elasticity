# Price_Elasticity_R
Calculating regular and cross price elasticity for products based on weekly sales data
  

## DATA REQUIREMENTS
You can find a sample dataset in this repository, based on a real sales data of offline sales of several products for a small company. Similar to typical sales for FMCG market the sales is reported on weekly level, both in value and volume.
Sales can be expressed in volume, value or units. Choosing the metric depends mostly on the ease of interpretation and business goals.  
To simplify - as with marketing mix modelling the more variables describing sales you will add to the regression model, the more valid your general results will be.    
  
## OVERVIEW
The main aim of the pricing studies is to learn about past impact of the price on sales and try to adjust future actions. Price elasticity is one of the basic metrics to achieve this and since you are here I guess you most likely are well aware of it.  
  
But just to be on the same page on what exactly are we talking about: price elasticity is the measurement of price impact on sales. Simply speaking it is a value describing by how many percent your sales will change if you will increase your price by 1%. Price elasticity of -2 means that by increasing the price by one percent we will loose two percent in sales.  
For our regression case it will be calculated using following formula formula:
#### PE = (ΔQ/ΔP) * (P/Q)  
Where (ΔQ/ΔP) is determined by regression coefficient and P and Q will be our mean Price and mean Sales.  
    
  
In full pricing studies (ex. Nielsen one) we can deliver many more insights than just this one metric.  
We can go through descriptive pricing overview (trends and architecture for each products), own and cross price elasticities and regular vs promo price elasticities. You will find them calculated or described below.  
  

### When to calculate it - selecting right products
Besides the regression significance output for a given variable we could in advance try to predict if given product is a good candidate for such study. There are two main constraints. 
It should have a stable distribution over time, at least ~30 weeks of constant sales. Otherwise we just don't have enough observations.
It should also have some variance in price over time. If the price was stable over past two years we won't be able to see any impact of changes on sales. There was simply no changes.

### Limitations
Due to limitations of regressions itself, price elasticity should not be treated in my opinion as a metric used to calculate exact price point for a given product or used for sales simulations. In most cases, as you will see in the results below, the regression coefficients values are just too unstable to treat this output as a sacred truth.  
With the right model validation and interpretation it could serve however as a good way to understand price sensitivity of our customers and identify products that potentially could benefit from price increase or decrease.  
  
More on limitations and things to keep in mind you can find here:  
http://www.cornerstonecapabilities.com/what-you-must-know-about-pricing-elasticity-and-the-danger-it-may-bring/
  
  
## DESCRIPTIVE ANALYSIS 
Code for all of the visualizations can be found in visualize_results.R  
  
### Price trend
This simple graph shows us price history across different products in a given time. We can easily see different price levels grouped across products, bigger discounts or regular price changes.  
  
<p align="center">
  <img src="https://github.com/MatCyt/Price_Elasticity_R/blob/master/charts/price_trend.png" alt="Price Trend"
       width="600" height="320">
 </p>
  
    
### Price architecture
Here we can see the price variance and range for a given SKU - this could be potentially replaced by showing the regular, promo and average price.
  
<p align="center">
  <img src="https://github.com/MatCyt/Price_Elasticity_R/blob/master/charts/price_architecture.png" alt="Price Architecture"
       width="600" height="400">
 </p>
  
    
## OWN PRICE ELASTICITY
The main points in calculating price elasticity are relatively simple.  
We run linear regression for each of the products, taking the sales and price over time as minimum input. Using the formula provided we calculate the price elasticity. Below you can find a loop (could easily be transformed into function) running linear model and first calculations for all products in typical sales dataset.  
  
``` R
# Run the model for each SKU in data frame
for (i in sku_list) {
  
  # select columns only for one SKU in that iteration
  input = df_casted %>%
    select(str_subset(names(df_casted), i)) 
  
  # create model name and add it to the list to be used later
  model_name = paste('model', i, sep = '_')
  model_names = c(model_names, list(model_name))
  
  # run model for given SKU
  model_iterated = lm(
    as.formula(paste(colnames(input)[2], # 1 for value, 2 for units
                     "~", paste(colnames(input)[3]))) # 3rd column for price
    , data = input)
  
  # Add model object to the list and change the naming
  models = c(models, list(model_iterated))
  
  # store the p value of the regular price for model validation
  models_results[ i, 'model'] = model_name
  models_results[ i, 'price_p_value'] = round(summary(model_iterated)$coefficients[8],6)
  models_results[ i, 'price_coef'] = summary(model_iterated)$coefficients[2]
  models_results[ i, 'mean_price'] = mean(input[[3]])
  models_results[ i, 'mean_sales'] = mean(input[[2]])
  
}
  ```
  
In addition we would need to add the final PE calculations to those outcomes. This could also easily be included into the main loop/function - was left out for clarity in this study.  
``` R
# Calculate the price elasticity value 
# Mark the results as significant with cutoff on p value at 0.2

df_own_price_elasticity = models_results %>%
  mutate(price_elasticity = price_coef * (mean_price / mean_sales),
         significance = ifelse(price_p_value < 0.2, 'significant', 'not significant')) %>% # TODO properly adjust the condition here
  arrange(desc(mean_sales))
}
  ```
  
Here we can see the output results. As mentioned before - depending of the model quality this should not be treated as a literal value of what will happen to our sales but they are a good indicator of price sensitive products. Also do remember that price elasticity is not a linear metric.
  
<p align="center">
  <img src="https://github.com/MatCyt/Price_Elasticity_R/blob/master/charts/regular_elasticities.png" alt="Own Elasticity"
       width="600" height="400">
 </p>
  
## CROSS PRICE ELASTICITY
Cross price elasticity tells us how our product was impacted by pricing of other SKUs and vice versa.  
It is essential to understand firstly if with your price discounts you are stealing from your competitors or you ar cannibalizing your other products. Secondly it will tell you at which competitors pricing you should pay special attention.  
  
Code below runs the model for all possible combination of products / variables and create a ready output.  
``` R
for (sku in sku_list) {
  
  target_variable = paste('sum_units', sku, sep = '_') # select target - here we model on units
  
  input = df_casted %>% # keep sales of sku as target and price of all other skus as other variables
    select(str_subset(names(df_casted), 'average_price'), target_variable) 
  
  model_name = paste(sku, 'model' ,sep = '_') 
  
  model_iterated = lm( # run models on all skus
    as.formula(paste(target_variable, 
                     "~ .")) 
    , data = input)
  
  
  ### Build end output and add line by line to final data frame with results
  
  # Extract model metrics and transform into desired format
  model_output = as.data.table(summary(model_iterated)$coefficients) %>%
    mutate(variables = rownames(summary(model_iterated)$coefficients)) %>%
    select(c(1,4,5)) %>%
    rename(p_value = 2) %>%
    filter(variables != '(Intercept)')

  # Calculate price elasticities - use price and sales from additional metrics by joining the tables
  elasticity_input = model_output %>%
    mutate(model = model_name,
           SKU = sub("^[^_]*_[^_]*_", "", variables)) %>%
    left_join(additional_metrics) %>%
    mutate(elasticity = Estimate * (mean_price / mean_sales),
           significant = ifelse(p_value < 0.2, 'significant', 'not significant'),
           own_price = ifelse(SKU == sku, 'yes', 'no')) %>%
    arrange(desc(variables)) %>% 
    # Add information about significance for each SKU (separately for cross and own) - see Readme for this point discussion
    # Change the elasticity value to 0 for each not significant variable
    mutate(elasticity = replace(elasticity, 
                                own_price == 'no' & (significant == 'not significant' | Estimate < 0), 0),
           elasticity = replace(elasticity,
                                own_price == 'yes' & (significant == 'not significant' | Estimate > 0), 0))
  
  elasticity_output = dcast(elasticity_input, model ~ variables, value.var = 'elasticity')
  
  # Add the results from the model to the final table
  cross_results = rbind(cross_results, elasticity_output)
  
}
```

Below you can find an output of cross price elasticity model (also could be presented as bar chart).  
Rows represent SKUs, being affected by pricing of SKUs shown in columns. The values are price elasticities of those relationships. The diagonal values represent own price elasticity.  
In this example we can see that SKU_H has own elasticity of -4,93 and its sales is being affected by price changes of SKUs B, D, I and H. So 1% of price decrease in SKU_I will cause 4,65% of price drop in SKU_H. At the same time this table allows us to quicky see how the price change of SKU_H (green highlight) will affect prices of other products (like strong impact on sales of SKU_G).
  
  
<p align="center">
  <img src="https://github.com/MatCyt/Price_Elasticity/blob/master/charts/cross_price_elasticity_explained.R.png" alt="Price Trend"
       width="800" height="250">
 </p>


## PROMO VS REGULAR PRICE 
One last thing you can achieve with price data is to calculate regular vs promo price elasticities. So how sales of you product will respond to long term smaller changes in regular price vs deep temporary discounts.  
In this data I did not have right data available - so unfortunatelly you will not find here a ready code and output.  
  
If you will however find a dataset like this the process if fairly similar to calculating the own price elasticity. The main interesting difference is in the insights it could bring. You can easily identify products that could benefit more from price discounts vs regular price change or those for which you have actually space to consider increase in base price and dropping the price promotions.  
  
## Materials
Here you can find some links and materials I found usefull along the way:

Code | Baseline - simple step by step price elasticity calculation with R:  
http://www.salemmarafi.com/code/price-elasticity-with-r/

Code | Price elasticity in R with non-linear relations included:  
https://www.r-bloggers.com/food-for-regression-using-sales-data-to-identify-price-elasticity/

Theory | Price elasticity limitations and things to keep in mind during interpretation:  
http://www.cornerstonecapabilities.com/what-you-must-know-about-pricing-elasticity-and-the-danger-it-may-bring/

Theory | Cross price elasticity and cannibalization:  
https://www.ashokcharan.com/Marketing-Analytics/~pm-cannibalization.php

Longer theory | Whole chapter on pricing studies and different method / models:  
https://www.ashokcharan.com/Marketing-Analytics/~pr-what-price.php

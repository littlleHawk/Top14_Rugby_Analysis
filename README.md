# [Read the Full Report Here](https://github.com/littlleHawk/Top14_Rugby_Analysis/blob/main/documents/478%20Final_Sakina%20Lord.pdf)
*The full report includes the full report with Graphics and Code appendicies. A truncated version follows*

# Introduction

The chosen dataset contains rugby match results and player data from the French Top 14 league. Every professional rugby match I have attended in person has been part of this league, which includes Union Bordeaux-Bègles, the team from my host city in France. Compared to more widely studied competitions such as HSBC Sevens or the Six Nations, the Top 14 is relatively underexplored. Personal interest also influenced this choice, as I played rugby union during the summer and it remains my favorite “ball sport.”

This project uses two datasets: one containing 18 seasons of French [**Top 14 Rugby Union results**](https://www.kaggle.com/datasets/thomasbarrepitous/french-league-rugby-union-results-from-2005-2023) , and another with [**per-player information**]([https://www.kaggle.com/datasets/patricknaylor/its-rugby-player-data](https://www.kaggle.com/datasets/patricknaylor/its-rugby-player-data)). These datasets are referred to as `matches` and `players` respectively. The third significant dataset used in this analysis is the `joined` dataset which expands the `matches` data so that each row is one player and match combination, and joins this expanded dataset with `players` to add that individual’s play statistics.

# Data evaluation

### Discussion of Variables

#### Top 14

The Top 14 dataset contains per-game data for 3,338 matches over 18 seasons involving 26 teams. In the raw dataset, all variables are character type. During cleaning, I converted *home_score* and *away_score* to numeric types as well as factored all the remaining character types. I also created:

* *delayed_game* — 1 if the match was postponed due to COVID-19, 0 otherwise

* *score_diff* — the score difference, positive if the home team won, negative if the away team won

* *winning_team* — indicates which team won

Additional key variables include *Season* (the two-year season span), *date* (match day), *home_team* and *away_team*, and *home_1-23* and *away_1-23* (the players for each team in that match).

Top 14 seasons are played over weekends denoted as *day* J1 through J26, with each team having 13 home and 13 away games. The top six teams progress to the final phase, where 3rd-6th place teams compete in *Barrages*. The winner of the 3rd/6th place match faces the 2nd-ranked team, and the winner of the 4th/5th place match faces the 1st-ranked team in the *Semi*, both played at neutral venues. The *Finale* is contested between the winners of the semifinals at a neutral stadium, which may affect how the model interprets home and away designations. The *Access Match* pits the 13th-ranked team against the second division champion for a place in the next Top 14 season. These differences in tournament days were important later in calculating Elo scores for each team.

#### Players

The Players dataset contains per-player information, including *birthdate*, *position*, and *team* for a given *competition* and *year*. It also tracks performance metrics such as *total_points_scored*, *matches_played*, *tries_scored*, and *matches_started*.

Data originates from [**itsrugby.fr**](http://itsrugby.fr), so I was able to compare the short column names in Kaggle and rename fields with something more descriptive. *player_id*, scraped from each player’s URL, was cleaned to contain only the player name in uppercase, aligning it with the Top 14 dataset when creating the `joined` set. `joined` contains:

* Character variables which were factored: *position*, *team*

* Character variables which were removed due to redundancy: *player_id*, *birthdate*, *competition*

* Numeric variables: all performance stats and *birth_year* obtained from *birthdate*

Players data aggregates per-competition metrics, whereas the Top 14 dataset contains per-game statistics. This motivates a potential join between the two datasets. Before filtering for Top 14 matches, the dataset included 5,968 players across 26 years and 86 competitions, totaling 51,211 rows. Some entries contain erroneous data, such as a player listed in a 2097 competition (*player_id* thomas-lainault-3896).

### Data Cleaning

A key decision was whether to use only Top14 player data or include matches from other leagues. I decided to only used data on the Top14 league so I could focus on match data as well. Filtering `players` for only the Top14 competition reduces the player dataset from 51,211 entries to 4,355, removing players from Irish, English, Welsh, and Scottish leagues such as the Six Nations. When converting scores to numeric type, I encountered "Delayed" and "TBD" values. TBD scores correspond to future matches and are removed, as were the “Delayed” matches because they were not important to the analysis.

A major challenge in merging the datasets is that many Top 14 players do not appear in the Players dataset. The raw datasets only share player names as identifiers, but in different formats. Top 14 contains over 4,000 unique players, while the cleaned Players dataset contains only 1,048, leaving individual statistics missing for 3,154 players. This gap exists across all seasons from 2005–2022, so truncating by season is not an option. Missing two-thirds of player data presents a substantial limitation if player stats are used for modeling. In the final `joined` set, 102,854 player/match combinations had completely missing player statistics data.

# Methods

Due to the large number of missing data points, a majority of the work in this project was spent researching and implementing imputation. I used the `mice` package which does Multivariate Imputation by Chained Equations by way of a Gibbs sampler. The`mice` package contains within it multiple methods of imputation depending on the type of missing data, such as whether it is binary, an ordered or unordered categorical, or numeric, and the distribution of the nonmissing data of that variable. The imputation in this analysis is primarily predictive mean modeling which takes samples from the data to build a subset of complete values whose response matches that of the observations with missing values. It then randomly selects a value from this subset to fill the missing value in the target observation. Predictive mean matching (`pmm`) is effective for a wide range of data and, since it is based on data already present in the set, will not produce imputations outside the observed data range which can be both good in that the values are plausible and bad in that the values may miss an important set of values that might lie outside the range. Thus, predictive mean matching is not appropriate for values MNAR (Missing Not at Random) where the missingness is related to the unobserved data. I was able to use it in this analysis as my values are missing completely at random (MCAR) as the missingness was not related to the missing values; they were missing due to simply not being chosen in the scrape when the data was collected.

I first tested a fandom forest versus predictive mean matching approach on the five missing *birth_year* values in the `players` dataset and found that the random forest approach gave imputations whose distribution better matched that of the available data.

After diagnosing the missingness and looking at the distributions and types of available data to select the method of imputation in the `joined` dataset, I determined `pmm` to be the most appropriate method for most of my numerical data. The categorical variables *team* and *position* were imputed using a random forest and Polytomous logistic regression, respectively. Polytomous logistic regression is used for categorical variables with more than 3 levels (*position* has 10) and is essentially a multinomial logistic regression. Random Forest imputation was said to do better than polytomous logistic regression for categories with a large number of levels and low sparsity; *teams* has 26 levels and is full in the available data.

After imputing the data to fill in the 102,854 rows with missing data, I created a single tree to try to predict the score difference using `joined_complete` which is the`joined` data completed using the imputed values.

# Analysis results

Cross validation was used to select the predictive model as well as tune all final models. The Elastic Net model to predict `home_win` was tuned internally using `cv.glmnet()` with an $\alpha$ of 0.5. This cross validation found the accuracy of predictions to be ~ 73%, as was also the case for LASSO, Ridge, and Random forest, but their accuracies were a little lower. Though their coefficients were small, the imputed values aggregated to be used as per-match team statistics were present in the Elastic Net model, as seen in the table of Elastic Net Coefficients. The Random Forest predictive model was selected for the highest accuracy in the initial 5x2 K-fold cross validation and was later tuned to select `mtry` and `ntree` using 5-fold cross validation across a grid of 4 potential `mtry` values and 10 potential forest sizes. Tuning these values brought the average predictive accuracy of the model from just over 73% using 4 predictors and a forest size of 5000 to ~74.8% as seen in the Accuracy table. The other tree-based models in the initial model selection cross validation models showed higher variance across folds than the `glmnet`based models which might be due to the high number of predictors when a large amount of the data are factors with many levels. 

Apart from the predictive Random Forest, tree-based models did not do very well throughout this analysis. The single trees produced relied solely on one predictor, as seen in and had few terminal nodes which pooled many potential outcomes, resulting likely in high error rates as seen in figures 5 and 7. The pruned `home_wins` prediction tree in fig 15 only contains one split based on the probability of a home win calculated using the Elo values of that match. The imputation based on random forests, however did well as can be seen in the Observed vs Imputed: `birth_year` in the difference of how the imputed `birth_year` values using the `rf`method follows the original data distribution better than those produced using `pmm`.

# Discussion of final models and analysis.

### Predictive Model: Random Forest

To choose a predictive model, I ran 10 iterations of 5x2 K-fold cross validation on nine different possible models to predict the binary outcome of whether or not the home team will win. The models analysed included GLM, LASSO, Ridge, Elastic Net, Single Tree, Random Forest, BAgged Random Forest, Boosted Random Forest, and BART. For this analysis, I added in an Elo score for each team calculated using all seasons leading up to a match. For each match, I added the probability of a home win calculated using the home team and away team Elo scores as well as incorporated aggregate team stats including the average birth year and total player statistics (*matches_played*, *tries_scored*, etc.) for each team's per-match lineup.

Based on the K-fold cross validation, the regular Random Forest model performed with the highest accuracy. The Elastic Net model was not far behind. Further improvements were be made to the random forest to tune the number of predictors and number of trees hyperparameters to further improve the model. This tuning revealed that the ideal number of predictor variables to choose from at each split, `mtry`, is 10 predictors and the ideal forest size, `ntree`, is 9500 .  

### Inferential Model: Single Tree

For my inferential model, I initially built a single tree to predict *score_diff* from the `joined_complete`. The most complex tree produced is seen in fig 5. It is only split on the different teams and did not use any other predictor. Upon cross validation, it was found that a tree of size 6, the most complex tree, was the best predictor of *score_diff*. A pruned tree of 3 is in fig 7 and is shown to have a larger MSE than the more complex tree. A major limitation of this model is the large bins to predict score difference itself because a large set of teams will result in the same *score_diff* even if the teams are on different ends of values that average to the mean *score_diff* at their terminal node. However, this model does help to show which teams tend to have the largest gap in score. The values at the terminal nodes can help to inform which teams have historically been stronger or weaker than their opponents as well as to what magnitude.

I inadvertently created another inferential model when building the predictive model. I did not expect the Elastic Net model to do almost as well as a Random Forest in prediction of `home_win`. I included it in the cross validation to potentially see how much more a random forest would improve upon the GLM options. The most important variables in the Elastic Net model was the intercept and the probability of a home win, calculated using the Elo score. The intercept is the most significant coefficient, which could indicate a baseline higher probability of a home win. This makes sense given the tendance of the data towards home wins which indicates a home advantage. Based on this model, home teams tend to win the most in day 4 and both Narbonne and CA Brive win much more often on their own turf.

# Conclusions

In this project, the most important lessons came from managing extensive missingness and learning how to perform large-scale imputation. The initial knit of the document took nearly four hours, driven by repeated MICE imputations and multiple rounds of cross-validation across complex models. Through this process, I gained a deeper understanding of how R handles factored variables, particularly when interpreting the largest coefficients in the Ridge and Elastic Net models. I also saw firsthand that modeling offers multiple pathways to similar goals, whether evaluating predictor importance, tuning hyperparameters, or selecting among competing model classes.
  
Working with datasets containing substantial sparsity proved especially challenging. In the `players` dataset, where events such as red cards or kick conversions are rare, imputation tended to oversmooth these patterns, filling in zeros where real variability should exist. The `joined` dataset—where roughly 70% of rows lacked multiple fields which seemed almost impossible to impute, and to some extent that concern was justified. I worried the resulting values would be unusable, but diagnostic checks suggested that at least some structure was preserved. Even so, imputation at this scale likely alters the data in ways that limit inferential validity and predictive reliability. The fact that the best models still achieved accuracy above 70% was surprising and somewhat reassuring, even if that performance is modest given the uncertainty introduced by so much estimated data.

In the future, I would like to incorporate additional methods such as VIF analysis and boosted random forest models. I initially considered VIF when examining the missingness mechanism of `birth_year`, but cross-validation ultimately made the Ridge penalty selection straightforward without it. Still, VIF could have clarified multicollinearity among predictors in the Elastic Net model, possibly improving its performance. Boosted models may also provide better predictive power and more nuanced variable importance measures. Overall, this project highlighted both the limitations and the unexpected strengths of modeling in the presence of extreme missingness.

# Sources

Sas, W. (21 July, 2025). *Elo Calculator*. Omni Calculator [https://www.omnicalculator.com/sports/elo](https://www.omnicalculator.com/sports/elo)

Van Buuren, S. (2018). *Flexible Imputation of Missing Data*. [https://stefvanbuuren.name/fimd/sec-pmm.html](https://stefvanbuuren.name/fimd/sec-pmm.html)

Van Buuren, S. (27 May, 2025). *Multivariate Imputation by Chained Equations.* [https://cran.r-project.org/web/packages/mice/refman/mice.html#mice](https://cran.r-project.org/web/packages/mice/refman/mice.html#mice)

Enders, C. (2024). *Modern Missing Data Analysis*. CenterStat. [Lecture Notes] [https://centerstat.org/wp-content/uploads/2024/04/MISS-Notes.pdf](https://centerstat.org/wp-content/uploads/2024/04/MISS-Notes.pdf)

ISLR Version 2



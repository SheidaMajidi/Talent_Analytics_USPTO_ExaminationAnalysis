---
date: 2024-02-09
output:
  html_document: default
  word_document: default
title: Final Project - USPTO Examination Analysis
---

Introduction

This analysis aims to explore organizational and social factors
affecting the length of patent application prosecution and examiner
attrition at the U.S. Patent and Trademark Office (USPTO), with a
particular focus on the role of gender, race, and ethnicity.

\`\`\`{r setup, include=FALSE} knitr::opts_chunk\$set(echo = TRUE)
library(tidyverse) library(lubridate) library(arrow) library(lubridate)
library(stringr) library(skimr) library(gender) library(wru)


    Data Loading

    First, we need to load the dataset.

    ```{r load-data}

    # Read the dataset
    applications <- read.csv("/Users/sheidamajidi/Desktop/Winter2024/COURSES/ORGB671/Final Project/app_data_starter.csv")
    applications

Data Preparation

We need to prepare the data by ensuring correct data types and creating
necessary features.

\`\`\`{r data-preparation}

applications \<- applications %\>% mutate(filing_date =
ymd(filing_date), patent_issue_date = ymd(patent_issue_date),
abandon_date = ymd(abandon_date), prosecution_length =
if_else(!is.na(patent_issue_date),
as.numeric(difftime(patent_issue_date, filing_date, units = "days")),
as.numeric(difftime(abandon_date, filing_date, units = "days"))))


    Data Cleaning and Exploration
    We'll examine the dataset for missing values and summarize its key statistics.


    ```{r missing-value}
    # Check for missing values
    summary(applications)

    # Quick summary of data columns
    skim(applications)

## Handling those missing values:

Imputation for prosecution_length:

For prosecution_length, since it's numerical, using the median might be
more robust to outliers than the mean.

\`\`\`{r imputation-for-prosecution-length} median_prosecution_length
\<-
median(applications$prosecution_length, na.rm = TRUE) applications$prosecution_length\[is.na(applications\$prosecution_length)\]
\<- median_prosecution_length




    Examiner Gender and Race Estimation


    ```{r examiner-gender-and-race-estimation}
    library(gender)
    library(dplyr)

    # Step 1: Generate gender predictions
    examiner_names <- applications %>%
      distinct(examiner_name_first) 

    # Use gender() on the unique list of first names
    gender_predictions <- gender(examiner_names$examiner_name_first, method = "ssa", years = c(1940, 2020))

    # Convert gender_predictions to a dataframe and prepare for join
    gender_df <- as.data.frame(gender_predictions) %>%
      rename(examiner_name_first = name) %>%
      select(examiner_name_first, gender)

    # Step 2: Join gender predictions back to applications
    applications <- applications %>%
      left_join(gender_df, by = "examiner_name_first")

`{r print-examiner-gender-and-race-estimation-result} # Print the result to check the first few rows, including the newly added gender column head(applications)`

Examiner Race Estimation

First, we need to start by predicting race based on surname using the
WRU package:

\`\`\`{r race-prediction-based-on-surname-1} library(dplyr) library(wru)

# Step 1: Get unique surnames

examiner_surnames \<- applications %\>% distinct(examiner_name_last)
%\>% rename(surname = examiner_name_last)

# Preparing the voter_file with surnames

#voter_file \<- applications %\>% \# distinct(examiner_name_last) %\>%
\# mutate(surname = tolower(examiner_name_last)) %\>% \# select(surname)

# Call to predict_race() adjusted for simplicity

#race_predictions \<- predict_race(voter_file, census = "2010",
surname.only = TRUE)

# Step 2: Use predict_race() to estimate race

race_predictions \<- predict_race(examiner_surnames, surname.only =
TRUE)


    ``` {r print-columns-for-race-predictions}
    print(colnames(race_predictions))

\`\`\`{r race-prediction-based-on-surname-2} \# Process the race
predictions to identify the most probable race race_predictions \<-
race_predictions %\>% rowwise() %\>% mutate(most_probable_race =
case_when( pred.whi == max(c(pred.whi, pred.bla, pred.his, pred.asi,
pred.oth), na.rm = TRUE) \~ "White", pred.bla == max(c(pred.whi,
pred.bla, pred.his, pred.asi, pred.oth), na.rm = TRUE) \~ "Black or
African American", pred.his == max(c(pred.whi, pred.bla, pred.his,
pred.asi, pred.oth), na.rm = TRUE) \~ "Hispanic", pred.asi ==
max(c(pred.whi, pred.bla, pred.his, pred.asi, pred.oth), na.rm = TRUE)
\~ "Asian", pred.oth == max(c(pred.whi, pred.bla, pred.his, pred.asi,
pred.oth), na.rm = TRUE) \~ "Other", TRUE \~ "Unknown" )) %\>% ungroup()




    ```{r race-prediction-based-on-surname-3}
    # Join the race predictions back to the applications dataframe
    # 'examiner_name_last' in 'applications' matches 'surname' in 'race_predictions'
    applications_with_race <- applications %>%
      left_join(race_predictions %>% select(surname, most_probable_race), by = c("examiner_name_last" = "surname"))

    # Note: The above operation selects only the relevant columns ('surname' and 'most_probable_race')


    # View the first few rows of the updated dataframe to check the join results
    head(applications_with_race)

Dropping outliers

\`\`\`{r data-preparation-drop-outliers} \# Calculating the IQR for
prosecution_length Q1 \<-
quantile(applications_with_race$prosecution_length, 0.25, na.rm = TRUE) Q3 <- quantile(applications_with_race$prosecution_length,
0.75, na.rm = TRUE) IQR \<- Q3 - Q1

# Defining the lower and upper bounds for what's considered an outlier

lower_bound \<- Q1 - 1.5 \* IQR upper_bound \<- Q3 + 1.5 \* IQR

# Filtering out the outliers

applications_with_race \<- applications_with_race %\>%
filter(prosecution_length \>= lower_bound & prosecution_length \<=
upper_bound)

# Checking the result after dropping outliers

summary(applications_with_race\$prosecution_length)








    ##############
    Part 1 - 1
    ##############


    1. Organizational and Social Factors Associated with the Length of Patent Application Prosecution
    For this analysis, we consider factors such as examiner's art unit (examiner_art_unit), the USPC class/subclass, and demographic attributes (gender, most_probable_race).

    Exploratory Analysis:

    ```{r organizational-and-social-factors-exploratory-analysis}

    library(ggplot2)

    # Distribution of prosecution lengths
    ggplot(applications_with_race, aes(x = prosecution_length)) +
      geom_histogram(binwidth = 30, fill = "blue", color = "black") +
      labs(title = "Distribution of Prosecution Lengths", x = "Prosecution Length (days)", y = "Frequency")

    # Prosecution length by gender
    ggplot(applications_with_race, aes(x = gender.x, y = prosecution_length, fill = gender.x)) +
      geom_boxplot() +
      labs(title = "Prosecution Length by Gender", x = "Gender", y = "Prosecution Length (days)")

    # Prosecution length by race
    ggplot(applications_with_race, aes(x = most_probable_race, y = prosecution_length, fill = most_probable_race)) +
      geom_boxplot() +
      labs(title = "Prosecution Length by Race", x = "Race", y = "Prosecution Length (days)") +
      theme(axis.text.x = element_text(angle = 45, hjust = 1))

Regression Analysis A linear regression can be used to quantify the
impact of these factors on prosecution_length.

\`\`\`{r organizational-and-social-factors-regression-analysis}
lm_result \<- lm(prosecution_length \~ examiner_art_unit + uspc_class +
gender.x + most_probable_race, data = applications_with_race)
summary(lm_result)

    ##############
    Part 1 -2 
    ##############

    2. Organizational and Social Factors Associated with Examiner Attrition

    As there's no straightforward attrition flag or direct indicator of examiner leaving, and considering attrition is meant to reflect whether an examiner has stopped working on applications (which from this dataset might not be directly inferable), we are considering alternative approaches to approximate this concept.
    Without a clear attrition indicator, we'll focus on what can be analyzed given the dataset.

    Alternative Analysis Approach:

    Since attrition directly cannot be assessed, let's pivot towards understanding the factors that influence application outcomes (e.g., issued patents vs. abandoned applications), which might provide insights into examiner behavior or organizational processes affecting application processing times or outcomes.

    Analyzing Factors Influencing Application Outcomes:

    We use the disposal_type as a proxy to differentiate between applications that were successfully issued a patent versus those abandoned or otherwise disposed of. This will allow us to analyze how different factors, including examiner demographics and organizational units, may influence these outcomes.

    1- Preparing Data: 
    First, we create a binary outcome variable based on disposal_type.

    ```{r organizational-and-social-factors-w-examiner-attrition-prepare-data}
    applications_with_race <- applications_with_race %>%
      mutate(outcome = ifelse(disposal_type == "ISS", 1, 0)) # "ISS" indicates issued patents

2- Logistic Regression for Analyzing Influencing Factors: Since we don't
have attrition_flag, we'll adjust the focus to the outcome variable just
created.

\`\`\`{r
organizational-and-social-factors-w-examiner-attrition-logistic-regression}
\# Logistic regression to explore factors influencing application
outcomes glm_outcomes \<- glm(outcome \~ examiner_art_unit +
tenure_days + gender.x + race, family = binomial(link = "logit"), data =
applications_with_race)

summary(glm_outcomes)



    3- Indirect Analysis for Examiner Attrition
    Given the limitations, we'll focus on exploring patterns that might suggest attrition-like behavior, such as examiners with shorter tenures possibly having different outcome patterns or being associated with specific organizational or social factors.

    3 - 1 - Aggregate Analysis on Tenure and Outcomes
    First, we explore the relationship between tenure_days and application outcomes, assuming that changes in tenure_days distribution might reflect on engagement or attrition-like behavior.

    ```{r organizational-and-social-factors-w-examiner-attrition-indirect-analysis}
    # Aggregate analysis to explore tenure distribution across different application outcomes
    applications_with_race %>%
      group_by(outcome) %>%
      summarise(Average_Tenure = mean(tenure_days, na.rm = TRUE),
                Median_Tenure = median(tenure_days, na.rm = TRUE)) %>%
      print()

4.  Exploratory Analysis on Tenure by Gender and Race Exploring
    tenure_days distribution across gender.x and race might also offer
    insights into differing patterns that could indirectly relate to
    attrition or engagement dynamics.

\`\`\`{r
organizational-and-social-factors-w-examiner-attrition-EDA-on-tenure-by-gender-and-race}
\# Tenure distribution by gender ggplot(applications_with_race, aes(x =
gender.x, y = tenure_days)) + geom_boxplot() + labs(title = "Tenure
Distribution by Gender", x = "Gender", y = "Tenure Days")

# Tenure distribution by race

ggplot(applications_with_race, aes(x = race, y = tenure_days)) +
geom_boxplot() + labs(title = "Tenure Distribution by Race", x = "Race",
y = "Tenure Days") + theme(axis.text.x = element_text(angle = 45, hjust
= 1))



    ##############
    Part 1 - 3 
    ##############

    3- Role of gender, race and ethnicity here in the processes


    Analyzing the Impact of Gender, Race, and Ethnicity
    Given the results from your logistic regression model for application outcomes and the EDA conducted for tenure distributions, you can interpret the roles of gender, race, and ethnicity as follows:

    1. Interpretation from Logistic Regression Model
    The coefficients from the logistic regression model (glm_outcomes) give us quantitative insights into how gender and race are associated with the likelihood of an application being issued a patent (outcome).

    ```{r impact-of-gender-race-ethnicity}
    # Recap the summary of the logistic regression model for reference
    summary(glm_outcomes)

########################### 

Part 1 - 3 - Interpretation \###########################

From the output of summary(glm_outcomes), we can interpret the
coefficients related to gender.x and race as follows:

Gender (gender.x): Coefficients for gender variables (e.g.,
gender.xfemale, gender.xmale) indicate the influence of an examiner's
gender on the likelihood of a patent application being issued. A
positive coefficient suggests that being of that gender increases the
likelihood of an application resulting in an issuance compared to the
baseline gender category, while a negative coefficient suggests a
decrease.

Race (race): Similarly, coefficients for different race categories show
how the race of an examiner might impact the outcome of patent
applications. Positive values indicate an increased likelihood of
issuance for examiners of that race, while negative values indicate a
decreased likelihood, both relative to the baseline race category.

\`\`\`{r impact-of-gender-race-ethnicity-Additional} \# Exploring
interaction effects (example) glm_interactions \<- glm(outcome \~
examiner_art_unit \* gender.x \* race + tenure_days, family =
binomial(link = "logit"), data = applications_with_race)
summary(glm_interactions)

\`\`\`

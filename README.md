# Faircare Analytics

Team members: Kehinde Soetan, Ricky Lee, Tam Cheetham-West, Souradeep Thakur

## Introduction

Hospital readmissions are a major hurdle in healthcare and can be incredibly expensive. According to [this article](https://kffhealthnews.org/news/medicare-readmissions-penalties-2015/) based on 2014 Medicare data, approximately 2 million patients got readmitted within a year, with an estimated Medicare cost of $26 billion. It is however estimated that about $17 billion of that can be attributed to potentially preventable readmissions. High rates of readmission often signify issues with the quality of care received during initial stay, a lack of patient education, and suboptimal post-discharge support. In this project, we aim to
develop a machine learning model that predicts whether a patient is at risk of being readmitted or not, based on their electronic health records (EHR).

## Dataset

Our [dataset](https://archive.ics.uci.edu/dataset/296/diabetes+130-us+hospitals+for+years+1999-2008) contains clinical care data from 130 US hospitals and integrated delivery networks spanning ten years (1999-2008). All patient records in the dataset correspond to patients diagnosed with diabetes who stayed up to 14 days in the hospital. The dataset contains 101766 patient encounters and has 47 features. Here is a brief summary of the features:

| Feature      | Type | Description         | Missing |
| ------------ | ---- | ------------------- | ------- |
| encounter_id |  ID    | UID of an encounter |   No   |
| patient_nbr  |  ID    | UID of a patient |     No    |
| race | Categorical  | Demographic; values: Caucasian,<br>Asian, AfricanAmerican, Hispanic, and other. | 2% |
| gender | Categorical | Demographic; values: male, female, and unknown/invalid | No |
| age | Categorical | Demographic; values binned as [0-10), [10-20), ..., [90-100) | No
| weight | Categorical | Weight in lbs. | 97% |
| admission_type_id | Categorical | Integer codes with 9 distinct values; for example, 1 = emergency. | No |
| discharge_disposition_id | Categorical | Integer codes with 29 distinct values; for example, 1 = discharged to home. | No |
| admission_source_id | Categorical | 21 distinct integer codes | No |
| time_in_hospital | Integer | Number of days between admission and discharge | No |
| payer_code | Categorical | 23 distinct integer codes | 52% |
medical_specialty | Categorical | 84 distinct integer codes; indicates the specialty of the admitting physician. | 53% |
| num_lab_procedures | Integer | Number of lab tests undergone during the encounter. | No |
| num_procedure | Integer | Number of non-lab procedures undergone during the encounter. | No |
| num_medications | Integer | Number of distinct medications administered during the encounter. |
| num_outpatient | Integer | Number of outpatient visits in the year preceding the encounter. | No |
| number_emergency | Integer | Number of emergency visits in the past year preceding the encounter. | No |
| number_inpatient | Integer | Number of inpatient visits in the year preceding the encounter. | No |
| diag_1 | Categorical | Primary diagnosis: first three characters of ICD9 diagnosis codes (848 distinct values). | <1% |
| diag_2 | Categorical | Secondary diagnosis: first three characters of ICD9 diagnosis codes (923 distinct values). | <1% |
| diag_3 | Categorical | Additional secondary diagnosis: first three characters of ICD9 diagnosis codes (954 distinct values). | 1% |
| number_diagnosis | Categorical | Number of diagnosis in the system. | No |
| max_glu_serum | Categorical | Test result with values: >200, >300, Norm (normal), and None if not measured. | No |
| A1Cresult | Categorical | HbA1c test result; values: >7%, >8%, Norm, and None if not measured. | No |
| 23 drug features (e.g.- metformin,<br>insulin, etc.) | Categorical | Values: up=dosage increased, steady=dosage unchanged, <br> down=dosage decreased, None=not prescribed.| No |

The dataset was originally introduced and used in the paper: [Impact of HbA1c Measurement on Hospital Readmission Rates: Analysis of 70,000 Clinical Database Patient Record](http://www.hindawi.com/journals/bmri/2014/781670/). It has since been incorporated into the ucimlrepo package.


## Data Processing

We drop the features: weight, payer_code, and medical_specialty since they missing more than 50% of the values. Additionally, a total of 16 drugs in the dataset were never prescribed for over 95% of all encounters; we drop these as well.

### Exploratory data analysis

Here is how the data is distributed over the target and demographic features.
![Target distribution](./figs/target_dist.png)
![Age distribution](./figs/age_v_count.png)
![Race distribution](./figs/race_dist.png)

The main objectives of our EDA are to visualize any notable correlations between different features and readmissions. We also check if the features themselves are correlated to others. 

![Numerical features vs Readmission frequency](./figs/num_1_v_readmit.png "Numerical features vs Readmission frequency")
![Correlations](./figs/corr.png)

None of the features seem to be significantly correlated to others. The only exception is the features time_in_hospital and num_medication, which is expected. We also established that the industry standard codes such as admission_type_id, discharge_disposition_id, admission_source_id and the ICD9 diagnosis codes have no correlation with the readmission frequency. We will discuss how to effectively encode with these features in the next section.

### Feature engineering 

#### Grouping codes

We group the features admission_type_id, discharge_disposition_id, admission_source_id, diag_1, diag_2, and diag_3. As stated earlier, the diagnosis codes appear as the first three characters of the alphanumeric [ICD9 codes](https://en.wikipedia.org/wiki/List_of_ICD-9_codes). We group these diagnosis codes into 19 distinct categories such as, circulatory, respiratory, digestive, infections, etc. The other industry standard codes are also grouped into categories. For instance, admission_type_id is grouped into urgent care, non-urgent care, and unknown.

#### Integer encodings
Some of the categorical variables such as race and gender are one-hot encoded. We encode the ages as follows, [0-10) as 0, [10-20) as 1, [20-30) as 3, and so on. The drug features (metformin, repaglinide, etc.) have four distinct values: up, steady, down, and no which correspond to dosage increased, unchanged, decreased and not prescribed respectively. We encode these as follows, up as 3, steady as 2, down as 1, and no as 0. Similarly, the test result variables, A1Cresult (HbA1c test result) and max_glu_serum (maximum glucose serum test) have four distinct values. Take max_glu_serum for instance. The four distinct values are encoded as follows, >300 mg/dL as 3, >200 mg/dL as 2, Norm (~120 mg/dL) as 1, and No (test not performed) as 0.

#### Additional features

A key observation for this project was the fact that even though we have $\sim 10^5$ data points, there are only $\sim 7\times 10^4$ unique patients. This fact is mentioned in the [paper](http://www.hindawi.com/journals/bmri/2014/781670/) and used in this [project](https://github.com/lelandburrill/diabetes_readmission). This allowed us to add some patient historical data. For example, we added the number of distinct diagnoses a patient has had as well as the total time spent in hospitals. We added an additional 13 features corresponding to diagnoses that are responsible for the highest number of readmissions.

## Models and Results
### Models
Here is a list of the models that we used:
- Logistic regression (base model)
- k-Nearest Neighbor (k-NN) classifier
- XGBoost classifier
- RandomForest
### Results
Here is a summary of the results (note that the class 1 corresponds to readmitted in <30 days).
| Model | MSE | AUC score | Accuracy | Precision | Recall | F1-score |
|-------|-----|----------|-----------|-----------|--------|----------|
|Baseline: Linear regression| 0.1168 | 0.7193 | 0.8832| 0 : 0.89<br>1 : 0.39 | 0 : 0.99<br>1 : 0.04 | 0 : 0.94<br>1 : 0.07 |
| k-NN | 0.1148 | 0.6640 | 0.8852 | 0 : 0.89<br>1 : 0.48 | 0 : 1.00<br> 1 : 0.02 | 0 : 0.94<br>1 : 0.04 |
| RandomForest | 0.1137 | 0.8158 | 0.8863 | 0 : 0.89<br>1 : 0.58 | 0 : 1.00<br> 1 : 0.03 | 0 : 0.94<br>1 : 0.05 |
| Final: XGBoost | 0.1124 | 0.8206 | 0.8876 | 0 : 0.89<br>1 : 0.57 | 0 : 0.99<br>1 : 0.07 | 0 : 0.94<br>1 : 0.13 |

### Final model

Extensive feature engineering and model optimization revealed the calibrated XGBoost as the best performing model, with a minimal edge over the RandomForest classifier. k-NearestNeighbors was the worst performing model, only managing to outperform the baseline model on unprocessed data. Here is a comparison of all the models,

![ROC Curves](./figs/roc_all.png)

Here are the feature importance scores,

| Feature                |   Importance |
|:-----------------------|-------------:|
| pt_inp_tot             |    0.158818  |
| number_inpatient       |    0.0687674 |
| pt_diag_tot            |    0.0667006 |
| discharge_longterm_ind |    0.0288018 |
| pt_diag_ct             |    0.0276208 |

It appears that features pertaining to inpatient visits and different diagnosis characteristics (total number of diagnoses and number of distinct diagnoses) are crucial for the model.

We also wanted to ensure that the model does not have biases towards different demographic groups. For instance, the races of the patients in the dataset are extremely imbalanced. Here is a glimpse at different metrics for different demographic groups,

![metrics vs age](./figs/met_by_age.png)
![metrics vs race](./figs/met_by_race.png)
![metrics vs gender](./figs/met_by_gender.png)

*Note:* Binary classification experiments were also performed, targeting readmission within 30 days or greater than 30 days compared to no readmission. This approach offered better control over evaluation metrics due to a less severe class imbalance.

## Future Steps
1. A ternary classification with classes readmitted in <30 days, readmitted after 30 days, and not readmitted.
2. Use the model on [MIMIC-III](https://physionet.org/content/mimiciii/1.4/) and [MIMIC-IV](https://physionet.org/content/mimiciv/). Both of these are large and robust datasets and would probably lead to a significantly improved model.

## Repo details

*data/* : Contains all data files used or generated throughout the project. They are all in CSV format.

*figs/* : Contains all plots and figures generated from different notebooks. Some of these figures are used in the README.

*figs/plot_data/* : Contains some additional data used to create plots.


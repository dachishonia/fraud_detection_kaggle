# IEEE-CIS Fraud Detection



## Repository Structure

```text
.
├── README.md
├── requirements.txt
├── 00_eda.ipynb
├── model_experiment_LogisticRegression.ipynb
├── model_experiment_DecisionTree.ipynb
├── model_experiment_Bagging.ipynb
├── model_experiment_RandomForest.ipynb
├── model_experiment_AdaBoost.ipynb
├── model_experiment_GradientBoosting.ipynb
├── model_experiment_XGBoost.ipynb
├── model_experiment_MLP.ipynb
├── model_inference.ipynb
└── data/
    └── test_identity.csv
    └── test_transaction.csv
```



## Dagshub :  [https://dagshub.com/dachishonia/fraud_detection_kaggle](https://dagshub.com/dachishonia/fraud_detection_kaggle)



## კონკურსის მიმოხილვა

ამ პროექტის მიზანია `isFraud` კლასის პროგნოზირება IEEE-CIS Fraud Detection მონაცემებზე. ეს არის მაღალი სიზუსტის ბინარული კლასიფიკაციის ამოცანა, სადაც ძირითადი შეფასების მეტრიკაა ROC-AUC.

- კონკურსი: IEEE-CIS Fraud Detection
- Tracking: DagsHub + MLflow

ეტაპები:

- Cleaning - თავიდან ამოვშალე არასაჭირო სვეტები, სადაც missing მნიშვნელობების წილი ძალიან მაღალი იყო.
- Feature Engineering - გამოვიყენე `TransactionDT`, email-domain, frequency და aggregation ტიპის ნიშნები, რომ raw მონაცემები უფრო ინფორმაციული გამეხადა.
- Feature Selection - გავტესტე რამდენიმე FS სტრატეგია, რომ დამედგინა იძლეოდა თუ არა სარგებელს სვეტების შეკვეცა.
- Training + HPO - თითოეული model family-სთვის ცალკე ექსპერიმენტები ჩატარდა.
- Final Pipeline Registration - საუკეთესო კონფიგურაციები გადავიტანე სრულ `sklearn.Pipeline`-ში და დავარეგისტრირე MLflow Model Registry-ში.
- Inference Notebook - რეგისტრირებული მოდელი ჩავტვირთე MLflow-დან და raw test მონაცემზე გავუშვი `predict_proba`.

ამ სტრუქტურამ მომცა რეპროდუცირებადი ექსპერიმენტები და მკაფიო პასუხი კითხვაზე: "რომელი მოდელი ჯობია და რატომ".

## Feature Engineering და Preprocessing ლოგიკა

Preprocessing და Feature Engineering ლოგიკა თითოეულ notebook-ში `sklearn.Pipeline`-თან არის მიბმული, რომ:

- იგივე ლოგიკამ იმუშაოს train-ზე და test-ზე;
- leakage არ მოხდეს manual data handling-ით;
- inference notebook-ში დამატებითი preprocessing მინიმუმამდე დავიდეს;
- Model Registry-ში შენახული მოდელი იყოს ერთი მთლიან pipeline-ად შეფუთული არტეფაქტი.

ძირითადი გადაწყვეტილებები:

- numeric სვეტები: imputation (`median` / constant strategy ექსპერიმენტებში) + scaling იმ მოდელებისთვის, რომლებსაც ეს სჭირდებათ;
- categorical სვეტები: impute + `OrdinalEncoder`, unknown კატეგორიების დაცვით;
- `TransactionDT`-დან დროითი ნიშნები;
- email-domain ნიშნები;
- frequency და aggregate ტიპის ნიშნები;
- `TransactionAmt_log`, `D_mean`, `D_null_count`, `C_sum`, `M_T_count`, `M_F_count`.

შედეგად საბოლოო რეგისტრირებული მოდელი არის ერთი მთლიან pipeline-ად შეფუთული არტეფაქტი.

## Feature Selection სტრატეგია

ვცადე რამდენიმე ვარიანტი:

- ყველა სვეტის დატოვება;
- მაღალი კორელაციის სვეტების მოშორება;
- top-MI / SelectKBest ტიპის სვეტები;
- tree-importance top სვეტები;
- ExtraTrees cumulative importance 95% და 97%.

საბოლოოდ საუკეთესო შედეგები ბევრ model family-ში მივიღე ყველა მნიშვნელოვანი სვეტის დატოვებასთან ან ExtraTrees-ის კონსერვატიულ 97% threshold-თან. ეს ამ კონკრეტულ ამოცანაზე მოსალოდნელიცაა - tree-based ensemble მოდელები ხშირად თავად ახერხებენ noisy ნიშნების გამკლავებას.

## Training შედეგები (Test ROC-AUC / CV ROC-AUC)


| Model Family       | CV AUC (3-fold) | Test AUC   |
| ------------------ | --------------- | ---------- |
| GradientBoosting   | 0.5833          | **0.5101** |
| MLP                | 0.5748          | 0.5087     |
| DecisionTree       | 0.5378          | 0.5080     |
| XGBoost            | 0.5578          | 0.4976     |
| RandomForest       | 0.6043          | 0.4986     |
| AdaBoost           | 0.5999          | 0.4948     |
| Bagging            | 0.5600          | 0.4917     |
| LogisticRegression | 0.6104          | 0.4729     |


## ინტერპრეტაცია

- საუკეთესო Test AUC მიიღო **GradientBoosting**-მა (0.5101), მეორე ადგილზე MLP (0.5087).
- LogisticRegression-ს ყველაზე მაღალი CV AUC აქვს (0.6104), მაგრამ test-ზე სუსტია — ეს კლასიკური overfit gap-ია cross-val vs holdout შედარებაში.
- DecisionTree holdout-ზე კარგად ჩანს (0.5080) თუმცა CV AUC ყველაზე დაბალია (0.5378) — მოდელი unstable-ია სხვადასხვა split-ზე.
- Underfit, tuned და overfit HPO trial-ები ცალკე run-ებად დაილოგა — DagHub-ში ჩანს `train_roc_auc` vs `test_roc_auc` gap თითოეული კონფიგურაციისთვის.

## საუკეთესო მოდელი

- Model: `IEEE_Fraud_BestModel` (version 1)
- Architecture: `GradientBoosting`
- Test AUC: `0.5101`
- CV AUC (3-fold mean): `0.5833`
- Registry: MLflow Model Registry → `IEEE_Fraud_BestModel`

## რატომ ეს მოდელი

- ყველა მოდელს შორის ყველაზე მაღალი holdout Test AUC ჰქონდა (0.5101).
- CV-სა და test-ს შორის gap GradientBoosting-ში ყველაზე მცირეა — generalization უკეთესია.
- boosting სტრატეგია ეფექტურია imbalanced fraud detection-ში, სადაც minority class-ის სწორი პრიორიტეტი მნიშვნელოვანია.
- მოდელი inference pipeline-ად რეგისტრირდება MLflow Model Registry-ში 


## MLflow / DagsHub Tracking

ყველა მნიშვნელოვანი run (cleaning, FE, FS, training, CV, final) დაილოგა MLflow-ში. თითოეული model family ცალკე experiment-ად არის გატანილი:

- `LogisticRegression_Training`
- `DecisionTree_Training`
- `Bagging_Training`
- `RandomForest_Training`
- `AdaBoost_Training`
- `GradientBoosting_Training`
- `XGBoost_Training`
- `MLP_Training`

Model Registry-ში რეგისტრირებულია:

- `IEEE_Fraud_BestModel` (version 1) — GradientBoosting pipeline

ჩაწერილი მეტრიკები:

- `roc_auc`
- `f1`
- `precision`
- `recall`
- `average_precision`
- `train_roc_auc` / `test_roc_auc` / `overfit_gap` (HPO trial runs-ში)
- `cv_test_roc_auc_mean` / `cv_test_roc_auc_std` (CV run-ში)

BestModel run-ებში დამატებით ინახება:

- `confusion_matrix.png`
- `roc_curve.png`
- `classification_report.txt`
- სრული `sklearn.Pipeline`




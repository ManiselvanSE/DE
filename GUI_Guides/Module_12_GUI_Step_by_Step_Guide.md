# Module 12: Machine Learning for Data Engineering - GUI Step-by-Step Guide

## Overview
This module covers AWS machine learning services essential for data engineering workflows. You'll learn to build churn prediction models with SageMaker, analyze text with Comprehend, and extract metadata from images using Rekognition.

**Duration:** 4 hours  
**Cost:** $100-200/month (depending on usage)  
**Difficulty:** Intermediate to Advanced

---

## Exercise 12.1: SageMaker Churn Prediction Pipeline

### Introduction
Build an end-to-end machine learning pipeline for customer churn prediction using Amazon SageMaker. This exercise covers data preparation, feature engineering, model training with hyperparameter tuning, deployment, and automated retraining.

**What You'll Build:**
- Customer churn dataset in S3
- SageMaker Processing job for feature engineering
- XGBoost model with hyperparameter tuning
- Real-time prediction endpoint
- Lambda function for batch predictions
- Automated retraining pipeline with EventBridge

**Duration:** 90 minutes  
**Cost:** ~$50-100/month (primarily endpoint hosting)

---

### Part 1: Create and Upload Customer Data

#### Step 1: Create S3 Bucket for ML Data
1. Open the **AWS Console** and navigate to **S3**
2. Click **Create bucket**
3. Configure bucket:
   - **Bucket name:** `ml-churn-data-[your-account-id]` (must be globally unique)
   - **Region:** us-east-1
   - **Block all public access:** Keep checked
   - **Bucket Versioning:** Enable
   - **Encryption:** Server-side encryption with Amazon S3 managed keys (SSE-S3)
4. Click **Create bucket**

#### Step 2: Generate Customer Churn Data
1. On your local machine, create a Python script `generate_churn_data.py`:
```python
import pandas as pd
import numpy as np
from datetime import datetime, timedelta
import boto3

# Set random seed for reproducibility
np.random.seed(42)

# Generate synthetic customer data
n_customers = 10000

data = {
    'customer_id': [f'CUST{str(i).zfill(6)}' for i in range(1, n_customers + 1)],
    'account_age_months': np.random.randint(1, 120, n_customers),
    'monthly_charges': np.random.uniform(20, 200, n_customers).round(2),
    'total_charges': np.random.uniform(100, 10000, n_customers).round(2),
    'contract_type': np.random.choice(['Month-to-Month', 'One Year', 'Two Year'], n_customers),
    'payment_method': np.random.choice(['Credit Card', 'Bank Transfer', 'Electronic Check', 'Mailed Check'], n_customers),
    'customer_service_calls': np.random.randint(0, 10, n_customers),
    'tech_support_tickets': np.random.randint(0, 8, n_customers),
    'streaming_services': np.random.randint(0, 5, n_customers),
    'internet_service': np.random.choice(['DSL', 'Fiber', 'None'], n_customers),
    'online_backup': np.random.choice(['Yes', 'No'], n_customers),
    'device_protection': np.random.choice(['Yes', 'No'], n_customers),
    'paperless_billing': np.random.choice(['Yes', 'No'], n_customers),
    'avg_session_minutes': np.random.uniform(30, 600, n_customers).round(1)
}

# Create churn label based on risk factors
churn_probability = (
    (data['contract_type'] == 'Month-to-Month').astype(int) * 0.3 +
    (data['customer_service_calls'] > 5).astype(int) * 0.2 +
    (data['payment_method'] == 'Electronic Check').astype(int) * 0.15 +
    (data['account_age_months'] < 12) * 0.2 +
    np.random.uniform(0, 0.15, n_customers)
)

data['churn'] = (churn_probability > 0.5).astype(int)

# Create DataFrame
df = pd.DataFrame(data)

# Split into train (70%), validation (15%), test (15%)
train_size = int(0.7 * len(df))
val_size = int(0.15 * len(df))

train_df = df[:train_size]
val_df = df[train_size:train_size + val_size]
test_df = df[train_size + val_size:]

# Save to CSV files
train_df.to_csv('train_data.csv', index=False)
val_df.to_csv('validation_data.csv', index=False)
test_df.to_csv('test_data.csv', index=False)

# Also save a version for batch prediction
batch_df = test_df.drop('churn', axis=1).head(100)
batch_df.to_csv('batch_prediction_input.csv', index=False)

print(f"Generated datasets:")
print(f"  Training: {len(train_df)} samples ({train_df['churn'].sum()} churned)")
print(f"  Validation: {len(val_df)} samples ({val_df['churn'].sum()} churned)")
print(f"  Test: {len(test_df)} samples ({test_df['churn'].sum()} churned)")
print(f"  Batch input: {len(batch_df)} samples")
print(f"\nChurn rate: {df['churn'].mean()*100:.1f}%")

# Upload to S3
s3 = boto3.client('s3')
bucket_name = 'ml-churn-data-YOUR-ACCOUNT-ID'  # Replace with your bucket name

files = ['train_data.csv', 'validation_data.csv', 'test_data.csv', 'batch_prediction_input.csv']
for file in files:
    s3.upload_file(file, bucket_name, f'raw-data/{file}')
    print(f"Uploaded {file} to s3://{bucket_name}/raw-data/{file}")
```
2. Install dependencies: `pip install pandas numpy boto3`
3. Update `bucket_name` with your actual bucket name
4. Run: `python generate_churn_data.py`

#### Step 3: Verify Data in S3
1. Navigate to **S3** console
2. Click on your bucket `ml-churn-data-[account-id]`
3. Navigate to `raw-data/` folder
4. Verify all 4 CSV files are present:
   - `train_data.csv`
   - `validation_data.csv`
   - `test_data.csv`
   - `batch_prediction_input.csv`

---

### Part 2: Create SageMaker Processing Job for Feature Engineering

#### Step 4: Create IAM Role for SageMaker
1. Navigate to **IAM** service
2. Click **Roles** in the left sidebar
3. Click **Create role**
4. Select trusted entity:
   - **Trusted entity type:** AWS service
   - **Use case:** SageMaker
   - Select **SageMaker - Execution**
5. Click **Next**
6. Add permissions (these should be auto-selected):
   - `AmazonSageMakerFullAccess`
   - `AmazonS3FullAccess`
7. Click **Next**
8. Name the role: `SageMaker-Execution-Role`
9. Click **Create role**

#### Step 5: Create Feature Engineering Script
1. Create a processing script `preprocessing.py` locally:
```python
import pandas as pd
import numpy as np
from sklearn.preprocessing import LabelEncoder, StandardScaler
import argparse
import os

def preprocess_data(input_file, output_file, is_training=True):
    """
    Feature engineering for churn prediction
    """
    # Read data
    df = pd.read_csv(input_file)
    
    # Feature engineering
    # 1. Create engagement score
    df['engagement_score'] = (
        df['streaming_services'] * 2 + 
        df['avg_session_minutes'] / 100 + 
        (df['online_backup'] == 'Yes').astype(int) +
        (df['device_protection'] == 'Yes').astype(int)
    )
    
    # 2. Calculate charge per month ratio
    df['charge_ratio'] = df['total_charges'] / (df['account_age_months'] + 1)
    
    # 3. Create risk score based on service calls
    df['support_risk'] = df['customer_service_calls'] + df['tech_support_tickets']
    
    # 4. Encode categorical variables
    categorical_cols = ['contract_type', 'payment_method', 'internet_service', 
                       'online_backup', 'device_protection', 'paperless_billing']
    
    label_encoders = {}
    for col in categorical_cols:
        le = LabelEncoder()
        df[col + '_encoded'] = le.fit_transform(df[col])
        label_encoders[col] = le
    
    # Select features for model
    feature_cols = [
        'account_age_months', 'monthly_charges', 'charge_ratio',
        'customer_service_calls', 'tech_support_tickets', 'support_risk',
        'streaming_services', 'engagement_score', 'avg_session_minutes',
        'contract_type_encoded', 'payment_method_encoded', 
        'internet_service_encoded', 'online_backup_encoded',
        'device_protection_encoded', 'paperless_billing_encoded'
    ]
    
    # Scale numerical features
    numerical_cols = ['monthly_charges', 'charge_ratio', 'engagement_score', 
                     'avg_session_minutes', 'account_age_months']
    
    scaler = StandardScaler()
    df[numerical_cols] = scaler.fit_transform(df[numerical_cols])
    
    # Prepare final dataset
    if is_training and 'churn' in df.columns:
        # For XGBoost, target must be first column
        output_df = df[['churn'] + feature_cols]
    else:
        output_df = df[feature_cols]
    
    # Save processed data
    output_df.to_csv(output_file, index=False, header=False)
    print(f"Processed {len(df)} records")
    print(f"Features: {len(feature_cols)}")
    if is_training and 'churn' in df.columns:
        print(f"Churn rate: {df['churn'].mean()*100:.1f}%")

if __name__ == '__main__':
    parser = argparse.ArgumentParser()
    parser.add_argument('--train-file', type=str, default='/opt/ml/processing/input/train/train_data.csv')
    parser.add_argument('--val-file', type=str, default='/opt/ml/processing/input/validation/validation_data.csv')
    parser.add_argument('--test-file', type=str, default='/opt/ml/processing/input/test/test_data.csv')
    parser.add_argument('--output-dir', type=str, default='/opt/ml/processing/output')
    
    args = parser.parse_args()
    
    # Process each dataset
    if os.path.exists(args.train_file):
        preprocess_data(args.train_file, f'{args.output_dir}/train/train.csv', is_training=True)
    
    if os.path.exists(args.val_file):
        preprocess_data(args.val_file, f'{args.output_dir}/validation/validation.csv', is_training=True)
    
    if os.path.exists(args.test_file):
        preprocess_data(args.test_file, f'{args.output_dir}/test/test.csv', is_training=True)
    
    print("Preprocessing complete!")
```
2. Upload to S3:
```bash
aws s3 cp preprocessing.py s3://ml-churn-data-YOUR-ACCOUNT-ID/code/preprocessing.py
```

#### Step 6: Create SageMaker Notebook Instance (Optional - for running processing job)
1. Navigate to **SageMaker**
2. In the left sidebar, click **Notebook** → **Notebook instances**
3. Click **Create notebook instance**
4. Configure:
   - **Notebook instance name:** `churn-ml-notebook`
   - **Notebook instance type:** ml.t3.medium
   - **Elastic inference:** None
   - **IAM role:** Select `SageMaker-Execution-Role`
5. Expand **Git repositories** (optional)
6. Click **Create notebook instance**
7. Wait 3-5 minutes for status to be "InService"

#### Step 7: Run SageMaker Processing Job
1. Click **Open JupyterLab** on your notebook instance
2. Create a new notebook (Python 3 kernel)
3. Run this code in cells:

```python
import boto3
import sagemaker
from sagemaker.processing import ScriptProcessor, ProcessingInput, ProcessingOutput
from sagemaker import get_execution_role

# Setup
region = boto3.Session().region_name
role = get_execution_role()
bucket = 'ml-churn-data-YOUR-ACCOUNT-ID'  # Replace with your bucket

# Create script processor
script_processor = ScriptProcessor(
    role=role,
    image_uri=f'683313688378.dkr.ecr.{region}.amazonaws.com/sagemaker-scikit-learn:1.2-1-cpu-py3',
    command=['python3'],
    instance_count=1,
    instance_type='ml.m5.xlarge',
    base_job_name='churn-preprocessing'
)

# Run processing job
script_processor.run(
    code=f's3://{bucket}/code/preprocessing.py',
    inputs=[
        ProcessingInput(
            source=f's3://{bucket}/raw-data/train_data.csv',
            destination='/opt/ml/processing/input/train'
        ),
        ProcessingInput(
            source=f's3://{bucket}/raw-data/validation_data.csv',
            destination='/opt/ml/processing/input/validation'
        ),
        ProcessingInput(
            source=f's3://{bucket}/raw-data/test_data.csv',
            destination='/opt/ml/processing/input/test'
        )
    ],
    outputs=[
        ProcessingOutput(
            source='/opt/ml/processing/output/train',
            destination=f's3://{bucket}/processed-data/train'
        ),
        ProcessingOutput(
            source='/opt/ml/processing/output/validation',
            destination=f's3://{bucket}/processed-data/validation'
        ),
        ProcessingOutput(
            source='/opt/ml/processing/output/test',
            destination=f's3://{bucket}/processed-data/test'
        )
    ]
)

print("Processing job completed!")
print(f"Processed data available at: s3://{bucket}/processed-data/")
```
4. Wait 5-10 minutes for processing job to complete
5. Verify output in S3 under `processed-data/` folder

---

### Part 3: Train XGBoost Model with Hyperparameter Tuning

#### Step 8: Configure Hyperparameter Tuning Job
1. In your SageMaker notebook, create a new cell:

```python
from sagemaker.estimator import Estimator
from sagemaker.tuner import HyperparameterTuner, IntegerParameter, ContinuousParameter
from sagemaker import image_uris

# Get XGBoost container
container = image_uris.retrieve('xgboost', region, version='1.7-1')

# Define base estimator
xgb_estimator = Estimator(
    image_uri=container,
    role=role,
    instance_count=1,
    instance_type='ml.m5.xlarge',
    output_path=f's3://{bucket}/model-output/',
    base_job_name='churn-xgboost'
)

# Set static hyperparameters
xgb_estimator.set_hyperparameters(
    objective='binary:logistic',
    eval_metric='auc',
    num_round=100,
    early_stopping_rounds=10
)

# Define hyperparameter ranges to tune
hyperparameter_ranges = {
    'max_depth': IntegerParameter(3, 10),
    'eta': ContinuousParameter(0.01, 0.3),
    'subsample': ContinuousParameter(0.5, 1.0),
    'colsample_bytree': ContinuousParameter(0.5, 1.0),
    'min_child_weight': IntegerParameter(1, 10),
    'gamma': ContinuousParameter(0, 5)
}

# Define objective metric
objective_metric_name = 'validation:auc'

# Create tuner
tuner = HyperparameterTuner(
    xgb_estimator,
    objective_metric_name,
    hyperparameter_ranges,
    max_jobs=10,
    max_parallel_jobs=2,
    strategy='Bayesian',
    objective_type='Maximize'
)

print("Starting hyperparameter tuning job...")
```

#### Step 9: Launch Tuning Job
1. Continue in the same notebook:

```python
from sagemaker.inputs import TrainingInput

# Define training and validation data
train_input = TrainingInput(
    s3_data=f's3://{bucket}/processed-data/train/train.csv',
    content_type='text/csv'
)

validation_input = TrainingInput(
    s3_data=f's3://{bucket}/processed-data/validation/validation.csv',
    content_type='text/csv'
)

# Launch tuning job
tuner.fit(
    inputs={
        'train': train_input,
        'validation': validation_input
    },
    wait=False
)

print(f"Tuning job name: {tuner.latest_tuning_job.name}")
print("Check progress in SageMaker Console → Training → Hyperparameter tuning jobs")
```
2. Note the tuning job name
3. This will take 30-60 minutes to complete

#### Step 10: Monitor Tuning Job Progress
1. Navigate to **SageMaker** console
2. Click **Training** → **Hyperparameter tuning jobs** in left sidebar
3. Click on your tuning job
4. Monitor the progress:
   - **Best training job** updates as jobs complete
   - **Training jobs** tab shows all individual jobs
   - **Best objective metric** shows the best AUC score achieved
5. Wait for status to be "Completed"

#### Step 11: Retrieve Best Model
1. In your notebook, run:

```python
# Wait for tuning to complete (if not already)
tuner.wait()

# Get best training job
best_job_name = tuner.best_training_job()
print(f"Best training job: {best_job_name}")

# Get best hyperparameters
best_estimator = tuner.best_estimator()
print(f"\nBest hyperparameters:")
for param, value in tuner.best_estimator().hyperparameters().items():
    print(f"  {param}: {value}")

# Get best metrics
best_metrics = tuner.best_training_job()
print(f"\nBest validation AUC: Check CloudWatch or training job details")
```

---

### Part 4: Deploy Model as Real-Time Endpoint

#### Step 12: Deploy Model Endpoint
1. In your notebook:

```python
# Deploy best model
predictor = tuner.deploy(
    initial_instance_count=1,
    instance_type='ml.m5.large',
    endpoint_name='churn-prediction-endpoint'
)

print(f"Endpoint deployed: churn-prediction-endpoint")
print(f"Endpoint ARN: {predictor.endpoint_arn}")
```
2. Deployment takes 5-10 minutes
3. Monitor in **SageMaker** → **Inference** → **Endpoints**

#### Step 13: Test Real-Time Predictions
1. In your notebook:

```python
from sagemaker.serializers import CSVSerializer
from sagemaker.deserializers import JSONDeserializer
import numpy as np

# Configure predictor
predictor.serializer = CSVSerializer()
predictor.deserializer = JSONDeserializer()

# Load test data
import pandas as pd
test_data = pd.read_csv(f's3://{bucket}/processed-data/test/test.csv', header=None)

# Get features (skip first column which is the label)
test_features = test_data.iloc[:5, 1:].values

# Make predictions
predictions = predictor.predict(test_features)
print("Sample predictions:")
for i, pred in enumerate(predictions['predictions']):
    churn_prob = pred['score']
    print(f"  Customer {i+1}: {churn_prob:.2%} churn probability")
```

#### Step 14: Configure Endpoint Auto-Scaling (Optional)
1. Navigate to **SageMaker** → **Inference** → **Endpoints**
2. Click on `churn-prediction-endpoint`
3. Scroll to **Endpoint runtime settings**
4. Click **Configure auto scaling**
5. Configure:
   - **Variant name:** Select the variant
   - **Minimum capacity:** 1
   - **Maximum capacity:** 3
   - **Target metric:** SageMakerVariantInvocationsPerInstance
   - **Target value:** 1000
   - **Scale-in cool down:** 300 seconds
   - **Scale-out cool down:** 60 seconds
6. Click **Save**

---

### Part 5: Create Lambda for Batch Predictions

#### Step 15: Create IAM Role for Lambda
1. Navigate to **IAM** → **Roles**
2. Click **Create role**
3. Select:
   - **Trusted entity type:** AWS service
   - **Use case:** Lambda
4. Click **Next**
5. Add permissions:
   - `AmazonS3FullAccess`
   - `AmazonSageMakerFullAccess`
6. Click **Next**
7. Role name: `Lambda-SageMaker-Execution-Role`
8. Click **Create role**

#### Step 16: Create Batch Prediction Lambda Function
1. Navigate to **Lambda**
2. Click **Create function**
3. Configure:
   - **Function name:** `batch-churn-predictions`
   - **Runtime:** Python 3.11
   - **Architecture:** x86_64
   - **Execution role:** Use existing role → `Lambda-SageMaker-Execution-Role`
4. Click **Create function**

#### Step 17: Add Lambda Function Code
1. In the Lambda code editor, replace the code:

```python
import json
import boto3
import csv
from io import StringIO
from datetime import datetime

s3 = boto3.client('s3')
sagemaker_runtime = boto3.client('sagemaker-runtime')

ENDPOINT_NAME = 'churn-prediction-endpoint'
BUCKET_NAME = 'ml-churn-data-YOUR-ACCOUNT-ID'  # Replace

def lambda_handler(event, context):
    """
    Process batch prediction requests
    """
    try:
        # Get input file from event or use default
        input_key = event.get('input_key', 'raw-data/batch_prediction_input.csv')
        
        # Read input CSV from S3
        response = s3.get_object(Bucket=BUCKET_NAME, Key=input_key)
        csv_content = response['Body'].read().decode('utf-8')
        
        # Parse CSV
        csv_reader = csv.DictReader(StringIO(csv_content))
        customers = list(csv_reader)
        
        # Simple feature engineering (same as preprocessing script)
        predictions_data = []
        
        for customer in customers:
            # Feature engineering
            engagement_score = (
                float(customer.get('streaming_services', 0)) * 2 + 
                float(customer.get('avg_session_minutes', 0)) / 100
            )
            
            charge_ratio = (
                float(customer.get('total_charges', 0)) / 
                (float(customer.get('account_age_months', 1)) + 1)
            )
            
            support_risk = (
                int(customer.get('customer_service_calls', 0)) + 
                int(customer.get('tech_support_tickets', 0))
            )
            
            # Encode categoricals (simplified - use same encoding as training)
            contract_encoded = {
                'Month-to-Month': 0, 'One Year': 1, 'Two Year': 2
            }.get(customer.get('contract_type'), 0)
            
            payment_encoded = {
                'Credit Card': 0, 'Bank Transfer': 1, 
                'Electronic Check': 2, 'Mailed Check': 3
            }.get(customer.get('payment_method'), 0)
            
            # Create feature vector (must match training order)
            features = [
                float(customer.get('account_age_months', 0)),
                float(customer.get('monthly_charges', 0)),
                charge_ratio,
                int(customer.get('customer_service_calls', 0)),
                int(customer.get('tech_support_tickets', 0)),
                support_risk,
                int(customer.get('streaming_services', 0)),
                engagement_score,
                float(customer.get('avg_session_minutes', 0)),
                contract_encoded,
                payment_encoded,
                0, 0, 0, 0  # Simplified - other encoded features
            ]
            
            # Make prediction
            payload = ','.join(map(str, features))
            response = sagemaker_runtime.invoke_endpoint(
                EndpointName=ENDPOINT_NAME,
                ContentType='text/csv',
                Body=payload
            )
            
            result = json.loads(response['Body'].read().decode())
            churn_probability = result['predictions'][0]['score']
            
            predictions_data.append({
                'customer_id': customer.get('customer_id'),
                'churn_probability': churn_probability,
                'churn_prediction': 1 if churn_probability > 0.5 else 0,
                'risk_level': 'High' if churn_probability > 0.7 else 
                             'Medium' if churn_probability > 0.4 else 'Low'
            })
        
        # Save predictions to S3
        output_key = f"predictions/batch_predictions_{datetime.now().strftime('%Y%m%d_%H%M%S')}.json"
        
        s3.put_object(
            Bucket=BUCKET_NAME,
            Key=output_key,
            Body=json.dumps(predictions_data, indent=2),
            ContentType='application/json'
        )
        
        # Summary statistics
        high_risk = sum(1 for p in predictions_data if p['churn_probability'] > 0.7)
        medium_risk = sum(1 for p in predictions_data if 0.4 < p['churn_probability'] <= 0.7)
        low_risk = sum(1 for p in predictions_data if p['churn_probability'] <= 0.4)
        
        return {
            'statusCode': 200,
            'body': json.dumps({
                'message': 'Batch predictions completed',
                'total_customers': len(predictions_data),
                'high_risk': high_risk,
                'medium_risk': medium_risk,
                'low_risk': low_risk,
                'output_file': f's3://{BUCKET_NAME}/{output_key}',
                'predictions': predictions_data[:5]  # Sample
            })
        }
        
    except Exception as e:
        print(f"Error: {str(e)}")
        return {
            'statusCode': 500,
            'body': json.dumps({'error': str(e)})
        }
```
2. Update `BUCKET_NAME` with your actual bucket
3. Click **Deploy**

#### Step 18: Configure Lambda Timeout
1. Click **Configuration** tab
2. Click **General configuration** → **Edit**
3. Set **Timeout:** 5 minutes
4. Set **Memory:** 512 MB
5. Click **Save**

#### Step 19: Test Batch Predictions
1. Click **Test** tab
2. Create test event:
```json
{
  "input_key": "raw-data/batch_prediction_input.csv"
}
```
3. Click **Test**
4. Review results showing churn predictions
5. Check S3 for output file under `predictions/` folder

---

### Part 6: Set Up Automated Retraining with EventBridge

#### Step 20: Create Lambda for Model Retraining
1. Navigate to **Lambda**
2. Click **Create function**
3. Configure:
   - **Function name:** `trigger-model-retraining`
   - **Runtime:** Python 3.11
   - **Execution role:** Use existing role → `Lambda-SageMaker-Execution-Role`
4. Click **Create function**
5. Add code:

```python
import json
import boto3
from datetime import datetime

sagemaker = boto3.client('sagemaker')

def lambda_handler(event, context):
    """
    Trigger SageMaker training job for model retraining
    """
    try:
        bucket = 'ml-churn-data-YOUR-ACCOUNT-ID'  # Replace
        role_arn = 'arn:aws:iam::YOUR-ACCOUNT-ID:role/SageMaker-Execution-Role'  # Replace
        
        job_name = f"churn-retrain-{datetime.now().strftime('%Y%m%d-%H%M%S')}"
        
        training_params = {
            'TrainingJobName': job_name,
            'RoleArn': role_arn,
            'AlgorithmSpecification': {
                'TrainingImage': '683313688378.dkr.ecr.us-east-1.amazonaws.com/sagemaker-xgboost:1.7-1',
                'TrainingInputMode': 'File'
            },
            'InputDataConfig': [
                {
                    'ChannelName': 'train',
                    'DataSource': {
                        'S3DataSource': {
                            'S3DataType': 'S3Prefix',
                            'S3Uri': f's3://{bucket}/processed-data/train/',
                            'S3DataDistributionType': 'FullyReplicated'
                        }
                    },
                    'ContentType': 'text/csv'
                },
                {
                    'ChannelName': 'validation',
                    'DataSource': {
                        'S3DataSource': {
                            'S3DataType': 'S3Prefix',
                            'S3Uri': f's3://{bucket}/processed-data/validation/',
                            'S3DataDistributionType': 'FullyReplicated'
                        }
                    },
                    'ContentType': 'text/csv'
                }
            ],
            'OutputDataConfig': {
                'S3OutputPath': f's3://{bucket}/model-output/'
            },
            'ResourceConfig': {
                'InstanceType': 'ml.m5.xlarge',
                'InstanceCount': 1,
                'VolumeSizeInGB': 10
            },
            'StoppingCondition': {
                'MaxRuntimeInSeconds': 3600
            },
            'HyperParameters': {
                'objective': 'binary:logistic',
                'eval_metric': 'auc',
                'num_round': '100',
                'max_depth': '6',
                'eta': '0.2',
                'subsample': '0.8',
                'colsample_bytree': '0.8'
            }
        }
        
        response = sagemaker.create_training_job(**training_params)
        
        return {
            'statusCode': 200,
            'body': json.dumps({
                'message': 'Retraining job started',
                'job_name': job_name,
                'job_arn': response['TrainingJobArn']
            })
        }
        
    except Exception as e:
        print(f"Error: {str(e)}")
        return {
            'statusCode': 500,
            'body': json.dumps({'error': str(e)})
        }
```
6. Update bucket name and role ARN
7. Click **Deploy**

#### Step 21: Create EventBridge Rule for Weekly Retraining
1. Navigate to **EventBridge**
2. Click **Rules** in left sidebar
3. Click **Create rule**
4. Configure rule:
   - **Name:** `weekly-model-retraining`
   - **Description:** `Trigger model retraining every week`
   - **Event bus:** default
   - **Rule type:** Schedule
5. Click **Next**
6. Define schedule:
   - **Schedule pattern:** Rate-based schedule
   - **Rate expression:** `rate(7 days)`
   - Or use cron: `cron(0 2 ? * SUN *)` (2 AM every Sunday)
7. Click **Next**
8. Select target:
   - **Target types:** AWS service
   - **Select a target:** Lambda function
   - **Function:** `trigger-model-retraining`
9. Click **Next**
10. Review and click **Create rule**

#### Step 22: Test Automated Retraining
1. Go to **EventBridge** → **Rules**
2. Select `weekly-model-retraining`
3. Click **Actions** → **Disable** (to prevent actual weekly runs during testing)
4. Navigate to **Lambda** → `trigger-model-retraining`
5. Click **Test** with empty event `{}`
6. Check **SageMaker** → **Training** → **Training jobs** for new job

---

### Verification Checklist

- [ ] S3 bucket created with customer churn data
- [ ] Training, validation, and test datasets generated
- [ ] SageMaker Processing job completed successfully
- [ ] Feature-engineered data available in S3
- [ ] Hyperparameter tuning job completed (10 training jobs)
- [ ] Best model identified with hyperparameters
- [ ] Real-time endpoint deployed and accessible
- [ ] Endpoint successfully returns predictions
- [ ] Batch prediction Lambda function working
- [ ] Batch predictions saved to S3
- [ ] Automated retraining Lambda created
- [ ] EventBridge rule configured for weekly retraining

### Architecture Benefits

**MLOps Best Practices:**
- Automated feature engineering with SageMaker Processing
- Hyperparameter optimization for better model performance
- Separate training, validation, and test datasets
- Automated retraining for model freshness
- Scalable real-time and batch inference

**Cost Optimization:**
- Use spot instances for training jobs (70% savings)
- Auto-scaling endpoints based on traffic
- Batch transform for large-scale predictions (cheaper than real-time)
- Lifecycle policies to delete old models

**Performance:**
- Real-time inference: < 100ms latency
- Batch predictions: 1000s of predictions per minute
- Hyperparameter tuning: 10-30% model improvement

---

### Cleanup (Important!)

1. **Delete SageMaker Endpoint:**
   - SageMaker → Endpoints → Select endpoint → Delete

2. **Delete Model:**
   - SageMaker → Inference → Models → Delete

3. **Delete Endpoint Configuration:**
   - SageMaker → Inference → Endpoint configurations → Delete

4. **Stop Notebook Instance:**
   - SageMaker → Notebook instances → Stop → Delete

5. **Delete Lambda Functions:**
   - Lambda → Functions → Delete both functions

6. **Delete EventBridge Rule:**
   - EventBridge → Rules → Delete

7. **Delete S3 Bucket:**
   - S3 → Empty bucket → Delete bucket

8. **Delete IAM Roles:**
   - IAM → Roles → Delete both roles

---

## Exercise 12.2: Comprehend Feedback Analytics Pipeline

### Introduction
Build an automated text analytics pipeline using Amazon Comprehend to analyze customer feedback. Extract sentiment, key phrases, entities, and train a custom classifier for categorizing feedback.

**What You'll Build:**
- Customer feedback dataset
- Lambda function for Comprehend analysis
- Glue Data Catalog for structured results
- Athena queries for analytics
- Custom Comprehend classifier
- QuickSight dashboard for visualization

**Duration:** 75 minutes  
**Cost:** ~$30-50/month

---

### Part 1: Generate and Store Customer Feedback Data

#### Step 1: Create S3 Bucket for Feedback Data
1. Navigate to **S3**
2. Click **Create bucket**
3. Configure:
   - **Bucket name:** `customer-feedback-analysis-[account-id]`
   - **Region:** us-east-1
   - **Block public access:** Keep checked
4. Click **Create bucket**

#### Step 2: Generate Sample Feedback Data
1. Create `generate_feedback.py` locally:

```python
import json
import boto3
from datetime import datetime, timedelta
import random

# Sample feedback templates
positive_feedback = [
    "Excellent service! Very satisfied with the product quality and delivery time.",
    "The customer support team was incredibly helpful and resolved my issue quickly.",
    "Love this product! It exceeded my expectations. Will definitely buy again.",
    "Great experience from start to finish. Highly recommend!",
    "Outstanding quality and very reasonable price. Five stars!"
]

negative_feedback = [
    "Terrible experience. The product arrived damaged and customer service was unhelpful.",
    "Very disappointed with the quality. Not worth the price at all.",
    "Worst purchase ever. Item doesn't work as described and return process is complicated.",
    "Poor customer service. My calls were ignored and emails went unanswered.",
    "Product broke after one week. Completely unsatisfied with this purchase."
]

neutral_feedback = [
    "The product is okay. Nothing special but does the job.",
    "Average experience. Met my basic expectations but nothing more.",
    "It's fine for the price. Not great but not terrible either.",
    "Decent product. Some features work well, others could be improved.",
    "Standard quality. No complaints but no praises either."
]

# Generate feedback data
feedbacks = []
for i in range(1000):
    sentiment_choice = random.choices(
        ['POSITIVE', 'NEGATIVE', 'NEUTRAL'],
        weights=[0.5, 0.3, 0.2]
    )[0]
    
    if sentiment_choice == 'POSITIVE':
        text = random.choice(positive_feedback)
    elif sentiment_choice == 'NEGATIVE':
        text = random.choice(negative_feedback)
    else:
        text = random.choice(neutral_feedback)
    
    feedback = {
        'feedback_id': f'FB{str(i+1).zfill(6)}',
        'customer_id': f'CUST{random.randint(1, 500):06d}',
        'product_id': f'PROD{random.randint(1, 50):04d}',
        'feedback_text': text,
        'rating': random.randint(1, 5),
        'timestamp': (datetime.now() - timedelta(days=random.randint(0, 90))).isoformat(),
        'channel': random.choice(['Email', 'Web', 'Mobile App', 'Phone'])
    }
    feedbacks.append(feedback)

# Save to file
with open('customer_feedback.json', 'w') as f:
    for feedback in feedbacks:
        f.write(json.dumps(feedback) + '\n')

print(f"Generated {len(feedbacks)} feedback entries")

# Upload to S3
s3 = boto3.client('s3')
bucket_name = 'customer-feedback-analysis-YOUR-ACCOUNT-ID'  # Replace

with open('customer_feedback.json', 'rb') as f:
    s3.upload_fileobj(f, bucket_name, 'raw-feedback/customer_feedback.json')

print(f"Uploaded to s3://{bucket_name}/raw-feedback/customer_feedback.json")
```
2. Update bucket name and run the script
3. Verify file in S3

---

### Part 2: Create Lambda for Comprehend Analysis

#### Step 3: Create IAM Role for Comprehend Lambda
1. Navigate to **IAM** → **Roles**
2. Click **Create role**
3. Select AWS service → Lambda
4. Add permissions:
   - `AmazonS3FullAccess`
   - `ComprehendFullAccess`
   - `AWSGlueServiceRole`
5. Role name: `Lambda-Comprehend-Execution-Role`
6. Click **Create role**

#### Step 4: Create Comprehend Analysis Lambda
1. Navigate to **Lambda**
2. Click **Create function**
3. Configure:
   - **Function name:** `analyze-customer-feedback`
   - **Runtime:** Python 3.11
   - **Execution role:** Use existing → `Lambda-Comprehend-Execution-Role`
4. Click **Create function**

#### Step 5: Add Lambda Code for Comprehend Analysis
1. Replace the Lambda code:

```python
import json
import boto3
from datetime import datetime

s3 = boto3.client('s3')
comprehend = boto3.client('comprehend')

OUTPUT_BUCKET = 'customer-feedback-analysis-YOUR-ACCOUNT-ID'  # Replace

def lambda_handler(event, context):
    """
    Analyze customer feedback using Amazon Comprehend
    """
    try:
        # Get S3 bucket and key from event
        bucket = event['Records'][0]['s3']['bucket']['name']
        key = event['Records'][0]['s3']['object']['key']
        
        # Read feedback data
        response = s3.get_object(Bucket=bucket, Key=key)
        content = response['Body'].read().decode('utf-8')
        
        feedbacks = [json.loads(line) for line in content.strip().split('\n')]
        
        analyzed_results = []
        
        for feedback in feedbacks[:100]:  # Limit for demo
            text = feedback['feedback_text']
            
            # Sentiment Analysis
            sentiment_response = comprehend.detect_sentiment(
                Text=text,
                LanguageCode='en'
            )
            
            # Key Phrases
            keyphrases_response = comprehend.detect_key_phrases(
                Text=text,
                LanguageCode='en'
            )
            
            # Entity Recognition
            entities_response = comprehend.detect_entities(
                Text=text,
                LanguageCode='en'
            )
            
            # Combine results
            analyzed = {
                'feedback_id': feedback['feedback_id'],
                'customer_id': feedback['customer_id'],
                'product_id': feedback['product_id'],
                'feedback_text': text,
                'rating': feedback['rating'],
                'timestamp': feedback['timestamp'],
                'channel': feedback['channel'],
                'sentiment': sentiment_response['Sentiment'],
                'sentiment_scores': sentiment_response['SentimentScore'],
                'key_phrases': [
                    {'text': kp['Text'], 'score': kp['Score']} 
                    for kp in keyphrases_response['KeyPhrases']
                ],
                'entities': [
                    {'text': e['Text'], 'type': e['Type'], 'score': e['Score']} 
                    for e in entities_response['Entities']
                ],
                'analysis_timestamp': datetime.now().isoformat()
            }
            
            analyzed_results.append(analyzed)
        
        # Save analyzed results
        output_key = f"analyzed-feedback/{datetime.now().strftime('%Y/%m/%d')}/analyzed_{datetime.now().strftime('%H%M%S')}.json"
        
        s3.put_object(
            Bucket=OUTPUT_BUCKET,
            Key=output_key,
            Body='\n'.join([json.dumps(r) for r in analyzed_results]),
            ContentType='application/json'
        )
        
        # Generate summary
        sentiment_counts = {}
        for result in analyzed_results:
            sentiment = result['sentiment']
            sentiment_counts[sentiment] = sentiment_counts.get(sentiment, 0) + 1
        
        return {
            'statusCode': 200,
            'body': json.dumps({
                'message': 'Analysis complete',
                'feedbacks_analyzed': len(analyzed_results),
                'sentiment_distribution': sentiment_counts,
                'output_location': f's3://{OUTPUT_BUCKET}/{output_key}'
            })
        }
        
    except Exception as e:
        print(f"Error: {str(e)}")
        return {
            'statusCode': 500,
            'body': json.dumps({'error': str(e)})
        }
```
2. Update `OUTPUT_BUCKET`
3. Click **Deploy**

#### Step 6: Configure S3 Trigger
1. Click **Add trigger**
2. Select **S3**
3. Configure:
   - **Bucket:** `customer-feedback-analysis-[account-id]`
   - **Event type:** All object create events
   - **Prefix:** `raw-feedback/`
   - **Suffix:** `.json`
4. Check acknowledgment box
5. Click **Add**

#### Step 7: Configure Lambda Settings
1. **Configuration** → **General configuration** → **Edit**
2. Set **Timeout:** 5 minutes
3. Set **Memory:** 512 MB
4. Click **Save**

#### Step 8: Test Comprehend Analysis
1. Re-upload feedback file to S3 to trigger Lambda:
```bash
aws s3 cp customer_feedback.json s3://customer-feedback-analysis-YOUR-ACCOUNT-ID/raw-feedback/customer_feedback_new.json
```
2. Check Lambda logs in CloudWatch
3. Verify analyzed data in S3 under `analyzed-feedback/` folder

---

### Part 3: Set Up Glue Catalog and Athena Queries

#### Step 9: Create Glue Database
1. Navigate to **AWS Glue**
2. Click **Databases** in left sidebar
3. Click **Add database**
4. Configure:
   - **Name:** `customer_feedback_db`
   - **Description:** `Database for customer feedback analytics`
5. Click **Create database**

#### Step 10: Create Glue Crawler
1. Click **Crawlers** in left sidebar
2. Click **Create crawler**
3. **Name:** `feedback-crawler`
4. Click **Next**
5. **Data source:**
   - **Is your data already mapped:** Not yet
   - Click **Add a data source**
   - **S3 path:** `s3://customer-feedback-analysis-[account-id]/analyzed-feedback/`
   - **Subsequent crawler runs:** Crawl all sub-folders
   - Click **Add an S3 data source**
6. Click **Next**
7. **IAM role:** Create new IAM role → `AWSGlueServiceRole-Feedback`
8. Click **Next**
9. **Target database:** Select `customer_feedback_db`
10. **Crawler schedule:** On demand
11. Click **Next**
12. Review and click **Create crawler**

#### Step 11: Run Crawler
1. Select the crawler
2. Click **Run**
3. Wait 1-2 minutes for completion
4. Click **Tables** in left sidebar
5. Verify table was created

#### Step 12: Query Data with Athena
1. Navigate to **Athena**
2. If first time, set up query result location:
   - Click **Settings** → **Manage**
   - Set location: `s3://customer-feedback-analysis-[account-id]/athena-results/`
   - Click **Save**
3. Select database: `customer_feedback_db`
4. Run queries:

```sql
-- Overall sentiment distribution
SELECT 
    sentiment,
    COUNT(*) as count,
    ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER(), 2) as percentage
FROM analyzed_feedback
GROUP BY sentiment
ORDER BY count DESC;
```

```sql
-- Average rating by sentiment
SELECT 
    sentiment,
    AVG(rating) as avg_rating,
    COUNT(*) as feedback_count
FROM analyzed_feedback
GROUP BY sentiment;
```

```sql
-- Top key phrases
SELECT 
    kp.text as key_phrase,
    COUNT(*) as frequency
FROM analyzed_feedback
CROSS JOIN UNNEST(key_phrases) as t(kp)
GROUP BY kp.text
ORDER BY frequency DESC
LIMIT 20;
```

```sql
-- Feedback by channel and sentiment
SELECT 
    channel,
    sentiment,
    COUNT(*) as count
FROM analyzed_feedback
GROUP BY channel, sentiment
ORDER BY channel, count DESC;
```

---

### Part 4: Train Custom Comprehend Classifier

#### Step 13: Prepare Training Data for Custom Classifier
1. Create `prepare_classifier_data.py`:

```python
import json
import boto3
import csv

# Categories for feedback classification
categories = ['Product Quality', 'Customer Service', 'Delivery', 'Pricing', 'General']

# Generate labeled training data
training_data = []

product_quality_texts = [
    "The product quality is excellent and durable",
    "Poor quality materials used in manufacturing",
    "Item works perfectly as described",
    "Product broke after first use"
]

customer_service_texts = [
    "Customer support was very helpful",
    "Support team resolved my issue quickly",
    "No response from customer service",
    "Staff was rude and unhelpful"
]

delivery_texts = [
    "Fast delivery and well packaged",
    "Package arrived damaged",
    "Delivery was delayed by two weeks",
    "Excellent shipping speed"
]

pricing_texts = [
    "Great value for money",
    "Too expensive for the quality",
    "Fair pricing compared to competitors",
    "Overpriced product"
]

general_texts = [
    "Overall good experience",
    "Would recommend to others",
    "Not satisfied with purchase",
    "Average product nothing special"
]

# Combine and create CSV
all_texts = (
    [(t, 'Product Quality') for t in product_quality_texts * 50] +
    [(t, 'Customer Service') for t in customer_service_texts * 50] +
    [(t, 'Delivery') for t in delivery_texts * 50] +
    [(t, 'Pricing') for t in pricing_texts * 50] +
    [(t, 'General') for t in general_texts * 50]
)

# Save to CSV (Comprehend format: label,text)
with open('classifier_training_data.csv', 'w', newline='', encoding='utf-8') as f:
    writer = csv.writer(f)
    for text, label in all_texts:
        writer.writerow([label, text])

print(f"Generated {len(all_texts)} training samples")

# Upload to S3
s3 = boto3.client('s3')
bucket = 'customer-feedback-analysis-YOUR-ACCOUNT-ID'

s3.upload_file('classifier_training_data.csv', bucket, 'classifier-training/training_data.csv')
print(f"Uploaded to s3://{bucket}/classifier-training/training_data.csv")
```
2. Run the script

#### Step 14: Create IAM Role for Comprehend
1. Navigate to **IAM** → **Roles**
2. Click **Create role**
3. Select **AWS service** → **Comprehend**
4. Add permission: `ComprehendFullAccess`
5. Add inline policy for S3 access:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket",
        "s3:PutObject"
      ],
      "Resource": [
        "arn:aws:s3:::customer-feedback-analysis-*",
        "arn:aws:s3:::customer-feedback-analysis-*/*"
      ]
    }
  ]
}
```
6. Role name: `Comprehend-Classifier-Role`
7. Click **Create role**

#### Step 15: Create Custom Classifier
1. Navigate to **Amazon Comprehend**
2. Click **Custom classification** in left sidebar
3. Click **Train classifier**
4. Configure:
   - **Classifier name:** `feedback-category-classifier`
   - **Language:** English
   - **Classifier mode:** Multi-class mode
   - **Data format:** CSV file
   - **Training data location:**
     - **S3 location:** `s3://customer-feedback-analysis-[account-id]/classifier-training/`
   - **IAM role:** `Comprehend-Classifier-Role`
   - **Output data location:** `s3://customer-feedback-analysis-[account-id]/classifier-output/`
5. Click **Train classifier**
6. Training takes 30-60 minutes

#### Step 16: Monitor Classifier Training
1. Go to **Comprehend** → **Custom classification**
2. Click on your classifier
3. Monitor **Status** (changes from Training → Trained)
4. Once trained, note the **Classifier ARN**

#### Step 17: Test Custom Classifier
1. Create test script `test_classifier.py`:

```python
import boto3

comprehend = boto3.client('comprehend')

classifier_arn = 'arn:aws:comprehend:us-east-1:ACCOUNT-ID:document-classifier/feedback-category-classifier'

test_texts = [
    "The product quality exceeded my expectations",
    "Delivery was very slow and package was damaged",
    "Customer service rep was extremely helpful",
    "Too expensive for what you get",
    "Overall satisfied with my purchase"
]

for text in test_texts:
    response = comprehend.classify_document(
        Text=text,
        EndpointArn=classifier_arn  # Or use ClassifierArn for real-time
    )
    
    print(f"\nText: {text}")
    print(f"Category: {response['Classes'][0]['Name']}")
    print(f"Confidence: {response['Classes'][0]['Score']:.2%}")
```

---

### Part 5: Build QuickSight Dashboard

#### Step 18: Set Up QuickSight
1. Navigate to **QuickSight**
2. If first time, click **Sign up for QuickSight**
3. Select **Standard edition**
4. Configure:
   - **QuickSight account name:** Choose unique name
   - **Email:** Your email
5. Grant S3 access to your feedback bucket
6. Click **Finish**

#### Step 19: Create Dataset from Athena
1. Click **Datasets** in left sidebar
2. Click **New dataset**
3. Select **Athena**
4. Configure:
   - **Data source name:** `feedback-analysis`
   - Click **Create data source**
5. Select database: `customer_feedback_db`
6. Select table: `analyzed_feedback`
7. Click **Select**
8. Choose **Directly query your data**
9. Click **Visualize**

#### Step 20: Create Visualizations
1. **Sentiment Distribution Pie Chart:**
   - Visual type: Pie chart
   - Group by: sentiment
   - Value: Count
   
2. **Feedback Over Time:**
   - Visual type: Line chart
   - X-axis: timestamp (aggregated by day)
   - Value: Count
   - Color: sentiment

3. **Average Rating by Sentiment:**
   - Visual type: Bar chart
   - Y-axis: sentiment
   - Value: rating (average)

4. **Channel Performance:**
   - Visual type: Stacked bar chart
   - Y-axis: channel
   - Value: Count
   - Group: sentiment

5. **Top Key Phrases (requires custom SQL):**
   - Create new dataset with custom SQL from Athena query
   - Visual type: Word cloud or bar chart

#### Step 21: Publish Dashboard
1. Click **Share** → **Publish dashboard**
2. Dashboard name: `Customer Feedback Analytics`
3. Click **Publish**
4. Share with team members as needed

---

### Verification Checklist

- [ ] S3 bucket created for feedback data
- [ ] Sample feedback data generated and uploaded
- [ ] Lambda function for Comprehend analysis deployed
- [ ] S3 trigger configured for automatic analysis
- [ ] Sentiment, key phrases, and entities extracted
- [ ] Glue database and crawler created
- [ ] Athena queries returning results
- [ ] Custom classifier training data prepared
- [ ] Custom Comprehend classifier trained successfully
- [ ] Custom classifier tested and working
- [ ] QuickSight dashboard created with visualizations
- [ ] Dashboard shows sentiment trends and insights

### Architecture Benefits

**Automated Text Analytics:**
- Real-time sentiment analysis on feedback
- Automatic key phrase and entity extraction
- Custom classification for business categories
- Scalable to millions of documents

**Business Insights:**
- Identify trending issues from key phrases
- Track sentiment over time
- Categorize feedback automatically
- Measure customer satisfaction by channel

**Cost Efficiency:**
- Pay per analysis (no idle infrastructure)
- Batch processing for cost optimization
- QuickSight serverless analytics
- Glue crawler automates schema discovery

---

### Cleanup

1. **Delete QuickSight Resources:**
   - QuickSight → Dashboards → Delete
   - QuickSight → Datasets → Delete

2. **Delete Custom Classifier:**
   - Comprehend → Custom classification → Delete

3. **Delete Lambda Function:**
   - Lambda → Functions → Delete

4. **Delete Glue Resources:**
   - Glue → Crawlers → Delete
   - Glue → Tables → Delete
   - Glue → Databases → Delete

5. **Delete S3 Bucket:**
   - S3 → Empty → Delete

6. **Delete IAM Roles:**
   - IAM → Roles → Delete created roles

---

## Exercise 12.3: Rekognition Media Metadata Extraction

### Introduction
Build an automated media analysis pipeline using Amazon Rekognition to extract metadata from images and videos. Detect objects, faces, text, celebrities, and implement content moderation.

**What You'll Build:**
- S3 event-driven architecture
- Lambda for Rekognition analysis
- DynamoDB for metadata storage
- API Gateway for search functionality
- Content moderation workflow with SNS alerts

**Duration:** 60 minutes  
**Cost:** ~$20-40/month

---

### Part 1: Configure S3 Event Triggers

#### Step 1: Create S3 Bucket for Media Files
1. Navigate to **S3**
2. Click **Create bucket**
3. Configure:
   - **Bucket name:** `media-analysis-[account-id]`
   - **Region:** us-east-1
   - **Block public access:** Keep checked
4. Click **Create bucket**

#### Step 2: Create Folder Structure
1. Click on the bucket
2. Click **Create folder**
3. Create folders:
   - `uploads/images/`
   - `uploads/videos/`
   - `processed/`

#### Step 3: Upload Sample Images
1. Download or create sample images (landscapes, people, text)
2. Upload to `uploads/images/` folder
3. Ensure variety: product photos, people, documents, scenes

---

### Part 2: Create DynamoDB Table for Metadata

#### Step 4: Create DynamoDB Table
1. Navigate to **DynamoDB**
2. Click **Create table**
3. Configure:
   - **Table name:** `MediaMetadata`
   - **Partition key:** `media_id` (String)
   - **Sort key:** `analysis_type` (String)
   - **Table settings:** Default settings
4. Click **Create table**

#### Step 5: Create GSI for Search
1. Click on the table
2. Click **Indexes** tab
3. Click **Create index**
4. Configure:
   - **Partition key:** `label` (String)
   - **Sort key:** `confidence` (Number)
   - **Index name:** `LabelIndex`
5. Click **Create index**

---

### Part 3: Create Lambda for Rekognition Analysis

#### Step 6: Create IAM Role for Rekognition Lambda
1. Navigate to **IAM** → **Roles**
2. Click **Create role**
3. Select AWS service → Lambda
4. Add permissions:
   - `AmazonS3FullAccess`
   - `AmazonRekognitionFullAccess`
   - `AmazonDynamoDBFullAccess`
   - `AmazonSNSFullAccess`
5. Role name: `Lambda-Rekognition-Execution-Role`
6. Click **Create role**

#### Step 7: Create Rekognition Analysis Lambda
1. Navigate to **Lambda**
2. Click **Create function**
3. Configure:
   - **Function name:** `analyze-media-rekognition`
   - **Runtime:** Python 3.11
   - **Execution role:** Use existing → `Lambda-Rekognition-Execution-Role`
4. Click **Create function**

#### Step 8: Add Lambda Code
1. Replace the code:

```python
import json
import boto3
from datetime import datetime
import uuid

s3 = boto3.client('s3')
rekognition = boto3.client('rekognition')
dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('MediaMetadata')

def lambda_handler(event, context):
    """
    Analyze media files with Amazon Rekognition
    """
    try:
        # Get S3 event details
        bucket = event['Records'][0]['s3']['bucket']['name']
        key = event['Records'][0]['s3']['object']['key']
        
        media_id = str(uuid.uuid4())
        
        # Determine media type
        is_image = key.lower().endswith(('.jpg', '.jpeg', '.png'))
        is_video = key.lower().endswith(('.mp4', '.mov'))
        
        results = {
            'media_id': media_id,
            'bucket': bucket,
            'key': key,
            'timestamp': datetime.now().isoformat()
        }
        
        if is_image:
            # Object and Scene Detection
            labels_response = rekognition.detect_labels(
                Image={'S3Object': {'Bucket': bucket, 'Name': key}},
                MaxLabels=10,
                MinConfidence=75
            )
            
            results['labels'] = [
                {'Name': label['Name'], 'Confidence': label['Confidence']}
                for label in labels_response['Labels']
            ]
            
            # Store each label in DynamoDB
            for label in results['labels']:
                table.put_item(Item={
                    'media_id': media_id,
                    'analysis_type': 'LABEL',
                    'label': label['Name'],
                    'confidence': int(label['Confidence']),
                    's3_bucket': bucket,
                    's3_key': key,
                    'timestamp': results['timestamp']
                })
            
            # Face Detection
            try:
                faces_response = rekognition.detect_faces(
                    Image={'S3Object': {'Bucket': bucket, 'Name': key}},
                    Attributes=['ALL']
                )
                
                results['faces'] = []
                for idx, face in enumerate(faces_response['FaceDetails']):
                    face_data = {
                        'FaceId': idx,
                        'Confidence': face['Confidence'],
                        'AgeRange': face.get('AgeRange', {}),
                        'Gender': face.get('Gender', {}),
                        'Emotions': face.get('Emotions', [])[:3],  # Top 3
                        'Smile': face.get('Smile', {}),
                        'Eyeglasses': face.get('Eyeglasses', {}),
                        'Beard': face.get('Beard', {})
                    }
                    results['faces'].append(face_data)
                    
                    # Store in DynamoDB
                    table.put_item(Item={
                        'media_id': media_id,
                        'analysis_type': 'FACE',
                        'label': f'Face_{idx}',
                        'confidence': int(face['Confidence']),
                        'metadata': json.dumps(face_data),
                        's3_bucket': bucket,
                        's3_key': key,
                        'timestamp': results['timestamp']
                    })
                    
            except Exception as e:
                print(f"Face detection error: {str(e)}")
                results['faces'] = []
            
            # Text Detection (OCR)
            try:
                text_response = rekognition.detect_text(
                    Image={'S3Object': {'Bucket': bucket, 'Name': key}}
                )
                
                results['text'] = [
                    {'Text': text['DetectedText'], 'Confidence': text['Confidence'], 'Type': text['Type']}
                    for text in text_response['TextDetections']
                    if text['Type'] == 'LINE'  # Only lines, not individual words
                ]
                
                # Store in DynamoDB
                for text_item in results['text']:
                    table.put_item(Item={
                        'media_id': media_id,
                        'analysis_type': 'TEXT',
                        'label': text_item['Text'],
                        'confidence': int(text_item['Confidence']),
                        's3_bucket': bucket,
                        's3_key': key,
                        'timestamp': results['timestamp']
                    })
                    
            except Exception as e:
                print(f"Text detection error: {str(e)}")
                results['text'] = []
            
            # Celebrity Recognition
            try:
                celebs_response = rekognition.recognize_celebrities(
                    Image={'S3Object': {'Bucket': bucket, 'Name': key}}
                )
                
                results['celebrities'] = [
                    {'Name': celeb['Name'], 'Confidence': celeb['MatchConfidence']}
                    for celeb in celebs_response['CelebrityFaces']
                ]
                
                # Store in DynamoDB
                for celeb in results['celebrities']:
                    table.put_item(Item={
                        'media_id': media_id,
                        'analysis_type': 'CELEBRITY',
                        'label': celeb['Name'],
                        'confidence': int(celeb['Confidence']),
                        's3_bucket': bucket,
                        's3_key': key,
                        'timestamp': results['timestamp']
                    })
                    
            except Exception as e:
                print(f"Celebrity recognition error: {str(e)}")
                results['celebrities'] = []
            
            # Content Moderation
            try:
                moderation_response = rekognition.detect_moderation_labels(
                    Image={'S3Object': {'Bucket': bucket, 'Name': key}},
                    MinConfidence=50
                )
                
                results['moderation'] = [
                    {'Name': label['Name'], 'Confidence': label['Confidence'], 
                     'ParentName': label.get('ParentName', '')}
                    for label in moderation_response['ModerationLabels']
                ]
                
                # Store in DynamoDB
                for mod_label in results['moderation']:
                    table.put_item(Item={
                        'media_id': media_id,
                        'analysis_type': 'MODERATION',
                        'label': mod_label['Name'],
                        'confidence': int(mod_label['Confidence']),
                        'parent_category': mod_label.get('ParentName', ''),
                        's3_bucket': bucket,
                        's3_key': key,
                        'timestamp': results['timestamp']
                    })
                
                # Alert if inappropriate content detected
                if results['moderation']:
                    send_moderation_alert(media_id, bucket, key, results['moderation'])
                    
            except Exception as e:
                print(f"Moderation error: {str(e)}")
                results['moderation'] = []
        
        # Store summary in DynamoDB
        table.put_item(Item={
            'media_id': media_id,
            'analysis_type': 'SUMMARY',
            'label': 'PROCESSED',
            'confidence': 100,
            's3_bucket': bucket,
            's3_key': key,
            'total_labels': len(results.get('labels', [])),
            'total_faces': len(results.get('faces', [])),
            'total_text': len(results.get('text', [])),
            'has_celebrities': len(results.get('celebrities', [])) > 0,
            'has_inappropriate_content': len(results.get('moderation', [])) > 0,
            'timestamp': results['timestamp']
        })
        
        return {
            'statusCode': 200,
            'body': json.dumps({
                'message': 'Analysis complete',
                'media_id': media_id,
                'results': results
            })
        }
        
    except Exception as e:
        print(f"Error: {str(e)}")
        import traceback
        traceback.print_exc()
        return {
            'statusCode': 500,
            'body': json.dumps({'error': str(e)})
        }

def send_moderation_alert(media_id, bucket, key, moderation_labels):
    """Send SNS alert for inappropriate content"""
    try:
        sns = boto3.client('sns')
        topic_arn = 'arn:aws:sns:us-east-1:YOUR-ACCOUNT-ID:content-moderation-alerts'
        
        message = f"""
        CONTENT MODERATION ALERT
        
        Media ID: {media_id}
        Location: s3://{bucket}/{key}
        
        Detected Issues:
        """
        
        for label in moderation_labels:
            message += f"\n- {label['Name']} ({label['Confidence']:.1f}% confidence)"
        
        sns.publish(
            TopicArn=topic_arn,
            Subject='Content Moderation Alert',
            Message=message
        )
    except Exception as e:
        print(f"SNS alert error: {str(e)}")
```
2. Click **Deploy**

#### Step 9: Configure Lambda Settings
1. **Configuration** → **General configuration**
2. Set **Timeout:** 5 minutes
3. Set **Memory:** 1024 MB
4. Click **Save**

#### Step 10: Add S3 Trigger
1. Click **Add trigger**
2. Select **S3**
3. Configure:
   - **Bucket:** `media-analysis-[account-id]`
   - **Event types:** All object create events
   - **Prefix:** `uploads/images/`
   - **Suffix:** `.jpg` (repeat for `.jpeg`, `.png`)
4. Click **Add**

---

### Part 4: Create SNS Topic for Content Moderation

#### Step 11: Create SNS Topic
1. Navigate to **SNS**
2. Click **Topics** → **Create topic**
3. Configure:
   - **Type:** Standard
   - **Name:** `content-moderation-alerts`
4. Click **Create topic**
5. Note the **Topic ARN**

#### Step 12: Create Email Subscription
1. Click **Create subscription**
2. Configure:
   - **Protocol:** Email
   - **Endpoint:** Your email address
3. Click **Create subscription**
4. Check email and confirm subscription

#### Step 13: Update Lambda with SNS ARN
1. Go back to Lambda function
2. Update the `topic_arn` in `send_moderation_alert` function
3. Click **Deploy**

---

### Part 5: Build Search API with API Gateway

#### Step 14: Create Lambda for Search
1. Navigate to **Lambda**
2. Click **Create function**
3. Configure:
   - **Function name:** `search-media-metadata`
   - **Runtime:** Python 3.11
   - **Execution role:** Use existing → `Lambda-Rekognition-Execution-Role`
4. Add code:

```python
import json
import boto3
from boto3.dynamodb.conditions import Key

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('MediaMetadata')

def lambda_handler(event, context):
    """
    Search media by labels, text, or faces
    """
    try:
        # Get query parameters
        params = event.get('queryStringParameters', {})
        search_type = params.get('type', 'label')  # label, text, face, celebrity
        search_query = params.get('query', '')
        min_confidence = int(params.get('min_confidence', 75))
        
        if not search_query:
            return {
                'statusCode': 400,
                'headers': {'Content-Type': 'application/json'},
                'body': json.dumps({'error': 'Query parameter required'})
            }
        
        # Search using GSI
        response = table.query(
            IndexName='LabelIndex',
            KeyConditionExpression=Key('label').eq(search_query) & 
                                  Key('confidence').gte(min_confidence)
        )
        
        results = response.get('Items', [])
        
        # Group by media_id
        media_results = {}
        for item in results:
            media_id = item['media_id']
            if media_id not in media_results:
                media_results[media_id] = {
                    'media_id': media_id,
                    's3_location': f"s3://{item['s3_bucket']}/{item['s3_key']}",
                    'matches': []
                }
            media_results[media_id]['matches'].append({
                'type': item['analysis_type'],
                'label': item['label'],
                'confidence': item['confidence']
            })
        
        return {
            'statusCode': 200,
            'headers': {'Content-Type': 'application/json'},
            'body': json.dumps({
                'query': search_query,
                'total_results': len(media_results),
                'results': list(media_results.values())
            })
        }
        
    except Exception as e:
        print(f"Error: {str(e)}")
        return {
            'statusCode': 500,
            'headers': {'Content-Type': 'application/json'},
            'body': json.dumps({'error': str(e)})
        }
```
5. Click **Deploy**

#### Step 15: Create API Gateway
1. Navigate to **API Gateway**
2. Click **Create API**
3. Select **REST API** (not private) → **Build**
4. Configure:
   - **Protocol:** REST
   - **Create new API:** New API
   - **API name:** `MediaSearchAPI`
   - **Endpoint Type:** Regional
5. Click **Create API**

#### Step 16: Create Search Resource and Method
1. Click **Actions** → **Create Resource**
2. Configure:
   - **Resource Name:** `search`
   - **Resource Path:** `/search`
3. Click **Create Resource**
4. Select `/search` resource
5. Click **Actions** → **Create Method** → Select **GET**
6. Configure integration:
   - **Integration type:** Lambda Function
   - **Lambda Function:** `search-media-metadata`
7. Click **Save** → **OK**

#### Step 17: Enable CORS
1. Select `/search` resource
2. Click **Actions** → **Enable CORS**
3. Click **Enable CORS and replace existing CORS headers**
4. Click **Yes, replace existing values**

#### Step 18: Deploy API
1. Click **Actions** → **Deploy API**
2. **Deployment stage:** [New Stage]
3. **Stage name:** `prod`
4. Click **Deploy**
5. Copy the **Invoke URL** (e.g., `https://xxxxx.execute-api.us-east-1.amazonaws.com/prod`)

#### Step 19: Test Search API
1. Test in browser or curl:
```bash
curl "https://YOUR-API-ID.execute-api.us-east-1.amazonaws.com/prod/search?query=Dog&min_confidence=80"
```
2. Or create a simple HTML test page:

```html
<!DOCTYPE html>
<html>
<head>
    <title>Media Search</title>
</head>
<body>
    <h1>Search Media Metadata</h1>
    <input type="text" id="searchQuery" placeholder="Enter label (e.g., Dog, Person)">
    <button onclick="searchMedia()">Search</button>
    <div id="results"></div>
    
    <script>
        async function searchMedia() {
            const query = document.getElementById('searchQuery').value;
            const apiUrl = 'https://YOUR-API-ID.execute-api.us-east-1.amazonaws.com/prod/search';
            
            const response = await fetch(`${apiUrl}?query=${query}&min_confidence=75`);
            const data = await response.json();
            
            document.getElementById('results').innerHTML = 
                '<pre>' + JSON.stringify(data, null, 2) + '</pre>';
        }
    </script>
</body>
</html>
```

---

### Part 6: Test Complete Workflow

#### Step 20: Upload Test Images and Verify
1. Upload various images to `s3://media-analysis-[account-id]/uploads/images/`
2. Wait 30-60 seconds for processing
3. Check DynamoDB table for metadata entries
4. Test search API with detected labels
5. If inappropriate content uploaded, verify SNS alert received

#### Step 21: Create Summary Dashboard Lambda
1. Create one more Lambda for analytics:

```python
import json
import boto3
from collections import Counter

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('MediaMetadata')

def lambda_handler(event, context):
    """
    Get analytics summary of processed media
    """
    # Scan table for summaries
    response = table.scan(
        FilterExpression='analysis_type = :type',
        ExpressionAttributeValues={':type': 'SUMMARY'}
    )
    
    items = response['Items']
    
    # Calculate statistics
    total_media = len(items)
    total_with_faces = sum(1 for item in items if item.get('total_faces', 0) > 0)
    total_with_text = sum(1 for item in items if item.get('total_text', 0) > 0)
    total_with_celebs = sum(1 for item in items if item.get('has_celebrities'))
    total_flagged = sum(1 for item in items if item.get('has_inappropriate_content'))
    
    # Get most common labels
    all_labels = table.scan(
        FilterExpression='analysis_type = :type',
        ExpressionAttributeValues={':type': 'LABEL'}
    )
    
    label_counts = Counter([item['label'] for item in all_labels['Items']])
    
    return {
        'statusCode': 200,
        'body': json.dumps({
            'total_media_processed': total_media,
            'media_with_faces': total_with_faces,
            'media_with_text': total_with_text,
            'media_with_celebrities': total_with_celebs,
            'media_flagged': total_flagged,
            'top_labels': dict(label_counts.most_common(10))
        })
    }
```

---

### Verification Checklist

- [ ] S3 bucket created for media storage
- [ ] DynamoDB table created for metadata
- [ ] Lambda function for Rekognition analysis deployed
- [ ] S3 trigger configured for automatic processing
- [ ] Labels detected and stored in DynamoDB
- [ ] Face detection working with attributes
- [ ] Text detection (OCR) extracting text from images
- [ ] Celebrity recognition identifying famous people
- [ ] Content moderation flagging inappropriate content
- [ ] SNS alerts sent for moderation issues
- [ ] Search API created with API Gateway
- [ ] Search functionality working for labels
- [ ] Analytics summary available

### Architecture Benefits

**Automated Media Intelligence:**
- Event-driven processing (no polling)
- Multiple analysis types in parallel
- Scalable to millions of images
- Real-time metadata extraction

**Content Safety:**
- Automatic moderation of uploads
- Instant alerts for violations
- Confidence thresholds configurable
- Compliance-ready architecture

**Searchability:**
- Rich metadata for every image
- Fast lookups via DynamoDB GSI
- RESTful API for integrations
- Support for complex queries

**Use Cases:**
- Digital asset management
- E-commerce product tagging
- Social media content moderation
- Surveillance and security
- Media library organization

---

### Cleanup

1. **Delete API Gateway:**
   - API Gateway → APIs → Delete

2. **Delete Lambda Functions:**
   - Lambda → Functions → Delete all three functions

3. **Delete SNS Topic:**
   - SNS → Topics → Delete

4. **Delete DynamoDB Table:**
   - DynamoDB → Tables → Delete

5. **Delete S3 Bucket:**
   - S3 → Empty → Delete

6. **Delete IAM Roles:**
   - IAM → Roles → Delete created roles

---

## Summary

### What You've Learned

**Exercise 12.1 - SageMaker Churn Prediction:**
- Built end-to-end ML pipeline from data to deployment
- Performed feature engineering with SageMaker Processing
- Optimized model with hyperparameter tuning
- Deployed production endpoint with auto-scaling
- Created batch prediction workflow
- Automated retraining with EventBridge

**Exercise 12.2 - Comprehend Text Analytics:**
- Extracted sentiment from customer feedback
- Identified key phrases and entities automatically
- Built Glue catalog for structured text data
- Queried insights with Athena SQL
- Trained custom classifier for categorization
- Visualized trends in QuickSight dashboard

**Exercise 12.3 - Rekognition Media Analysis:**
- Processed images with multiple Rekognition APIs
- Detected objects, faces, text, and celebrities
- Implemented content moderation workflow
- Stored metadata in DynamoDB for fast retrieval
- Built search API with API Gateway
- Created event-driven processing pipeline

### Key Takeaways

1. **SageMaker provides complete MLOps platform** - from notebooks to production endpoints with built-in scaling

2. **Comprehend simplifies NLP** - sentiment, entities, and custom classification without ML expertise

3. **Rekognition enables vision AI** - multiple detection types with single API call, production-ready accuracy

4. **Event-driven ML workflows** - S3 triggers + Lambda enable automatic processing at scale

5. **Metadata is valuable** - extracted insights enable search, analytics, and business intelligence

### Real-World Applications

**E-commerce:**
- Product recommendations based on churn prediction
- Automated product image tagging
- Review sentiment analysis for quality monitoring

**Media & Entertainment:**
- Content moderation for user uploads
- Celebrity detection in photos/videos
- Automated video cataloging and search

**Customer Service:**
- Sentiment tracking across feedback channels
- Issue categorization with custom classifiers
- Proactive churn prevention campaigns

**Healthcare:**
- Medical image analysis and classification
- Patient feedback sentiment monitoring
- Document text extraction (OCR)

### Cost Optimization Tips

1. **Use batch transform for large-scale predictions** instead of real-time endpoints (10x cheaper)
2. **Stop/delete SageMaker endpoints** when not in use (biggest cost driver)
3. **Use Comprehend async APIs** for large documents (lower cost)
4. **Cache Rekognition results** in DynamoDB to avoid re-analysis
5. **Leverage free tiers**: 
   - SageMaker: 250 hours/month (2 months)
   - Comprehend: 50K units/month (12 months)
   - Rekognition: 5,000 images/month (12 months)

### Performance Benchmarks

**SageMaker:**
- Training job: 5-20 minutes for typical datasets
- Hyperparameter tuning: 30-120 minutes (10 jobs)
- Real-time inference: <100ms p99 latency
- Batch transform: 1000s of predictions/minute

**Comprehend:**
- Sentiment analysis: <1 second per document
- Entity detection: <2 seconds per document
- Custom classification: 30-60 minutes training time

**Rekognition:**
- Label detection: <500ms per image
- Face analysis: <1 second per image
- Text detection: <1 second per image
- Video analysis: 1-2x video duration

---

## Exam Alignment

**DEA-C01 Coverage:**

**Domain 1: Data Ingestion and Transformation (34%)**
- Feature engineering pipelines
- Data preprocessing for ML
- Text and image data transformation

**Domain 2: Data Store Management (26%)**
- DynamoDB for ML metadata
- S3 for training data and models
- Glue catalog for ML datasets

**Domain 3: Data Operations and Support (22%)**
- SageMaker endpoint monitoring
- Automated retraining workflows
- ML pipeline orchestration

**Domain 4: Data Security and Governance (18%)**
- Content moderation and compliance
- IAM roles for ML services
- Data encryption for sensitive ML data

### Practice Scenarios

**Scenario 1:** A company needs to predict customer churn with 85%+ accuracy. Which AWS services would you use and why?
- Answer: SageMaker for model training, hyperparameter tuning for optimization, real-time endpoint for predictions

**Scenario 2:** You need to analyze 10,000 customer reviews daily for sentiment and categorize by topic. What's the most cost-effective approach?
- Answer: Comprehend async API for sentiment, custom classifier for topics, Lambda for orchestration, Athena for analysis

**Scenario 3:** A social media platform needs to moderate user-uploaded images in real-time. How would you architect this?
- Answer: S3 events → Lambda → Rekognition moderation → DynamoDB → SNS alerts for violations

---

**Total Cost Summary:**
- Exercise 12.1: $50-100/month (endpoint hosting is main cost)
- Exercise 12.2: $30-50/month (Comprehend API + QuickSight)
- Exercise 12.3: $20-40/month (Rekognition API + DynamoDB)
- **Total: $100-200/month** (delete endpoints when not in use to save 80%)

**Estimated Time Investment:** 4 hours (90 + 75 + 60 minutes)

**Free Tier Eligible:** Yes (first 12 months for Comprehend/Rekognition, 2 months for SageMaker)

---

*This guide is part of the AWS Data Engineer Associate certification preparation series. Practice these exercises in your own AWS account to gain hands-on experience with machine learning services for data engineering.*

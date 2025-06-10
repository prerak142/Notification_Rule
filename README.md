Climate Resilient Farms of the Future: Notification Rules Engine
Overview
This project builds a weather monitoring and notification system for farms, using AWS services to ingest weather data, evaluate rules, and send alerts. It supports farms in Udaipur (udaipur_farm1) and other locations, leveraging multiple weather APIs (OpenWeather, WeatherAPI, Yr.no, Open-Meteo) and AWS Free Tier resources.
Components

WeatherDataIngestion Lambda (new): Fetches current and forecast weather data from multiple APIs and stores it in PostgreSQL RDS.
RulesEngine Lambda (alert): Evaluates rules stored in DynamoDB against weather data and sends notifications via SNS.
RuleApiLambda: Provides an API to create and manage rules via API Gateway.
PostgreSQL RDS: Stores weather data (weather-db.cfaciysiukn7.ap-south-1.rds.amazonaws.com).
DynamoDB: Stores rules (WeatherRules table).
SNS: Sends email and SMS notifications (arn:aws:sns:ap-south-1:580075786360:weather-alerts).
API Gateway: Exposes the rule creation API (https://9qzfpocell.execute-api.ap-south-1.amazonaws.com/prod/rules).

Project Structure
climate-resilient-farms/
├── lambda/
│   ├── weather_data_ingestion/
│   │   ├── weather_data_ingestion.py
│   │   └── requirements.txt
│   ├── rules_engine/
│   │   ├── rules_engine.py
│   │   └── requirements.txt
│   └── rule_api_lambda/
│       ├── rule_api_lambda.py
│       └── requirements.txt
├── db_schema/
│   ├── postgresql_schema.sql
│   └── dynamodb_schema.json
├── configuration.txt
├── template.yaml
├── README.md

Configuration Details

configuration.txt: Describes the architecture of the WeatherDataIngestion and RulesEngine Lambdas, including runtime, dependencies, layers, environment variables, and interactions with AWS services.

Database Schema
PostgreSQL (RDS)
The PostgreSQL database has two tables: current_weather and forecast_weather, with indexes for efficient querying.
current_weather

id: SERIAL PRIMARY KEY
source: TEXT NOT NULL (e.g., 'openweather')
farm_id: TEXT NOT NULL (e.g., 'udaipur_farm1')
location: GEOGRAPHY(Point, 4326) NOT NULL (latitude, longitude)
timestamp: TIMESTAMPTZ NOT NULL (when the data was recorded)
temperature_c: REAL (temperature in Celsius)
humidity_percent: REAL (humidity percentage)
wind_speed_mps: REAL (wind speed in meters per second)
wind_direction_deg: REAL (wind direction in degrees)
rainfall_mm: REAL (rainfall in millimeters)
solar_radiation_wm2: REAL (solar radiation in watts per square meter)

Indexes:

idx_current_weather_farm: On farm_id
idx_current_weather_time: On timestamp
idx_current_weather_location: On location (using GIST)
idx_current_source_time: On (source, timestamp)
Unique constraint: (farm_id, source, timestamp)

forecast_weather

id: SERIAL PRIMARY KEY
source: TEXT NOT NULL (e.g., 'openweather')
farm_id: TEXT NOT NULL (e.g., 'udaipur_farm1')
forecast_for: TIMESTAMPTZ NOT NULL (forecast timestamp)
fetched_at: TIMESTAMPTZ NOT NULL (when the forecast was fetched)
location: GEOGRAPHY(Point, 4326) NOT NULL (latitude, longitude)
temperature_c: REAL (forecasted temperature in Celsius)
humidity_percent: REAL (forecasted humidity percentage)
wind_speed_mps: REAL (forecasted wind speed in meters per second)
wind_direction_deg: REAL (forecasted wind direction in degrees)
rainfall_mm: REAL (forecasted rainfall in millimeters)
chance_of_rain_percent: REAL (chance of rain percentage)

Indexes:

idx_forecast_weather_farm: On farm_id
idx_forecast_weather_time: On forecast_for
idx_forecast_weather_location: On location (using GIST)
idx_forecast_source_time: On (source, fetched_at)
Unique constraint: (farm_id, source, forecast_for)

Schema File: See db_schema/postgresql_schema.sql for the full schema.
DynamoDB (WeatherRules)
The WeatherRules table stores rules for weather conditions and actions.

Partition Key: farm_id (String)
Sort Key: stakeholder (String)
Attributes:
rule_id: String (unique identifier for the rule)
name: String (rule name)
priority: String (rule priority for ordering)
data_type: String (e.g., 'forecast' or 'current')
conditions: Map (nested conditions with operators like AND, OR, RATE>, DAY_DIFF>)
actions: List (list of actions like email or SMS notifications)
stop_on_match: Boolean (whether to stop evaluating further rules)



Global Secondary Index:

StakeholderIndex: (farm_id, stakeholder)

Schema File: See db_schema/dynamodb_schema.json for the table definition.
Configuration
Environment Variables
The Lambdas require the following environment variables:
WeatherDataIngestion (new)

OPENWEATHER_API_KEY: API key for OpenWeather (get from https://openweathermap.org/)
WEATHERAPI_API_KEY: API key for WeatherAPI (get from https://www.weatherapi.com/)
OPEN_METEO_URL: Open-Meteo API URL (e.g., https://api.open-meteo.com/v1/forecast)
YR_NO_URL: Yr.no API URL (e.g., https://api.met.no/weatherapi/locationforecast/2.0/compact?lat=26.9124&lon=75.7873)
DB_HOST: RDS endpoint (e.g., weather-db.cfaciysiukn7.ap-south-1.rds.amazonaws.com)
DB_PORT: RDS port (default: 5432)
DB_NAME: Database name (e.g., postgres)
DB_USER: Database user (e.g., postgres)
DB_PASS: Database password (store securely, e.g., in AWS Secrets Manager)
SNS_TOPIC_ARN: SNS topic ARN (e.g., arn:aws:sns:ap-south-1:580075786360:weather-alerts)

RulesEngine (alert)

DB_HOST, DB_PORT, DB_NAME, DB_USER, DB_PASS: Same as above
API_KEY: Custom API key (purpose unclear, possibly for an external service)
RULE_ID: Specific rule identifier (e.g., 872e9c71-2a16-b908-6b3a-7cc3af9c5b06/1)

RuleApiLambda

No additional environment variables required.

AWS Resources

RDS: db.t3.micro instance, <20 GB storage.
DynamoDB: WeatherRules table, <25 GB storage, 1 RCU/WCU.
Lambda: 128 MB memory, <1M requests/month.
SNS: <1,000 notifications/month.
API Gateway: <1M requests/month.
S3: Used for caching (optional), <5 GB storage.

Setup Instructions
Prerequisites

AWS account with Free Tier access.
git installed on your local machine or AWS CloudShell.
Python 3.12 for Lambda runtime (local installation for dependency management).
AWS CLI configured with appropriate permissions.
AWS SAM CLI for deploying the SAM template (optional, if using SAM for deployment).

Steps

Clone the Repository:
git clone https://github.com/<your-username>/climate-resilient-farms.git
cd climate-resilient-farms


Set Up the Database:

PostgreSQL RDS:
Create an RDS instance in ap-south-1 (db.t3.micro, PostgreSQL 15+).
Connect to the database:psql -h <rds-endpoint> -U postgres -d postgres


Run the schema script:psql -h <rds-endpoint> -U postgres -d postgres -f db_schema/postgresql_schema.sql




DynamoDB:
Create the WeatherRules table using the AWS CLI:aws dynamodb create-table --cli-input-json file://db_schema/dynamodb_schema.json --region ap-south-1






Prepare the Lambdas:

Install dependencies for each Lambda (excluding psycopg2, which is provided by the layer):pip install -r lambda/weather_data_ingestion/requirements.txt -t lambda/weather_data_ingestion/
pip install -r lambda/rules_engine/requirements.txt -t lambda/rules_engine/
pip install -r lambda/rule_api_lambda/requirements.txt -t lambda/rule_api_lambda/


Zip each Lambda directory:cd lambda/weather_data_ingestion
zip -r ../weather_data_ingestion.zip .
cd ../rules_engine
zip -r ../rules_engine.zip .
cd ../rule_api_lambda
zip -r ../rule_api_lambda.zip .
cd ../..




Deploy Using AWS SAM (Recommended):

Use the provided SAM template (template.yaml) to deploy the Lambdas:sam deploy --template-file template.yaml --stack-name climate-resilient-farms --region ap-south-1 --capabilities CAPABILITY_IAM


Note: The SAM template already includes the layers:
arn:aws:lambda:ap-south-1:770693421928:layer:Klayers-p312-psycopg2-binary:1 (provides psycopg2)
arn:aws:lambda:ap-south-1:336392948345:layer:AWSSDKPandas-Python312:16 (provides boto3 and related libraries)




Alternative: Deploy Using AWS CLI:

If not using SAM, deploy each Lambda manually:aws lambda create-function --function-name WeatherDataIngestion \
  --zip-file fileb://lambda/weather_data_ingestion.zip \
  --handler weather_data_ingestion.lambda_handler \
  --runtime python3.12 \
  --role arn:aws:iam::<account-id>:role/lambda-execution-role \
  --region ap-south-1 \
  --layers arn:aws:lambda:ap-south-1:770693421928:layer:Klayers-p312-psycopg2-binary:1 arn:aws:lambda:ap-south-1:336392948345:layer:AWSSDKPandas-Python312:16

aws lambda create-function --function-name RulesEngine \
  --zip-file fileb://lambda/rules_engine.zip \
  --handler rules_engine.lambda_handler \
  --runtime python3.12 \
  --role arn:aws:iam::<account-id>:role/lambda-execution-role \
  --region ap-south-1 \
  --layers arn:aws:lambda:ap-south-1:770693421928:layer:Klayers-p312-psycopg2-binary:1 arn:aws:lambda:ap-south-1:336392948345:layer:AWSSDKPandas-Python312:16

aws lambda create-function --function-name RuleApiLambda \
  --zip-file fileb://lambda/rule_api_lambda.zip \
  --handler rule_api_lambda.lambda_handler \
  --runtime python3.12 \
  --role arn:aws:iam::<account-id>:role/lambda-execution-role \
  --region ap-south-1 \
  --layers arn:aws:lambda:ap-south-1:770693421928:layer:Klayers-p312-psycopg2-binary:1 arn:aws:lambda:ap-south-1:336392948345:layer:AWSSDKPandas-Python312:16


Set environment variables for each Lambda in the AWS Console (as listed in the SAM template).


Set Up API Gateway:

Create an API Gateway REST API.
Add a POST /rules endpoint linked to RuleApiLambda.
Deploy the API and note the URL (e.g., https://9qzfpocell.execute-api.ap-south-1.amazonaws.com/prod/rules).


Set Up SNS:

Create an SNS topic (weather-alerts) and note the ARN.
Subscribe email or SMS endpoints to the topic.


Test the System:

Invoke WeatherDataIngestion to fetch weather data.
Create a rule via API Gateway:curl -X POST https://<api-gateway-url>/prod/rules \
  -H "Content-Type: application/json" \
  -d '{"farm_id": "udaipur_farm1", "stakeholder": "field", "rule": {"name": "Test Rule", "priority": "10", "data_type": "forecast", "conditions": {"metric": "temperature_c", "operator": ">", "value": 35}, "actions": [{"type": "email", "message": "High temperature alert"}]}}'


Invoke RulesEngine to evaluate rules and send notifications.




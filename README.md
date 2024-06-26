[![Deploy with Vercel](https://vercel.com/button)](https://vercel.com/new/clone?repository-url=https%3A%2F%2Fgithub.com%2Fruf-io%2Fneon-cold-start&env=CONNECTION_STRING&envDescription=Connection%20string%20returned%20by%20the%20setup%20step)

This is a [Neon](http://neon.tech) tool to benchmark cold starts.

## Getting Started

1. Install the dependencies:
    ```bash
    npm install
    ```
2. Create an `.env` file using `.env.example` as template.
3. Setup the project:
    ```bash
    npm run setup
    ```
4. Run a single benchmark:
    ```bash
    npm run benchmark
    ```
5. Run the app:
    ```bash
    npm run serve
    ```
6. Open [http://localhost:3000](http://localhost:3000) with your browser to view the results. 

### Benchmark recurrently

The following command will configure AWS to schedule a benchmark using Lambda every 30 minutes:

```bash
# Make sure to have installed and configured the latest AWS CLI (https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html).
npm run deploy
```

<details>
<summary>Instructions</summary>
<br>

```bash
# 1. Load env. vars:
source .env

# 2. Zip the code:
zip -j lambda.zip ./setup/index.js && zip -j lambda.zip ./setup/config.json && zip -rq lambda.zip node_modules -x "*next*" -x "typescript" -x "*chartjs*"

# 3. Create a role and attach the policy:
ROLE_ARN=$(aws iam create-role --role-name neon-benchmark-lambda-execute-role --assume-role-policy-document file://setup/trust-policy.json --query 'Role.Arn' --output text)
aws iam attach-role-policy --role-name neon-benchmark-lambda-execute-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaRole
aws iam attach-role-policy --role-name neon-benchmark-lambda-execute-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

# 4. Upload the lambda code:
LAMBDA_ARN=$(aws lambda create-function --function-name BenchmarkRunner --runtime nodejs20.x --role $ROLE_ARN --handler index.handler --timeout 240 --zip-file fileb://lambda.zip --query 'FunctionArn' --output text --environment Variables={API_KEY=$API_KEY})

# 5. Schedule every 30 minutes:
aws scheduler create-schedule \
    --name NeonColdBenchmarkScheduler \
    --schedule-expression "rate(30 minutes)" \
    --target "{\"RoleArn\": \"$ROLE_ARN\", \"Arn\":\"$LAMBDA_ARN\" }" \
    --flexible-time-window '{ "Mode": "FLEXIBLE", "MaximumWindowInMinutes": 15}'
```
</details>

This will generate enough datapoints throughout the day to calculate the average time for your own cold starts.

## The problem

Neon suspends the database compute to save resources after five minutes of inactivity. The next time the database needs to process a query, the compute will need to resume. This is known as the cold-start problem. Understanding how much time takes the cold-start is the idea for this project.

## Setup

The setup command (`npm run setup`) will create a new Neon project with multiple branches configured in the `/setup/config.json` file. The project's _main_ branch will store the benchmark results, while the other branches will be used just for benchmarking.

## Benchmark

The benchmark is a Lambda function that suspends the compute resources of a branch and runs a benchmark query using `pg`, a [Node.JS Postgres client](https://github.com/brianc/node-postgres). The benchmark takes between two and three minutes. An example of a branch using the TimescaleDB extension would be as follows:

```json
{
    "name": "Timescale",
    "description": "Contains the TimescaleDB extension installed.",
    "setupQueries": [
        "CREATE EXTENSION \"timescaledb\";",
        "CREATE TABLE IF NOT EXISTS series (serie_num INT);",
        "INSERT INTO series VALUES (generate_series(0, 1000));",
        "CREATE INDEX IF NOT EXISTS series_idx ON series (serie_num);"
    ],
    "benchmarkQuery": "SELECT * FROM series WHERE serie_num = 10;"
}
```

After the benchmark, the results are stored in the _main_ branch in the following table:

```sql
CREATE TABLE IF NOT EXISTS benchmarks (
    id TEXT, -- Benchmark branch ID.
    initial_timestamp TIMESTAMP, -- Useful to group benchmarks from the same run.
    duration INT, -- Benchmark duration expressed in ms. 
    ts TIMESTAMP -- Benchmark timestamp.
);
```

## Application

The web app will query the benchmarks stored in the _main_ branch, calculate basic metrics (average, maximum, minimum), and display them on a chart to give an overview of the cold start durations.

## Learn More

- [Neon Documentation](https://neon.tech/docs/introduction).
- [Cold starts](https://neon.tech/blog/cold-starts-just-got-hot).

Your feedback and contributions are welcome!
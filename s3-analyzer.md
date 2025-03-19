### Automating the weekly audit of publicly accessible S3 buckets.

![devsecops-on-steroids](images/s3-analyzer/0.png)

[This tutorial on my blog](https://igorzhivilo.com//2025/03/19/s3-analyzer/)


#### Why is a weekly review of the company’s publicly accessible S3 buckets necessary?

To address this question, I will outline the top five risks associated with exposing a public S3 bucket.
#### Top 5 Risks of Exposing a Public S3 Bucket
1. **Data Breaches** – Unauthorized users can access sensitive data, leading to leaks of personal, financial, or proprietary information.
2. **Malware Distribution** – Attackers can upload and serve malicious files from the bucket, tricking users into downloading compromised content.
3. **Data Tampering** – If write permissions are misconfigured, attackers can modify or delete files, leading to data integrity issues.
4. **Credential Exposure** – Publicly accessible configuration files, API keys, or credentials stored in the bucket can be exploited for further attacks.
5. **Compliance Violations** – Exposing regulated data (e.g., PII, financial records) can result in legal penalties and non-compliance with industry standards like GDPR, HIPAA, or PCI DSS.

Here are some notable examples of companies that suffered data breaches due to misconfigured public S3 buckets:

1. **Capital One (2019)** – A misconfigured S3 bucket exposed sensitive data of over 100 million customers, including Social Security numbers and bank account details.
2. **Facebook (2019)** – Third-party app developers left **540 million** Facebook user records exposed in public S3 buckets, including account names and comments.
3. **Verizon (2017)** – A security lapse by a third-party vendor exposed **14 million** Verizon customer records, including phone numbers and account PINs.
4. **Accenture (2017)** – A publicly accessible S3 bucket contained **sensitive internal data**, including passwords and secret API keys, posing a major risk.
5. **GoDaddy (2018)** – Public S3 bucket misconfiguration exposed **configuration files**, putting millions of customers at risk of service disruption and phishing attacks.


These incidents highlight the importance of **regularly reviewing S3 bucket permissions** to prevent unauthorized data access. 🚨

#### Implementation Plan:
1. Schedule a Kubernetes job to run weekly.
2. Use AWS IAM Analyzer to detect publicly accessible S3 buckets.
3. Deliver the results to a designated Slack channel.

##### 1. Create the Trusted Policy JSON File
First, create a trusted policy that allows a Kubernetes Service Account to assume this role.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::<AWS_ACCOUNT_ID>:oidc-provider/<OIDC_PROVIDER>"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "<OIDC_PROVIDER>:sub": "system:serviceaccount:s3-analyzer:s3-analyzer-sa"
                }
            }
        }
    ]
}
```

Replace:
* <AWS_ACCOUNT_ID> with your AWS account ID.
* <OIDC_PROVIDER> with your EKS OIDC provider URL (find it using aws eks describe-cluster --name <CLUSTER_NAME> --query "cluster.identity.oidc.issuer" --output text).
* Ensure the serviceaccount matches the one used in your Kubernetes job.


##### 2. Create the IAM Role (s3-analyzer-role)
Run the following command to create the IAM role with the trust policy:

```bash
aws iam create-role \
    --role-name s3-analyzer-role \
    --assume-role-policy-document file://trust-policy.json
```

✔️ This creates the role and applies the trust policy, allowing Kubernetes workloads to assume it.

##### 3. Create IAM Permissions for the Role

Now, create an IAM policy that grants access to IAM Access Analyzer and S3 permissions.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "access-analyzer:ListFindings",
                "access-analyzer:GetFinding"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:ListAllMyBuckets",
                "s3:GetBucketAcl"
            ],
            "Resource": "*"
        }
    ]
}
```

##### 4. Create and Attach the Policy
* Create the policy:

```bash
aws iam create-policy --policy-name s3-analyzer-policy --policy-document file://iam-policy.json
```

This returns a policy ARN (e.g., arn:aws:iam::<AWS_ACCOUNT_ID>:policy/s3-analyzer-policy).

* Attach the policy to the role:

```bash
aws iam attach-role-policy \
    --role-name s3-analyzer-role \
    --policy-arn arn:aws:iam::<AWS_ACCOUNT_ID>:policy/s3-analyzer-policy
```

##### 5. Verify the Role

To confirm the role and its permissions, run:

```bash
aws iam get-role --role-name s3-analyzer-role
aws iam list-attached-role-policies --role-name s3-analyzer-role
```

Now, your Kubernetes CronJob can assume the IAM role and run AWS Access Analyzer to check for public S3 buckets.

##### 6. Create a Kubernetes CronJob
The following Kubernetes CronJob will execute a script once a week to run IAM Access Analyzer and send the results to a Slack channel.

k8s-cronjob.yaml

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: s3-analyzer
  namespace: s3-analyzer
spec:
  schedule: "0 0 * * 0"  # Runs every Sunday at midnight UTC
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: s3-analyzer-sa  # Ensure this SA has IAM permissions
          restartPolicy: Never
          tolerations:
          - key: CriticalAddonsOnly
            operator: Exists
          containers:
          - name: iam-analyzer
            image: amazon/aws-cli:latest
            command:
              - /bin/sh
              - -c
              - |
                echo "Running AWS IAM Access Analyzer..."

                # Run the analyzer to list public buckets
                aws accessanalyzer list-findings \
                  --analyzer-arn "$ANALYZER_ARN" \
                  --filter '{"resourceType": {"eq": ["AWS::S3::Bucket"]}, "isPublic": {"eq": ["true"]}}' \
                  --region us-east-1 \
                  --query "findings[*].resource" > findings.json

                # Send findings to Slack if there are public buckets
                if [ -s findings.json ]; then
                  # install jq
                  yum install -y jq
                  jq --version
                  
                  export BUCKETS=$(cat findings.json | jq -r '. | join("\n")')
                  echo "\nPublic buckets discovered: $BUCKETS\nSending results to Slack..."

                  curl -X POST -H 'Content-type: application/json' \
                    --data "{\"text\": \"AWS IAM Access Analyzer detected public S3 buckets:\n$BUCKETS\"}" \
                    $SLACK_WEBHOOK_URL
                else
                  echo "No public S3 Buckets found."
                fi
            env:
              - name: AWS_REGION
                value: "us-east-1"
              - name: SLACK_WEBHOOK_URL
                valueFrom:
                  secretKeyRef:
                    name: slack-webhook
                    key: url  # Store your Slack webhook in a Kubernetes Secret
              - name: ANALYZER_ARN
                valueFrom:
                  secretKeyRef:
                    name: analyzer-arn
                    key: arn # Store your analyzer's arn in a Kubernetes Secret
              

```

##### Notes
* AWS_REGION: put proper AWS region
* ANALYZER_ARN:  to get analyzer ARN run in console: 'aws accessanalyzer list-analyzers --query "analyzers[*].arn" --output text'
* SLACK_WEBHOOK_URL: The webhook URL follows this format - https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX

##### 7. Create the Kubernetes Service Account
This service account will assume an IAM role with permissions to run the IAM Access Analyzer.
k8s-serviceaccount.yaml

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: s3-analyzer-sa
  namespace: s3-analyzer
  annotations:
    eks.amazonaws.com/role-arn: "arn:aws:iam::<AWS_ACCOUNT_ID>:role/s3-analyzer-role"
```

##### 8. Apply the Kubernetes Resources
Once your AWS IAM role and permissions are set up, apply the Kubernetes configurations:

###### create k8s secret for slack-webhook
```bash
kubectl create secret generic slack-webhook \
    --from-literal=url=<YOUR_WEBHOOK_URL> \
    --namespace s3-analyzer
```

> "To Set Up a Slack Webhook:
- Go to your Slack workspace and create an Incoming Webhook:
- Open Slack → Apps → Incoming Webhooks (Slack Webhook Setup)
- Create a new webhook and choose a Slack channel where alerts will be sent.
- Copy the Webhook URL"
- The webhook URL follows this format: https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX

###### create k8s secret for analyzer-arn
```bash
kubectl create secret generic analyzer-arn \
    --from-literal=arn=<ANALYZER_ARN> \
    --namespace s3-analyzer
```


```bash
kubectl apply -f k8s-serviceaccount.yaml
kubectl apply -f k8s-cronjob.yaml
```

##### 9. Verifying the CronJob
To check the logs and verify that it ran correctly:
```bash
kubectl get cronjobs
kubectl get pods --selector=job-name=s3-public-check
kubectl logs <pod-name>
```
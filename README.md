# building-an-automated-cloud-security-remediation-system-on-aws

This project is about building an automated cloud security remediation system on AWS that can detect and immediately fix unsafe IAM role configurations without human intervention. The project builds a real-time AWS security enforcement system that automatically detects and fixes overprivileged IAM roles using IAM Access Analyzer, EventBridge, Lambda, and CloudWatch.

The lab goal is to create a Cloud Security Posture Management (CSPM) solution that:

i). Detects insecure IAM trust policies.
ii). Generates a security finding.
iii). Routes the finding through an event-driven pipeline.
iv). Automatically remediates the role within seconds.
v). Produces audit logs proving what was changed.

In summary, this is the to-do list for the lab: 

<img width="1003" height="654" alt="image" src="https://github.com/user-attachments/assets/0f6adfcd-d97b-47cd-897d-244398be13c0" />

The objective of the lab is to build an Automated Cloud Security Posture Management (CSPM) pipeline that:

i). detects insecure IAM roles using AWS IAM Access Analyzer.
ii). routes security findings through Amazon EventBridge.
iii). triggers a Lambda remediation function automatically.
iv). fixes the vulnerable trust policy within seconds.
v). logs all remediation actions for auditing and compliance.

This is the end-to-end workflow:  

<img width="999" height="760" alt="image" src="https://github.com/user-attachments/assets/9ce9fa65-ea39-4f2c-95e9-693850a38038" />

This lab will be done entirely on AWS CloudShell. 

---------------------------------------------------------------------------------------------------------------------------------------------------------------------------

**Step 1: Verify your identity on AWS CloudShell **

Log in to your AWS Console > click the CloudShell icon > verify your identity with this command: aws sts get-caller-identity

You should see details like this: 

<img width="798" height="236" alt="image" src="https://github.com/user-attachments/assets/cb70e645-bbe0-472e-b771-43498327723e" />

**Step 2: Enable Access Analyzer**


If you already have an access analyzer, skip this step. But if that's not the case, use this command to create a new one: 

aws accessanalyzer create-analyzer \
    --analyzer-name external-access-analyzer \
    --type ACCOUNT

Here's my Access Analyzer: 

<img width="1368" height="494" alt="image" src="https://github.com/user-attachments/assets/d79a6644-85b2-48d8-91f8-8bc05a97f560" />

**Step 3: Create the vulnerable IAM role**

Attach this trust policy file: 

cat > trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "*"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

Then create the role using this command: 

aws iam create-role \
  --role-name CloudScale-Audit-Test-Role \
  --assume-role-policy-document file://trust-policy.json

This is the result of both steps: 

<img width="1128" height="1110" alt="image" src="https://github.com/user-attachments/assets/668a3330-584d-4eee-b4fb-44c75957d3cf" />


**Step 4: Create Lambda execution role**

This is the trust document:

cat > lambda-trust.json <<EOF
{
  "Version":"2012-10-17",
  "Statement":[
    {
      "Effect":"Allow",
      "Principal":{
        "Service":"lambda.amazonaws.com"
      },
      "Action":"sts:AssumeRole"
    }
  ]
}
EOF

Then create a role for Lambda to perform its duties using this command: 

aws iam create-role \
  --role-name AccessAnalyzerRemediationRole \
  --assume-role-policy-document file://lambda-trust.json

The result: 

<img width="1169" height="1139" alt="image" src="https://github.com/user-attachments/assets/a05c2652-1d0b-4bf3-b418-cca661263ebc" />

**Step 5: Create permissions policy**

This is the file:

cat > remediation-policy.json <<EOF
{
  "Version":"2012-10-17",
  "Statement":[
    {
      "Effect":"Allow",
      "Action":[
        "iam:GetRole",
        "iam:UpdateAssumeRolePolicy"
      ],
      "Resource":"*"
    },
    {
      "Effect":"Allow",
      "Action":[
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource":"*"
    }
  ]
}
EOF

Then attach the policy with this command: 

aws iam put-role-policy \
  --role-name AccessAnalyzerRemediationRole \
  --policy-name RemediationPolicy \
  --policy-document file://remediation-policy.json

The result: 

<img width="1072" height="846" alt="image" src="https://github.com/user-attachments/assets/41f0a119-9feb-4cd6-98d7-1f8988b37ae7" />


**Step 6: Create the Lambda Function**

Create the directory with this command: 

mkdir remediation
cd remediation

Then create Lambda code with this: 

cat > lambda_function.py <<EOF
import json
import boto3

iam = boto3.client('iam')

def lambda_handler(event, context):

    role_name = "CloudScale-Audit-Test-Role"

    safe_policy = {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Principal": {
                    "AWS":
                    "arn:aws:iam::369607682818:root"
                },
                "Action": "sts:AssumeRole"
            }
        ]
    }

    iam.update_assume_role_policy(
        RoleName=role_name,
        PolicyDocument=json.dumps(safe_policy)
    )

    print(f"Remediated {role_name}")

    return {
        "statusCode": 200
    }
EOF

_**Note: replace the account ID with yours.**_

The result: 

<img width="1081" height="1071" alt="image" src="https://github.com/user-attachments/assets/99cfd83c-46d5-4159-a9e3-d7285fe3c694" />

Then package the Lambda code into a zip file with this command: zip function.zip lambda_function.py

The result: 

<img width="720" height="74" alt="image" src="https://github.com/user-attachments/assets/4269cf1f-300c-4f74-8eb9-bba1442a2588" />


**Step 7: Create Lambda**

Get the role ARN of the Lambda with this command: 

aws iam get-role \
  --role-name AccessAnalyzerRemediationRole \
  --query Role.Arn \
  --output text

Then create the Lambda with this command: 

aws lambda create-function \
  --function-name FixUnsafeTrustPolicy \
  --runtime python3.12 \
  --role arn:aws:iam::369607682818:role/AccessAnalyzerRemediationRole \
  --handler lambda_function.lambda_handler \
  --zip-file fileb://function.zip

**_Note: replace --role with your ARN._**

The result: 

<img width="1809" height="1299" alt="image" src="https://github.com/user-attachments/assets/278f9fb6-b400-4c09-9755-a9ef91e3c236" />

**Step 8: Create EventBridge Rule**

Create the file: 

cat > access-analyzer-pattern.json <<EOF
{
  "source": [
    "aws.access-analyzer"
  ]
}
EOF

Then create the rule: 

aws events put-rule \
  --name AccessAnalyzerRemediationRule \
  --event-pattern file://access-analyzer-pattern.json

Get Lambda ARN with this command: 

aws lambda get-function \
  --function-name FixUnsafeTrustPolicy \
  --query Configuration.FunctionArn \
  --output text

Specify the target:

aws events put-targets \
  --rule AccessAnalyzerRemediationRule \
  --targets Id=1,Arn=arn:aws:lambda:us-east-1:369607682818:function:FixUnsafeTrustPolicy

The result of the steps: 
 
<img width="1374" height="757" alt="image" src="https://github.com/user-attachments/assets/1a953cf7-7e0d-4826-a04d-a0f34164d8d7" />

**Step 9: Allow EventBridge to Invoke Lambda**

Use this command to permit EventBridge to invoke Lambda:

aws lambda add-permission \
  --function-name FixUnsafeTrustPolicy \
  --statement-id EventBridgeInvoke \
  --action lambda:InvokeFunction \
  --principal events.amazonaws.com \
  --source-arn arn:aws:events:us-east-1:369607682818:rule/AccessAnalyzerRemediationRule

_**Note: replace the --source-arn with yours**._

The result: 

<img width="3312" height="290" alt="image" src="https://github.com/user-attachments/assets/a06dec19-a80c-48ce-ad31-2e53e429dda8" />

**Step 10: Verify Access Analyzer finding**

First, get your Access Analyzer ARN with this command: 

aws accessanalyzer list-analyzers \
  --query "analyzers[0].arn" \
  --output text

Then use this command to verify the finding: 

aws accessanalyzer list-findings \
--analyzer-arn arn:aws:access-analyzer:us-east-1:369607682818:analyzer/panopticon-analyzer

The result: 

<img width="1803" height="1448" alt="image" src="https://github.com/user-attachments/assets/224a5f4b-5122-42d4-903d-11413824e743" />

As the image shows, the result shows a finding related to the resource and the vulnerable role ("AWS":"*"). This suggests the Access Analyzer found the anomaly. 

**Step 11: Test Lambda manually**

Before waiting for Access Analyzer, invoke Lambda manually:

aws lambda invoke \
  --function-name FixUnsafeTrustPolicy \
  output.json

The result: 

<img width="701" height="223" alt="image" src="https://github.com/user-attachments/assets/0647336e-4d89-44ac-a510-8796e1a1a53a" />

**Step 12: Verify the policy was fixed**

Use this command to retrieve the trust policy:

aws iam get-role \
--role-name CloudScale-Audit-Test-Role \
--query Role.AssumeRolePolicyDocument

The result: 

<img width="974" height="442" alt="image" src="https://github.com/user-attachments/assets/9ad96f19-3c4b-476d-b355-c8f105a36e81" />

Before remediation, the policy showed "AWS":"*" but after remediation, it showed "AWS":"arn:aws:iam::369607682818:root" showing the policy has been fixed.

**Step 13: Check CloudWatch logs**

List the log groups with this command: 

aws logs describe-log-groups \
  --log-group-name-prefix "/aws/lambda/FixUnsafeTrustPolicy"

The result:

<img width="1510" height="497" alt="image" src="https://github.com/user-attachments/assets/0629a10d-f4a1-45a7-b394-7a8186b59d8f" />

Find streams with this command: 

aws logs describe-log-streams \
--log-group-name "/aws/lambda/FixUnsafeTrustPolicy"

The result: 

<img width="2267" height="467" alt="image" src="https://github.com/user-attachments/assets/431afb5f-bc88-4a80-88cd-d2a2822c7bdd" />

View the logs with this command: 

aws logs get-log-events \
  --log-group-name "/aws/lambda/FixUnsafeTrustPolicy" \
  --log-stream-name '2026/10/28/[$LATEST]67b284f99d9f4693b7b80b2ae0257a63'

_**Note: replace the --log-stream-name with yours. **

The result: 

<img width="2861" height="1022" alt="image" src="https://github.com/user-attachments/assets/431abc2b-bf2e-402c-a6a1-a29c02c4770f" />


The key line in the result is the "Remediated CloudScale-Audit-Test-Role\n" which shows the vulnerable policy has been fixed.

# Lambda Architecture Errors

This repository is a reproducible example of exec errors being thrown by valid handler code and valid cloudformation template when migrating from x86_64 to arm64 (graviton) when paired with a Lambda Extension.

The extension provided in this case is the Datadog Agent, which has versions compiled for both x86_64 and ARM.

Used here, they are `"arn:aws:lambda:us-east-1:464622532012:layer:Datadog-Extension-ARM:29"`
and
`"arn:aws:lambda:us-east-1:464622532012:layer:Datadog-Extension:29"`

For a brief (10 second) window during deployment, the function throws launch errors due to AWS Lambda mismatching the architecture of the Lambda function with the architecture of the Lambda Extension; even though the CloudFormation template used to update the Lambda function is correct:

```json
"HelloLambdaFunction": {
      "Type": "AWS::Lambda::Function",
      "Properties": {
        "Code": {
          "S3Bucket": {
            "Ref": "ServerlessDeploymentBucket"
          },
          "S3Key": "serverless/arch-bug/dev/1667400038982-2022-11-02T14:40:38.982Z/arch-bug.zip"
        },
        "Handler": "/opt/nodejs/node_modules/datadog-lambda-js/handler.handler",
        "Runtime": "nodejs16.x",
        "FunctionName": "arch-bug-dev-hello",
        "MemorySize": 1024,
        "Timeout": 6,
        "Architectures": [
          "arm64"
        ],
        "Tags": [
          {
            "Key": "dd_sls_plugin",
            "Value": "v5.9.0"
          }
        ],
        "Environment": {
          "Variables": {
            "DD_API_KEY": "abcd-1234",
            "DD_SITE": "datadoghq.com",
            "DD_TRACE_ENABLED": true,
            "DD_MERGE_XRAY_TRACES": false,
            "DD_LOGS_INJECTION": true,
            "DD_SERVERLESS_LOGS_ENABLED": true,
            "DD_CAPTURE_LAMBDA_PAYLOAD": false,
            "DD_SERVICE": "arch-bug",
            "DD_ENV": "dev",
            "DD_LAMBDA_HANDLER": "handler.hello"
          }
        },
        "Role": {
          "Fn::GetAtt": [
            "IamRoleLambdaExecution",
            "Arn"
          ]
        },
        "Layers": [
          "arn:aws:lambda:us-east-1:464622532012:layer:Datadog-Node16-x:84",
          "arn:aws:lambda:us-east-1:464622532012:layer:Datadog-Extension-ARM:29"
        ]
      },
      "DependsOn": [
        "HelloLogGroup"
      ]
    },
```

## Steps to Reproduce

1. `npm install -g serverless` (use 2.61 or newer)
2. `npm install`
3. `serverless deploy`
4. Now, copy the API Gateway URL and curl it (or just navigate to it in your browser). It should work.
5. Finally, uncomment the `architecture` block in `serverless.yml`
6. Run `serverless deploy` again, while repeatedly curling/refreshing the browser tab.

Result: You'll receive 500 errors and see this in the logs:

```
2022-11-02T10:41:30.814-04:00	EXTENSION Name: datadog-agent State: LaunchError Events: [] Error Type: UnknownError

2022-11-02T10:41:30.814-04:00	START RequestId: 3273a1b6-267c-45b4-8ce8-1a82c1953653 Version: $LATEST

2022-11-02T10:41:30.815-04:00	RequestId: 3273a1b6-267c-45b4-8ce8-1a82c1953653 Error: fork/exec /opt/extensions/datadog-agent: exec format error Extension.LaunchError

2022-11-02T10:41:30.815-04:00	END RequestId: 3273a1b6-267c-45b4-8ce8-1a82c1953653
```

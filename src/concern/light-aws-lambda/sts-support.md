# STS Support

The `lambda-invoker` module supports AWS Security Token Service (STS) to obtain temporary, limited-privilege *assumed-role* credentials for invoking Lambda functions. This is an alternative to directly using long-lived static IAM access keys and and it is a fundamental component of AWS Identity and Access Management (IAM) used to enhance security by following the principle of least privilege.

## Key Features

*   **Temporary Credentials**: Provides short-lived credentials (access key, secret key, and token) that expire, reducing risks from compromised keys.
*   **AssumeRole**: Obtains temporary credentials for cross-account access or delegated permissions.
*   **Automatic Managed Refresh**: The `LambdaFunctionHandler` leverages the AWS SDK's `StsAssumeRoleCredentialsProvider` to handle token refresh automatically and asynchronously.

## Configuration

To enable STS support, you need to add the following configuration to your `lambda-invoker.yml`:

*   **stsEnabled**: Set to `true` to enable STS support. Default is `false`.
*   **roleArn**: The ARN of the IAM role to assume.
*   **roleSessionName**: An identifier for the assumed role session. Default is `light-gateway-session`.
*   **durationSeconds**: The duration, in seconds, of the role session. Default is `3600` (1 hour).

### Example `lambda-invoker.yml`

```yaml
region: us-east-1
# other configuration properties...

stsEnabled: true
roleArn: arn:aws:iam::123456789012:role/LambdaInvokerRole
roleSessionName: gateway-session
durationSeconds: 3600
```

## How it Works

When `stsEnabled` is set to `true`, the `LambdaFunctionHandler` initializes an `StsAssumeRoleCredentialsProvider`. 

This provider uses the **AWS Default Credential Chain** (e.g., EC2 instance profile, ECS task role, environment variables, or local profile) as the *source* credentials to call the `AssumeRole` operation.

The returned temporary credentials (access key, secret key, and session token) are then used by the `LambdaAsyncClient` to sign invocation requests.

### Automatic Refreshment

The `StsAssumeRoleCredentialsProvider` automatically manages the lifecycle of the temporary credentials. It preemptively and asynchronously refreshes the session before it expires, ensuring zero downtime and omitting the need for manual refresh logic in the application code.

## IAM Policy Requirements

1. **Source Credentials Permission**: The IAM entity (e.g., EC2 instance profile) that the light-4j application is running as must have the `sts:AssumeRole` permission for the target `roleArn`.
2. **Target Role Trust Relationship**: The target role (`roleArn`) must have a trust relationship (Principal) that allows the application's source IAM entity to assume it.

### Example Source IAM Policy

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "sts:AssumeRole",
            "Resource": "arn:aws:iam::123456789012:role/LambdaInvokerRole"
        }
    ]
}
```

# STS Support

The `lambda-invoker` module supports AWS Security Token Service (STS) to obtain temporary, limited-privilege security credentials for invoking Lambda functions. This is an alternative to providing long-term IAM access keys and and it is a fundamental component of AWS Identity and Access Management (IAM) used to enhance security by minimizing the need for long-term access keys.

## Key Features

*   **Temporary Credentials**: Provides short-lived credentials (access key, secret key, and token) that expire, reducing risks from compromised keys.
*   **AssumeRole**: Obtains temporary credentials for cross-account access or delegated permissions.
*   **Automatic Refresh**: The `LambdaFunctionHandler` automatically refreshes the temporary credentials before they expire.

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

When `stsEnabled` is set to `true`, the `LambdaFunctionHandler` use the `StsClient` to call the `AssumeRole` operation with the configured `roleArn`, `roleSessionName`, and `durationSeconds`.

The returned temporary credentials (access key, secret key, and session token) are then used to initialize the `LambdaAsyncClient`.

### Automatic Refreshment

The `LambdaFunctionHandler` checks the expiration time of the cached STS credentials before each Lambda invocation. If the credentials are within a 5-minute buffer of expiring, the handler automatically initiates a new `AssumeRole` request to refresh the credentials and re-initializes the `LambdaAsyncClient`.

## IAM Policy Requirements

The IAM entity (user or role) that the light-4j application is running as must have the `sts:AssumeRole` permission for the target `roleArn`.

### Example IAM Policy

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

Additionally, the target role must have a trust relationship that allows the application's IAM entity to assume it.

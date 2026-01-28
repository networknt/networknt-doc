# Lambda Invoker

The `lambda-invoker` module provides a way to invoke AWS Lambda functions from a light-4j application. It includes a configuration class and an HTTP handler that can be used to proxy requests to Lambda functions.

## Core Components

### LambdaInvokerConfig
The configuration class for the Lambda invoker. It loads settings from `lambda-invoker.yml`.
*   **region**: The AWS region where the Lambda functions are deployed.
*   **endpointOverride**: Optional URL to override the default AWS Lambda endpoint.
*   **apiCallTimeout**: Timeout for the entire API call in milliseconds.
*   **apiCallAttemptTimeout**: Timeout for each individual API call attempt in milliseconds.
*   **maxRetries**: Maximum number of retries for the invocation.
*   **maxConcurrency**: Maximum number of concurrent requests to Lambda.
*   **functions**: A map of endpoints to Lambda function names or ARNs.
*   **metricsInjection**: Whether to inject Lambda response time metrics into the metrics handler.

### LambdaFunctionHandler
An HTTP handler that proxies requests to AWS Lambda functions based on the configured mapping.
*   **Behavior**: It converts the incoming `HttpServerExchange` into an `APIGatewayProxyRequestEvent`, invokes the configured Lambda function asynchronously using the `LambdaAsyncClient`, and then converts the `APIGatewayProxyResponseEvent` back into the HTTP response.
*   **Metrics**: If enabled, it records the total time spent in the Lambda invocation and injects it into the metrics handler.

## Configuration

Example `lambda-invoker.yml`:

```yaml
region: us-east-1
apiCallTimeout: 60000
apiCallAttemptTimeout: 20000
maxRetries: 2
maxConcurrency: 50
functions:
  /v1/pets: petstore-function
  /v1/users: user-service-function
metricsInjection: true
metricsName: lambda-response
```

## Usage

To use the `LambdaFunctionHandler`, add it to your `handler.yml` chain:

```yaml
- com.networknt.aws.lambda.LambdaFunctionHandler@lambda
```

And configure the paths in `handler.yml`:

```yaml
paths:
  - path: '/v1/pets'
    method: 'GET'
    handler:
      - lambda
```

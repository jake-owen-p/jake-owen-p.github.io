## Lambdas

### Extremely Simple Serverless File

```yml
service: hello-ts-lambda

frameworkVersion: ">=1.1.0 <2.0.0"

provider:
  name: aws
  runtime: nodejs12.x

functions:
  garmentFinder:
    handler: dist/index.handler
    events:
      - http:
          path: ping
          method: get
```

### Invoking Lambdas Locally
```bash
serverless invoke local --function garmentFinder
```
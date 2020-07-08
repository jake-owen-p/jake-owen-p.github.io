# Serverless

### Basic serverless.yml file

```yml
service: GarmentFinder.API

frameworkVersion: ">=1.1.0 <2.0.0"

provider:
  stackName: garment-finder-api
  name: aws
  runtime: nodejs12.x
  region: eu-west-1



functions:
  garmentFinder:
    name: find-garments
    handler: dist/index.handler
    events:
      - http:
          path: findgarments
          method: post
```

### Invoking Lambdas Locally
```bash
serverless invoke local --function garmentFinder
```

### Access needed for `serverless deploy`
(Given full access for each service so may be more needed)
- cloudformation:ValidateTemplate
- s3:CreateBucket
- logs:DescribeLogGroups
- apigateway:POST
- iam:GetRole
- lambda:GetFunction
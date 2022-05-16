<div align="center">
	<br/>
	<br/>
  <h1>
	<img height="140" src="assets/alma-cdk-openapix.svg" alt="Alma CDK OpenApiX" />
  <br/>
  <br/>
  </h1>

  ```sh
  npm i -D @alma-cdk/openapix
  ```

  <div align="left">

  Generate AWS Api Gateway REST APIs via [OpenAPI](https://www.openapis.org/) (a.k.a. Swagger) Schema Definitions by consuming "clean" OpenAPI schemas and inject `x-amazon-apigateway-` extensions with type-safety.

  </div>
  <br/>
</div>


<br/>

## Work in Progress

![experimental](https://img.shields.io/badge/stability-experimental-yellow "Stability: Experimental")

🚧 &nbsp;**Do not use for production critial stuff! This construct is still very much work in progress and breaking changes may occur.** 🚧


<br/>

## Getting Started

```ts
const fn = new lambda.Function(this, "fn", {
  handler: "index.handler",
  runtime: lambda.Runtime.NODEJS_14_X,
  code: lambda.Code.fromInline('export function handler() { return { statusCode: 200, body: JSON.stringify("hello")} }'),
});

const pkName = 'item';

const table = new dynamodb.Table(this, 'table', {
  partitionKey: {
    type: dynamodb.AttributeType.STRING,
    name: pkName,
  }
});

const role = new iam.Role(this, "role", {
  assumedBy: new iam.ServicePrincipal('apigateway.amazonaws.com'),
});

table.grantReadData(role);

const apiDefinition = new OpenApiXDefinition(this, {
  upload: false, // by default add as inline Body, set to true to use as BodyS3Location
  source: './schema.yaml',
  integrations: {

    // Mock Integration
    '/mock': { 'GET': new OpenApiXMock(this) },

    // AWS Lambda Proxy integration
    '/message': { 'GET': new OpenApiXLambda(this, fn) },

    // HTTP Proxy integration
    '/ext': { 'GET': new OpenApiXHttp(this, "https://example.com") },

    // Direct integration to AWS Service
    '/item': { 'GET': new OpenApiXService(this, {
          service: 'dynamodb',
          action: 'GetItem',
          options: {
            credentialsRole: role,
            requestTemplates: {
              'application/json': JSON.stringify({
                "TableName": table.tableName,
                "Key": {
                  [pkName]: {
                    "S": "$input.params('item')"
                  }
                }
              }),
            },
          },
      }),
    },

  },

  injectPaths: { "info.title": "FancyPantsAPI" },
  rejectPaths: ['info.version'],

  // TODO add validators...
})


new apigateway.SpecRestApi(this, 'api', {
  apiDefinition,
});

// Also supports new OpenApiX()
```

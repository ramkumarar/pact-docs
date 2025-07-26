# PactFlow Bidirectional Contract Testing - Compatibility Checks Reference

## Overview

In bidirectional contract testing, PactFlow performs cross-contract validation by comparing consumer contracts (Pact files) against provider contracts (OpenAPI specifications). This ensures that consumer expectations are a valid subset of what the provider's OpenAPI specification defines.

**Key Principle**: The consumer contract must be compatible with the provider's OpenAPI specification - consumers can only use what providers explicitly offer.

> **Important**: PactFlow can only validate based on the contracts it receives. If the consumer contract doesn't capture all the interactions the consumer actually uses, compatibility checks may incorrectly indicate it's safe to deploy when breaking changes exist in uncaptured API calls.

## Bidirectional Testing Context

Unlike traditional consumer-driven contract testing where providers implement consumer requirements, bidirectional testing validates that:

1. **Consumer contracts** (generated from actual consumer tests) are compatible with
2. **Provider contracts** (OpenAPI specifications defining provider capabilities)
3. **Provider implementations** actually match their OpenAPI specifications

This creates a **specification-first** approach where the OpenAPI spec serves as the contract authority.

## Failure Conditions

There are three conditions that could result in an invalid (or failed) integration:

1. **Provider self-verification test results indicate failure** - The provider build ran some tests against the OAS and the result was a failure
2. **Consumer Pact verification results failed** - The Pact consumer tests failed, and the consumer is not compatible with the published Pact file
3. **Consumer Pact file is not compatible with the OAS** - Cross-contract validation failed

## Compatibility Checks

For the specific validations and checks performed, refer to OAS features and testing.

## Interpreting Verification Result Failures

Verification results are automatically pre-generated when a consumer contract is published against a number of common provider versions (such as deployed versions). They are also generated dynamically when can-i-deploy is invoked for a given set of application versions and target environments.

Compatibility results are visible from the user interface, in the API and in the output of can-i-deploy.

### User Interface

Compatibility results are listed on the detail view page, grouped by the relevant API resource in the OpenAPI document. They are also on the consumer contract tab, grouped by the interactions defined in the consumer contract.

### Can I Deploy

When can-i-deploy is called, you will get a table of results that shows which applications are compatible and the verification URL which explains the results.

#### Output from can-i-deploy

```
CONSUMER                       | C.VERSION          | PROVIDER                       | P.VERSION          | SUCCESS? | RESULT#
-------------------------------|--------------------|---------------------------------|--------------------|----------|--------
pactflow-example-consumer-w... | 5785fb8+1622544123 | pactflow-example-provider-r... | 7f3d83f+1622544125 | true     | 1

VERIFICATION RESULTS
--------------------
1. https://testdemo.pactflow.io/hal-browser/browser.html#https://testdemo.pactflow.io/contracts/provider/pactflow-example-bi-directional-provider-restassured/version/7f3d83f%2B1622544125/consumer/pactflow-example-bi-directional-consumer-wiremock/pact-version/b421f8d1c0691e8304492c716e546427c4267c7f/verification-results (success)
```

Clicking on the verification results will take you to the (API) resource in PactFlow and show you the detailed analysis.

> **Note**: In some cases, results may not yet have been generated and entries in the above table will be unknown. To address this, you should consider polling via the `--retry-while-unknown` and `--retry-interval` flags.

## API Resources

### Response Object

- **summary**: Whether or not the verification was successful
- **crossContractVerificationResults**: This element contains the results of comparing the mock (pact contract) to the OpenAPI specification
- **providerContractVerificationResults**: This contains the results of the provider verification, including the tool used to verify it, whether the test passed or failed and the base64 encoded OAS contract

### Successful Result

```json
{
  "summary": {
    "success": true
  },
  "crossContractVerificationResults": {
    "success": true,
    "results": {
      "errors": [],
      "warnings": []
    },
    "verificationDate": "2021-06-01T10:42:30.980+00:00",
    "verifier": "pactflow-swagger-mock-validator",
    "verifierVersion": "10.0.0"
  },
  "providerContractVerificationResults": {
    "success": true,
    "content": "dGVzdGVkIHZpYSBSZXN0QXNzdXJlZAo=",
    "contentType": "text/plain",
    "verifier": "verifier"
  },
  "_links": {
    "self": {
      "title": "Cross contract and Provider Contract verification results",
      "href": "https://testdemo.pactflow.io/contracts/provider/pactflow-example-bi-directional-provider-restassured/version/7f3d83f%2B1622544125/consumer/pactflow-example-bi-directional-consumer-wiremock/pact-version/b421f8d1c0691e8304492c716e546427c4267c7f/verification-results"
    }
  }
}
```

### Failure Result

```json
{
  "summary": {
    "success": false
  },
  "crossContractVerificationResults": {
    "success": false,
    "results": {
      "errors": [{
        "code": "request.body.incompatible",
        "message": "Request body is incompatible with the request body schema in the spec file: should NOT have additional properties",
        "mockDetails": {
          "interactionDescription": "POST_/products_7436b06b-b387-4535-9d3a-da149d9826ba",
          "interactionState": "[none]",
          "location": "[root].interactions[0].request.body",
          "mockFile": "pact",
          "value": {
            "id": "27",
            "name": "pizza",
            "type": "food",
            "price": 27
          }
        },
        "source": "spec-mock-validation",
        "specDetails": {
          "location": "[root].paths./products.post.requestBody.content.application/json.schema.additionalProperties",
          "pathMethod": "post",
          "pathName": "/products",
          "specFile": "oas",
          "value": false
        },
        "type": "error"
      }, {
        "code": "response.body.incompatible",
        "message": "Response body is incompatible with the response body schema in the spec file: should NOT have additional properties - price",
        "mockDetails": {
          "interactionDescription": "GET_/products_646e1d83-da87-4155-9a43-3b24a2014cf3",
          "interactionState": "[none]",
          "location": "[root].interactions[1].response.body[0]",
          "mockFile": "pact",
          "value": {
            "name": "pizza",
            "id": "10",
            "type": "food",
            "price": 100
          }
        },
        "source": "spec-mock-validation",
        "specDetails": {
          "location": "[root].paths./products.get.responses.200.content.application/json; charset=utf-8.schema.items.additionalProperties",
          "pathMethod": "get",
          "pathName": "/products",
          "specFile": "oas",
          "value": false
        },
        "type": "error"
      }],
      "failureReason": "Mock file \"pact\" is not compatible with spec file \"oas\"",
      "warnings": []
    },
    "verificationDate": "2021-06-02T01:27:54.815+00:00",
    "verifier": "pactflow-swagger-mock-validator",
    "verifierVersion": "10.0.0"
  },
  "providerContractVerificationResults": {
    "success": true,
    "content": "dGVzdGVkIHZpYSBSZXN0QXNzdXJlZAo=",
    "contentType": "text/plain",
    "verifier": "verifier"
  }
}
```

## Understanding Error Components

In the case of a failure, the following elements of the error are most helpful in diagnosing the problem:

### message
Summary of the problem. In most cases, you can understand the problem immediately. As above, you can see there are two errors - one for the request body, and another for the response body. In both cases (the request and response body), there is an additional unexpected property `price` expected by the consumer.

### mockDetails
Contains details of the Consumer Contract (mock) that are problematic, including the path to the interaction in the contract and the request/response details.

### specDetails
Contains details of the Provider Contract (spec) that are problematic, including the path to the component of the resource in the OpenAPI specification the mock is incompatible with.

## Contract Compatibility Errors

All errors and warnings are written from the consumer's perspective, referencing a "spec file" (the OpenAPI Document).

For example, if the consumer calls an unknown endpoint `/products/10`, the error code will be:
`request.path-or-method.unknown` and the corresponding message:
`Path or method not defined in spec file: GET /products/10`

All errors contain 3 major components to aid with debugging:

| Component | Description |
|-----------|-------------|
| **Code** | The category of error or warning discovered. Warnings do not fail the compatibility check, but indicate a potential misunderstanding. |
| **Message** | A description of the violation, usually in the form of a JSON Schema validation error |
| **Interaction Path** | A JSON-path like syntax that can help you reference the specific property in the interaction, within the consumer contract that is incompatible with the provider contract |

## Error Codes Reference

A code describes the category of the problem, such as an incorrect header or an unexpected property.

> **Note**: "spec" refers to the provider contract/OpenAPI Document. "Interaction" refers to the problematic interaction in the pact file.

### Request Errors

| Error Code | Type | Description | Fix |
|------------|------|-------------|-----|
| `request.accept.incompatible` | Error | The Accept header in the interaction does not match any of the mime-types in the OpenAPI document | Check your consumer test to ensure the expected mime type matches an acceptable mime type in the provider contract |
| `request.accept.unknown` | Warning | There is an Accept header in your interaction, but the OpenAPI Document does not return any content | This is a redundant header that should be removed. If there is an expected body, this is likely to be an error |
| `request.authorization.missing` | Error | The interaction lacks an Authorization query or header, but is required by the spec file | Update the pact test to ensure it has an appropriate authorization scheme |
| `request.body.incompatible` | Error | The request body in the interaction is incompatible with the request body schema in the spec | Review the JSON schema validation message |
| `request.body.unknown` | Warning | No matching schema could be found for the request body | Your API Provider does not expect a request body, or the request body does not match one of the allowed schemas |
| `request.content-type.incompatible` | Error | Request Content-Type header is incompatible with the mime-types the spec accepts to consume | Confirm that your pact test is sending the correct content type |
| `request.content-type.missing` | Warning | Request content type header is not defined but spec specifies mime-types to consume | You should explicitly set the Accept header in your pact test |
| `request.content-type.unknown` | Warning | Request content-type header is defined but the spec does not specify any mime-types to consume | The request body is redundant. Check if the spec should be accepting one, or remove it from your Pact test |
| `request.header.incompatible` | Error | The interaction request header is incompatible with the spec | Review the JSON Schema validation message |
| `request.header.unknown` | Warning | Request header is not defined in the spec file | Remove the redundant header or ensure it is defined in the spec |
| `request.path-or-method.unknown` | Error | The Path or method used in the interaction is not defined in spec file | Check the correct resource is being used |
| `request.query.incompatible` | Error | The request query in the interaction is incompatible with the spec | Review the JSON Schema validation message |
| `request.query.unknown` | Warning | The query parameter in the interaction is not defined in the spec file | Review if the query parameter is valid |

### Response Errors

| Error Code | Type | Description | Fix |
|------------|------|-------------|-----|
| `response.body.incompatible` | Error | The response body in the interaction is incompatible with the spec | Review the JSON Schema validation message |
| `response.body.unknown` | Warning | No matching schema was found for response body | Your API Provider does not return a response body, or the response body does not match one of the allowed schemas |
| `response.content-type.incompatible` | Error | Response Content-Type header is incompatible with the mime-types the spec defines to produce | Confirm that your pact test is consuming the correct content type |
| `response.content-type.unknown` | Warning | Response content-type header is defined but the spec does not specify any mime-types to produce | The response body is redundant. Check if the spec should be returning one, or remove it from your Pact test |
| `response.header.incompatible` | Error | The response header in the interaction is incompatible with the spec | Review the JSON Schema validation message |
| `response.header.unknown` | Warning | Response header is not defined in the spec file | Remove the redundant header or ensure it is defined in the spec |
| `response.status.default` | Warning | The interaction is using a response status code that is a default response in the spec file | Avoid default responses as they increase the ambiguity of possible valid response types |
| `response.status.unknown` | Error | The expected response status code is not defined in the spec | Check the spec to see what valid status codes are returned |

## Error Message Categories

There are broadly three categories of error:

1. **Unknown** - values that aren't defined in the spec, but referenced in the interaction
2. **Missing** - values that are required in the spec, but are missing in the interaction  
3. **Incompatible** - values that are both defined in the spec and referenced in the interaction, but are incompatible

For the ones deemed "incompatible", they will usually be accompanied by a detailed error message explaining the mismatch, which will be JSON Schema errors.

## Error Path Syntax

Error paths use a JSONPath-like syntax with an associated value, to help you navigate from the error to the specific problematic component within the Pact interaction.

### Examples

- `[root].interactions[0].response.body = {"id":"09","type":"CREDIT_CARD","name":"Gem Visa","price":99.99}` - the response body in the first pact interaction is incompatible
- `[root].interactions[0].response.headers.access-control-allow-origin = "*"` - the Access-Control-Allow-Origin header in the first pact interaction
- `[root].interactions[1].request.query.id = "2"` - the id query parameter in the 2nd pact interaction
- `[root].interactions[3].request.path` - the request path in the third pact interaction

## Common Error Types

### additionalProperties

When a pact test expects a response body, it may ask for a subset of what the provider can provide - this is perfectly acceptable. However it cannot ask for a property not present in the spec - this will cause failure.

**Example Error:**
```
Response body is incompatible with the response body schema in the spec file: must NOT have additional properties - foo
```

In JSON Schema terminology, `foo` is an "additional property" not defined in the schema.

**Fix:** The problematic property should be removed from the pact interaction, or supplied by the spec.

### unevaluatedProperties (allOf)

When a pact test expects to send a request body or to receive a response body, the body must match any defined schemas.

If the `allOf` keyword is used, we must treat all the schemas as a single composite schema. As per the additionalProperties checks, if a property is expected that is not part of this composite schema, a similar error will be returned:

**Example Error:**
```
Response body is incompatible with the response body schema in the spec file: must NOT have unevaluated properties
```

**Fix:** The problematic property should be removed from the pact interaction, or supplied by the spec.

## Bidirectional Testing Troubleshooting Workflow

### 1. Identify the Contract Mismatch
- **Error Code**: Understand the category (request/response, body/header/path)
- **Error Message**: Get the specific validation failure details
- **Contract Source**: Determine if the issue is in consumer expectations or provider specification

### 2. Analyze the Contracts
- **Consumer Contract (mockDetails)**: Review the Pact interaction that's failing
- **Provider Contract (specDetails)**: Check the corresponding OpenAPI specification
- **Implementation Gap**: Verify if the provider implementation matches the OpenAPI spec

### 3. Determine Resolution Strategy

#### Option A: Update Consumer Contract (Most Common)
- Consumer is expecting something not offered by the provider
- **Action**: Modify consumer tests to align with provider capabilities
- **Example**: Remove additional properties, use correct content types

#### Option B: Update Provider Contract (OpenAPI Spec)
- Provider specification is incomplete or incorrect
- **Action**: Update OpenAPI spec to include missing capabilities
- **Verification**: Ensure provider implementation supports the updated spec

#### Option C: Update Provider Implementation
- Provider implementation doesn't match its OpenAPI specification
- **Action**: Fix provider code or update OpenAPI spec to match reality
- **Validation**: Run provider contract tests to verify alignment

### 4. Bidirectional Testing Validation Steps

1. **Fix the Contract**: Apply the chosen resolution strategy
2. **Regenerate Consumer Contract**: Run consumer tests to generate new Pact file
3. **Validate Provider Contract**: Ensure provider tests pass against OpenAPI spec
4. **Cross-Contract Check**: Verify compatibility in PactFlow
5. **Deployment Safety**: Run `can-i-deploy` before releasing

## Best Practices for Avoiding Compatibility Issues

1. **Keep Consumer Tests Minimal** - Only test what your consumer actually uses
2. **Avoid Additional Properties** - Don't expect fields not defined in the OpenAPI spec
3. **Use Correct Content Types** - Ensure Accept and Content-Type headers match the spec
4. **Handle Optional Fields Properly** - Use appropriate matchers for optional response fields
5. **Coordinate with Provider Team** - Ensure OpenAPI spec accurately reflects the API behavior
6. **Regular Validation** - Run compatibility checks frequently during development

## Related Documentation

- [Bidirectional Contract Testing Guide](bidirectional_contract_testing_guide.md) - Core concepts and workflow
- [Spring Boot Consumer Testing Best Practices](spring_boot_consumer_testing_best_practices.md) - Consumer implementation guide
- [Spring Boot Provider Testing Best Practices](spring_boot_provider_testing_best_practices.md) - Provider implementation guide
- [PactFlow Test Scenarios](pactflow_test_scenarios.md) - Real-world testing scenarios
- [Scenario Examples](scenarios/pactflow_test_scenarios.md) - Sample test files and reports
# Pactflow Bidirectional Contract Testing - Comprehensive Test Report

## Overview
This document demonstrates bidirectional contract testing with Pactflow using a User Management API. It shows how contract testing catches breaking changes between API providers and consumers through real-world scenarios with actual error messages.

## API Evolution Summary
- **Version 1.0.0** (`openapi.yaml`): Basic user creation with `name`, `age`, `email`
- **Version 2.0.0** (`openapi-breaking.yaml`): Adds mandatory `department` field with enum constraints

## Test Scenarios & Results

### Scenario 1: Perfect Match (Baseline) ✅
**Status**: Both provider and consumer contracts match perfectly

**Provider Spec**: `openapi.yaml` (v1.0.0)
```yaml
CreateUserRequest:
  type: object
  required: [name, age, email]
  properties:
    name: { type: string, minLength: 1, maxLength: 100 }
    age: { type: integer, minimum: 0, maximum: 110 }
    email: { type: string, format: email }
  additionalProperties: false
```

**Consumer Pact**: `scenario1_consumer_pact.json`
- Tests valid user creation with all required fields
- Tests error scenarios (invalid age, invalid email)
- Expects 201 success and 400 error responses

**Result**: ✅ **NO ERRORS** - Perfect contract alignment

---

### Scenario 2: Missing Mandatory Fields ❌
**Status**: Consumer removes mandatory fields from expectations

**Provider Spec**: `openapi.yaml` (unchanged - still requires name, age, email)

**Consumer Changes**: Attempts to create users without required fields:
- Request without `email` field
- Request without `name` field  
- Request with only `age` field

**Actual Errors**:
```json
[
  {
    "code": "request.body.incompatible",
    "message": "Request body is incompatible with the request body schema in the spec file: must have required property 'email'",
    "mockDetails": {
      "interactionDescription": "Create user without email (missing mandatory field)",
      "value": {"name": "John Doe", "age": 30}
    },
    "specDetails": {
      "location": "CreateUserRequest.required",
      "value": ["name", "age", "email"]
    }
  },
  {
    "code": "request.body.incompatible", 
    "message": "Request body is incompatible with the request body schema in the spec file: must have required property 'name'",
    "mockDetails": {
      "interactionDescription": "Create user without name (missing mandatory field)",
      "value": {"age": 25}
    }
  }
]
```

**Result**: ❌ **CONTRACT VERIFICATION FAILS** - Consumer breaking change detected

---

### Scenario 3: Consumer Adds Unknown Fields ⚠️
**Status**: Consumer adds new fields unknown to provider

**Provider Spec**: `openapi.yaml` with `additionalProperties: false` (strict mode)

**Consumer Changes**: Adds unknown fields to requests/responses:
- `phoneNumber` field
- `address` object with street, city, zipCode
- `department`, `jobTitle`, and `preferences` object

**Actual Errors**:
```json
[
  {
    "code": "request.body.incompatible",
    "message": "Request body is incompatible with the request body schema in the spec file: must NOT have additional properties - phoneNumber",
    "mockDetails": {
      "interactionDescription": "Create user with additional phone number field",
      "value": {
        "name": "John Doe",
        "age": 30,
        "email": "john.doe@example.com",
        "phoneNumber": "+1-555-123-4567"
      }
    },
    "specDetails": {
      "location": "CreateUserRequest.additionalProperties",
      "value": false
    }
  },
  {
    "code": "response.body.incompatible",
    "message": "Response body is incompatible with the response body schema in the spec file: must NOT have additional properties - phoneNumber"
  }
]
```

**Additional Errors for Multiple Fields**:
- `department`: "must NOT have additional properties - department"
- `jobTitle`: "must NOT have additional properties - jobTitle" 
- `preferences`: "must NOT have additional properties - preferences"
- `address`: "must NOT have additional properties - address"

**Result**: ❌ **CONTRACT VERIFICATION FAILS** - Provider's strict validation rejects unknown fields

---

### Scenario 4: Provider Adds Mandatory Field ❌
**Status**: Provider evolves to require new mandatory field unknown to consumer

**Provider Spec**: `openapi-breaking.yaml` (v2.0.0)
```yaml
CreateUserRequest:
  type: object
  required: [name, age, email, department]  # department now mandatory
  properties:
    # ... existing fields ...
    department:
      type: string
      minLength: 1
      maxLength: 50
      enum: ["Engineering", "Marketing", "Sales", "HR", "Finance", "Operations"]
```

**Consumer Pact**: Still uses old contract (no department field)

**Actual Errors**:
```json
[
  {
    "code": "request.body.incompatible",
    "message": "Request body is incompatible with the request body schema in the spec file: must have required property 'department'",
    "mockDetails": {
      "interactionDescription": "Create user with old contract (no department field)",
      "value": {
        "name": "John Doe",
        "age": 30,
        "email": "john.doe@example.com"
      }
    },
    "specDetails": {
      "location": "CreateUserRequest.required",
      "value": ["name", "age", "email", "department"]
    }
  }
]
```

**Result**: ❌ **CONTRACT VERIFICATION FAILS** - Provider breaking change detected

## Contract Testing Summary

| Scenario | Provider | Consumer | Result | Error Count | Issue Type |
|----------|----------|----------|---------|-------------|------------|
| 1 | v1.0.0 Baseline | Matching | ✅ Pass | 0 | None |
| 2 | v1.0.0 Baseline | Missing fields | ❌ Fail | 6 | Consumer breaking change |
| 3 | v1.0.0 Baseline | Extra fields | ❌ Fail | 8 | Provider strictness |
| 4 | v2.0.0 New mandatory | Old baseline | ❌ Fail | 2 | Provider breaking change |

## Key Insights from Error Analysis

### 1. Missing Required Fields (Scenario 2)
- **Impact**: Consumer cannot fulfill provider's basic requirements
- **Error Pattern**: `must have required property 'fieldName'`
- **Detection**: Immediate validation failure at request level
- **Fix**: Consumer must include all required fields

### 2. Additional Properties Rejection (Scenario 3)  
- **Impact**: Provider's strict schema rejects consumer innovations
- **Error Pattern**: `must NOT have additional properties - fieldName`
- **Detection**: Both request and response validation failures
- **Fix**: Provider should allow `additionalProperties: true` or explicitly define new fields

### 3. New Mandatory Requirements (Scenario 4)
- **Impact**: Existing consumers break when provider adds requirements
- **Error Pattern**: `must have required property 'newField'`  
- **Detection**: Request validation failure for missing new requirements
- **Fix**: Provider should make new fields optional initially, or version the API

## OpenAPI Schema Comparison

### Version 1.0.0 (Baseline)
```yaml
required: [name, age, email]
additionalProperties: false
```

### Version 2.0.0 (Breaking Change)
```yaml
required: [name, age, email, department]  # Added department
additionalProperties: false
properties:
  department:
    type: string
    enum: ["Engineering", "Marketing", "Sales", "HR", "Finance", "Operations"]
```

## Pactflow Commands for Testing

```bash
# Publish consumer pacts
pact-broker publish ./pacts --consumer-app-version 1.0.0 --branch main

# Verify provider against pacts  
mvn pact:verify

# Can-i-deploy check
pact-broker can-i-deploy --pacticipant user-consumer --version 1.0.0 --to-environment production
```

## Best Practices Learned

1. **Backward Compatibility**: New required fields break existing consumers
2. **Schema Flexibility**: `additionalProperties: false` prevents consumer innovation
3. **Gradual Evolution**: Make new fields optional first, then mandatory in later versions
4. **Contract Testing Value**: Catches breaking changes before production deployment
5. **Error Clarity**: Pactflow provides detailed error messages for debugging contract mismatches

This comprehensive analysis demonstrates how bidirectional contract testing with Pactflow effectively identifies and prevents API compatibility issues across different evolution scenarios.
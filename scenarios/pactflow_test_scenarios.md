# Pactflow Bidirectional Testing Scenarios

## Overview
This demonstrates bidirectional contract testing with Pactflow using a simple User Creation API. The API creates users with mandatory fields (name, age, email) and returns a generated ID along with user attributes.

## API Contract Details
- **Endpoint**: POST /users
- **Success Response**: 201 with user data including generated ID
- **Error Response**: 400 for invalid age (<0 or >110) or invalid email format
- **Mandatory Fields**: name, age, email (in baseline scenario)

## Scenario Descriptions

### Scenario 1: Perfect Match (Baseline) ✅
**Status**: Both provider and consumer contracts match perfectly

**Provider OpenAPI Spec**:
- Defines user creation with mandatory fields: name, age, email
- Validates age between 0-110
- Validates email format
- Returns 201 with generated ID or 400 for validation errors

**Consumer Pact**:
- Expects same mandatory fields in request
- Expects same response structure with ID
- Tests both success (201) and error (400) scenarios
- Includes validation for negative age, high age (>110), and invalid email

**Expected Result**: ✅ Contract verification passes

---

### Scenario 2: Missing Mandatory Fields ❌
**Status**: Consumer removes mandatory fields from expectations

**Consumer Changes**:
- Requests without email field
- Requests without name field  
- Requests with only age field
- Still expects 201 success responses

**Provider OpenAPI Spec**: Unchanged (still requires all fields)

**Expected Result**: ❌ Contract verification fails
- Provider requires email but consumer doesn't send it
- Provider requires name but consumer doesn't send it
- This demonstrates breaking changes when mandatory fields are missing

---

### Scenario 3: Consumer Adds Unknown Fields ⚠️
**Status**: Consumer adds new fields unknown to provider

**Consumer Changes**:
- Adds phoneNumber field to requests/responses
- Adds address object with street, city, zipCode
- Adds department, jobTitle, and preferences object
- Expects these fields to be returned in response

**Provider OpenAPI Spec**: 
- Unchanged (doesn't know about new fields)
- Uses `additionalProperties: false` (strict mode)

**Expected Result**: ⚠️ Depends on provider implementation
- If provider uses strict validation: ❌ Fails (400 error for unknown fields)
- If provider ignores unknown fields: ⚠️ Partial success (fields ignored in response)
- Highlights the importance of provider flexibility vs strict validation

---

### Scenario 4: Provider Adds Mandatory Field ❌
**Status**: Provider evolves to require new mandatory field unknown to consumer

**Provider Changes**:
- OpenAPI spec now requires "department" as mandatory field
- Updated version 2.0.0
- Department must be one of: Engineering, Marketing, Sales, HR, Finance, Operations
- Returns 400 error when department is missing

**Consumer Pact**: 
- Still uses old contract (no department field)
- Expects 201 success responses
- Unaware of new mandatory requirement

**Expected Result**: ❌ Contract verification fails
- Consumer doesn't send required department field
- Provider returns 400 (missing department) but consumer expects 201
- Demonstrates breaking changes when providers add mandatory fields

## Contract Testing Outcomes

| Scenario | Provider Contract | Consumer Contract | Verification Result | Issue Type |
|----------|------------------|-------------------|-------------------|------------|
| 1 | Baseline | Baseline | ✅ Pass | None |
| 2 | Baseline | Missing fields | ❌ Fail | Consumer breaking change |
| 3 | Baseline | Extra fields | ⚠️ Depends | Provider strictness |
| 4 | New mandatory field | Baseline | ❌ Fail | Provider breaking change |

## Key Learnings

1. **Perfect Match**: Both sides must agree on mandatory fields and response structure
2. **Missing Fields**: Removing mandatory fields breaks the contract from consumer side
3. **Extra Fields**: Success depends on provider's flexibility with unknown fields
4. **New Mandatory Fields**: Adding required fields breaks existing consumers


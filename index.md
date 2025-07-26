# Bidirectional Contract Testing Documentation

Welcome to the comprehensive guide for implementing bidirectional contract testing with PactFlow in Spring Boot applications.

## 🚀 Quick Start

New to bidirectional contract testing? Start here:

1. **[Overview](bidirectional_contract_testing_guide.md)** - Understand the concepts and workflow
2. **[Consumer Testing](spring_boot_consumer_testing_best_practices.md)** - Implement consumer-side contract tests
3. **[Provider Testing](spring_boot_provider_testing_best_practices.md)** - Implement provider-side contract tests
4. **[Compatibility Checks](pactflow_compatibility_checks_reference.md)** - Debug and resolve contract issues

## 📚 Documentation Structure

### Getting Started
- **[Bidirectional Contract Testing Guide](bidirectional_contract_testing_guide.md)** - Core concepts, workflows, and comparison with traditional testing approaches

### Implementation Guides
- **[Spring Boot Consumer Testing Best Practices](spring_boot_consumer_testing_best_practices.md)** - Complete guide for writing consumer contract tests using Pact, HTTP Interfaces, and RestClient
- **[Spring Boot Provider Testing Best Practices](spring_boot_provider_testing_best_practices.md)** - Comprehensive provider testing with REST Assured and OpenAPI validation

### Testing & Scenarios
- **[PactFlow Test Scenarios](pactflow_test_scenarios.md)** - Real-world testing scenarios and expected outcomes
- **[Scenario Examples](scenarios/)** - Sample OpenAPI specs, Pact files, and test reports

### Troubleshooting & Reference
- **[PactFlow Compatibility Checks Reference](pactflow_compatibility_checks_reference.md)** - Complete error code reference and troubleshooting guide

## 🎯 Key Features Covered

### ✅ Consumer Testing
- Pact consumer tests with Spring Boot
- HTTP Interfaces and RestClient integration
- JDK HttpClient and Apache HttpClient support
- Error handling and validation
- System properties configuration

### ✅ Provider Testing  
- REST Assured integration with Spring Boot
- OpenAPI validation filters
- Custom validators for error scenarios
- Test organization and best practices
- Performance and load testing

### ✅ Bidirectional Workflow
- Cross-contract validation
- PactFlow integration
- Deployment safety checks
- Contract publishing and versioning

### ✅ Troubleshooting
- Complete error code reference
- JSON Schema validation errors
- Step-by-step debugging workflow
- Common pitfalls and solutions

## 🔧 Tools & Technologies

This documentation covers integration with:

- **Spring Boot 6.x** - Modern Spring framework features
- **PactFlow** - Contract testing platform
- **Pact JVM** - Consumer contract testing
- **REST Assured** - Provider API testing
- **OpenAPI 3.x** - API specification format
- **Maven/Gradle** - Build tool integration

## 🚦 Testing Approach

The documentation follows a **specification-first** approach where:

1. **Provider teams** create OpenAPI specifications
2. **Consumer teams** generate contracts from actual usage
3. **PactFlow** validates compatibility between contracts
4. **Teams deploy safely** using `can-i-deploy` checks

## 📖 How to Use This Documentation

### For New Teams
1. Start with the [Bidirectional Contract Testing Guide](bidirectional_contract_testing_guide.md)
2. Follow the [Consumer Testing Guide](spring_boot_consumer_testing_best_practices.md) for your consumer applications
3. Implement [Provider Testing](spring_boot_provider_testing_best_practices.md) for your APIs
4. Use the [Compatibility Checks Reference](pactflow_compatibility_checks_reference.md) when issues arise

### For Existing Teams
- **Troubleshooting**: Jump to [Compatibility Checks Reference](pactflow_compatibility_checks_reference.md)
- **Specific Scenarios**: Check [Test Scenarios](pactflow_test_scenarios.md)
- **Implementation Details**: Refer to Consumer or Provider testing guides

### For Platform Teams
- Review all guides for comprehensive understanding
- Use scenario examples for training and onboarding
- Customize configuration examples for your environment

## 🤝 Contributing

This documentation is designed to be:
- **Practical** - Real-world examples you can use immediately
- **Comprehensive** - Covers all aspects of bidirectional contract testing
- **Maintainable** - Clear structure and cross-references
- **Searchable** - Optimized for TechDocs/Backstage integration

---

**Next Steps**: Start with the [Bidirectional Contract Testing Guide](bidirectional_contract_testing_guide.md) to understand the core concepts and workflow.
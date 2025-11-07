# Validator Architecture Overview

This document provides a high-level overview of the terraform-provider-validatefx architecture, explaining how validators, Terraform functions, and integration tests connect.

## Overview

The terraform-provider-validatefx is built with a layered architecture that separates validation logic from Terraform-specific function interfaces. This design promotes code reusability, testability, and maintainability.

```
PlusTerraform Configuration
(uses provider.validatefx.* functions)
Arrow Down
internal/provider/provider.go
(registers functions via Functions())
Arrow Down
internal/functions/*.go
(wraps validators as Terraform functions)
- NewBase64Function()
- NewCIDRFunction()
- newStringValidationFunction() helper
Arrow Down
internal/validators/*.go
(core validation logic)
- Base64Validator()
- CIDRValidator()
- implements frameworkvalidator.String
```

## Architecture Flow

### 1. Validators (internal/validators/)

Validators contain the core validation logic and are framework-agnostic implementations that follow the Terraform Plugin Framework's validator interface.

**Structure:**
- Each validator is defined in its own file (e.g., `base64.go`, `cidr.go`)
- Implements the `frameworkvalidator.String` interface
- Provides `Description()`, `MarkdownDescription()`, and `ValidateString()` methods
- Returns validation errors via `resp.Diagnostics.AddAttributeError()`

**Key Points:**
- Validators handle null/unknown values gracefully
- Error messages are clear and actionable
- Each validator includes comprehensive unit tests in `*_test.go` files

### 2. Functions (internal/functions/)

Functions wrap validators and expose them as callable Terraform functions. They use helper functions to reduce boilerplate code.

**Structure:**
- Each function file corresponds to a validator (e.g., `base64.go` -> `Base64Validator()`)
- Uses `newStringValidationFunction()` helper from `common.go`
- Returns `function.Function` interface

**Helper Function:** `internal/functions/common.go`

The `newStringValidationFunction()` helper creates a consistent wrapper that:
- Defines the function signature (string input -> boolean output)
- Handles null/unknown input propagation
- Invokes the underlying validator
- Converts validation results to boolean return values

### 3. Provider Registration (internal/provider/)

The provider registers all available functions so Terraform can discover and use them.

**Key File:** `internal/provider/provider.go`

The `Functions()` method returns all available functions.

**Registry:** `internal/functions/registry.go`

Contains `ProviderFunctionFactories()` which returns a slice of all function constructors.

### 4. Integration Tests (integration/)

Integration tests verify the entire flow works end-to-end with real Terraform configurations.

**Structure:**
- `integration/main.tf` contains Terraform code that calls validation functions
- Tests run in Docker containers for consistency
- Execute via `make integration`

## Extension Points

### Adding a New Validator

To add a new validator, follow this workflow (as documented in `CONTRIBUTING.md`):

1. **Create the Validator** (`internal/validators/<name>.go`)
   - Implement the validator struct and interface methods
   - Add comprehensive tests in `internal/validators/<name>_test.go`

2. **Create the Function** (`internal/functions/<name>.go`)
   - Use `newStringValidationFunction()` helper
   - Add tests in `internal/functions/<name>_test.go`

3. **Register the Function** (`internal/functions/registry.go`)
   - Add `New<Name>Function` to the `ProviderFunctionFactories()` slice

4. **Add Examples** (`examples/functions/<name>/`)
   - Create example Terraform configurations

5. **Add Integration Tests** (`integration/main.tf`)
   - Add test cases for the new function

6. **Generate Documentation** 
   - Run `make docs` to generate function documentation
   - Update README if needed

## Coding Conventions

### File Naming
- Validator files: `internal/validators/<validation_name>.go`
- Function files: `internal/functions/<validation_name>.go`
- Test files: `*_test.go` alongside implementation files

### Testing
- Use table-driven tests for comprehensive coverage
- Test success cases, failure cases, and edge cases (null, unknown, empty)
- Integration tests should cover real-world usage scenarios

### Error Handling
- Use `resp.Diagnostics.AddAttributeError()` for validation failures
- Provide clear, actionable error messages
- Include the problematic value in error messages when safe

### Null/Unknown Handling
- Always check for null or unknown values early
- Return `types.BoolUnknown()` for unknown inputs in functions
- Validators should return early without errors for null/unknown values

## Key Files Reference

| Path | Purpose |
|------|----------|
| `internal/validators/*.go` | Core validation logic implementations |
| `internal/validators/*_test.go` | Validator unit tests |
| `internal/functions/*.go` | Terraform function wrappers |
| `internal/functions/common.go` | Shared helper functions |
| `internal/functions/registry.go` | Function registration |
| `internal/provider/provider.go` | Provider implementation and setup |
| `integration/main.tf` | End-to-end integration tests |
| `examples/functions/*/` | Usage examples for each function |
| `CONTRIBUTING.md` | Detailed contribution workflow |

## Development Workflow

1. **Setup**: `make deps` to install dependencies
2. **Development**: Write validator and function code
3. **Testing**: `make test` for unit tests
4. **Formatting**: `make fmt` to format code
5. **Linting**: `make lint` to check code quality
6. **Documentation**: `make docs` to generate documentation
7. **Integration**: `make integration` to run end-to-end tests
8. **Validation**: `make validate` runs the full CI pipeline locally

## Additional Resources

- [CONTRIBUTING.md](../CONTRIBUTING.md) - Detailed contribution guide
- [CODE_OF_CONDUCT.md](../CODE_OF_CONDUCT.md) - Community guidelines
- [SECURITY.md](../SECURITY.md) - Security policy
- [Terraform Plugin Framework Documentation](https://developer.hashicorp.com/terraform/plugin/framework)

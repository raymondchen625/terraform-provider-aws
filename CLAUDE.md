# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is the **Terraform AWS Provider** - a large-scale Go project that enables Terraform to manage AWS resources. The provider covers 259+ AWS services and uses a hybrid architecture with both Plugin SDK v2 (legacy) and Plugin Framework (modern) implementations multiplexed together.

## Development Commands

### Building and Installing

```bash
# Build the provider (installs to GOPATH)
make build

# Build for specific Go version (from .go-version: 1.24.8)
GO_VER=go1.24.8 make build

# Ensure Go version is installed
make prereq-go
```

### Running Tests

```bash
# Run acceptance tests for a specific service (use PKG or K interchangeably)
make testacc PKG=iam T=TestAccIAMRole_basic
make testacc K=s3 T=TestAccS3Bucket_basic

# Run all tests for a service
make testacc PKG=ec2

# Run acceptance tests with shorter alias
make t K=lambda T=TestAccLambda

# Run unit tests (non-acceptance)
make test PKG=iam

# Run acceptance tests with -short flag
make testacc-short K=s3

# Run quick sanity smoke tests
make sane

# Set parallelism (default 20)
make testacc PKG=iam P=10

# Run test with specific count
TEST_COUNT=5 make testacc PKG=s3 T=TestAccS3Bucket_basic
```

**Note**: Several EC2 service aliases are automatically mapped:
- `PKG=ebs` → `internal/service/ec2`
- `PKG=vpc` → `internal/service/ec2`
- `PKG=ipam` → `internal/service/ec2`
- `PKG=transitgateway` → `internal/service/ec2`
- `PKG=vpnclient` → `internal/service/ec2`
- `PKG=vpnsite` → `internal/service/ec2`
- `PKG=wavelength` → `internal/service/ec2`

### Code Quality and Linting

```bash
# Quick fix for common issues (runs fmt, lint fixes, imports, etc.)
make quick-fix K=s3

# Format code
make fmt

# Fix imports
make fix-imports

# Run all linters
make golangci-lint PKG=s3

# Run specific linter checks
make golangci-lint1 K=ec2
make provider-lint K=iam

# Fix acceptance test formatting
make testacc-lint-fix

# Run semgrep checks
make semgrep-code-quality
make semgrep-fix K=lambda

# Modern Go code fixes
make modern-fix K=dynamodb
```

### Code Generation

```bash
# Run all code generators
make gen

# Generate and check for differences (CI check)
make gen-check

# Create new resource/data source using skaff
cd skaff && go run . resource -n my_resource -s myservice
cd skaff && go run . datasource -n my_data_source -s myservice
```

### Sweepers (Test Cleanup)

```bash
# Run sweepers to clean up test resources
make sweep SWEEPARGS=-sweep-run=aws_iam_role

# Run sweepers with failures allowed
make sweeper
```

### Tools Installation

```bash
# Install all development tools
make tools

# Install skaff scaffolding tool
make skaff

# Clean and reinstall everything
make clean
```

## Architecture

### Dual Provider Implementation

The provider uses a **multiplexed architecture** combining two patterns:

1. **Plugin SDK v2** (Legacy) - `internal/provider/sdkv2/`
   - Traditional `schema.Resource` with CRUD functions
   - Most existing resources use this pattern
   - Example: `ec2_instance`, `s3_bucket`

2. **Plugin Framework** (Modern) - `internal/provider/framework/`
   - Typed models with modern Go patterns
   - New resources should use this pattern
   - Better type safety and composition
   - Example: `ec2_capacity_block_reservation`

Both are multiplexed via `tf5muxserver` into a single provider.

### Service Organization

Services are located in `internal/service/<service_name>/` with standard structure:

```
internal/service/<service>/
├── service_package_gen.go    # Generated service registration (DO NOT EDIT)
├── <resource>.go              # Resource implementation
├── <resource>_test.go         # Acceptance tests
├── <data_source>.go          # Data source implementation
├── find.go                   # Common AWS API finder functions
├── status.go                 # Status check functions
├── wait.go                   # Waiter/retry functions
├── sweep.go                  # Test resource cleanup
└── tags_gen.go               # Generated tagging code (DO NOT EDIT)
```

Large services like EC2 have resources grouped by feature area (e.g., `ebs_*.go`, `vpc_*.go`).

### Code Generation

The provider heavily uses code generation. Generated files have `_gen.go` suffix or header comments - **DO NOT manually edit them**. Instead:

- Modify generator code in `internal/generate/`
- Use annotations in resource files (e.g., `// @SDKResource`, `// @Tags`)
- Run `make gen` to regenerate

### Resource Annotations

Resources use comment annotations for code generation:

```go
// @SDKResource("aws_instance", name="Instance")
// @Tags(identifierAttribute="id")
// @Testing(importIgnore="user_data")
```

### Key Architectural Patterns

1. **Single API Object per Resource** - Each resource maps 1:1 to an AWS API object
2. **No Cross-Service Resources** - Resources only interact with their primary AWS service
3. **Separate Policy Resources** - IAM policies are separate resources
4. **Immutable Infrastructure** - Designed for Infrastructure-as-Code patterns
5. **Framework Composition** - Modern resources use composition:
   ```go
   type myResource struct {
       framework.ResourceWithModel[...]
       framework.WithTimeouts
       framework.WithImportByID
   }
   ```

## Common Development Workflows

### Adding a New Resource

Use the `skaff` tool for scaffolding:

```bash
cd skaff
go run . resource -n my_new_resource -s myservice
```

This generates fully-commented resource and test files following best practices. See `docs/add-a-new-resource.md` for details.

### Working on an Existing Service

```bash
# Quick iteration cycle for a service
make quick-fix K=s3        # Format, lint, fix imports
make t K=s3 T=TestAcc...   # Run specific test
make testacc K=s3          # Run all tests
```

### Understanding Test Patterns

Acceptance tests (`_test.go` files) use the pattern:

```go
func TestAccServiceResource_basic(t *testing.T) {
    ctx := acctest.Context(t)
    // Test implementation
}
```

- Test names: `TestAcc<Service><Resource>_<testCase>`
- Use `acctest.Context(t)` for context with timeout
- Resource naming: Use `rName := sdkacctest.RandomWithPrefix(acctest.ResourcePrefix)`
- Check functions: `resource.TestCheckResourceAttr()`, `resource.TestCheckResourceAttrPair()`

### Running Single Tests

```bash
# Full test name
make t K=iam T=TestAccIAMRole_basic

# Pattern matching (runs all matching tests)
make t K=iam T=TestAccIAMRole_

# Multiple tests
make testacc PKG=iam T='TestAccIAMRole_basic|TestAccIAMRole_namePrefix'
```

## Key Directories

- `internal/service/` - All AWS service implementations (259+ services)
- `internal/conns/` - AWS client connections and configuration
- `internal/flex/` - Type conversion utilities (AWS SDK ↔ Terraform)
- `internal/verify/` - Input validation functions
- `internal/tfresource/` - Resource state management helpers
- `internal/errs/` - Error handling utilities
- `internal/tags/` - Standardized tag management
- `internal/acctest/` - Acceptance test utilities and helpers
- `internal/generate/` - Code generation tools
- `skaff/` - Resource/data source scaffolding tool
- `names/` - AWS service name constants and metadata
- `docs/` - Comprehensive developer documentation

## Important Files

- `GNUmakefile` - All development commands
- `.go-version` - Required Go version (1.24.8)
- `go.mod` - Go module dependencies
- `docs/makefile-cheat-sheet.md` - Quick reference for make targets
- `docs/running-and-writing-acceptance-tests.md` - Test guidance

## Environment Variables

Required for acceptance tests:

```bash
export TF_ACC=1                          # Enable acceptance tests
export AWS_DEFAULT_REGION=us-west-2      # AWS region
# AWS credentials via standard AWS SDK methods (env vars, ~/.aws/credentials, etc.)
```

Optional:

```bash
export TF_LOG=DEBUG                      # Enable Terraform logging
export TF_ACC_ASSUME_ROLE_ARN=...       # Use specific IAM role
```

See `docs/acc-test-environment-variables.md` for comprehensive list.

## Common Patterns and Conventions

### Error Handling

```go
// Framework resources
return diag.Errorf("error message: %s", err)

// SDK v2 resources
return fmt.Errorf("error message: %s", err)

// Use errs package for error checking
if errs.IsNotFound(err) { ... }
```

### AWS API Calls

```go
// Get client from context
client := meta.(*conns.AWSClient).ServiceConn(ctx)

// Use finders for read operations
resource, err := tfservice.FindResourceByID(ctx, client, id)
if tfresource.NotFound(err) {
    // Handle not found
}

// Use waiters for eventual consistency
_, err = tfservice.WaitResourceCreated(ctx, client, id, timeout)
```

### Tagging

Resources with tags use the `@Tags` annotation:

```go
// @Tags(identifierAttribute="id")
// @Tags(identifierAttribute="arn")
```

This generates tag handling code automatically. Use `tags.TagsIn(ctx)` and `tags.TagsOut(ctx)` in CRUD operations.

## Testing Philosophy

- **Acceptance tests are primary** - Unit tests are minimal
- Tests hit real AWS APIs (require AWS credentials)
- Use sweepers to clean up leaked resources
- Tests must be idempotent and isolated
- Always test Create, Read, Update (if applicable), Delete, and Import
- Include disappears tests for resource deletion handling

## Submitting Changes

When creating commits:

1. Run `make quick-fix K=<service>` to fix common issues
2. Ensure all tests pass: `make testacc K=<service>`
3. Regenerate code if needed: `make gen`
4. Follow commit message format from recent commits (see `git log`)
5. Generated files in commits are expected - don't worry about their size

## Performance Notes

- The codebase is **very large** (~259 services, extensive test coverage)
- `make testacc` runs can take hours for large services like EC2
- Use `T=` parameter to run specific tests during development
- Use `P=` to adjust parallelism (default 20, lower if hitting AWS rate limits)
- CI runs comprehensive checks; local development focuses on affected services

## Additional Resources

- Comprehensive documentation in `docs/` directory
- Contributing guide: https://hashicorp.github.io/terraform-provider-aws/
- Development environment setup: `docs/development-environment.md`
- Provider design principles: `docs/provider-design.md`
- FAQ: `docs/faq.md`

## Migration to Framework

The provider is gradually migrating from SDK v2 to Plugin Framework:

- **New resources**: Use Framework pattern
- **Existing resources**: SDK v2 until explicitly migrated
- See `docs/terraform-plugin-migrations.md` for migration guidance
- Both patterns coexist seamlessly via multiplexing

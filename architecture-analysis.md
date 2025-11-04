# Terraform AWS Provider - Architecture Analysis

## Project Overview

**Project**: Terraform AWS Provider
**Language**: Go 1.24.8
**Type**: Large-scale Infrastructure-as-Code provider
**Scope**: 259+ AWS services
**Architecture**: Hybrid multiplexed provider (Plugin SDK v2 + Plugin Framework)

## Architecture

### 1. Multiplexed Dual-Provider Architecture

The provider uses a sophisticated **multiplexed architecture** that combines two Terraform plugin patterns into a single provider:

```
┌─────────────────────────────────────────┐
│     Terraform Core (Protocol v5)        │
└──────────────┬──────────────────────────┘
               │
      ┌────────▼────────┐
      │  tf5muxserver   │  (Multiplexer)
      └────────┬────────┘
               │
       ┌───────┴────────┐
       │                │
┌──────▼─────┐   ┌─────▼──────┐
│  SDKv2     │   │ Framework  │
│ (Primary)  │   │(Secondary) │
│  Legacy    │   │   Modern   │
└────────────┘   └────────────┘
```

**Implementation** (`internal/provider/factory.go`):
- Uses `tf5muxserver.NewMuxServer()` to combine both providers
- SDK v2 provider is **primary** (returns metadata, handles most resources)
- Plugin Framework provider is **secondary** (new resources, modern patterns)
- Both share the same configuration and AWS client connection

### 2. Service Package Organization

Each AWS service is organized under `internal/service/<service>/`:

```
internal/service/<service>/
├── service_package_gen.go    # Generated service registration
├── <resource>.go              # Resource implementations
├── <resource>_test.go         # Acceptance tests
├── <data_source>.go          # Data source implementations
├── find.go                   # AWS API finder functions
├── status.go                 # Status check functions
├── wait.go                   # Waiter/retry logic
├── sweep.go                  # Test resource cleanup
└── tags_gen.go               # Generated tagging code
```

**Design Principles**:
- Single API object per resource (1:1 mapping)
- No cross-service resource dependencies
- Separate policy resources (IAM policies are standalone)
- Immutable infrastructure pattern

### 3. Provider Initialization Flow

```go
main.go
  └─> provider.ProtoV5ProviderServerFactory()
       ├─> sdkv2.NewProvider() [Primary]
       │    └─> Registers SDK v2 resources/data sources
       ├─> framework.NewProvider() [Secondary]
       │    └─> Registers Framework resources/data sources
       └─> tf5muxserver.NewMuxServer()
            └─> Returns multiplexed provider server
```

## Core Frameworks

### 1. Terraform Plugin SDK v2 (Legacy Pattern)

**Location**: `internal/provider/sdkv2/`

**Characteristics**:
- Traditional CRUD function-based approach
- Uses `schema.Resource` with `CreateContext`, `ReadContext`, `UpdateContext`, `DeleteContext`
- Most existing resources (majority of 259 services)
- Map-based configuration handling

**Example Pattern**:
```go
&schema.Resource{
    CreateContext: resourceCreate,
    ReadContext:   resourceRead,
    UpdateContext: resourceUpdate,
    DeleteContext: resourceDelete,
    Schema: map[string]*schema.Schema{...},
}
```

### 2. Terraform Plugin Framework (Modern Pattern)

**Location**: `internal/provider/framework/`

**Characteristics**:
- Type-safe with Go structs
- Composition-based architecture
- Better validation and diagnostics
- New resources should use this pattern
- Framework interceptors for cross-cutting concerns

**Example Pattern**:
```go
type resourceWithModel struct {
    framework.ResourceWithModel[resourceModel]
    framework.WithTimeouts
    framework.WithImportByID
}
```

**Key Interfaces**:
- `provider.ProviderWithActions`
- `provider.ProviderWithFunctions`
- `provider.ProviderWithEphemeralResources`
- `provider.ProviderWithListResources`

### 3. Terraform Plugin Mux

**Package**: `github.com/hashicorp/terraform-plugin-mux/tf5muxserver`

**Purpose**: Seamlessly combines SDK v2 and Framework providers into single provider
**Version**: Protocol v5 (tfprotov5)

## Key Libraries and Dependencies

### AWS SDK Libraries

```go
// AWS SDK v2 - Core
github.com/aws/aws-sdk-go-v2 v1.39.4
github.com/aws/aws-sdk-go-v2/config v1.31.15
github.com/aws/aws-sdk-go-v2/credentials v1.18.19

// 259+ AWS Service Clients (examples):
github.com/aws/aws-sdk-go-v2/service/ec2 v1.258.1
github.com/aws/aws-sdk-go-v2/service/s3 v1.88.7
github.com/aws/aws-sdk-go-v2/service/iam v1.49.0
github.com/aws/aws-sdk-go-v2/service/lambda v1.79.0
// ... and 255+ more service clients
```

**Note**: Uses AWS SDK Go v2 (not v1), with individual service client packages for each AWS service.

### Terraform Plugin Libraries

```go
// Plugin Framework (Modern)
github.com/hashicorp/terraform-plugin-framework v1.16.1
github.com/hashicorp/terraform-plugin-framework-jsontypes v0.2.0
github.com/hashicorp/terraform-plugin-framework-timeouts v0.7.0
github.com/hashicorp/terraform-plugin-framework-timetypes v0.5.0
github.com/hashicorp/terraform-plugin-framework-validators v0.19.0

// Plugin SDK v2 (Legacy)
github.com/hashicorp/terraform-plugin-sdk/v2 v2.38.1

// Plugin Infrastructure
github.com/hashicorp/terraform-plugin-go v0.29.0
github.com/hashicorp/terraform-plugin-log v0.9.0
github.com/hashicorp/terraform-plugin-mux v0.21.0
github.com/hashicorp/terraform-plugin-testing v1.14.0-beta.1
```

### HashiCorp Ecosystem Libraries

```go
github.com/hashicorp/aws-sdk-go-base/v2 v2.0.0-beta.67  // AWS config/auth
github.com/hashicorp/awspolicyequivalence v1.7.0         // IAM policy comparison
github.com/hashicorp/go-cty v1.5.0                       // Type system
github.com/hashicorp/hcl/v2 v2.24.0                      // HCL parsing
github.com/hashicorp/go-multierror v1.1.1                // Error aggregation
github.com/hashicorp/go-cleanhttp v0.5.2                 // HTTP client
```

### Utility Libraries

```go
github.com/YakDriver/regexache v0.24.0           // Compiled regex caching
github.com/mitchellh/copystructure v1.2.0        // Deep copying
github.com/mitchellh/mapstructure v1.5.0         // Map to struct conversion
github.com/google/go-cmp v0.7.0                  // Deep comparison
github.com/shopspring/decimal v1.4.0             // Decimal math
github.com/gertd/go-pluralize v0.2.1             // String pluralization
gopkg.in/dnaeon/go-vcr.v4 v4.0.5                 // VCR testing support
```

### OpenTelemetry

```go
go.opentelemetry.io/otel v1.38.0
go.opentelemetry.io/contrib/instrumentation/github.com/aws/aws-sdk-go-v2/otelaws v0.63.0
```

## Build Tools and Development Infrastructure

### 1. Make-Based Build System

**File**: `GNUmakefile` (1036 lines)

**Key Make Targets**:

```makefile
# Building
make build              # Build and install provider
make go-build          # Build provider binary

# Testing
make test              # Unit tests
make testacc           # Acceptance tests (AWS API calls)
make t                 # Short alias for testacc
make sane             # Quick smoke tests
make sweep            # Clean up test resources

# Code Quality
make fmt               # Format code (gofmt)
make golangci-lint     # Run all linters (5 configurations)
make provider-lint     # Custom provider linters
make semgrep          # Semgrep security/quality checks

# Code Generation
make gen               # Run all generators
make gen-check         # Verify generated code matches

# Quick Iteration
make quick-fix K=<service>  # Format, lint, fix imports, modernize
```

**Variables**:
- `PKG=<service>` or `K=<service>` - Service name
- `T=<pattern>` - Test pattern
- `P=20` - Parallelism (default 20)
- `TEST_COUNT=1` - Test repetition count

### 2. Linting Infrastructure

**Multiple Linter Configurations**:

```
.ci/.golangci.yml      # Primary golangci-lint config
.ci/.golangci2.yml     # Additional checks
.ci/.golangci3.yml     # More checks
.ci/.golangci4.yml     # Even more checks
.ci/.golangci5.yml     # Final checks
```

**Linters Used**:
- **golangci-lint** - Meta-linter aggregator
- **providerlint** - Custom AWS provider-specific linter (`.ci/providerlint/`)
- **Semgrep** - Pattern-based code analysis (multiple rule files)
- **impi** - Import order checking
- **terrafmt** - Terraform HCL formatting in docs
- **tflint** - Terraform linting
- **actionlint** - GitHub Actions workflow linting
- **yamllint** - YAML validation
- **misspell** - Spell checking

### 3. Code Generation Tools

**Generators** (`internal/generate/`):

```go
//go:generate go run ./...   # Common pattern
```

**Generated Files** (marked with `_gen.go` suffix or header comments):
- `service_package_gen.go` - Service registration
- `tags_gen.go` - Tag handling code
- `attr_consts_gen.go` - Attribute name constants
- `consts_gen.go` - Service name constants

**Key Generator Tools**:
- **skaff** (`skaff/`) - Resource/data source scaffolding tool
- **stringer** - Enum string generation
- Service package generators

### 4. Testing Infrastructure

**Framework**: `github.com/hashicorp/terraform-plugin-testing`

**Test Types**:

1. **Acceptance Tests** (Primary):
   - Hit real AWS APIs
   - Require AWS credentials
   - Named `TestAcc<Service><Resource>_<testCase>`
   - Use `resource.Test()` framework
   - Parallelized (default: 20)

2. **Unit Tests**:
   - Minimal usage (provider focuses on acceptance tests)
   - Standard Go testing

3. **Sweepers**:
   - Clean up leaked test resources
   - Located in `sweep.go` files
   - Run with `make sweep`

**Test Utilities** (`internal/acctest/`):
- `acctest.Context(t)` - Test context with timeout
- `acctest.ResourcePrefix` - Consistent resource naming
- `resource.TestCheckResourceAttr()` - State verification
- VCR recording support for offline testing

**Example Acceptance Test Pattern**:
```go
func TestAccServiceResource_basic(t *testing.T) {
    ctx := acctest.Context(t)
    var resource ServiceResource
    rName := sdkacctest.RandomWithPrefix(acctest.ResourcePrefix)

    resource.Test(t, resource.TestCase{
        PreCheck:     func() { acctest.PreCheck(ctx, t) },
        Providers:    acctest.Providers,
        CheckDestroy: testAccCheckResourceDestroy(ctx),
        Steps: []resource.TestStep{
            {
                Config: testAccResourceConfig_basic(rName),
                Check: resource.ComposeTestCheckFunc(
                    testAccCheckResourceExists(ctx, resourceName, &resource),
                    resource.TestCheckResourceAttr(resourceName, "name", rName),
                ),
            },
        },
    })
}
```

### 5. Development Tools

**Installed via `make tools`**:

```bash
# Code Quality
golangci-lint    # Linter aggregator
providerlint     # Custom provider linter
impi            # Import order checker
terrafmt        # Terraform formatting
tflint          # Terraform linter

# Documentation
tfproviderdocs  # Provider docs checker
misspell        # Spell checker

# Other
copywrite       # License header management
actionlint      # GitHub Actions linter
gofumpt         # Stricter gofmt
changelog-build # Changelog generation
```

### 6. Continuous Integration

**CI Targets** (from `GNUmakefile`):

```makefile
make ci           # Full CI suite (all checks)
make ci-quick     # Faster CI subset
```

**CI Checks Include**:
- Code generation verification (`gen-check`)
- All linter suites (golangci-lint 1-5)
- Provider-specific lints
- Semgrep scans
- Documentation checks
- Website validation
- Copyright header checks
- Dependency verification
- Sweeper linking checks

## Internal Package Organization

### Core Internal Packages

```
internal/
├── conns/              # AWS client connections and config
├── provider/           # Provider implementations (sdkv2 + framework)
├── service/            # AWS service implementations (259+ services)
├── acctest/            # Acceptance test utilities
├── flex/               # Type conversion (AWS ↔ Terraform)
├── verify/             # Input validation functions
├── tfresource/         # Resource state management
├── errs/               # Error handling utilities
├── tags/               # Standardized tag management
├── framework/          # Framework utilities and helpers
├── generate/           # Code generators
└── sweep/              # Test resource sweepers
```

### Key Utilities

**Type Conversion** (`internal/flex/`):
- AWS SDK types ↔ Terraform types
- Flattening/expanding nested structures
- Null/zero value handling

**Error Handling** (`internal/errs/`):
- AWS error type checking
- `errs.IsNotFound(err)` - Check for 404 errors
- Error wrapping and context

**Tags** (`internal/tags/`):
- Unified tag handling across services
- Default tags support
- Tag ignore patterns

**Validation** (`internal/verify/`):
- ARN validation
- Region validation
- Account ID validation
- Custom validators

## Code Generation Patterns

### 1. Service Package Generation

**Trigger**: `//go:generate` directives in service directories

**Output**: `service_package_gen.go`

**Contains**:
- Resource registration
- Data source registration
- Service client initialization
- Service metadata

### 2. Tag Generation

**Annotation**: `// @Tags(identifierAttribute="id")`

**Output**: `tags_gen.go`

**Generates**:
- Tag CRUD operations
- Tag update functions
- Tag flattening/expanding

### 3. Resource Annotations

Resources use comment annotations for code generation:

```go
// @SDKResource("aws_instance", name="Instance")
// @Tags(identifierAttribute="id")
// @Testing(importIgnore="user_data")
```

## Provider Configuration

### AWS Authentication

**Methods Supported** (via `aws-sdk-go-base`):
- Environment variables (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`)
- Shared credentials file (`~/.aws/credentials`)
- IAM role assumption
- EC2 instance metadata
- Web identity federation

**Provider Block**:
```hcl
provider "aws" {
  region              = "us-west-2"
  access_key          = "..."
  secret_key          = "..."

  assume_role {
    role_arn = "..."
  }

  default_tags {
    tags = {
      Environment = "production"
    }
  }
}
```

## Resource Implementation Patterns

### SDK v2 Pattern (Legacy)

```go
func ResourceExample() *schema.Resource {
    return &schema.Resource{
        CreateContext: resourceExampleCreate,
        ReadContext:   resourceExampleRead,
        UpdateContext: resourceExampleUpdate,
        DeleteContext: resourceExampleDelete,

        Schema: map[string]*schema.Schema{
            "name": {
                Type:     schema.TypeString,
                Required: true,
            },
        },
    }
}

func resourceExampleCreate(ctx context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
    conn := meta.(*conns.AWSClient).ExampleClient(ctx)

    input := &example.CreateInput{
        Name: aws.String(d.Get("name").(string)),
    }

    output, err := conn.Create(ctx, input)
    if err != nil {
        return diag.Errorf("creating Example: %s", err)
    }

    d.SetId(aws.ToString(output.Id))

    return resourceExampleRead(ctx, d, meta)
}
```

### Framework Pattern (Modern)

```go
type resourceExample struct {
    framework.ResourceWithModel[resourceExampleModel]
    framework.WithTimeouts
    framework.WithImportByID
}

type resourceExampleModel struct {
    ID   types.String `tfsdk:"id"`
    Name types.String `tfsdk:"name"`
}

func (r *resourceExample) Create(ctx context.Context, req resource.CreateRequest, resp *resource.CreateResponse) {
    var data resourceExampleModel
    resp.Diagnostics.Append(req.Plan.Get(ctx, &data)...)

    conn := r.Meta().ExampleClient(ctx)

    input := &example.CreateInput{
        Name: data.Name.ValueStringPointer(),
    }

    output, err := conn.Create(ctx, input)
    if err != nil {
        resp.Diagnostics.AddError("creating Example", err.Error())
        return
    }

    data.ID = types.StringValue(aws.ToString(output.Id))

    resp.Diagnostics.Append(resp.State.Set(ctx, &data)...)
}
```

## Performance Considerations

### Parallelism

- **Default test parallelism**: 20 concurrent tests
- **Adjustable**: `make testacc P=10` for lower parallelism
- **Large services**: EC2 has 1000+ tests, can take hours

### Build Times

- **Full build**: Fast (Go compilation)
- **Code generation**: Moderate (depends on generators)
- **Full CI suite**: Extensive (all linters + checks)

### Provider Size

- **Binary size**: Large (259 services, thousands of resources)
- **Compilation**: Optimized with service package pattern
- **Memory**: Service clients initialized on-demand

## Security and Quality

### Security Scanning

- **Semgrep**: Multiple rulesets for security patterns
- **Custom rules**: AWS-specific security checks
- **Dependency scanning**: Via GitHub Actions

### Quality Gates

1. All tests must pass
2. Code generation must be current
3. All linters must pass
4. Documentation must be valid
5. Copyright headers present
6. No sweeper functions in provider binary

## Migration Strategy

### SDK v2 → Framework Migration

**Status**: Gradual migration in progress

**Guidelines**:
- **New resources**: Use Framework
- **Existing resources**: SDK v2 until explicitly migrated
- **Both coexist**: Via multiplexing
- **No breaking changes**: Seamless for users

**Migration Documentation**: `docs/terraform-plugin-migrations.md`

## Summary

The Terraform AWS Provider is a **massive, production-grade infrastructure project** with:

- ✅ **Hybrid architecture** supporting both legacy and modern Terraform patterns
- ✅ **259+ AWS services** with comprehensive resource coverage
- ✅ **Extensive code generation** reducing manual coding
- ✅ **Robust testing** with acceptance tests hitting real AWS APIs
- ✅ **Multi-layered linting** ensuring code quality
- ✅ **Make-based build system** with 100+ targets
- ✅ **Active development** with clear migration path to modern patterns

**Key Strengths**:
1. Multiplexed architecture enables gradual modernization
2. Service package pattern scales to 259+ services
3. Comprehensive code generation reduces boilerplate
4. Extensive testing infrastructure ensures reliability
5. Well-organized internal packages promote maintainability

**Technology Stack**:
- **Language**: Go 1.24.8
- **Primary Framework**: Terraform Plugin Framework (modern) + SDK v2 (legacy)
- **AWS SDK**: AWS SDK for Go v2
- **Build Tool**: GNU Make
- **Testing**: Terraform Plugin Testing framework
- **Linting**: golangci-lint + custom providerlint + Semgrep
- **CI/CD**: GitHub Actions (implied from actionlint usage)

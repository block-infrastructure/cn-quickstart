# Plan: C# / .NET Bindings for Canton Ledger API + Daml Codegen

## Context

Canton Network applications interact with the ledger via a **gRPC Ledger API v2**. Digital Asset provides official code generators for Java and JavaScript, but not for C#. An official `daml-net` package existed but was **archived January 2025** and only supported Ledger API v1.

This plan covers building three layers:
1. gRPC stub generation from Ledger API proto files
2. A thin .NET client wrapper for the Ledger API
3. A standalone CLI tool that generates type-safe C# bindings from compiled Daml packages (`.dar` files)

### Reference Implementations

| Project | Language | Status | Value |
|---------|----------|--------|-------|
| `go-daml` (github.com/noders-team/go-daml) | Go | Active, supports LF v2/v3 | Codegen architecture, DAR parsing |
| `rust-daml-bindings` (github.com/fujiapple852/rust-daml-bindings) | Rust | Archived | Most complete non-Java codegen (9 crates) |
| `daml-net` (github.com/digital-asset/daml-net) | C# | Archived (v1 only) | gRPC stub generation scripts, client wrapper patterns |
| CN Quickstart Java bindings | Java | Active | Target output structure to replicate |

### Key Technical Facts

- The Ledger API v2 proto files are located in `com/daml/ledger/api/v2/` package
- Proto files can be extracted from the Canton SDK or found in the project at `quickstart/backend/build/extracted-protos/main/com/daml/ledger/api/v2/`
- `.dar` files are ZIP archives containing `.dalf` files (protobuf-serialized Daml-LF AST)
- The Daml-LF proto schema is public at `daml-lf/archive/src/main/protobuf/com/digitalasset/daml_lf_dev/daml_lf.proto`
- The Java codegen uses a library called "transcode" (`com.digitalasset.transcode`) which provides `fromDynamicValue`/`toDynamicValue` converters and a `Dictionary` registry mapping identifiers to type converters

---

## Phase 1: Generate gRPC Stubs from Ledger API v2 Protos

### Goal
Produce C# service stubs and message classes for all Ledger API v2 services.

### Approach
Use `Grpc.Tools` (the standard .NET protobuf/gRPC compiler) to generate C# code from the Canton Ledger API `.proto` files.

### Input
Proto files from `com.daml.ledger.api.v2` package. These define the following services:

**Core Services:**

| Service | Key RPCs | Purpose |
|---------|----------|---------|
| `CommandService` | `SubmitAndWait`, `SubmitAndWaitForTransaction`, `SubmitAndWaitForUpdateId` | Synchronous command submission — submit and block until result |
| `CommandSubmissionService` | `Submit` | Async fire-and-forget command submission |
| `CommandCompletionService` | `CompletionStream` | Stream command completion status |
| `UpdateService` | `GetUpdates`, `GetTransactionById` | Stream ledger transactions, reassignments, topology |
| `StateService` | `GetActiveContracts`, `GetConnectedSynchronizers`, `GetLedgerEnd` | Active contract snapshots, ledger state |
| `EventQueryService` | `GetEventsByContractId` | Query events for a contract |
| `PackageService` | `ListPackages`, `GetPackage`, `GetPackageStatus` | Query deployed Daml packages |
| `VersionService` | `GetLedgerApiVersion` | API version and feature discovery |

**Admin Services (under `com.daml.ledger.api.v2.admin`):**

| Service | Key RPCs | Purpose |
|---------|----------|---------|
| `PartyManagementService` | `AllocateParty`, `GetParties`, `ListKnownParties` | Party lifecycle |
| `UserManagementService` | `CreateUser`, `GetUser`, `GrantRights`, `RevokeRights` | User provisioning |
| `PackageManagementService` | `UploadDarFile`, `ValidateDarFile`, `ListKnownPackages` | DAR upload/vetting |

**Key Protobuf Message Types (from `value.proto`, `commands.proto`, `event.proto`):**

```protobuf
// Wire format for all Daml values
message Value {
    oneof sum {
        Record record = 14;       // Records / template data
        Variant variant = 15;     // Sum types
        Enum enum = 16;           // Enum types
        string contract_id = 9;   // ContractId
        List list = 11;           // [a]
        sint64 int64 = 3;         // Int
        string numeric = 6;       // Numeric (decimal string)
        string text = 8;          // Text
        sfixed64 timestamp = 5;   // Time (microseconds since epoch)
        string party = 7;         // Party
        bool bool = 2;            // Bool
        Empty unit = 1;           // ()
        int32 date = 4;           // Date (days since epoch)
        Optional optional = 10;   // Optional a
        Map text_map = 12;        // TextMap a
        GenMap gen_map = 13;      // GenMap k v
    }
}

// Template/type identification
message Identifier {
    string package_id = 1;   // Package hash or name
    string module_name = 2;
    string entity_name = 3;
}

// Command submission
message Commands {
    string command_id = 3;
    repeated Command commands = 4;
    repeated string act_as = 9;
    repeated string read_as = 10;
    repeated DisclosedContract disclosed_contracts = 14;
}

message Command {
    oneof command {
        CreateCommand create = 1;
        ExerciseCommand exercise = 2;
        ExerciseByKeyCommand exercise_by_key = 4;
        CreateAndExerciseCommand create_and_exercise = 3;
    }
}
```

### Implementation Steps

1. Create a new .NET class library project (e.g., `Canton.Ledger.Api`)
2. Add NuGet references: `Grpc.Net.Client`, `Google.Protobuf`, `Grpc.Tools`
3. Copy or reference the Ledger API v2 `.proto` files
4. Configure `.csproj` to compile protos:

```xml
<ItemGroup>
    <Protobuf Include="protos/**/*.proto"
              GrpcServices="Client"
              ProtoRoot="protos" />
</ItemGroup>
```

5. Build — this produces all C# service stubs and message classes

### Output
A NuGet-publishable package containing all generated gRPC stubs for the Ledger API v2.

### Effort
Low — this is mechanical. The `Grpc.Tools` package handles everything.

---

## Phase 2: .NET Ledger Client Wrapper

### Goal
A thin, ergonomic C# client that wraps the raw gRPC stubs with authentication, command construction, and result parsing. Modelled after the Java `LedgerApi.java` in the quickstart (~200 lines).

### Architecture

The quickstart Java backend follows this pattern:
- **Reads** go through PQS (PostgreSQL via JDBC) — the .NET client does not need to replicate this; it's application-level
- **Writes** go through the Ledger API via gRPC — this is what the client wraps
- **Auth** is a bearer token passed via gRPC metadata interceptor

```
Application Code
    ↓ (type-safe C# calls)
CantonLedgerClient
    ↓ (constructs Commands protobuf, adds auth header)
gRPC Channel → CommandService / StateService / UpdateService
    ↓ (gRPC over HTTP/2)
Canton Participant Node
```

### Core Interface

```csharp
public class CantonLedgerClient : IDisposable
{
    // Constructor: channel setup + auth interceptor
    public CantonLedgerClient(string host, int port, Func<string> tokenProvider, string applicationId);

    // Create a contract
    Task<ContractId<T>> CreateAsync<T>(
        T template,
        string commandId,
        IReadOnlyList<string> actAs,
        IReadOnlyList<string>? readAs = null) where T : ITemplate;

    // Exercise a choice on a contract
    Task<TResult> ExerciseAsync<TTemplate, TChoice, TResult>(
        ContractId<TTemplate> contractId,
        TChoice choice,
        string commandId,
        IReadOnlyList<string> actAs,
        IReadOnlyList<string>? readAs = null,
        IReadOnlyList<DisclosedContract>? disclosedContracts = null)
        where TChoice : IChoice<TTemplate, TResult>;

    // Get active contracts snapshot
    IAsyncEnumerable<ActiveContract<T>> GetActiveContractsAsync<T>(
        IReadOnlyList<string> parties) where T : ITemplate;

    // Stream ledger updates
    IAsyncEnumerable<Transaction> GetUpdatesAsync(
        string beginOffset,
        IReadOnlyList<string> parties);

    // Archive a contract (convenience)
    Task ArchiveAsync<T>(ContractId<T> contractId, string commandId, IReadOnlyList<string> actAs);
}
```

### Key Types

```csharp
// Strongly-typed contract ID
public readonly record struct ContractId<T>(string Value);

// Party wrapper
public readonly record struct Party(string Value);

// Marker interfaces (implemented by generated code)
public interface ITemplate
{
    Identifier TemplateId { get; }
    Record ToValue();
    static abstract ITemplate FromValue(Value value);
}

public interface IChoice<TTemplate, TResult>
{
    string ChoiceName { get; }
    Identifier TemplateId { get; }
    Value ToValue();
}

// Template identifier
public record Identifier(string PackageId, string ModuleName, string EntityName);
```

### Auth Interceptor

Mirrors the Java pattern — injects bearer token into gRPC metadata:

```csharp
public class AuthInterceptor : Interceptor
{
    private readonly Func<string> _tokenProvider;

    public override AsyncUnaryCall<TResponse> AsyncUnaryCall<TRequest, TResponse>(
        TRequest request, ClientInterceptorContext<TRequest, TResponse> context,
        AsyncUnaryCallContinuation<TRequest, TResponse> continuation)
    {
        var headers = context.Options.Headers ?? new Metadata();
        headers.Add("Authorization", $"Bearer {_tokenProvider()}");
        // ... forward call with headers
    }
}
```

### Command Construction

The client builds `Commands` protobuf messages internally:

```csharp
// For ExerciseAsync:
var command = new Command
{
    Exercise = new ExerciseCommand
    {
        TemplateId = ToProtoIdentifier(choice.TemplateId),
        ContractId = contractId.Value,
        Choice = choice.ChoiceName,
        ChoiceArgument = choice.ToValue()
    }
};

var request = new SubmitAndWaitForTransactionRequest
{
    Commands = new Commands
    {
        CommandId = commandId,
        ApplicationId = _applicationId,
        Commands_ = { command },
        ActAs = { actAs },
        ReadAs = { readAs ?? actAs },
        DisclosedContracts = { disclosedContracts ?? [] }
    }
};
```

### Implementation Steps

1. Create a new .NET class library project (e.g., `Canton.Ledger.Client`)
2. Reference the Phase 1 gRPC stubs package
3. Implement `CantonLedgerClient` with:
   - Channel management (plaintext for dev, TLS for prod)
   - `AuthInterceptor` for bearer token injection
   - `CreateAsync` — builds `CreateCommand`, submits, extracts `ContractId` from result
   - `ExerciseAsync` — builds `ExerciseCommand`, submits, deserializes `exercise_result` via `TResult.FromValue()`
   - `GetActiveContractsAsync` — wraps `StateService.GetActiveContracts` streaming RPC
   - `GetUpdatesAsync` — wraps `UpdateService.GetUpdates` streaming RPC
4. Add error handling (gRPC status codes → typed exceptions)
5. Add OpenTelemetry tracing support (optional, mirrors Java's `@WithSpan`)

### Output
A NuGet-publishable client library. Usable immediately with manually constructed `Value`/`Record` types, or with generated bindings from Phase 3.

### Effort
Moderate — ~200-400 lines of core logic, plus error handling and streaming support.

---

## Phase 3: DAR → C# Code Generator (Standalone CLI Tool)

### Goal
A `dotnet tool` that reads compiled Daml packages (`.dar` files) and generates type-safe C# classes for each template, choice, record, variant, and enum.

### Why a CLI Tool (Not a C# Source Generator)

| Concern | Source Generator | CLI Tool |
|---------|-----------------|----------|
| **Input format** | Designed for C# syntax trees, not binary artifacts | No constraints on input format |
| **Dependencies** | Runs inside compiler process — fragile dependency conflicts with `Google.Protobuf` etc. | Standard .NET app — use any NuGet packages |
| **Debugging** | Must attach to Roslyn compiler process | Standard `dotnet run` / debugger attach |
| **Output visibility** | Ephemeral in-memory files — IDE support varies | Real `.cs` files on disk — full IDE support, readable diffs |
| **Build caching** | Roslyn may re-run on every keystroke | MSBuild `Inputs`/`Outputs` — skips when DAR unchanged |
| **Complexity ceiling** | Hard to debug complex Daml-LF type mapping | No constraints — use templates, multi-pass, etc. |
| **Precedent** | No Daml codegen uses source generators | All existing Daml codegens (Java, JS, Go, Rust) are CLI tools |

### DAR File Structure

A `.dar` file is a ZIP archive:

```
my-package-1.0.0.dar (ZIP)
├── META-INF/MANIFEST.MF                    # Lists main + dependency DALFs
├── mypkg-1.0.0-<package-hash>/
│   └── mypkg-1.0.0-<package-hash>.dalf     # Primary package (protobuf)
├── dependency-<hash>/
│   └── dependency-<hash>.dalf              # Dependency DALFs
└── *.daml                                   # Original source (optional)
```

Each `.dalf` is a serialized `DamlLf.Archive` protobuf message.

### Daml-LF AST Structure (Protobuf)

The Daml-LF proto schema defines:

```
DamlLf.Archive
  ├── hash (package ID / SHA-256)
  ├── hash_function
  └── payload → ArchivePayload
        ├── minor (LF version)
        └── Package
              ├── metadata (name, version)
              └── modules[] → Module
                    ├── name (DottedName)
                    ├── data_types[] → DefDataType
                    │     ├── name
                    │     ├── serializable (bool)
                    │     └── DataCons: Record | Variant | Enum
                    ├── templates[] → DefTemplate
                    │     ├── tycon (template name)
                    │     ├── param (type of template data)
                    │     ├── precond (ensure clause)
                    │     ├── signatories
                    │     ├── observers
                    │     ├── choices[] → TemplateChoice
                    │     │     ├── name
                    │     │     ├── consuming (bool)
                    │     │     ├── arg_binder (parameter type)
                    │     │     ├── ret_type (return type)
                    │     │     ├── controllers
                    │     │     └── update (choice body — not needed for codegen)
                    │     └── key (optional contract key definition)
                    └── values[] → DefValue (functions — not needed for codegen)
```

### Processing Pipeline

```
1. Open .dar (ZIP)
2. Read META-INF/MANIFEST.MF → identify main DALF
3. Parse main .dalf → DamlLf.Archive (protobuf deserialize)
4. Extract Package → iterate modules
5. For each Module:
   a. For each DefDataType → emit C# record, variant, or enum
   b. For each DefTemplate → emit C# template class with nested choice classes
6. Emit Registry.cs (identifier → type mapping, like Java's Daml.ENTITIES)
```

### Type Mapping

| Daml Type | C# Type | Protobuf Wire (`Value.Sum`) |
|-----------|---------|-----------------------------|
| `Party` | `Party` (readonly record struct wrapping string) | `party` (string) |
| `Text` | `string` | `text` (string) |
| `Int` | `long` | `int64` (sint64) |
| `Numeric` / `Decimal` | `decimal` | `numeric` (string, e.g. "17.42") |
| `Bool` | `bool` | `bool` |
| `Time` | `DateTimeOffset` | `timestamp` (sfixed64, microseconds since epoch) |
| `Date` | `DateOnly` | `date` (int32, days since epoch) |
| `RelTime` | `TimeSpan` | `int64` (microseconds) |
| `ContractId a` | `ContractId<T>` | `contract_id` (string) |
| `Optional a` | `T?` (nullable) | `optional` (wrapped Value or empty) |
| `[a]` | `IReadOnlyList<T>` | `list` (repeated Value) |
| `TextMap a` | `IReadOnlyDictionary<string, T>` | `text_map` (repeated key-value) |
| `GenMap k v` | `IReadOnlyDictionary<TKey, TValue>` | `gen_map` (repeated key-value pairs) |
| Record | C# class with `init` properties | `record` (Identifier + RecordField[]) |
| Variant | Abstract base + subclass per constructor | `variant` (Identifier + constructor + Value) |
| Enum | C# `enum` | `enum` (Identifier + constructor) |
| `()` (Unit) | `Unit` (singleton) | `unit` (Empty) |

### Generated Code Structure

**Per template (e.g., `License`):**

```csharp
namespace MyApp.Daml.Licensing;

public sealed class License : ITemplate
{
    public static readonly Identifier TemplateId = new(
        "be51e4c405300fb65cddcf53e84486e39192615bf71676fafb8f098cc11b9725",
        "Licensing.License",
        "License");

    public Identifier ITemplate.TemplateId => TemplateId;

    // Template fields (from DefTemplate → param record)
    public required Party Provider { get; init; }
    public required Party User { get; init; }
    public required DateTimeOffset ExpiresAt { get; init; }
    public required long LicenseNum { get; init; }
    public required LicenseParams Params { get; init; }

    // Protobuf serialization
    public Record ToValue() { /* field-by-field Record construction */ }
    public static License FromValue(Value value) { /* field-by-field extraction */ }

    // Nested choice: License_Renew (nonconsuming)
    public sealed class License_Renew : IChoice<License, ContractId<LicenseRenewalRequest>>
    {
        public string ChoiceName => "License_Renew";
        public Identifier TemplateId => License.TemplateId;
        public bool IsConsuming => false;

        public required string RequestId { get; init; }
        public required InstrumentId LicenseFeeInstrumentId { get; init; }
        public required decimal LicenseFeeAmount { get; init; }
        public required TimeSpan LicenseExtensionDuration { get; init; }
        public required DateTimeOffset RequestedAt { get; init; }
        public required DateTimeOffset PrepareUntil { get; init; }
        public required DateTimeOffset SettleBefore { get; init; }
        public required string Description { get; init; }

        public Value ToValue() { /* ... */ }
        public static ContractId<LicenseRenewalRequest> ResultFromValue(Value value) { /* ... */ }
    }

    // Nested choice: License_Expire (consuming, default)
    public sealed class License_Expire : IChoice<License, Unit>
    {
        public string ChoiceName => "License_Expire";
        public Identifier TemplateId => License.TemplateId;
        public bool IsConsuming => true;

        public required Party Actor { get; init; }
        public required Metadata Meta { get; init; }

        public Value ToValue() { /* ... */ }
        public static Unit ResultFromValue(Value value) => Unit.Value;
    }
}
```

**Per record type (e.g., `LicenseParams`):**

```csharp
public sealed class LicenseParams
{
    public required Metadata Meta { get; init; }

    public Record ToValue() { /* ... */ }
    public static LicenseParams FromValue(Value value) { /* ... */ }
}
```

**Per variant type:**

```csharp
public abstract class MyVariant
{
    public abstract Value ToValue();
    public static MyVariant FromValue(Value value) { /* switch on constructor */ }

    public sealed class ConstructorA : MyVariant { /* fields, ToValue */ }
    public sealed class ConstructorB : MyVariant { /* fields, ToValue */ }
}
```

**Registry (like Java's `Daml.ENTITIES`):**

```csharp
public static class DamlRegistry
{
    public static readonly IReadOnlyDictionary<Identifier, TemplateInfo> Templates = new Dictionary<Identifier, TemplateInfo>
    {
        [License.TemplateId] = new(
            typeof(License),
            License.FromValue,
            new Dictionary<string, ChoiceInfo>
            {
                ["License_Renew"] = new(typeof(License.License_Renew), isConsuming: false),
                ["License_Expire"] = new(typeof(License.License_Expire), isConsuming: true),
                ["Archive"] = new(typeof(Archive), isConsuming: true),
            }),
        // ... other templates
    };
}
```

### CLI Tool Invocation

```bash
# Install as global tool
dotnet tool install --global Canton.Daml.Codegen

# Generate C# from a DAR
dotnet daml-codegen \
    --dar ./my-package-1.0.0.dar \
    --output ./Generated/ \
    --namespace MyApp.Daml
```

### MSBuild Integration

Wire into the consuming project's `.csproj` so codegen runs automatically on build when the DAR changes:

```xml
<Target Name="DamlCodegen" BeforeTargets="BeforeCompile"
        Inputs="../daml/.daml/dist/my-package-1.0.0.dar"
        Outputs="Generated/.daml-codegen-stamp">
    <Exec Command="dotnet daml-codegen --dar %(Inputs.Identity) --output Generated/ --namespace MyApp.Daml" />
    <Touch Files="Generated/.daml-codegen-stamp" AlwaysCreate="true" />
</Target>

<ItemGroup>
    <Compile Include="Generated/**/*.cs" />
</ItemGroup>
```

MSBuild compares `Inputs` (DAR timestamp) against `Outputs` (stamp file) and **skips the target entirely** when the DAR hasn't changed.

### Implementation Steps

1. Create a .NET console app (e.g., `Canton.Daml.Codegen`)
2. Generate C# stubs for the Daml-LF protobuf schema (`daml_lf.proto`) — this allows parsing `.dalf` files
3. Implement DAR reader: open ZIP → read MANIFEST → extract main `.dalf`
4. Implement DALF parser: deserialize `DamlLf.Archive` → walk `Package.Modules`
5. Implement type resolver: map Daml-LF types to C# types (handle records, variants, enums, parameterized types)
6. Implement C# emitter: for each `DefDataType` and `DefTemplate`, emit `.cs` files with `ToValue`/`FromValue` methods
7. Emit registry class
8. Package as `dotnet tool` for distribution

### Effort
This is the most substantial phase. Key complexity areas:
- **Parameterized types**: Daml supports `data Pair a b = Pair a b` — requires generic C# classes with serialization delegates
- **Variant types**: Must generate abstract base + subclasses + `FromValue` dispatch
- **Cross-package references**: Templates can reference types from dependency DARs — need to resolve identifiers across packages
- **Daml-LF version differences**: LF v2 vs v3 have schema differences — the parser needs to handle both

---

## End-to-End Usage Example

After all three phases, a developer would use the bindings like this:

```csharp
// Setup
var client = new CantonLedgerClient(
    host: "localhost",
    port: 6865,
    tokenProvider: () => myAuthToken,
    applicationId: "my-app");

// Create a contract
var installRequest = new AppInstallRequest
{
    Provider = new Party("AppProvider::1220abc..."),
    User = new Party("AppUser::1220def..."),
    Meta = new Metadata(new Dictionary<string, string>())
};

var requestId = await client.CreateAsync(
    installRequest,
    commandId: Guid.NewGuid().ToString(),
    actAs: new[] { "AppUser::1220def..." });

// Exercise a choice
var result = await client.ExerciseAsync(
    requestId,
    new AppInstallRequest.AppInstallRequest_Accept
    {
        InstallMeta = new Metadata(new Dictionary<string, string>()),
        Meta = new Metadata(new Dictionary<string, string>())
    },
    commandId: Guid.NewGuid().ToString(),
    actAs: new[] { "AppProvider::1220abc..." });

// result is ContractId<AppInstall>

// Query active contracts
await foreach (var contract in client.GetActiveContractsAsync<License>(
    parties: new[] { "AppProvider::1220abc..." }))
{
    Console.WriteLine($"License #{contract.Data.LicenseNum} expires {contract.Data.ExpiresAt}");
}
```

---

## Summary

| Phase | Deliverable | Effort | Depends On |
|-------|------------|--------|------------|
| 1 | `Canton.Ledger.Api` — gRPC stubs NuGet package | Low | Proto files from Canton SDK |
| 2 | `Canton.Ledger.Client` — Ergonomic .NET client | Moderate | Phase 1 |
| 3 | `Canton.Daml.Codegen` — CLI tool for DAR → C# | High | Phase 1 (for proto value types), Daml-LF proto schema |

Phases 1+2 alone give a working .NET client (manually constructing `Value`/`Record` protobuf types). Phase 3 makes it type-safe and ergonomic.

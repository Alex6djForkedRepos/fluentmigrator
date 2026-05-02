# Custom Extensions

You can create custom extension methods to encapsulate common patterns and reduce code duplication in your migrations. This approach helps maintain consistency across your migration scripts and makes complex operations reusable.

## Simple Extension Methods

The simplest form of custom extension wraps existing fluent API calls into reusable patterns:

```csharp
public static class MigrationExtensions
{
    public static ICreateTableColumnOptionOrWithColumnSyntax WithIdColumn(
        this ICreateTableWithColumnSyntax tableWithColumnSyntax)
    {
        return tableWithColumnSyntax
            .WithColumn("Id")
            .AsInt32()
            .NotNullable()
            .PrimaryKey()
            .Identity();
    }

    public static ICreateTableColumnOptionOrWithColumnSyntax WithTimeStamps(
        this ICreateTableColumnOptionOrWithColumnSyntax tableWithColumnSyntax)
    {
        return tableWithColumnSyntax
            .WithColumn("CreatedAt")
                .AsDateTime()
                .NotNullable()
                .WithDefaultValue(SystemMethods.CurrentDateTime)
            .WithColumn("UpdatedAt")
                .AsDateTime()
                .Nullable();
    }

    public static ICreateTableColumnOptionOrWithColumnSyntax WithAuditFields(
        this ICreateTableColumnOptionOrWithColumnSyntax tableWithColumnSyntax)
    {
        return tableWithColumnSyntax
            .WithTimeStamps()
            .WithColumn("CreatedBy")
                .AsString(100)
                .Nullable()
            .WithColumn("UpdatedBy")
                .AsString(100)
                .Nullable();
    }
}
```

**Usage:**
```csharp
Create.Table("Products")
    .WithIdColumn()
    .WithColumn("Name").AsString(200).NotNullable()
    .WithColumn("Price").AsDecimal(10, 2).NotNullable()
    .WithAuditFields();
```

## Extending the Generator Pipeline with `ISupportAdditionalFeatures`

When you need to expose a database-specific option through the fluent API that the built-in FluentMigrator expressions don't support, use the `ISupportAdditionalFeatures` pattern. This lets you:

- Add new options to existing builder expressions (e.g. `CreateIndexExpression`, `CreateSchemaExpression`)
- Read those options in a custom generator subclass to produce database-specific SQL
- Test each layer independently

Several built-in expression and model types already implement `ISupportAdditionalFeatures`:
- `CreateSchemaExpression`, `CreateIndexExpression`, `DeleteIndexExpression`
- `CreateConstraintExpression`, `DeleteConstraintExpression`, `InsertDataExpression`
- `IndexDefinition`, `IndexColumnDefinition`, `ColumnDefinition`, `ConstraintDefinition`

The pattern is used throughout FluentMigrator itself — for example, `SqlServerExtensions.Authorization` sets a schema owner, and `PostgresExtensions.UsingBTree` sets an index algorithm — both use this same mechanism.

### The Three-Part Pattern

The `ISupportAdditionalFeatures` pattern consists of three components that work together:

1. **Feature keys** — string constants that identify your feature in the `AdditionalFeatures` dictionary
2. **Extension methods** — fluent API methods that store feature values using `SetAdditionalFeature`
3. **Custom generator** — a subclass that reads values using `GetAdditionalFeature` and generates SQL

### Worked Example: Schema AUTHORIZATION

This example mirrors how `SqlServerExtensions` adds `AUTHORIZATION owner` to a `CREATE SCHEMA` statement.

#### Step 1: Define Feature Keys

Store feature key constants in a dedicated class. Use a unique prefix to avoid collisions with other extensions:

```csharp
public static class MyDatabaseExtensions
{
    /// <summary>Key for the schema owner option.</summary>
    public const string SchemaOwner = "MyDatabase:SchemaOwner";

    private static string UnsupportedMethodMessage(string methodName)
        => $"The '{methodName}' method is not supported. The expression must implement ISupportAdditionalFeatures.";
}
```

#### Step 2: Write Extension Methods

Extension methods cast the builder to `ISupportAdditionalFeatures` and store the value. Always throw a descriptive `InvalidOperationException` when the cast fails so the developer gets a clear error message:

```csharp
using FluentMigrator.Builders.Create.Schema;
using FluentMigrator.Infrastructure;
using FluentMigrator.Infrastructure.Extensions;

public static partial class MyDatabaseExtensions
{
    /// <summary>
    /// Sets the schema owner during CREATE SCHEMA.
    /// </summary>
    public static ICreateSchemaOptionsSyntax OwnedBy(
        this ICreateSchemaOptionsSyntax expression,
        string ownerName)
    {
        var additionalFeatures = expression as ISupportAdditionalFeatures
            ?? throw new InvalidOperationException(
                UnsupportedMethodMessage(nameof(OwnedBy)));

        additionalFeatures.SetAdditionalFeature(SchemaOwner, ownerName);
        return expression;
    }
}
```

> **Tip:** `SetAdditionalFeature` and `GetAdditionalFeature` are extension methods on
> `ISupportAdditionalFeatures` from the `FluentMigrator.Infrastructure.Extensions` namespace.

#### Step 3: Extend the Generator

Subclass the generator for your target database and override the `Generate` overload that handles the relevant expression type. Read the additional feature value and incorporate it into the SQL:

```csharp
using FluentMigrator.Expressions;
using FluentMigrator.Infrastructure.Extensions;
using FluentMigrator.Runner.Generators.Postgres;
using Microsoft.Extensions.Options;

public class MyCustomPostgresGenerator : Postgres15_0Generator
{
    public MyCustomPostgresGenerator(
        PostgresQuoter quoter,
        IOptions<GeneratorOptions> generatorOptions,
        IPostgresTypeMap typeMap)
        : base(quoter, generatorOptions, typeMap)
    {
    }

    public override string Generate(CreateSchemaExpression expression)
    {
        // Build the base SQL (e.g. "CREATE SCHEMA "myschema";")
        var sql = base.Generate(expression);

        // Read the additional feature
        if (expression.TryGetAdditionalFeature<string>(
                MyDatabaseExtensions.SchemaOwner, out var owner)
            && !string.IsNullOrEmpty(owner))
        {
            // Append the OWNER clause before the trailing semicolon
            sql = sql.TrimEnd(';').TrimEnd() + $" AUTHORIZATION {Quoter.QuoteSchemaName(owner)};";
        }

        return sql;
    }

    // Declare which additional feature keys this generator supports
    // so that strict-mode compatibility checks work correctly.
    public override bool IsAdditionalFeatureSupported(string feature)
        => feature == MyDatabaseExtensions.SchemaOwner
        || base.IsAdditionalFeatureSupported(feature);
}
```

> **Why override `IsAdditionalFeatureSupported`?**
> In `CompatibilityMode.STRICT`, FluentMigrator validates that every additional feature on an
> expression is understood by the active generator. Returning `true` for your feature key prevents
> a "feature not supported" error when running migrations in strict mode.

#### Step 4: Register the Custom Generator with Dependency Injection

After calling `AddPostgres()`, replace the default generator registration so that the processor picks up your subclass:

```csharp
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.DependencyInjection.Extensions;

services.AddFluentMigratorCore()
    .ConfigureRunner(rb => rb
        .AddPostgres()
        .WithGlobalConnectionString("Host=localhost;Database=myapp;...")
        .ScanIn(typeof(MyMigration).Assembly).For.Migrations())
    // Replace the generator so the processor uses our custom version
    .ConfigureRunner(rb =>
    {
        rb.Services.Replace(ServiceDescriptor.Scoped<Postgres15_0Generator>(sp =>
            new MyCustomPostgresGenerator(
                sp.GetRequiredService<PostgresQuoter>(),
                sp.GetRequiredService<IOptions<GeneratorOptions>>(),
                sp.GetRequiredService<IPostgresTypeMap>())));
    });
```

#### Step 5: Use the Extension in Migrations

```csharp
[Migration(20240101120000)]
public class CreateReportingSchema : Migration
{
    public override void Up()
    {
        Create.Schema("reporting")
            .OwnedBy("reporting_role");   // custom extension
    }

    public override void Down()
    {
        Delete.Schema("reporting");
    }
}
```

---

### Worked Example: Index Storage Parameter (complex value)

For features that require more than a simple scalar value, store a model object in `AdditionalFeatures` instead of a plain string.

#### Define the Model

```csharp
public class IndexStorageOptions
{
    public string Method { get; set; } = "default";
    public int FillFactor { get; set; } = 90;
}
```

#### Define the Feature Key and Extension Method

```csharp
public static class MyDatabaseExtensions
{
    public const string IndexStorage = "MyDatabase:IndexStorage";

    public static ICreateIndexOptionsSyntax WithStorage(
        this ICreateIndexOptionsSyntax expression,
        string method,
        int fillFactor = 90)
    {
        var additionalFeatures = expression as ISupportAdditionalFeatures
            ?? throw new InvalidOperationException(
                UnsupportedMethodMessage(nameof(WithStorage)));

        additionalFeatures.SetAdditionalFeature(
            IndexStorage,
            new IndexStorageOptions { Method = method, FillFactor = fillFactor });

        return expression;
    }
}
```

#### Read the Model in the Generator

```csharp
public override string Generate(CreateIndexExpression expression)
{
    var sql = base.Generate(expression);

    var storage = expression.GetAdditionalFeature<IndexStorageOptions>(
        MyDatabaseExtensions.IndexStorage);

    if (storage is not null)
    {
        sql = sql.TrimEnd(';').TrimEnd()
            + $" WITH (STORAGE_METHOD = '{storage.Method}', FILLFACTOR = {storage.FillFactor});";
    }

    return sql;
}
```

---

## Writing Unit Tests

Testing custom extensions should cover two independent layers:

1. **Extension method tests** — verify that calling the extension method correctly stores the value in `AdditionalFeatures`
2. **Generator tests** — verify that the generator reads the stored value and produces the expected SQL

### Testing Extension Methods

Create the expression and builder directly without any DI or runner infrastructure:

```csharp
using FluentMigrator.Builders.Create.Schema;
using FluentMigrator.Expressions;
using FluentMigrator.Infrastructure.Extensions;
using NUnit.Framework;
using Shouldly;

[TestFixture]
public class MyDatabaseExtensionsTests
{
    [Test]
    public void OwnedBy_SetsSchemaOwnerAdditionalFeature()
    {
        var expression = new CreateSchemaExpression { SchemaName = "reporting" };
        var builder = new CreateSchemaExpressionBuilder(expression);

        // Call the extension method
        builder.OwnedBy("reporting_role");

        // Verify the additional feature was stored
        expression.TryGetAdditionalFeature<string>(
            MyDatabaseExtensions.SchemaOwner, out var owner).ShouldBeTrue();
        owner.ShouldBe("reporting_role");
    }

    [Test]
    public void OwnedBy_ThrowsWhenExpressionDoesNotImplementISupportAdditionalFeatures()
    {
        // Use a mock/stub that does NOT implement ISupportAdditionalFeatures
        var unsupported = new UnsupportedCreateSchemaSyntax();

        Should.Throw<InvalidOperationException>(() => unsupported.OwnedBy("owner"));
    }

    // Minimal stub that implements the interface but NOT ISupportAdditionalFeatures
    private class UnsupportedCreateSchemaSyntax : ICreateSchemaOptionsSyntax { }
}
```

### Testing the Generator

Instantiate the generator directly and build the expression by hand:

```csharp
using FluentMigrator.Expressions;
using FluentMigrator.Infrastructure.Extensions;
using FluentMigrator.Runner.Generators.Postgres;
using Microsoft.Extensions.Options;
using NUnit.Framework;
using Shouldly;

[TestFixture]
public class MyCustomPostgresGeneratorTests
{
    private MyCustomPostgresGenerator _generator;

    [SetUp]
    public void SetUp()
    {
        _generator = new MyCustomPostgresGenerator(
            new PostgresQuoter(),
            new OptionsWrapper<GeneratorOptions>(new GeneratorOptions()),
            new PostgresTypeMap());
    }

    [Test]
    public void Generate_CreateSchema_WithOwner_IncludesAuthorizationClause()
    {
        var expression = new CreateSchemaExpression { SchemaName = "reporting" };
        expression.SetAdditionalFeature(MyDatabaseExtensions.SchemaOwner, "reporting_role");

        var sql = _generator.Generate(expression);

        sql.ShouldBe("CREATE SCHEMA \"reporting\" AUTHORIZATION \"reporting_role\";");
    }

    [Test]
    public void Generate_CreateSchema_WithoutOwner_OmitsAuthorizationClause()
    {
        var expression = new CreateSchemaExpression { SchemaName = "reporting" };

        var sql = _generator.Generate(expression);

        sql.ShouldBe("CREATE SCHEMA \"reporting\";");
    }

    [Test]
    public void IsAdditionalFeatureSupported_ReturnsTrueForSchemaOwner()
    {
        _generator.IsAdditionalFeatureSupported(MyDatabaseExtensions.SchemaOwner).ShouldBeTrue();
    }
}
```

### Testing the Full Migration Pipeline

To test a complete migration end-to-end — including that the builder correctly populates the expression — use a mock `IMigrationContext`:

```csharp
using System.Collections.Generic;
using FluentMigrator.Expressions;
using FluentMigrator.Infrastructure;
using Microsoft.Extensions.DependencyInjection;
using Moq;
using NUnit.Framework;
using Shouldly;

[TestFixture]
public class CreateReportingSchemaTests
{
    [Test]
    public void Up_AddsCreateSchemaExpressionWithOwner()
    {
        var expressions = new List<IMigrationExpression>();

        var contextMock = new Mock<IMigrationContext>();
        contextMock.SetupGet(x => x.Expressions).Returns(expressions);
        contextMock.SetupGet(x => x.ServiceProvider)
            .Returns(new ServiceCollection().BuildServiceProvider());

        var migration = new CreateReportingSchema();
        migration.GetUpExpressions(contextMock.Object);

        var schemaExpr = expressions
            .OfType<CreateSchemaExpression>()
            .ShouldHaveSingleItem();

        schemaExpr.SchemaName.ShouldBe("reporting");
        schemaExpr.TryGetAdditionalFeature<string>(
            MyDatabaseExtensions.SchemaOwner, out var owner).ShouldBeTrue();
        owner.ShouldBe("reporting_role");
    }
}
```

> **Tip:** To keep migration tests focused, separate the concern of "does the migration build the
> right expression?" (tested above) from "does the generator produce the right SQL?" (tested in
> the generator test above). This makes failures much easier to diagnose.

---

## Best Practices

### Feature Key Naming

- **Use a unique namespace prefix** to avoid collisions with other extension packages:
  `"MyCompany:FeatureName"` rather than a bare `"FeatureName"`
- **Store keys as `public const string`** so generator code and extension code reference
  the same constant
- **Use `readonly` fields** (`public static readonly string`) when the feature key is
  dynamically composed

### Extension Method Design

- **Always guard the cast to `ISupportAdditionalFeatures`** with a descriptive `InvalidOperationException`
- **Return the original interface type** to preserve fluent method chaining
- **Prefer `SetAdditionalFeature<T>` over direct dictionary access** — it is type-safe and
  more readable
- **Use a model object** (a small class or record) when the feature requires more than one
  value; avoid packing multiple values into a single string

### Generator Implementation

- **Override `IsAdditionalFeatureSupported`** and return `true` for your feature keys, then
  fall through to `base.IsAdditionalFeatureSupported(feature)` for all others
- **Use `TryGetAdditionalFeature` when the feature is optional** so the generator remains
  backwards-compatible with expressions that do not set the feature
- **Use `GetAdditionalFeature` with a default value** when the feature has a sensible default

### Testing

- **Test the extension method and generator separately** — each layer can be exercised
  without the other
- **Use `CreateSchemaExpressionBuilder`, `CreateIndexExpressionBuilder`, etc. directly** in
  tests rather than running the full migration runner
- **Test with and without the additional feature** to ensure the generator works correctly
  in both cases
- **Test `IsAdditionalFeatureSupported`** to verify strict-mode compatibility

---

## Example: Complete Extension Library

```csharp
namespace YourProject.Migrations.Extensions
{
    public static class StandardExtensions
    {
        // Standard ID column
        public static ICreateTableColumnOptionOrWithColumnSyntax WithIdColumn(
            this ICreateTableWithColumnSyntax syntax) =>
            syntax.WithColumn("Id").AsInt32().NotNullable().PrimaryKey().Identity();

        // Audit fields
        public static ICreateTableColumnOptionOrWithColumnSyntax WithAuditFields(
            this ICreateTableColumnOptionOrWithColumnSyntax syntax) =>
            syntax
                .WithColumn("CreatedAt").AsDateTime().NotNullable()
                    .WithDefaultValue(SystemMethods.CurrentDateTime)
                .WithColumn("CreatedBy").AsString(100).Nullable()
                .WithColumn("UpdatedAt").AsDateTime().Nullable()
                .WithColumn("UpdatedBy").AsString(100).Nullable();

        // Soft delete
        public static ICreateTableColumnOptionOrWithColumnSyntax WithSoftDelete(
            this ICreateTableColumnOptionOrWithColumnSyntax syntax) =>
            syntax
                .WithColumn("IsDeleted").AsBoolean().NotNullable().WithDefaultValue(false)
                .WithColumn("DeletedAt").AsDateTime().Nullable()
                .WithColumn("DeletedBy").AsString(100).Nullable();
    }
}
```

Custom extensions are a powerful way to make your migrations more maintainable and reduce duplication while ensuring consistency across your database schema evolution.
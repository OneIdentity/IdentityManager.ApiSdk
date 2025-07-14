# Entity methods

This document provides an overview of the definition of entity methods, which is a type of API method that integrates directly with the Identity Manager object model.

## A simple entity method for reading data

This example shows how to build a simple entity method that integrates with object layer to load data from a specific database table.

```csharp
Method
    // Set the method URL.
    .Define("person/allcolumns")
    // Set the database table
    .FromTable("Person")
    .EnableRead()
    // This will include all enabled columns in the result
    .WithAllColumns()
```

You can specify the columns that should always be included in the result.

```csharp
Method.Define("person/specificcolumns")
    .FromTable("Person")
    .EnableRead()
    // Only include specific columns in the result.
    .WithResultColumns("FirstName", "LastName")

```

## CRUD (Create, read, update, delete) models

You can define any combination of the four CRUD operations on an entity methods using the `EnableCreate`, `EnableRead`, `EnableUpdate` and `EnableDelete` methods.

The base filter condition will also be applied when updating and deleting entities (but not on creating new ones). This example shows a method that can only read or delete entities that match the filter condition.

```csharp
Method.Define("deletiontest")
    .FromTable("DialogTag")
    .EnableRead()
    .WithWhereClause("Ident_DialogTag = 'DELETION TEST'")
    .EnableDelete()
```

Passing a filter clauses as a string is not the only option. Clauses can also be referenced from a predefined SQL statement:

```csharp
Method.Define("example")
   .FromTable("Person")
   .EnableRead()
   .WithClause(new LimitedSqlWhereClause("<id of the LimitedSQL object>"));
```

This call will allows the client to write to all columns. Note that object permission constraints still apply and cannot be overridden.

```csharp
.EnableUpdate()
.WithWritableAllColumns()
```

You can explicitly specify writable columns.

```csharp
.WithWritableColumns("FirstName", "LastName", "UID_PersonHead"));
```

The following code defines a rule that determines when a column should be writable. In this case, the `PhoneNumber` is writable only if the `IsExternal` flag is set. (Note that object permission constraints still apply and cannot be overridden.)

```csharp
.WithWritableColumns("Phone", e => e.GetValue<bool>("IsExternal"))
```

## Interactive vs. delayed logic

By default, entity methods create *delayed-logic* entities. For many scenarios where users interact with an entity, and where the effective set of permissions is important, it is recommended to use *interactive* entities instead.

```csharp
.As(EntityType.Interactive)
```

## The data model API

The API Server creates an additional API for entity methods called the _data model API_. This can be enabled by setting `EnableDataModelApi`:

```csharp
builder.AddMethod(Method.Define("policies/violations")
    .FromTable("QERPolicyHasObject")
    .WithDescription("Returns a list of policy violations.")
    .EnableRead()
    .With(m =>
    {
        m.EnableDataModelApi = true;
    }
);
```

This will cause the server to provide an endpoint with the `/datamodel` suffix (`policies/violations/datamodel`) which provides information about:

- the list of available properties of this entity API,
- the list of available filters and their options,
- the list of available grouping providers,
- the list of view configurations.

## Entity method filters

The API Server's property model supports two basic types of filters for entity methods.

- A _static filter_ defines a static set of options from which the client can pick one. Since the set of options are static, they are emitted to the OpenAPI document. An example of a static filter is the "high risk" filter for attestation cases; the options are "true" and "false".
- A _dynamic filter_ is similar to a static filter. The set of options is calculated at run-time, and a dynamic filter can support more than one active value. Depending on the data situation, a client could see any number of available options for a dynamic filter, or even no options at all.

The Angular UI shows these filters in the filter sidesheet.

Each filter defines a URL parameter for the method that it is assigned to. Each value of the filter maps to a filter clause, typically a SQL WHERE clause.

Whenever using entity method filters, ensure that the `EnableDataModelApi` property of the API method is set to `true`.

### Adding a static filter

In this example, we're going to define an API returning policy violations. We want users to be able to filter the three possible states of a policy violation (pending, denied and granted). Because these three states are represented using 2 bit fields in the database (`IsDecisionMade` and `IsExceptionGranted`), it makes sense to define a filter that abstracts away the physical representation of the data.

```csharp
public void Build(IApiBuilder builder)
{
    builder.AddMethod(Method.Define("policies/violations")
        .FromTable("QERPolicyHasObject")
        .WithDescription("Returns a list of policy violations.")
        .EnableRead()
        .With(m =>
        {
            m.EnableDataModelApi = true;
            // group by pending/denied/granted state
            m.OptionalClauseProviders.Add(new StateFilter());
        })
    );
}

private class StateFilter : IPredefinedFilter
{
    public string UrlParameterName => "state";

    public string Description => "Filter by status";

    // This is the translatable display for the filter in the UI
    public MultiLanguageStringData Title { get; }
        = MultiLanguageStringData.FromWebTranslations("State");

    // Define a filter with three options: denied, pending and granted.
    public IDictionary<string, IPredefinedFilterOption> Options { get; }
        = new Dictionary<string, IPredefinedFilterOption>
        {
            {
                "denied", new PredefinedFilterOption
                {
                    // Each parameter value defines a SQL clause that is applied when the filter is selected
                    ClauseProvider = new ClauseProvider("IsDecisionMade=1 and IsExceptionGranted=0"),
                    DisplayName = MultiLanguageStringData.FromWebTranslations("Exception denied")
                }
            },
            {
                "pending", new PredefinedFilterOption
                {
                    ClauseProvider = new ClauseProvider("IsDecisionMade=0"),
                    DisplayName = MultiLanguageStringData.FromWebTranslations("Approval decision pending")
                }
            },
            {
                "granted", new PredefinedFilterOption
                {
                    ClauseProvider = new ClauseProvider("IsExceptionGranted=1"),
                    DisplayName = MultiLanguageStringData.FromWebTranslations("Exception granted")
                }
            }
        };
}
```

### Adding a dynamic filter

In this example, we're going to take the policy violation API, and add a filter by compliance framework (`ComplianceArea`).
The filter will show one option for each available compliance framework. The user can select more than one filter option.

```csharp
public void Build(IApiBuilder builder)
{
    var metadata = builder.Resolver.Resolve<IMetaData>();
    var areaTable = metadata.GetTable("ComplianceArea");
    builder.AddMethod(Method.Define("policies/violations")
        .FromTable("QERPolicyHasObject")
        .WithDescription("Returns a list of policy violations.")
        .EnableRead()
        .With(m =>
        {
            m.EnableDataModelApi = true;
            m.DynamicFilters.Add(new AreaFilter(areaTable));
        })
    );
}

internal class AreaFilter : IDynamicFilter
{
    public AreaFilter(IMetaTable areaTable)
    {
        Title = MultiLanguageStringData.FromTableDisplaySingular(areaTable);
    }

    // This character is used to separate filter option values. If this is null, then the user can only select
    // one option.
    public string Delimiter => ",";
    public string UrlParameterName => "uid_area";
    public string Description => "Filters by compliance framework";
    public MultiLanguageStringData Title { get; }
    public async Task<IReadOnlyList<DynamicFilterOption>> GetOptionsAsync(IRequest request, CancellationToken ct = default)
    {
        var frameworks = await request.Session.Source().GetCollectionAsync(
                Query.From("ComplianceArea").SelectDisplays(), ct)
            .ConfigureAwait(false);
        // Build one filter option for each ComplianceArea entity; taking the display and the primary key value.
        return frameworks.Select(p => new DynamicFilterOption
        {
            DisplayName = p.Display,
            Value = p.GetValue("UID_ComplianceArea").String
        }).ToArray();
    }
    public Task<IEnumerable<Clause>> GetClausesAsync(IRequest request, string filterValue, CancellationToken ct = default)
    {
        // Split the filter values by the delimiter.
        var uidList = filterValue.Split(Delimiter);
        // Build the filter SQL clause for policy violations filtered by compliance framework.
        var uidMatches = string.Format("uid_qerpolicy in (select uid_qerpolicy from qerpolicyinarea where {0})",
            request.Session.SqlFormatter().InClause("UID_ComplianceArea", ValType.String, uidList)
        );
        return Task.FromResult<IEnumerable<Clause>>(new[]
        {
            new WhereClause(uidMatches)
        });
    }
}
```

Do not use a dynamic filter when the set of options is very large, because the full list of options is sent to the client upon the initial request to the `/datamodel` API.

### Filter providers

_Important: Filter providers are available starting with Identity Manager 9.3._

A _filter provider_ defines a virtual filterable property that does not exist in the entity schema, it can be used to define filters that cannot be expressed using the entity schema alone. The Angular UI shows filter providers in the list of filterable properties on the "Custom filter" tab.

Note that filter providers are defined globally, not per API method.

This sample demonstrates how to register a filter provider, which allows the definition of a generic filter criterion. The creation of the SQL where clause is freely definable.

```csharp
public void Build(IApiBuilder builder)
{
    // The modifier service manages filter providers.
    var mod = builder.Resolver.Resolve<IModifierService>();
    // This is the display name of the filter.
    var display = new TranslatableString("Reporting to");
    // As an example, define a filter criterion to match all identities reporting
    // to a reference identity, as reported by the HelperHeadPerson table.
    mod.RegisterFilterProvider("Person",
        new PersonFilterProvider(display)
        {
            ValidReferencedTables = new[]
            {
                // The reference identity has to be selected from the Person table.
                new FkParentData { ParentTableName = "Person" }
            }
        });
}

private class PersonFilterProvider : FilterProvider
{
    public PersonFilterProvider(ITranslatable display) : base(
        // This is the technical identifier of our filter criterion
        "UID_PersonReportingTo",
        display,
        ValType.String)
    {
    }

    protected override string BuildFilterString(IFilterPropertyContext arg)
    {
        var val = arg.Value;
        // Create the base comparison "UID_PersonHead <operator> '...'" using the provided comparison operator
        var baseComparison = arg.SqlFormatter.UidComparison("UID_PersonHead", val.ToString(), arg.Operator.CompareOperator);
        // Wrap the base comparison in a nested SELECT
        return string.Format("UID_Person in (select uid_person from helperheadperson where {0})", baseComparison);
    }
}
```

## Grouping

To enable the grouping API, ensure that the `EnableGroupingApi` and `EnableDataModelApi` properties are set to `true`.

Use the `EnableGrouping` method to enable grouping on specific properties.

```csharp
Method.Define("person")
    .FromTable("Person")
    .EnableRead()
    // Enables default grouping behavior for the UID_Department column,
    // and allows the client to call person/group?by=UID_Department
    .EnableGrouping("UID_Department")
    .With(m =>
    {
        m.EnableDataModelApi = true;
        m.EnableGroupingApi = true;
    }
);
```

### Custom grouping providers

This example shows how to define a custom grouping provider. In this example, we're defining an API based on the `Person` table with a grouping option based on the `EntryDate`. The grouping provider defines two data groups:

- Identities that have joined since the beginning of the current month,
- All other identities.

```csharp
Method.Define("person")
    .FromTable("Person")
    .EnableRead()
    .With(m => m.EnableDataModelApi = true)
    // Add a custom group provider that defines a data group definition
    // "new hires" with two options: "this month" and "earlier".
    .With(m => m.GroupDefinitionProviders.Add(new CustomEntryDateGroupDefProvider()))
);

// ...

private class CustomEntryDateGroupDefProvider : IGroupDefinitionProvider
{
    public string Name { get; } = "newidentities";
    private readonly IGroupProvider _provider = new CustomEntryDateGroupProvider();
    public IGroupProvider GetGroupProvider(string value)
    {
        if (!string.Equals(value, "HireDate"))
            throw new ArgumentException("Unexpected value " + value);
        return _provider;
    }

    public async Task<IReadOnlyList<GroupDefinition>> GetGroupDefinitionsAsync(IRequest qr,
        CancellationToken ct = default)
    {
        // Using the request object, you can dynamically calculate groups here.
        // In this example, we just return a static array.
        return new[]
        {
            new GroupDefinition("Hire date", new GroupDefinitionOption("Hire date", "HireDate"))
        };
    }

    private class CustomEntryDateGroupProvider : IGroupProvider
    {
        public async Task<IReadOnlyList<Group>> GetGroupsAsync(string parentWhereClause, IRequest request, CancellationToken ct = default)
        {
            var now = DateTime.UtcNow;
            var startOfMonth = new DateTime(now.Year, now.Month, 1);
            return new[]
            {
                new Group("This month", new FilterData
                {
                    ColumnName = "EntryDate",
                    Type = FilterType.Compare,
                    CompareOp = CompareOperator.GreaterThan,
                    Value1 = startOfMonth
                }),
                new Group("Earlier", new FilterData
                {
                    ColumnName = "EntryDate",
                    Type = FilterType.Compare,
                    CompareOp = CompareOperator.LowerThan,
                    Value1 = startOfMonth
                })
            };
        }
    }
}

```

## Extended data

It is possible to extend entity methods with `ExtendedData` objects. These are customized data flows that integrate with the API endpoints for reading and writing data.

Typical use cases include:

- Reading and writing generic parameters (`DialogParameter` and `DialogParameterSet`) for entities. (Use the `WithParameterExtendedData` extension method for this use case.)
- Providing additional data that does not suit itself to be represented as properties of the entity model. For example, the out-of-the-box API for IT shop requests provides additional information about the approval workflow of each request as ExtendedData.

Extended data is added to an entity method by registering an extended data provider. The API includes interfaces for different types:
- Read-only (`IReadOnlyExtendedDataProvider<T>`, data is only sent to the client)
- Write-only (`IWriteOnlyExtendedDataProvider<T>`, data is only sent from the client to the server)
- Read-write (`IReadWriteExtendedDataProvider<TRead, TWrite>`, data is sent in both directions).

The classes referenced by the type parameters must be serializable and deserializable.

It is important to note that the API requests and responses always contain *exactly one* extended data object, regardless of how many entities are handled in a request.

The following sample shows a minimal extended data provider.

```csharp
public void Build(IApiBuilder builder)
{
    // Example with a custom ExtendedData provider
    builder.AddMethod(Method.Define("request_with_extendeddata")
        .FromTable("Person")
        .EnableRead()

        // include custom object
        .WithExtendedData(new ExampleExtendedDataProvider())
    );
}

private class ExampleExtendedDataProvider : IReadOnlyExtendedDataProvider<ExampleExtendedData>
{
    public async Task<ExampleExtendedData> GetExtendedDataAsync(IReadOnlyList<IEntity> entities,
        IRequest request, CancellationToken ct = default)
    {
        return new ExampleExtendedData();
    }

    public async Task ValidateAsync(IMethodValidationContext con, CancellationToken ct = default)
    {
        // include any code that runs at API compilation time.
    }
}

// This is the DTO type that is sent to the client.
public class ExampleExtendedData
{
    public string SomeProperty { get; set; }
}
```

This is an example of a write-only ExtendedData provider. There are two phases to writing data:

1. The _Apply_ phase, where the client may update the data object.
   The `ApplyAsync` method is called on the ExtendedData object for every update. In case of an interactive entity API endpoint, there may be more than one `ApplyAsync` call; one for each update call.
1. The _Commit_ phase is run when the client commits the change.
   The `CommitAsync` method is called on the ExtendedData object. If the client never commits the entity, or if an exception occurs, the `CommitAsync` method may never be called.

```csharp
public void Build(IApiBuilder builder)
{
    builder.AddMethod(Method.Define("test_with_writecheck")
        .FromTable("Person")
        .EnableRead()
        .EnableUpdate()
        .WithExtendedData(new ExampleExtendedDataWriteProvider())
    );
}

public class ExampleExtendedDataWriteProvider : IWriteOnlyExtendedDataProvider<ExampleDataObject>
{
    public Task<IWriteExtendedData<ExampleDataObject>> GetExtendedDataAsync(IReadOnlyList<IEntity> entities,
        IRequest request, CancellationToken ct = default)
    {
        return Task.FromResult<IWriteExtendedData<ExampleDataObject>>(new ExampleExtendedWriteData());
    }

    Task IValidatingMethod.ValidateAsync(IMethodValidationContext con, CancellationToken ct)
    {
        return NullTask.Instance;
    }

    private class ExampleExtendedWriteData : IWriteExtendedData<ExampleDataObject>
    {
        private ExampleDataObject CurrentValue { get; set; }

        public async Task ApplyAsync(IRequest qr, ExampleDataObject val, CancellationToken ct = default)
        {
            // The client has submitted a new value. In case of an interactive entity,
            // there is one call to ApplyAsync for each incremental change
            // before the final CommitAsync.
            CurrentValue = val;
        }

        public Task CommitAsync(IUnitOfWork unitOfWork, CancellationToken ct = default)
        {
            // The client wants to commit the change -> log the value
            unitOfWork.Session.GetLogSession().Debug("CommitAsync called with value: " + CurrentValue.Content);
            return NullTask.Instance;
        }
    }
}

public class ExampleDataObject
{
    /// <summary>
    /// Represents the content supplied by the client.
    /// </summary>
    public string Content { get; set; }
}
```

## Adding model-based properties

### Adding a foreign-key property

This example is based on a `Person` API and shows how to add each identity's primary department manager as a property.

```csharp
Method.Define("persondepartmentmanagers")
    .FromTable("Person")
    .EnableRead()
    // For every Person object, add a property for the primary department's manager.
	.WithCalculatedProperties(new FkProperty(
        // Property name for the client data model
        "DepartmentManager",

        // Foreign-key parent table name
		"Department",

		// Column name in the Department table
		"UID_PersonHead",

		// Connecting column name in the Person table
		"UID_Department")
		{
		    // Assign a display name and description to this property. (This is optional.)
			PropertyMetaData =
			{
			    Display = "Department manager",
				Description = "Manager of the primary department"
			}
		})
```

This example shows how to use child relation property to list every identity's direct reports as a property.

```csharp
// For every Person object, add a property listing the direct reports.
.WithCalculatedProperties(new CrProperty(
    // Property name for the client data model
    "Reports",
    // Foreign-key parent table name
    "Person",
    // Connecting column name
    "UID_PersonHead")));
```

### Calculated properties

```csharp
Method.Define("calculatedproperty")
    .FromTable("Person")
    .EnableRead()
    .WithResultColumns("InternalName")

    // Define a new property "InternalNameUpper":
    .WithCalculatedProperties(new CalculatedProperty<string>("InternalNameUpper",

        // Define how the property value is calculated:
        context =>

            // Obtain the data value through the entity of the context. In this example,
            // convert the "InternalName" property to upper-case:
            context.Entity.GetValue("InternalName").String.ToUpperInvariant())
    );
```

You can also define calculated properties that evaluate values in bulk mode. This is recommended if you need to run any expensive operations (such as accessing the database) for value calculation. This is an example of how to define a bulk-mode calculated property.

``` csharp
.WithCalculatedProperties(
    new CalculatedPropertyBulk<string>("SomeProperty", GetValuesForEntities)
);

// ...

private static IReadOnlyList<string> GetValuesForEntities(IBulkPropertyValueContext cx)
{
    // These are the entities for which the property value needs to be calculated.
    var entities = cx.Entities;
    // The result array must be of the same length as the input array.
    var result = new string[entities.Count];
    for (var idx = 0; idx < result.Length; idx++)
        result[idx] = "some value";
    return result;
}
```

You can also define a writable calculated property. The client can set values for these properties, and you can add API code to process the changed values.

```csharp
.WithCalculatedProperties(new CalculatedProperty(new PropertyMetaData("ArtificialProperty", false)))

// Now we subscribe to the BeforeSave event so that we can change the entity
// using the value of the artificial property.
.Subscribe(e =>
{
    if (e.Type == EntityProcessingType.BeforeSave)
    {
        var value = e.Entity.GetValue("ArtificialProperty");
        // now we can handle the value, and take action - for
        // example, change the entity.
    }
});
```

## Foreign-key candidates

Whenever a user has to make a selection of objects from a set of objects, the API has to provide this set, which are called _candidate_ objects in the API model.

You can filter the set of candidate objects on an individual foreign-key property like this.

```csharp
.WithFkWhereClause("UID_Person" /* name of the foreign-key property */,
    "Person" /* name of the parent table */,
    "IsInActive = 0" /* SQL filter condition for the Person objects */)
```

You can also define a dynamic condition and use values of the referencing entity to build a filter condition.

In this example, the candidate set for the State (`UID_DialogState`) property of a `Person` is filtered to match the selected country (`UID_DialogCountry`).

```csharp
.FromTable("Person")
.EnableUpdate()
.WithWritableColumns("UID_DialogState")
.WithFkWhereClause("UID_DialogState", "DialogState", context => {
    var uidCountry = context.Entity.GetValue("UID_DialogCountry").String;
    if (string.IsNullOrEmpty(uidCountry))
    {
        return null; // no filter
    }

    // filter the DialogState objects with the selected country
    return context.Request.Session.SqlFormatter().UidComparison("UID_DialogCountry", uidCountry);
})
```

### Hierarchy configuration

Candidates may be loaded from a hierarchical table. In this case, the usual semantics for hierarchical loading will apply and Angular client will try to load the root level first.

As shown above, you can filter the candidates using any SQL condition like this.

```csharp
.WithFkWhereClause("UID_Locality", "Locality", "CCC_SomeCondition = 1")
```

By doing this, the hierarchical loading model may fail because no candidates meet both conditions (they match the `CCC_SomeCondition = 1` filter _and_ they are on the root level of the `Locality` table).

The easiest solution to this problem is to return the candidates as a flat hierarchy instead.

```csharp
.WithFkCandidateConfiguration("LocationFilter", "Locality",
    a => a.Crud.Read.Hierarchy = new FlatHierarchy(), false)
```

There is also an option to use a partial sub-tree hierarchy with a single known root element. The UID of this element has to be supplied as a parameter. This can be configured like this:

```csharp
.WithFkCandidateConfiguration("LocationFilter", "Locality",
    a => a.Crud.Read.Hierarchy = new BaseTreeHierarchy("Locality", "UID_Locality", "UID_ParentLocality", "UID_OF_ROOT_LOCALITY", sqlformatter), false)
```

## Entity event model

Whenever an entity is loaded or created during the processing lifecycle of an entity method, you can subscribe to entity changes in order to react to entity events.

```csharp
public void Build(IApiBuilder builder)
{
    Method.Define("person")
        .FromTable("Person")
        .Subscribe(ProcessingEntity);
}

private void ProcessingEntity(IEntityProcessingContext cx)
{
    // Subscribe to a newly created entity
    if (cx.Type != EntityProcessingType.CreateNew)
        return;
    cx.Entity.Subscribe(new GenericAsyncObserver<EntityEventArgs>(args =>
    {
        if (args.Type == EntityEventType.ColumnChanged)
        {
            // Get the name of the changed column
            var columnArgs = (EntityColumnChangedEventArgs)args;
            var columnName = columnArgs.Columnname;
        }
    }));
}
```

In some cases, it is more efficient to process bulk events. When events are triggered for more than one entity, you can also subscribe to a bulk event.

```csharp
Method.Define("person")
    .FromTable("Person")
     .Subscribe(ProcessingEntities);

// ...

private void ProcessingEntities(IBulkEntityProcessingContext cx)
{
    var entities = cx.Entities;
}
```

## Modifying an entity method

Call the `ModifyCrudMethod` method to modify or extend an entity method.

```csharp
builder.ModifyCrudMethod(
	// This is the URL of the method to be modified.
	"person/specificcolumns",
    method =>
    {
        // Include one more property in the result.
        method.WithResultColumns("CentralAccount");

        // Add one more filter provider
        method.WhereClauseProviders.Add(new WhereClauseProvider((request, whereClause) =>
            // add another condition using "AND" operator
            request.Session.SqlFormatter().AndRelation(whereClause, "IsTemporaryDeactivated=0")));
    });
```

## Modifying property semantics

The `IModifierService` service manages modifications for an API project. You can use this service to modify the semantics of a database property. For example, you can:

- change the property's caption
- limit the property's set of permitted values (this set must be a subset of all permitted values)
- limit the set of assignable foreign-key values.

This example shows how to limit the possible values for an identity's `UID_DialogState` assignment.

```csharp
var mod = builder.Resolver.Resolve<IModifierService>();
// Obtain the modifiers for this column.
mod.GetPropertyModifiers("Person", "UID_DialogState")
    .Add(new PropertyModifier
    {
        // Add a modifier that limits the possible values for the UID_DialogState
        // column, depending on the value for UID_DialogCountry.
        FkWhereClauses =
        {
            new FkWhereClause(r => r.Request.Session.SqlFormatter().UidComparison("UID_DialogCountry",
                r.Entity.GetValue("UID_DialogCountry")))
        }
    });
```

If you wish to modify the semantics of a property only for a specific API method, you can use the `Modify` method of the API method. This sample shows how to apply a the same modification as above, but only for one specific method.

```csharp
public void Build(IApiBuilder builder)
{
    builder.AddMethod(Method.Define("modified")
        .FromTable("Person")
        .EnableCreate()
        .WithWritableColumns("UID_DialogCountry", "UID_DialogState")
        // Add a modifier that limits the possible values for the UID_DialogState
        // column, depending on the value for UID_DialogCountry.
        .Modify("UID_DialogState", mo => mo.FkWhereClauses.Add(
            new FkWhereClause(r => r.Request.Session
                .SqlFormatter().UidComparison("UID_DialogCountry",
                    r.Entity.GetValue("UID_DialogCountry")))))
        // This is an example for a custom display value provider;
        // converting an e-mail address to lower-case.
        .Modify("DefaultEMailAddress",
            modifier => modifier.DynamicModifiers.Add(
                new LowerCaseModifier(new ConvertToLowerCase())))
    );
}

private class ConvertToLowerCase : IDisplayValueProvider
{
    public async Task<string> GetDisplayValueAsync(IDisplayValueContext context, CancellationToken ct = default)
    {
        // get actual data value
        var value = await context.InnerColumn.GetValueAsync(ct).ConfigureAwait(false);
        // convert to lower-case
        return value?.ToString().ToLowerInvariant();
    }
}

private class LowerCaseModifier : IEntityColumnModifier
{
    private readonly EntityColumnModifierResult _modifier;
    public LowerCaseModifier(IDisplayValueProvider provider)
    {
        _modifier = new EntityColumnModifierResult
        {
            DisplayValueProvider = provider
        };
    }
    public EntityColumnModifierResult Get(IEntity entity)
    {
        return _modifier;
    }
}
```

This example shows how to apply modifiers dynamically. If an identity's `IsExternal` flag is set, the property `UID_FirmPartner` becomes mandatory.

``` csharp

public void Build(IApiBuilder builder)
{
    // Add a modifiers to the API method definition
    builder.AddMethod(Method.Define("person")
        .FromTable("Person")
        .EnableCreate()
        .WithResultColumns("UID_FirmPartner")
        .Modify("UID_FirmPartner", mod => mod.DynamicModifiers.Add(new MandatoryCompanyModifier())));
}

// sample modifier object
internal class MandatoryCompanyModifier : IEntityColumnModifier
{
    public EntityColumnModifierResult Get(IEntity entity)
    {
        // is it an external user? If so, then UID_FirmPartner becomes
        // a mandatory field
        if (entity.GetValue("IsExternal").Bool)
            return new EntityColumnModifierResult
            {
                MinLen = 1
            };
        // if not -> no modification
        return null;
    }
}
```

The following example of a dynamic modifier to restrict a date property to disallow dates that are in the past.

```csharp
private class DateNotInPastModifier : IEntityColumnModifier
{
    public EntityColumnModifierResult Get(IEntity entity)
    {
        // return a modifier with a constraint that defines that MinDate
        // should not be in the past
        return new EntityColumnModifierResult
        {
            ValueConstraint = new ValueConstraint
            {
                MinValue = DateTime.UtcNow
            }
        };
    }
}
```

## Type-safe method definitions

For some modules, a typed wrapper library `<module>.TypedWrappers.dll` is provided that contains types for the tables for a module. Reference the correct library and use the type directly to define an entity-based API method.

```csharp
// needs QBM.TypedWrappers.dll
Method.Define("test")
    .From<QBM.TypedWrappers.QBMVSystemOverview>()
    .EnableRead()
    // Use the column names directly as LINQ expressions.
    .WithResultColumns(x => x.Element, x => x.QualityOfValue, x => x.RecommendedValue)
```

The created entity model method implements the generic interface `ICrudModel<T>` where `T` must implement the interface `ITypedEntityWrapper`.

Please note: While generic equivalents are provided for some parts of the definition API, the generic definition API is not complete at this time. Because `ICrudModel<T>` inherits from `ICrudModel`, you can also use all the functionality of untyped API definition, so mixing typed and untyped definition code is possible.

## Changing an entity request at runtime

You can change the parameters of an entity request at runtime. For example, you can add filters or change the sorting.

``` csharp
Method.Define("person")
   .FromTable("Person")
   .EnableRead()
   .SubscribeProcessing((request, ct) => PrepareRequestAsync(request, ct))

// ...

private static async Task PrepareRequestAsync(IRequest request, CancellationToken ct)
{
    // Get the request's entity configuration
    var env = request.GetEntityCollectionRequest();

    // add a filter clause
    env.FilterClauses.Add(new WhereClause("IsInactive = 0"));
}
```

You can also define an entity method that is not bound to an object type. In this case, you have to set the object type at runtime.

``` csharp
Method.Define("generic")
   .FromTable() // this is an unbound entity method
   .EnableRead()
   .SubscribeProcessing((request, ct) => PrepareRequestAsync(request, ct))

// ...

private static async Task PrepareRequestAsync(IRequest request, CancellationToken ct)
{
    // Get the request's entity configuration
    var env = request.GetEntityCollectionRequest();

    // set the Person table as the data source
    env.Table = new MetaTableDescriptor(request.Session.MetaData().GetTable("Person"));
}
```

## Search modes

The API Server supports 3 possible search modes that define how search results are used when loading entities.

When the server receives a request to this API including a search term, it has to produce a result based on two sets of entities:

 - *A* is the set of entities returned by the search index, identified by their object keys.
 - *B* is the set of entities returned by the database query, filtered by the permission layer.

The result of the API query *C* must be the **intersection** of *A* and *B*.

The API Server supports 3 different `SearchMode`s.

- `Default`
- `SearchAfterLoad`
- `LoadCached`

The mode can be configured on the API level:

``` csharp
builder.AddMethod(Method.Define("requests")
    .FromTable("PersonWantsOrg")
    .EnableRead()
    .With(m => m.SearchMode = SearchMode.SearchAfterLoad)
);
```

Each mode has different performance characteristics and the best mode for each API depends on a combination of factors:
- the table size,
- the average expected result size
- the permission configuration.

Also consider that the number of search results returned by the index is limited by the `Common\Indexing\DefaultResultLimit` configuration parameter.

### Default mode

This mode works best when the size of *A* is comparable to *C*. In this mode, *B* is calculated and *A* is queried using the set *B* as a filter. Because the limitation is applied early on in the process, this can lead to the user seeing few search results even though there may be more.

### SearchAfterLoad mode

This mode can be considered the opposite of the default mode: First the entirety of *B* is loaded into memory. Then, *A* is evaluated using *B* as a filter.

This mode works best when *A* is large but *B* is comparatively small.

In the portal application, this mode is used for the request history API for regular (non-administrator users). Regular users only see their requests, so *B* is comparatively small. This behavior can be defined in code like in this example:

``` csharp
builder.AddMethod(Method.Define("requests")
    .FromTable("PersonWantsOrg")
    .EnableRead()
    .Subscribe(c =>
    {
        // Define the SearchMode per request

        var coll = c.Request.GetEntityCollectionRequest();
        var names = c.Request.Session.Principal.Groups.GetGroupNames().ToHashSet();

        // Check the user's permission groups
        var isAuditor = names.Contains("vi_4_AUDITING_AUDITOR");

        if (isAuditor)
        {
            // Sees all entities -> use default mode
            coll.SearchMode = EntitySearchMode.Default;
        }
        else
        {
            // User sees only a small subset -> use SearchAfterLoad mode
            coll.SearchMode = EntitySearchMode.SearchAfterLoad;
        }
    }
);
```

### LoadCached mode

Depending on the situation, *A* and *B* can both be large, but the intersection *C* may be small.

This mode addresses this situation, it loads *B* once and caches the result across different users and queries. (This mode only works when there is no viewing permission filter on the table. If there is, then the cache is bypassed for security reasons.)

The default service catalog API uses this mode. This works well because the base set of `AccProduct` entities lends itself well to cross-user caching.

# Composition API plugins

This document describes Composition API plugins.

Note that this type of plugins is different from API Server plugins because they also need to work outside of the API Server.

## Setting up a Composition API plugin

The steps to add a Composition API plugin are similar to setting up API Server plugins. The differences are as follows:

- Add references to `QER.CompositionApi.dll` and `QBM.CompositionApi.dll`.
- Do not add a reference to `QBM.CompositionApi.Server.dll`.
- Plugin file names have to match the pattern `*.CompositionApi.dll`.

## Registering additional Composition API services

To register additional services at initialization time, implement the `IBootstrapper` interface and register your service instances:

``` csharp
public class Bootstrapper : IBootstrapper
{
    public void RegisterServices(IServices services)
    {
        // Resolve the plugin service
        var pluginService = services.Resolve<IPluginService>();

        // This is an example showing the pattern
        pluginService.Register<IServiceInterface>(new ServiceImplementation());
    }
}
```

The `RegisterService` method will be called when the Composition API initializes its services.

## IT shop component parameters

The Composition API offers an interface to extend IT shop requests with *component parameters*. In the UI, these parameters act like request properties. However, they are completely independent and defined in code. The values of these parameters can be stored anywhere in the database.

This sample will show how to define such properties.

In a first step, we declare a shop component by defining a class implementing the `IShopComponent` interface. We assign it to specific service items by hard-coding their object keys and implementing the `IComponentAssignment<IShopComponent>` interface.

``` csharp
public class ShopLogic : IComponentAssignment<IShopComponent>, IShopComponent
{
    public ShopLogic()
    {
        Assignments = new Dictionary<string, IShopComponent>
        {
            {
                // Object key of the service item
                "<Key><T>AccProduct</T><P>CCC-abcdefabcde-fabcdefabcdefabcde</P></Key>",
                this
            }
        };
    }

    public IReadOnlyDictionary<string, IShopComponent> Assignments { get; }

    public string DisplayType => null;

    public bool CanRequest => true;

    public ICartItemLogic GetCartItemLogic(IEntity cartItem, ICartItemContext context)
    {
        return new AccessRequestLogic(cartItem);
    }

    public IPwoLogic GetPwoLogic(IEntity pwo, IPwoLogicContext context)
    {
        return new AccessRequestLogic(pwo);
    }
}
```

The class `AccessRequestLogic` defines the parameters for this service item. You can distinguish between three contexts:
- the request is in the shopping cart (`ICartItemLogic`)
- the request is being viewed by an approver (`IPwoLogic` with context `Approver`)
- the request is being viewed in the history (`IPwoLogic` with context `History`)

(This class covers all three cases, but there could be different logic for each case.)

``` csharp
private class AccessRequestLogic : ICartItemLogic, IPwoLogic
{
    private readonly IEntity _entity;
    public AccessRequestLogic(IEntity entity)
    {
        _entity = entity;
    }

    Task<string> IRequestBaseLogic.GetUiTextAsync(IEntity entity, ISession session, CancellationToken ct)
    {
        return Task.FromResult<string>(null);
    }

    Task IRequestBaseLogic.CommitAsync(IUnitOfWork unitOfWork, CancellationToken ct)
    {
        return NullTask.Instance;
    }

    public async Task<IReadOnlyList<IComponentParameter>> GetParametersAsync(ISession session,
        CancellationToken ct = default)
    {
        // This class returns just one parameter: we expose the value of the OrderDetail1 property.

        var result = new List<IComponentParameter>();
        var isReadOnly = _entity.Tablename != "ShoppingCartItem";
        result.Add(new ComponentParameter("OrderDetail1", _entity, isReadOnly));
        return result;
    }

    public Task<IReadOnlyList<ICartItemCheck>> CheckAsync(ICartItemCheckContext context, CancellationToken ct = default)
    {
        return NoChecks;
    }

    public Task ProcessAsync(CancellationToken ct = default)
    {
        return NullTask.Instance;
    }
}
```

The final step is the implementation of the `CompnentParameter` class which supplies the definition of a single parameter. In this case, we map the parameter value directly to a property of the `ShoppingCartItem`/`PersonWantsOrg` entity. However, the values can be read from and written to anywhere.

``` csharp
class ComponentParameter : IComponentParameter
{
    public ComponentParameter(string columnName, IEntity entity, bool isReadOnly)
    {
        Name = columnName;
        _entity = entity;
        _isReadOnly = isReadOnly;
    }
    public string Name { get; }

    private readonly IEntity _entity;
    private readonly bool _isReadOnly;

    public async Task<ParameterAdapter> GetDataAsync(CancellationToken ct = default)
    {
        var p = new ParameterAdapter
        {
            Name = Name,
            Type = ValType.String,
            IsReadOnly = _isReadOnly || !_entity.Columns[Name].CanEdit,
            Display = "Component Parameter",
            // Get the current value and expose it to the client.
            Value = await _entity.GetValueAsync<string>(Name, ct).ConfigureAwait(false)
        };
        return p;
    }

    Task IComponentParameter.ApplyAsync(object val, CancellationToken ct)
    {
        // Save the value provided by the client in the entity
        return _entity.PutValueAsync(Name, val, ct);
    }

    Task IComponentParameter.CommitAsync(IUnitOfWork unit, CancellationToken ct)
    {
        return unit.PutAsync(_entity, cancellationToken: ct);
    }
}
```

The `ParameterAdapter` class also allows defining more advanced parameters. For example, you can define a list of allowed values.

``` csharp
parameterAdapter.LimitedValues = new LimitedValuesCollection
{
    new LimitedValue("Security", "Security"),
    new LimitedValue("Distribution", "Distribution")
};
```

Alternatively, you can define the parameter as a foreign-key selection from a source table.

``` csharp
var table = await _session.MetaData().GetTableAsync("Person", ct).ConfigureAwait(false);
// Configure the selection of an identity.
parameterAdapter.QueryColumns = new[] {table.Columns["UID_Person"]};
// Filter only internal active identities.
parameterAdapter.QueryWhereClause = "IsExternal=0 and IsInActive=0";
```

## Optional service items in the IT Shop

In addition to the service item dependencies defined in the configuration, you can dynamically define optional service items in code. To do that, add a class implementing `IServiceItemDependencyProvider` and register it with the plugin service.

Each `IServiceItemDependencyProvider` is called with the following information:
- the current session,
- the list of recipients (as UIDs),
- the UID of the requested service item,
- and the check mode.

The check mode indicates the context of the call. The code is called while the user adds the service item to the shopping cart (but before the cart item is actually created). It is also called during the shopping cart validation to ensure that the optional items are still valid.

In the return value, you can return any number of optional service items, and any combination of recipients.

``` csharp
public class Bootstrapper : IBootstrapper
{
    public void RegisterServices(IServices services)
    {
        // Resolve the plugin service
        var pluginService = services.Resolve<IPluginService>();

        // This is an example showing the pattern
        pluginService.Register<IServiceItemDependencyProvider>(new CustomDependencyProvider());
    }
}

public class CustomDependencyProvider : IServiceItemDependencyProvider
{
    public async Task<IReadOnlyList<ServiceItemDependency>> GetOptionalServiceItemsAsync(ISession session, string[] uidPersonRecipients,
        string uidAccProduct, DependencyMode mode, CancellationToken ct = default)
    {
        // If the user requested a different product, exit
        if (uidAccProduct != "UID-OF-MY-ACCPRODUCT")
        {
            return Array.Empty<ServiceItemDependency>();
        }

        // insert your code to calculate the optional service item(s) here.
        
        var uidOptionalProduct = "UID-OF-MY-OPTIONAL-ACCPRODUCT";

        return new[] {
            new ServiceItemDependency {
                UidAccProduct = uidOptionalProduct,
                // assuming the dependency is valid for every one of the selected recipients:
                UidPersonsValidFor = uidPersonRecipients
            }
        };
    }
}
```

## Shopping cart checks

You can define a plugin to add custom logic to the shopping cart check.

In a Composition API plugin, add a public class implementing the `ICartCheckProvider` interface to integrate with the cart check.

Another class implementing `ICartCheck` is responsible for creating check classes for the individual cart items. The `ICartItemCheck` object implements the actual checking logic for a single item. The object has access to a `ICartItemCheckContext` that provides the user's session and the cart item being checked.

```csharp
public class CartCheckProvider : ICartCheckProvider, IKnownTypeProvider
{
    public Task<IReadOnlyList<ICartCheck>> GetCartChecksAsync(ICartCheckContext cart, CancellationToken ct = default)
    {
        return Task.FromResult<IReadOnlyList<ICartCheck>>(new[]
        {
            new SampleCartCheck()
        });
    }

    // This method defines implementation types of an interface that are known to be safe
	// for (de-)serialization.
    public IReadOnlyDictionary<Type, IReadOnlyList<Type>> GetTypes()
    {
        return new Dictionary<Type, IReadOnlyList<Type>>
        {
            {
                typeof(ICartItemCheck),
                new[]
                {
                    typeof(SampleCheck)
                }
            }
        };
    }
}

public class SampleCartCheck : ICartCheck
{
    private SampleCheck _check;

    // This method is called when a shopping cart must be checked.
    public Task<IReadOnlyList<ICartItemCheck>> CheckAsync(ICartItemCheckContext context, CancellationToken ct = default)
    {
        // Set up the individual ICartItemCheck object(s) and return them.
        _check = new SampleCheck(context);
        return Task.FromResult<IReadOnlyList<ICartItemCheck>>(new[]
        {
            _check
        });
    }

    public Task ProcessAsync(CancellationToken ct = default)
    {
        return _check.ProcessAsync(ct);
    }
}

public class SampleCheck : ICartItemCheck
{
    private readonly ICartItemCheckContext _context;

    public SampleCheck(ICartItemCheckContext context)
    {
        _context = context;
    }

    internal async Task ProcessAsync(CancellationToken ct = default)
    {
        var session = _context.Session;
        var cartitem = _context.Item;
        // Any logic required to calculate the check result should go here.
        // At the end we will have a result:
        Status = CheckStatus.Success; // ... or CheckStatus.Error
        // Set the description text to provide more information about the result.
        if (Status == CheckStatus.Success)
        {
            ResultText = "This check was successful";
        }
        else
        {
            ResultText = "This check failed";
        }
    }

    public string Id => "SampleCartItemCheck";
    public CheckStatus Status { get; set; } = CheckStatus.Pending;
    public string Title => "Sample customizable check";
    public string ResultText { get; set; }
    // Populate this property to append any serializable data to the result.
    public object Detail => null;
}

```

## Workflow recommendations

You can define a plugin to add custom recommendation logic.

In a Composition API plugin, add a public class implementing one of these interfaces:

- For IT shop workflows, use the interface `QER.CompositionApi.ITShop.Recommendation.IRecommendationProvider`.
- For attestation workflows, use the interface `ATT.CompositionApi.Recommendation.IRecommendationProvider`.

The recommendation provider is called during the calculation of the recommendation for a specific request/attestation procedure. The recommendation provider is responsible for calculating and returning a `RecommendationItem` describing the details of the calculation.

The `Weight` property is used to calculate an aggregated recommendation.

- If the sum of all `Weight` values is 0, the recommendation is `Approve`.
- If the sum of all `Weight` values is >= 1.0, the recommendation is `Deny`.
- Otherwise, there is no recommendation.

The following code shows an example for a recommendation provider implementation.

``` csharp
public class ExampleRecommendationProvider : IRecommendationProvider
{
    // Calculate the recommendation for this request.
    public async Task<TryResult<RecommendationItem>> BuildAsync(IEntity request, ISession session,
        CancellationToken ct = default)
    {
        // Return TryResult<RecommendationItem>.Failed if this type of recommendation
        // does not apply to this request and should be skipped.
        if (request.Display === "test-failed")
        {
            return TryResult<RecommendationItem>.Failed;
        }

        // Insert logic here to determine whether to increase the weight.
        var weight = 0.5;

        var item = new RecommendationItem
        {
            Id = nameof(ExampleRecommendationProvider),
            Weight = weight,
            Value = riskIndex,
            Title = "Recommendation example",
            DetailText = ""
        };

        return TryResult<RecommendationItem>.FromResult(item);
    }
}
```
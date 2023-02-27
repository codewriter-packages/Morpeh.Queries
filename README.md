# Morpeh.Queries [![Github license](https://img.shields.io/github/license/codewriter-packages/Morpeh.Events.svg?style=flat-square)](#) [![Unity 2020](https://img.shields.io/badge/Unity-2020+-2296F3.svg?style=flat-square)](#) ![GitHub package.json version](https://img.shields.io/github/package-json/v/actionk/Morpeh.Queries?style=flat-square)

Entity queries plugin (alternative to built-in filters) using lambdas for [Morpeh ECS](https://github.com/scellecs/morpeh).

## Example

```csharp
public class ExampleQuerySystem : QuerySystem
{
    protected override void Configure()
    {
        CreateQuery()
            .WithAll<PlayerComponent, ViewComponent, Reference<Transform>>()
            .WithNone<Dead>()
            .ForEach((Entity entity, ref Reference<Transform> transformRerefence, ref ViewComponent viewComponent) =>
            {
                testQueryComponent.value++;
            });
    }
}
```

## Comparison

### Before

Usually, the regular system in Morpeh is implemented this way:

```csharp
public class NoQueriesTestSystem : UpdateSystem
{
    private Filter filter;

    public override void OnAwake()
    {
        filter = World.Filter.With<TestComponent>();
    }

    public override void OnUpdate(float deltaTime)
    {
        foreach (var entity in filter)
        {
            ref var testQueryComponent = ref entity.GetComponent<TestComponent>();
            testQueryComponent.value++;
        }
    }
}
```

There will be `1 000 000` entities and `100` iterations of testing for this and the other examples;

Results: **14.43** seconds.

In order to optimize this, we can store a reference to the `Stash<T>` that contains all the components of type `TestComponent` for different entities:

```csharp
public class NoQueriesUsingStashTestSystem : UpdateSystem
{
    private Filter filter;
    private Stash<TestComponent> stash;

    public override void OnAwake()
    {
        filter = World.Filter.With<TestComponent>();
        stash = World.GetStash<TestComponent>();
    }

    public override void OnUpdate(float deltaTime)
    {
        foreach (var entity in filter)
        {
            ref var testQueryComponent = ref stash.Get(entity);
            testQueryComponent.value++;
        }
    }
}
```

Results: **9.05** seconds (-38%)

### After

In order to remove the boilerplate for acquiring the components and still have it optimized using Stashes, you can use the Queries from this plugin instead: 

```csharp
public class WithQueriesSystem : QuerySystem
{
    protected override void Configure()
    {
        CreateQuery()
            .With<TestComponent>()
            .ForEach((Entity entity, ref TestComponent testQueryComponent) =>
            {
                testQueryComponent.value++;
            });
    }
}
```

Results: **9.45** seconds (+5%)

As you can see, we're using a `QuerySystem` abstract class that implements the queries inside, therefore we have no `OnUpdate` method anymore. If you need the `deltaTime` though, you can acquire it using `protected float deltaTime` field in `QuerySystem`, which is updated every time `QuerySystem.OnUpdate()` is called.

Performance-wise, it's a bit slower than the optimized solution that we've looked previously (because of using lambdas), but still faster that the "default" one and is **much** smaller than both of them.

## Documentation

### CreateQuery()

CreateQuery() returns an object of type QueryConfigurer, that has many overloads for filtering that you can apply before describing the update lambda. Examples are as follows:

### .WithAll

Selects all the entities that have **all** of the specified components.


```csharp
CreateQuery()
    .WithAll<TestComponent, DamageComponent>()
    .ForEach(...)
CreateQuery()
    .WithAll<TestComponent, DamageComponent, PlayerComponent, ViewComponent>()
    .ForEach(...)
```

Supports up to 8 arguments (but you can extend it if you want).

Equivalents in Morpeh:
```csharp
Filter = Filter.With<TestComponent>().With<DamageComponent>();
Filter = Filter.With<TestComponent>().With<DamageComponent>().With<PlayerComponent>().With<ViewComponent>();
```

### .WithNone

Selects all the entities that have **none** of the specified components.

```csharp
CreateQuery()
    .WithNone<Dead, Inactive>()
    .ForEach(...)
CreateQuery()
    .WithNone<Dead, Inactive, PlayerComponent, ViewComponent>()
    .ForEach(...)
```

Supports up to 8 arguments (but you can extend it if you want).

Equivalents in Morpeh:
```csharp
Filter = Filter.Without<Dead>().Without<Inactive>();
Filter = Filter.Without<Dead>().Without<Inactive>().Without<PlayerComponent>().Without<ViewComponent>();
```

### .With<T>

Equivalent to Morpeh's `Filter.With<T>`.

### .Without<T>

Equivalent to Morpeh's `Filter.Without<T>`.

### .Also

You can specify your custom filter if you want:
```csharp
CreateQuery()
    .WithAll<TestComponent, DamageComponent>()
    .Also(filter => filter.Without<T>())
    .ForEach(...)
```

### Sequencing

Multiple filtering calls can be used in a sequence before describing the ForEach lambda:

```csharp
CreateQuery()
    .WithAll<TestComponent, DamageComponent>
    .WithNone<Dead, Inactive>()
    .ForEach(...)
```

### .ForEach

There are multiple supported options for describing a lambda:

```csharp
.ForEach<TestComponent>(ref TestComponent component)
.ForEach<TestComponent>(Entity entity, ref TestComponent component)
```

You can either receive the entity as the 1st parameter or you can just skip it if you only need the components.

#### Restrictions:

* You can only receive components as **ref**
* You can't receive Aspects

### Automatic Validation

Be default, the query engine applies checks when you create a query: all the components that you're using in `ForEach` should also be defined in a query using `With` or `Without` to guarantee that the components exist on the entities that the resulting `Filter` returns.

This validation **only happens once** when creating a query so it doesn't affect the performance of your `ForEach` method! 

However, if you're willing to disable the validation for some reason, you can use `.SkipValidation(true)` method:

```csharp
CreateQuery()
    .WithAll<TestComponent, DamageComponent>
    .SkipValidation(true)
    .ForEach(...)
```

### OnAwake & OnUpdate

You can override `OnAwake` & `OnUpdate` methods if you want to:

```charp
public override void OnAwake()
{
    base.OnAwake();
}

public override void OnUpdate(float newDeltaTime)
{
    base.OnUpdate(newDeltaTime);
}
```

Don't forget to call the base method, otherwise `Configure` and/or queries execution won't happen!


## License

Morpeh.Queries is [MIT licensed](./LICENSE.md).
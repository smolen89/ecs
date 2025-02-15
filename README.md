[![gitter](https://img.shields.io/gitter/room/leopotam/ecs.svg)](https://gitter.im/leopotam/ecs)
[![discord](https://img.shields.io/discord/404358247621853185.svg?label=discord)](https://discord.gg/5GZVde6)
[![license](https://img.shields.io/github/license/Leopotam/ecs.svg)](https://github.com/Leopotam/ecs/blob/develop/LICENSE)
# LeoECS - Simple lightweight C# Entity Component System framework
Performance, zero/small memory allocations/footprint, no dependencies on any game engine - main goals of this project.

> C#7.3 or above required for this framework.

> Tested on unity 2018.3 (not dependent on it) and contains assembly definition for compiling to separate assembly file for performance reason.

> **Important!** Dont forget to use `DEBUG` builds for development and `RELEASE` builds in production: all internal error checks / exception throwing works only in `DEBUG` builds and eleminated for performance reasons in `RELEASE`.

# Installation

## As unity module
This repository can be installed as unity module directly from git url. In this way new line should be added to `Packages/manifest.json`:
```
"com.leopotam.ecs": "https://github.com/Leopotam/ecs.git",
```
By default last released version will be used. If you need trunk / developing version then `develop` name of branch should be added after hash:
```
"com.leopotam.ecs": "https://github.com/Leopotam/ecs.git#develop",
```

## As source
If you can't / don't want to use unity modules, code can be downloaded as sources archive of required release from [Releases page](`https://github.com/Leopotam/ecs/releases`).

# Main parts of ecs

## Component
Container for user data without / with small logic inside. Can be used any user class without any additional inheritance:
```csharp
class WeaponComponent {
    public int Ammo;
    public string GunName;
}
```

> **Important!** Dont forget to manually init all fields of new added component. Default value initializers will not work due all components can be reused automatically multiple times through builtin pooling mechanism (no destroying / creating new instance for each request for performance reason).

> **Important!** Dont forget to cleanup reference links to instances of another components / engine classes before removing components from entity, otherwise it can lead to memory leaks.
>
> By default all `marshal-by-reference` typed fields of component (classes in common case) will be checked for null on removing attempt in `DEBUG`-mode. If you know that you have object instance that should be not null (preinited collections for example) - `[EcsIgnoreNullCheck]` attribute can be used for disabling these checks.

## Entity
Сontainer for components. Implemented with struct `EcsEntity` for wrapping internal identifiers:
```csharp
WeaponComponent myWeapon;
EcsEntity entity = _world.CreateEntityWith<WeaponComponent> (out myWeapon);
_world.RemoveEntity (entity);
```
Previous example can be simplified with new C# syntax:
```csharp
var entityId = _world.CreateEntityWith<WeaponComponent> (out var myWeapon);
_world.RemoveEntity (entityId);
```

Dont forget that `EcsWorld.CreateEntityWith` method has multiple overloaded versions:
```csharp
Component1 c1;
Component2 c2;
EcsEntity entityId = _world.CreateEntityWith<Component1, Component2> (out c1, out c2);
_world.RemoveEntity (entityId);
```

> **Important!** Entities without components on them will be automatically removed from `EcsWorld` right after finish execution of current system.

## System
Сontainer for logic for processing filtered entities. User class should implements `IEcsPreInitSystem`, `IEcsInitSystem` or / and `IEcsRunSystem` interfaces:
```csharp
class WeaponSystem : IEcsPreInitSystem, IEcsInitSystem {
    void IEcsPreInitSystem.PreInitialize () {
        // Will be called once during world initialization and before IEcsInitSystem.Initialize.
    }

    void IEcsInitSystem.Initialize () {
        // Will be called once during world initialization.
    }

    void IEcsInitSystem.Destroy () {
        // Will be called once during world destruction.
    }

    void IEcsPreInitSystem.PreDestroy () {
        // Will be called once during world destruction and after IEcsInitSystem.Destroy.
    }
}
```

```csharp
class HealthSystem : IEcsRunSystem {
    void IEcsRunSystem.Run () {
        // Will be called on each EcsSystems.Run() call.
    }
}
```

# Data injection
> **Important!** Will not work when LEOECS_DISABLE_INJECT preprocessor constant defined.

With `[EcsInject]` attribute over `IEcsSystem` class all compatible `EcsWorld` and `EcsFilter<T>` fields of instance of this class will be auto-initialized (auto-injected):
```csharp
[EcsInject]
class HealthSystem : IEcsSystem {
    EcsWorld _world = null;

    EcsFilter<WeaponComponent> _weaponFilter = null;
}
```
Instance of any custom type can be injected to all systems through `EcsSystems.Inject` method:
```csharp
var systems = new EcsSystems (world)
    .Add (new TestSystem1 ())
    .Add (new TestSystem2 ())
    .Add (new TestSystem3 ())
    .Inject (a)
    .Inject (b)
    .Inject (c)
    .Inject (d);
systems.Initialize ();
```
Each system will be scanned for compatible fields (can contains all of them or no one) with proper initialization.

> **Important!** Data injection for any user type can be used for sharing external data between systems.

## Data Injection with multiple EcsSystems

If you want to use multiple `EcsSystems` you can find strange behaviour with DI:

```csharp
class Component1 { }

[EcsInject]
class System1 : IEcsInitSystem {
    EcsWorld _world = null;

    void IEcsInitSystem.Initialize () {
        _world.CreateEntityWith<Component1> ();
    }

    void IEcsInitSystem.Destroy () { }    
}

[EcsInject]
class System2 : IEcsInitSystem {
    EcsFilter<Component1> _filter;

    void IEcsInitSystem.Initialize () {
        Debug.Log (_filter.GetEntitiesCount());
    }

    void IEcsInitSystem.Destroy () { }    
}

var systems1 = new EcsSystems (world);
var systems2 = new EcsSystems (world);
systems1.Add (new System1 ());
systems2.Add (new System2 ());
systems1.Initialize ();
systems2.Initialize ();
```
You will get "0" at console. Problem is that DI starts at `Initialize` method inside each `EcsSystems`. It means that any new `EcsFilter` instance (with lazy initialization) will be correctly injected only at current `EcsSystems`. 

To fix this behaviour startup code should be modified in this way:

```csharp
var systems1 = new EcsSystems (world);
var systems2 = new EcsSystems (world);
systems1.Add (new System1 ());
systems2.Add (new System2 ());
systems1.ProcessInjects();
systems2.ProcessInjects();
systems1.Initialize ();
systems2.Initialize ();
```
You should get "1" at console after fix.

# Special classes

## EcsFilter<T>
Container for keeping filtered entities with specified component list:
```csharp
[EcsInject]
class WeaponSystem : IEcsInitSystem, IEcsRunSystem {
    EcsWorld _world = null;

    // We wants to get entities with "WeaponComponent" and without "HealthComponent".
    EcsFilter<WeaponComponent>.Exclude<HealthComponent> _filter = null;

    void IEcsInitSystem.Initialize () {
        _world.CreateEntityWith<WeaponComponent> ();
    }

    void IEcsInitSystem.Destroy () { }

    void IEcsRunSystem.Run () {
        foreach (var i in _filter) {
            // entity that contains WeaponComponent.
            // Performance hint: use 'ref' prefixes for disable copying entity structure.
            ref var entity = ref _filter.Entities[i];

            // Components1 array will be automatically filled with instances of type "WeaponComponent".
            var weapon = _filter.Components1[i];
            weapon.Ammo = System.Math.Max (0, weapon.Ammo - 1);
        }
    }
}
```

All compatible entities will be stored at `filter.Entities` array.

> Important: `filter.Entities` array data should be used with `ref` prefixes for performance boost (no copying, marshal by reference).

> Important: `filter.Entities`, `filter.Components1`, `filter.Components2`, etc - can't be iterated with foreach-loop directly, use foreach-loop over `filter`.

All components from filter `Include` constraint will be stored at `filter.Components1`, `filter.Components2`, etc - in same order as they were used in filter type declaration.

If auto-storing to `filter.ComponentsX` collections not required (for example, for flag-based components without data), `EcsIgnoreInFilter` attribute can be used for decrease memory usage and increase performance:
```csharp
class Component1 { }

[EcsIgnoreInFilter]
class Component2 { }

[EcsInject]
class TestSystem : IEcsSystem {
    EcsFilter<Component1, Component2> _filter;

    public Test() {
        foreach (var i in _filter) {
            // its valid code.
            var component1 = _filter.Components1[i];

            // its invalid code due to _filter.Components2 is null for memory / performance reasons.
            var component2 = _filter.Components2[i];
        }
    }
}
```

> Important: Any filter supports up to 4 component types as "include" constraints and up to 2 component types as "exclude" constraints. Shorter constraints - better performance.

> Important: If you will try to use 2 filters with same components but in different order - you will get exception with detailed info about conflicted types, but only in `DEBUG` mode. In `RELEASE` mode all checks will be skipped.

## EcsWorld
Root level container for all entities / components, works like isolated environment.
> Important: Do not forget to call `EcsWorld.Dispose` method when instance will not be used anymore.

## EcsSystems
Group of systems to process `EcsWorld` instance:
```csharp
class Startup : MonoBehaviour {
    EcsSystems _systems;

    void OnEnable() {
        // create ecs environment.
        var world = new EcsWorld ();
        _systems = new EcsSystems(world)
            .Add (new WeaponSystem ());
        _systems.Initialize ();
    }
    
    void Update() {
        // process all dependent systems.
        _systems.Run ();
        // optional behaviour for one-frame components.
        _world.RemoveOneFrameComponents ();
    }

    void OnDisable() {
        // destroy systems logical group.
        _systems.Dispose ();
        _systems = null;
        // destroy world.
        _world.Dispose ();
        _world = null;
    }
}
```
> Important: Do not forget to call `EcsSystems.Dispose` method when instance will not be used anymore.

`EcsSystems` instance can be used as nested system (any types of `IEcsPreInitSystem`, `IEcsInitSystem` or `IEcsRunSystem` behaviours are supported):
```csharp
// initialization.
var nestedSystems = new EcsSystems (_world)
    .Add (new NestedSystem ());
// dont call nestedSystems.Initialize() here, rootSystems will do it automatically.

var rootSystems = new EcsSystems (_world)
    .Add (nestedSystems);
rootSystems.Initialize();

// update loop.
// dont call nestedSystems.Run() here, rootSystems will do it automatically.
rootSystems.Run();

// destroying.
// dont call nestedSystems.Dispose() here, rootSystems will do it automatically.
rootSystems.Dispose();
```

# Examples
[Snake game](https://github.com/Leopotam/ecs-snake)

[Pacman game](https://github.com/SH42913/pacmanecs)

# Extensions

[Reactive filters / systems](https://github.com/Leopotam/ecs-reactive)

[Multithreading support](https://github.com/Leopotam/ecs-threads)

[Unity integration](https://github.com/Leopotam/ecs-unityintegration)

[Unity uGui event bindings](https://github.com/Leopotam/ecs-ui)

[Engine independent types](https://github.com/Leopotam/ecs-types)

# License
The software released under the terms of the [MIT license](./LICENSE). Enjoy.

# Donate
Its free opensource software, but you can buy me a coffee:

<a href="https://www.buymeacoffee.com/leopotam" target="_blank"><img src="https://www.buymeacoffee.com/assets/img/custom_images/yellow_img.png" alt="Buy Me A Coffee" style="height: auto !important;width: auto !important;" ></a>

# FAQ

### My project complex enough, I need more than 256 components. How I can do it?

There are no components limit, but for performance / memory usage reason better to keep amount of components on each entity less or equals 8.

### I want to create alot of new entities with new components on start, how to speed up this process?

In this case custom component creator with predefined capacity can be used (for speed up 2x or more):

```csharp
class MyComponent { }

class Startup : Monobehaviour {
    EcsSystems _systems;

    void OnEnable() {
        var world = new MyWorld (_sharedData);
        
        EcsComponentPool<MyComponent>.Instance.SetCapacity (100000);
        EcsComponentPool<MyComponent>.Instance.SetCreator (() => new MyComponent());
        
        _systems = new EcsSystems(world)
            .Add (MySystem());
        _systems.Initialize();
    }
}
```

### I want to shrink allocated caches for my components, how I can do it?

In this case `EcsComponentPool<T>.Instance.Shrink` method can be used:

```csharp
class MyComponent1 { }
class MyComponent2 { }

class ShrinkComponents : Monobehaviour {
    void OnEnable() {
        EcsComponentPool<MyComponent1>.Instance.Shrink ();
        EcsComponentPool<MyComponent2>.Instance.Shrink ();
    }
}
```

### I want to process one system at MonoBehaviour.Update() and another - at MonoBehaviour.FixedUpdate(). How I can do it?

For splitting systems by `MonoBehaviour`-method multiple `EcsSystems` logical groups should be used:
```csharp
EcsSystems _update;
EcsSystems _fixedUpdate;

void OnEnable() {
    var world = new EcsWorld();
    _update = new EcsSystems(world).Add(new UpdateSystem());
    _fixedUpdate = new EcsSystems(world).Add(new FixedUpdateSystem());
}

void Update() {
    _update.Run();
}

void FixedUpdate() {
    _fixedUpdate.Run();
}
```

### I want to add new or get already exist component on entity, but `GetComponent` with checking result - so boring. How it can be simplified?

There is `EnsureComponent` method at `EcsWorld`, next 2 code blocks are identical:
```csharp
var c1 = _world.GetComponent<C1>(id);
if (c1 == null) {
    c1 = _world.AddComponent<C1>(id);
}
...
bool isNew;
var c1 = _world.EnsureComponent<C1>(id, out isNew);
```

### I want to add reaction on add / remove entity / components in `EcsWorld`. How I can do it?

It will add performance penalty and should be avoided. Anyway, **LEOECS_ENABLE_WORLD_EVENTS** preprocessor define can be used for this:
```csharp
class MyListener : IEcsWorldEventListener {
    public void OnEntityCreated (int entity) { }
    public void OnEntityRemoved (int entity) { }
    public void OnComponentAdded (int entity, object component) { }
    public void OnComponentRemoved (int entity, object component) { }
    public void OnWorldDestroyed (EcsWorld world) { }
}

// at init code.
var listener = new MyListener();
_world.AddEventListener(listener);

// at destroy code.
_world.RemoveEventListener(listener);
```

### I do not need dependency injection through `Reflection` (I heard, it's very slooooow! / I want to use my own way to inject). How I can do it?

Builtin Reflection-based DI can be removed with **LEOECS_DISABLE_INJECT** preprocessor define:
* No `EcsInject` attribute.
* No automatic injection for `EcsWorld` and `EcsFilter<T>` fields.
* Less code size.

`EcsWorld` should be injected somehow (for example, through constructor of system), `EcsFilter<T>` data can be requested through `EcsWorld.GetFilter<T>` method.

### I like how dependency injection works, but i want to skip some fields from initialization. How I can do it?

You can use `[EcsIgnoreinject]` attribute on any field of system:
```csharp
...
// will be injected.
EcsFilter<C1> _filter1 = null;

// will be skipped.
[EcsIgnoreInject]
EcsFilter<C2> _filter2 = null;
```

### I do not like foreach-loops, I know that for-loops are faster. How I can use it?

Current implementation of foreach-loop fast enough (custom enumerator, no memory allocation), small performance differences can be found on 10k items and more. Current version not support for-loop iterations anymore.

### I copy&paste my reset components code again and again. How I can do it in other manner?

If you want to simplify your code and keep reset-code in one place, you can use `IEcsAutoResetComponent` interface for components:
```csharp
class MyComponent : IEcsAutoResetComponent {
    public object LinkToAnotherComponent;

    void IEcsAutoResetComponent.Reset() {
        // Cleanup all marshal-by-reference fields here.
        LinkToAnotherComponent = null;
    }
}
```
This method will be automatically called after component removing from entity and before recycling to component pool.

### I want to add component to entity or get exists one, how I can simplify sequence of "var c = GetComponent(); if (c == null) {c = AddComponent()}"?

This sequence
```csharp
var c = _world.GetComponent<C1>(entityId);
if (c == null) {
    c = _world.AddComponent<C1>(entityId);
    // optional - init new instance.
}
```
Can be replaced with
```csharp
bool isNew;
var c = _world.EnsureComponent<C1>(entityId, out isNew);
if (isNew) {
    // optional - init new instance.
}
```

### I use components as events that works only one frame, then remove it at last system in execution sequence. It's boring, how I can automate it?

If you want to remove one-frame components without additional custom code, you can use `EcsOneFrame` attribute:
```csharp
[EcsOneFrame]
class MyComponent { }
```
> Important: Do not forget to call `EcsWorld.RemoveOneFrameComponents` method once after all `EcsSystems.Run` calls.

> Important: Do not forget that if one-frame component contains `marshal-by-reference` typed fields - this component should implements `IEcsAutoResetComponent` interface.

### I used reactive systems and filter events before, but now I can't find them. How I can get it back?

You can implement them by yourself with `EcsFilter.AddListener` / `EcsFilter.RemoveListener` methods or use default implementation, that can be found at [separate repo](https://github.com/Leopotam/ecs-reactive).

### I used `EcsWorld.Active` static instance as singleton for `EcsWorld`, but now I can't find it. How I can get it back?

You should use any custom singleton / service locator implementation for sharing `EcsWorld` as before. For example, `Service<T>` class from [Globals support repo](https://github.com/Leopotam/globals) can be used.

### I need more than 4 components in filter, how i can do it?

Check `EcsFilter<Inc1, Inc2, Inc3, Inc4>` class and create new class with more components in same manner.
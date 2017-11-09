# Architecture overview

This page has been created to make contributors' lives easier.

The internal architecture of InversifyJS is heavily influenced by [Ninject](https://github.com/ninject/Ninject). Being influenced does not mean that both architectures are identical. In fact, both architectures are quite different since C# and JavaScript are really different programming languages. However, the terms used to describe some elements of the library and the phases of resolution process are not that different.

# The resolution process
InversifyJS performs **3 mandatory operations** before resolving a dependency: 

- **Annotation**
- **Planning**
- **Middleware (optional)**
- **Resolution**
- **Activation (optional)**

In some cases there will be some **additional operations (middleware & activation)**.

The project's folder architecture groups the components that are related with one of the phases of the resolution process:
```
├── src
│   ├── annotation
│   │   ├── context.ts
│   │   ├── metadata.ts
│   │   ├── queryable_string.ts
│   │   ├── request.ts
│   │   └── target.ts
│   ├── bindings
│   │   ├── binding.ts
│   │   ├── binding_count.ts
│   │   └── binding_scope.ts
│   ├── constants
│   │   ├── error_msgs.ts
│   │   └── metadata_keys.ts
│   ├── decorators
│   │   ├── decorator_utils.ts
│   │   ├── inject.ts
│   │   ├── named.ts
│   │   ├── target_name.ts
│   │   └── tagged.ts
│   ├── interfaces
│   │   └── ...
│   ├── inversify.ts
│   ├── container
│   │   ├── container.ts
│   │   ├── key_value_pair.ts
│   │   ├── lookup.ts
│   │   ├── plan.ts
│   │   ├── planner.ts
│   │   └── resolver.ts
│   └── middleware
│       └── logger.ts
```

### Annotation Phase
The annotation phase reads the metadata generated by the decorators and transform it into a series of instances of the Request and Target classes. This Request and Target instances are then used to generate a resolution plan during the Planing Phase. 

### Planning Phase
When we invoke the following:
```js
var obj = container.get<SomeType>("SomeType");
```
We start a new resolution, which means that the container will create a new resolution context. The resolution context contains a reference to the container and a reference to a Plan.

The Plan is generated by an instance of the Planner class. The Plan contains a reference to the context and a reference to the a (root) Request. A requests represents a dependency which will be injected into a Target. 

Let's take a look to the following code snippet:

```js
@injectable()
class FooBar implements FooBarInterface {
  public foo : FooInterface;
  public bar : BarInterface;
  public log() {
    console.log("foobar");
  }
  constructor(
    @inject("FooInterface") foo : FooInterface, 
    @inject("BarInterface") bar : BarInterface
  ) {
    this.foo = foo;
    this.bar = bar;
  }
}

var foobar = container.get<FooBarInterface>("FooBarInterface");
```
The preceding code snippet will generate a new Context and a new Plan. The plan will contain a RootRequest with a null Target and two child Requests:
- The first child requests represents a dependency on `FooInterface` and its target is the constructor argument named `foo`.
- The second child requests represents a dependency on `BarInterface` and its target is the constructor argument named `bar`.

The following diagram can help you to understand the shape of a resolution Context and how all its internal parts reference each other:

![](http://i.imgur.com/NSSbPWy.png)

### Middleware Phase
If we have configured some Middleware it will be executed just before the resolution Phase takes place. Middleware can be used to develop some browser extensions that will allow us to display the resolution plan using some data visualisation tool like D3.js. This kind of tools will help developers to identify problems during the development process.

One example of middleware is the [inversify-logger-middleware](https://github.com/inversify/inversify-logger-middleware) which can be used to display the resolution plan and the time that took to create and resolve it in console:

![](http://i.imgur.com/iFAogro.png)

### Resolution Phase
The Plan is passed to an instance of the Resolver class. The Resolver will then proceed to resolve each of the dependencies in the Request tree starting with the leafs and finishing with the root request.

The resolution process can be executed synchronously or asynchronously which can help to boost performance.

### Activation Phase
Activation takes place after a dependency has been resolved. Just before it is added to the cache (if singleton) and injected. It is possible to add an event handler that is invoked just before activation is completed. This feature allows developers to do things like injecting a proxy to intercept all the calls to the properties or methods of that object.
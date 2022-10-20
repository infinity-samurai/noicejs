# Getting Started

This guide briefly describes the benefits of dependency injection, then details how to begin using
noicejs with abundant examples.

## Contents

- [Getting Started](#getting-started)
  - [Contents](#contents)
  - [Why Inject Dependencies](#why-inject-dependencies)
  - [Setup](#setup)
  - [Creating a Container](#creating-a-container)
  - [Registering a Module](#registering-a-module)
  - [Extending a Module](#extending-a-module)
  - [Requiring a Dependency](#requiring-a-dependency)
  - [Providing a Dependency](#providing-a-dependency)
  - [Populating Properties](#populating-properties)
  - [Additional Examples](#additional-examples)

## Why Inject Dependencies

This is a very brief explanation of what dependency injection does and how it may be useful. There is a tremendous
amount of existing literature that goes into far more detail on the topic. If you are looking for more information,
some good places to start are:

- [Clean Code Talks - Dependency Injection](http://misko.hevery.com/2008/11/11/clean-code-talks-dependency-injection/)
- [Google Testing Blog: When To Use Dependency Injection](https://testing.googleblog.com/2009/01/when-to-use-dependency-injection.html)
- [Software Engineering Stack Exchange - Why Should I Use Dependency Injection?](https://softwareengineering.stackexchange.com/a/381410)
- [Guice - Motivation](https://github.com/google/guice/wiki/Motivation)
- [Wikipedia - Dependency Injection](https://en.wikipedia.org/wiki/Dependency_injection#Overview)

The resolution performed by the container removes the need for the consumer to be aware of, or pass, dependencies
to the service. The service can change dependencies without any code changes to the consumer.

Compare the following examples:

```typescript
import { LocalFilesystem } from './local';

class Foo {
  constructor(options) {
    const {
      filesystem,
    } = options;

    this.data = filesystem.read(options.path);
  }
}

function main() {
  const foo = new Foo({
    filesystem: LocalFilesystem, /* must be passed here */
  });
}

function later() {
  const bar = new Foo({
    filesystem: LocalFilesystem, /* must be passed again */
  });
}
```

Every time a `new Foo` is created, the correct filesystem must be passed to it. If the filesystem changes,
due to a config setting or environment variable, each instance must be replaced with a function call or variable
to access that configuration, repeatedly invoking that logic in each location.

With dependency injection, implementations with a similar theme can be grouped in a module, and the module
replaced as a whole.

```typescript
import { Cache, Filesystem } from './interfaces';
import { LocalModule } from './local';
import { NetworkModule } from './network';

@Inject(Cache, Filesystem)
class Foo {
  protected readonly cache: Cache;
  protected readonly filesystem: Filesystem;

  constructor(options) {
    this.cache = options.cache;
    this.filesystem = options.filesystem;
  }

  get(path: string) {
    return options.cache.get(path, () => options.filesystem.get(path));
  }
}

function module() {
  if (process.env['DEBUG'] === 'TRUE') {
    return new LocalModule();
  } else {
    return new NetworkModule();
  }
}

function main() {
  const container = Container.from(module());
  await container.configure();

  const foo = await container.create(Foo); /* cache and filesystem are found and injected by container */
}
```

Neither `Foo` nor `main` knows what kind of `Cache` the `LocalModule` provides or what kind of `Filesystem` the
`NetworkModule` provides, nor do they care. The container resolves and injects the correct implementation as
needed and the application logic moves to providing the correct modules for the current configuration.

More technically, dependency injection inverts the chain of control between a class and its dependencies, by
transferring responsibility for providing those dependencies to a container which discovers and resolves dependencies
before injecting them into the original class.

## Setup

Install noicejs with your package manager of choice:

```shell
> yarn add -D noicejs
```

If you will be bundling the output with a tool like Rollup or Webpack, you can install noicejs as a devDependency.

## Creating a Container

The container is the central interface of noicejs. Containers are responsible for creating new objects and resolving
dependencies from their modules.

Most containers are created from a list of modules, but they may also be created empty:

```typescript
import { Container } from 'noicejs';

async function main() {
  const container = Container.from();
  await container.configure();
}
```

While an empty container will not be have any dependencies to inject, it can still create new instances of
classes without dependencies:

```typescript
import { Container } from 'noicejs';

class Foo { }

async function main() {
  const container = Container.from();
  await container.configure();

  const foo = await container.create(Foo);
  console.log(foo instanceof Foo);
  // prints: true
}
```

However, this container will not be able to create instances of a class that does have dependencies:

```typescript
import { Container, Inject } from 'noicejs';

@Inject('foo')
class Bar { }

async function main() {
  const container = Container.from();
  await container.configure();

  try {
    const bar = await container.create(Bar);
  } catch (err) {
    console.error(err);
    // prints: TODO
  }
}
```

## Registering a Module

The container does not provide any dependencies of its own, instead resolving them from an ordered
list of modules.

While most modules are subclasses that `extend Module`, a quick start module is provided in the
library: the `MapModule`. The `MapModule` constructor takes a dictionary or `Map` of named dependencies,
binds them as constructors or instances, and then provides them as usual.

The `MapModule` is meant to provide a quick start and does not support all of the dependency types
that the container does, especially factory methods. Please see [the `@Provides` decorator](#providing-a-dependency)
for further details.

Registering a module that provides some named dependency `'foo'` will allow the container to resolve it:

```typescript
import { Container, Inject, MapModule } from 'noicejs';

@Inject('foo')
class Bar {
  constructor(options) {
    console.log(options.foo);
  }
}

async function main() {
  const container = Container.from(new MapModule({
    providers: {
      foo: 3,
    },
  }));
  await container.configure();

  const bar = await container.create(Bar);
  // prints: 3
}
```

While many of these examples use named dependencies for semantic meaning, modules can provide a few
different kinds of contracts:

- constructors
- string names
- symbol names

Symbols are the most unique and can be used in a relatively type-safe manner by declaring an interface
of available symbols:

```typescript
import { BaseOptions, Container, Inject } from 'noicejs';
import { Cache } from './interfaces';

const INJECT_CACHE = Symbol('inject-cache');
interface InjectedOptions extends BaseOptions {
  [INJECT_CACHE]?: Cache; // these may not be injected and so should be optional
}

@Inject(INJECT_CACHE)
class Foo {
  constructor(options: InjectedOptions) {
    this.cache = mustExist(options[INJECT_CACHE]); // throw if required dependencies are missing
  }
}
```

## Extending a Module

In order to resolve a dependency, one of the modules within the container needs to provide it. Modules represent
a small set of dependencies with a common theme.

Modules bind most of their dependencies during configuration:

```typescript
import { Module } from 'noicejs';

class NetworkThing {}

class NetworkModule extends Module {
  public async configure(options) {
    await super.configure(options);

    this.bind('foo').toConstructor(NetworkThing);
  }
}
```

Modules must be instantiated before the container is created:

```typescript
import { Container } from 'noicejs';
import { NetworkModule, NetworkThing } from 'the-last-example';

async function main() {
  const container = Container.from(new NetworkModule());
  await container.configure();

  const foo = await container.create('foo');
  console.log(foo instanceOf NetworkThing);
}
```

Modules are not useful on their own and can be passed directly into the container's control.

## Requiring a Dependency

Now that a module exists to provide the contract `foo`, some other class needs to depend on
having an instance of `foo`:

```typescript
import { Container, Inject } from 'noicejs';
import { NetworkModule, NetworkThing } from 'the-module-example';

@Inject('foo')
class Bar {
  constructor(options) {
    console.log(options.container, options.foo);
  }
}

async function main() {
  const container = Container.from(new NetworkModule());
  await container.configure();

  const bar = await container.create(Bar);
  // prints: Container {}, NetworkThing {}
}
```

The decorator is optional and classes may be annotated manually instead, dependencies are a
constructor property using a particular `symbol`.

## Providing a Dependency

Modules can provide a few different kinds of dependencies:

- constructor
- factory
- instance

While constructors are created when needed and factories are invoked, instances are simply
returned as they were provided and are the closest equivalent to a traditional singleton that
can exist within DI.

```typescript
import { Container, Inject, Module } from 'noicejs';

class RandomModule extends Module {
  public async configure(options) {
    await super.configure(options);

    this.bind('foo').toFactory(async () => Math.random());
    this.bind('bar').toInstance(3);
  }
}

@Inject('foo', 'bar')
class FooBar {
  constructor(options) {
    console.log(options.foo, options.bar);
  }
}

async function main() {
  const container = Container.from(new RandomModule());
  await container.configure();

  const foobar = await container.create(FooBar);
  // prints: N, 3
}
```

When the factory implementation fits better in a method, that method can be decorated as
a provider:

```typescript
import { Container, Inject, Module } from 'noicejs';

class RandomModule extends Module {
  public async configure(options) {
    await super.configure(options);

    this.bind('bar').toInstance(3);
  }

  @Provides('foo')
  public async createFoo(options) {
    return Math.random();
  }
}
```

This module's provider method is equivalent to the previous `toFactory` binding. As before, the
decorator is optional and attaches metadata to the method function with a symbol. There are not
decorators for constructor or instance bindings, since they fit well in the fluent form.

The `@Provides` decorator can be used to extend the `MapModule` with factories. This module
is equivalent to the previous example's `RandomModule`:

```typescript
import { Container, Inject, Module } from 'noicejs';

class RandomModule extends MapModule {
  constructor() {
    super({
      providers: {
        bar: 3,
      },
    });
  }

  @Provides('foo')
  public async createFoo(options) {
    return Math.random();
  }
}
```

## Populating Properties

While injecting constructor parameters covers many common use-cases, there are many cases where you may want to
assign those parameters to properties within the newly-constructed object. In particular, this pattern is common
enough to have a helper:

```typescript
import { Container, Inject, Module } from 'noicejs';

@Inject(INJECT_COUNTER, INJECT_EVENT, INJECT_LOGGER, INJECT_RANDOM)
class FooService {
  constructor(options: InjectedOptions) {
    this.container = options.container;
    this.counter = mustExist(options[INJECT_COUNTER]);
    this.event = mustExist(options[INJECT_EVENT]);
    this.random = mustExist(options[INJECT_RANDOM]);

    // some other logic, maybe using those properties
    this.logger = makeServiceLogger(options[INJECT_LOGGER], this);
  }
}
```

This can be simplified with the `@Field` decorator and `injectFields` helper. Fields are not automatically added to
the list of dependencies with `@Inject`, although that may be changed in a future version. A more specific decorator
is provided that _does_ merge the lists, `@InjectWithFields`.

An equivalent to the previous example using those decorators would look more like:

```typescript
import { Container, Field, InjectWithFields, Module } from 'noicejs';

@InjectWithFields(INJECT_LOGGER) // the logger is not a simple field, it uses some additional computation
class FooService {
  @Field('container')
  public container: Container;

  @Field(INJECT_COUNTER)
  public counter: Counter;

  @Field(INJECT_EVENT)
  public event: EventBus;

  @Field(INJECT_RANDOM)
  public random: RandomSource;

  constructor(options: InjectedOptions) {
    injectFields(this, options);

    // some other logic, maybe using those properties
    this.logger = makeServiceLogger(options[INJECT_LOGGER], this);
  }
}
```

## Additional Examples

For additional examples, please see:

- [the `docs/examples` directory](examples)
- [the unit tests](../test)
- [this text adventure engine](https://github.com/ssube/textual-engine)

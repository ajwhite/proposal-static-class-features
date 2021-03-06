# Static class features

Champion: Daniel Ehrenberg

Stage 2

This proposal tracks the "static" (i.e., of the constructor) aspects of the [class fields](http://github.com/tc39/proposal-class-fields) and [private methods](https://github.com/tc39/proposal-private-methods) proposals. In the November 2017 TC39 meeting, the static dimensions of these proposals were demoted to Stage 2, to be broken out into a separate proposal while the instance dimensions remain at Stage 3. This repository is created to explore the aspects which are still at Stage 2, to determine how to proceed to Stage 3 with some version of them.

The current proposal is similar to the previous semantics, with one modification:
1. **Public and private static fields are initialized only once in the class where they are defined, not additionally in subclasses**. [see below](https://github.com/tc39/proposal-static-class-features/blob/master/README.md#static-field-initialization)
1. **Private static methods can be called with the superclass or any subclasses as the receiver; otherwise, TypeError** (NEW)
1. **Private static fields can only be referenced with the class as the receiver; TypeError when used with anything else**. [see below](https://github.com/tc39/proposal-static-class-features/blob/master/README.md#static-private-access-on-subclasses)

Note that private fields and methods are only accessible within the class body where they are defined; the new change here only pertains to what happens when methods in that superclass body is invoked with a receiver which is a subclass constructor.

## Static public fields

Like static public methods, static public fields take a common idiom which was possible to write without class syntax and make it more ergonomic, have more declarative-feeling syntax (although the semantics are quite imperative), and allow free ordering with other class elements.

### Semantics

Define an own property on the constructor which is set to the value of the initializer expression. The initializer is evaluated in a scope where the binding of the class is available--unlike in computed property names, the class can be referred to from inside initializers without leading to a ReferenceError.

Alternate semantics are discussed in ["Initializing fields on subclasses"](#initializing-fields-on-subclasses). However, they are not proposed here, for reasons detailed in that section. These semantics are justified in more detail in ["Static field initialization"](#static-field-initialization).

### Use case

```js
class CustomDate {
  // ...
  static epoch = new CustomDate(0)
}
```

JavaScript programmers can currently accomplish the same thing through this syntax:

```js
class CustomDate {
  // ...
}
CustomDate.epoch = new CustomDate(0);
```

Declaring static properties in the class body is hoped to be cleaner and doing a better job of meeting programmer expectations of what classes should be for. It's claimed that the latter workaround is a common idiom, and it would be a nice convenience for programmers if the property declaration could be lifted into the class body, matching how methods are placed there.

### Why only initialize static fields once

Kevin Gibbons [raised a concern](https://github.com/tc39/proposal-class-fields/issues/43#issuecomment-340517955) that JavaScript programmers may not be used to having objects with mutated properties which are exposed on the prototype chain, the way that static fields are inherited and may be overwritten. Some reasons why this is not that bad:
- Many JS classes add a property to a constructor after a class definition. A subclass of Number, for example, would make this issue observable.
- Lots of current educational materials, e.g., by Kyle Simpson and Eric Elliott, explain directly how prototypical inheritance of data properties in JS works to newer programmers.
- This proposal is more conservative and going with the grain of JS by not adding a new time when code runs for subclassing, preserving identities as you'd expect, etc.

### Semantics in an edge case with Set and inheritance

Example of how these semantics work out with subclassing (this is not a recommended use of constructors as stateful objects, but it shows the semantic edge cases):

```js
static Counter {
  static count = 0;
  static inc() { this.count++; }
}
class SubCounter extends Counter { }

Counter.hasOwnProperty("count");  // true
SubCounter.hasOwnProperty("count");  // false

Counter.count; // 0, own property
SubCounter.count; // 0, inherited

Counter.inc();  // undefined
Counter.count;  // 1, own property
SubCounter.count;  // 1, inherited

// ++ will read up the prototype chain and write an own property
SubCounter.inc();

Counter.hasOwnProperty("count");  // true
SubCounter.hasOwnProperty("count");  // true

Counter.count;  // 1, own property
SubCounter.count;  // 2, own property

Counter.inc(); Counter.inc();
Counter.count;  // 3, own property
SubCounter.count;  // 2, own property
```

See [Initializing fields on subclasses](#initializing-fields-on-subclasses) for more details on an alternative proposed semantics and why it was not selected.

## Static private methods and accessors

### Semantics

Like [instance private methods](https://github.com/tc39/proposal-private-methods/blob/master/DETAILS.md), static private methods are conceptually non-writable private fields on objects whose value is the method. There is no prototype chain access in private fields or methods; instead, the methods are installed in each object that would be able to access the method through prototype chain lookups.

In the case of instance private methods, this is instances of the class and subclasses; for static private methods, this is the constructor and subclass constructors. Invoking a private method on an object where it is missing causes a TypeError.

In specification terms, when a subclass is created which extends another class, the non-writable and accessor private 'fields' (namely, the private methods and accessors) of the super class are copied to the subclass. This makes it so that the private methods and accessors can be called on the subclass.

### Use case

Static private methods can be useful whenever there is shared behavior to extract into a function which uses private fields, but which doesn't work cleanly as an instance method. For example, multiple factory static methods may share part of their implementation, including parts which run both before and after construction of the instance. See [#4](https://github.com/tc39/proposal-static-class-features/issues/4) for more context about the following example.

```js
export const registry = new JSDOMRegistry();

export class JSDOM {
  #createdBy;
  
  #registerWithRegistry() {
    // ... elided ...
  }
 
  async static fromURL(url, options = {}) {
    url = normalizeFromURLOptions(url, options);
    
    const body = await getBodyFromURL(url);
    return JSDOM.#finalizeFactoryCreated(body, options, "fromURL");
  }
  
  static async fromFile(filename, options = {}) {
    const body = await getBodyFromFilename(filename);
    return JSDOM.#finalizeFactoryCreated(body, options, "fromFile");
  }
  
  static #finalizeFactoryCreated(body, options, factoryName) {
    normalizeOptions(options);
    let jsdom = new JSDOM(body, options):
    jsdom.#createdBy = factoryName;
    jsdom.#registerWithRegistry(registry);
    return jsdom;
  }
}
```

In [Issue #1](https://github.com/tc39/proposal-static-class-features/issues/1), there is further discussion about whether this feature is well-motivated. In particular, static private methods can typically be replaced by either lexically scoped function declarations outside the class declaration, or by private instance methods. However, the current proposal is to include them, due to the use cases in [#4](https://github.com/tc39/proposal-static-class-features/issues/4).

### Subclassing use case

As described above, static private methods may be invoked with subclass instances as the receiver. Note that this does *not* mean that subclass bodies may call private methods from superclasses--the methods are private, not protected, and it would be a SyntaxError to call the method from a place where it's not syntactically present.

In the below example, a public static method `from` is refactored, using a private static method. This private static method has an extension point which can be overridden by subclasses, which is that it calls the `of` method. All of this is in service of factory functions for creating new instances. If the private method `#from` were only callable with `MyArray` as the receiver, a `TypeError` would result.

```js
class MyArray {
  static #from(obj) {
    this.of.apply(...obj);
  }
  static from(obj) {
    // This function gets large and complex, with parts shared with
    // other static methods, so an inner portion #from is factored out
    return this.#from(obj);
  }
}

class SubMyArray extends MyArray {
  static of(...args) {
    let obj = new SubMyArray();
    let i = 0;
    for (let arg of args) {
      obj[i] = arg;
      i++;
    }
  }
}

let subarr = MySubArray.from([1, 2, 3]);
```

## Static private fields

### Semantics

Unlike static private methods, static private fields are private fields just of the constructor. If a static private field is read with the receiver being a subclass, a TypeError will be triggered.

As with static public fields, the initializer is evaluated in a scope where the binding of the class is available--unlike in computed property names, the class can be referred to from inside initializers without leading to a ReferenceError. As described in [Why only initialize static fields once](#why-only-initialize-static-fields-once), the initializer is evaluated only once.

### Use case

```js
class ColorFinder {
  static #red = "#ff0000";
  static #blue = "#00ff00";
  static #green = "#0000ff";
  
  static colorName(name) {
    switch (name) {
      case "red": return ColorFinder.#red;
      case "blue": return ColorFinder.#blue;
      case "green": return ColorFinder.#green;
      default: throw new RangeError("unknown color");
    }
  }
  
  // Somehow use colorName
}
```

In [Issue #1](https://github.com/tc39/proposal-static-class-features/issues/1), there is further discussion about whether this feature is well-motivated. In particular, static private fields can typically be subsumed by lexically scoped variables outside the class declaration. Static private fields are motivated mostly by a desire to allow free ordering of things inside classes and consistency with other class features. It's also possible that some initialzer expressions may want to refer to private names declared inside the class, e.g., in [this gist](https://gist.github.com/littledan/19c09a09d2afe7558cdfd6fdae18f956).

The behavior of private static methods have, in copying to subclass constructors, does not apply to private static fields. This is due to the complexity and lack of a good option for semantics which are analogous to static public fields, as explained below.

### TypeError case

Justin Ridgewell [expressed concern](https://github.com/tc39/proposal-class-fields/issues/43) about the TypeError that results from static private field access from subclasses. Here's an example of that TypeError, which occurs when code ported from the above static public fields example is switched to private fields:

```js
static Counter {
  static #count = 0;
  static inc() { this.#count++; }
  static get count() { return this.#count; }
}
class SubCounter extends Counter { }

Counter.inc();  // undefined
Counter.count;  // 1

SubCounter.inc();  // TypeError

Counter.count;  // 1
SubCounter.count;  // TypeError
```

A TypeError is used here because no acceptable alternative would have the semantics which are analogous to static public fields. Some alternatives are discussed below.

#### Why this TypeError is not so bad

The above concern is hoped to be not too serious because:
- Programmers can avoid the issue by instead writing `ClassName.#field`. This phrasing should be easier to understand, anyway--no need to worry about what `this` refers to.
- It is not so bad to repeat the class name when accessing a private static field. When implementing a recursive function, the name of the function needs to be repeated; this case is similar.
- It is statically known whether a private name refers to a static or instance-related class field. Therefore, implementations should be able to make helpful error messages for instance issues that say "TypeError: The private field #foo is only present on instances of ClassName, but it was accessed on an object which was not an instance", or, "TypeError: The static private method #bar is only present on the class ClassName; but it was accessed on a subclass or other object", etc.
- Linters could flag uses of `foo.#field` where `foo` is not the name of the class and `#field` is a static private field, and discourage its use that way.
- Type systems which want to be slightly more accepting could trigger an error on any class with a class which is subclassed and has public method which uses a private method or field without the class's name being the receiver. In the case of TypeScript, [this is already not polymorphic](https://github.com/Microsoft/TypeScript/issues/5863) so it would already flag instances of `this.#field` for a private static method call within a public static method.
- Beginners can just learn the rule (helped by linters, type systems and error messages) to refer to the class when calling static private methods. Advanced users can learn the simple mental model that corresponds to the specification: private things are represented by a WeakMap, and public things by ordinary properties. When it's by the instance, the WeakMap has a key for each instance; when it's static, the WeakMap has a single key, for the constructor.
- A guaranteed TypeError when the code runs on the subclass is a relatively easy to debug failure mode. This is how JS responds, at a high level, when you access an undefined variable as well, or read a property on undefined. A silent other interpretation would be the real footgun.
- TC39 has worked hard on other guaranteed exceptions, such as the "temporal dead zone"--a ReferenceError from reading `let`- or `const`-declared variables whose declaration has not yet been reached. Here, a runtime error was considered very useful for programmers and a way of preventing a "footgun". In this case, we're getting such a guaranteed error.

## Programmer mental model

These proposed semantics are designed to be learnable by beginners as well as understandable at a deep level by experts, without any mismatch. The semantics here aren't just "something that works well in the spec", but rather a design which remains intuitive and consistent through multiple lenses.

### Beginner, simplified semantics

To make something inaccessible from outside the class, put a `#` at the beginning of the name. When a field or method has `static`, that means it is on the class, not on instances. To access static class elements, use the syntax `ClassName.element` or `ClassName.#element`, depending on whether the name contains a `#`. Note that the latter case is available only inside the class declaration. If you make a mistake in accessing private class elements, JS will throw `TypeError` when the code is run to make it easier to catch the programming error. To catch errors earlier in development, try using a linter or type system.

### Expert, complete semantics

Public and private class elements are analogous to each other, with the difference being that public class elements are implemented as properties on the object, and private class elements are like a WeakMap, or internal slot. The rest of this section is written in terms of a WeakMap, but could be phrased equally in terms of internal slots.

Public class elements are defined with \[\[DefineOwnProperty]] when they are added to the relevant object, and private class elements have a corresponding WeakMap, mapping objects to their values, which an object is added to when the element is defined for the object. For instance fields, this relevant object is the instance; for static fields, this relevant object is the constructor.

Private methods are a special case--they are made to be inherited, either from the prototype or a superclass, but the prototype chain is not accessed when a private method is read. Instead of inheriting, private methods are added as own fields on all of the relevant objects, whether it's all instances inheriting from the class for ordinary methods, or all subclasses of a constructor for a static method.

Static fields are syntax sugar for defining a property or private field of a constructor after the rest of the class definition runs. As such, they are able to access or instantiate the class in their initializer. Static fields are installed on just the particular constructor, whether as an own property or a key in a singleton WeakMap.

The above semantics are orthogonal and consistent among private vs public, static vs instance, and field vs method, making it easy for advanced programmers to do appropriate refactorings between features.

## "Back pocket" alternatives

These alternatives are not currently proposed by the champion, but are considered possibly feasible alternatives.

### Omit static private fields

The most realistic "hazard" cases for private access on subclasses was in the use of private static methods from subclasses. Because private methods are immutable, they don't suffer the same mismatch as static fields, where semantics need to be defined when they are overwritten (which is ultimately an issue making it difficult to support private static fields on subclasses). For this reason, private methods could just be installed on subclass constructors and made callable with those as a receiver.

You could argue that the different set of "relevant objects" for private static methods is somewhat analogous to the way that the set of objects is different for private instance methods vs public instance methods (the set of instances vs the prototype, or the constructor and subclasses vs the constructor). In effect, these semantics provide the equivalent of a "private prototype chain walk" for the state of the prototype chain when the instance was constructed. This alternative would meet many of the presented use cases and avoid the "hazard" semantics, but it is a bit ad-hoc and doesn't explain why static private fields are omitted.

### Allow static private fields to be read from subclasses

If we want to get static private fields to be as close as possible to static public fields when it comes to inheritance semantics: The shadowing property of writes to properties on the prototype chain is very weird and complicated, and wouldn't make sense to replicate; on the other hand, reads up the prototype chain are simple and intelligible. We could allow reads, but not writes, to take place from subclasses. The reads would reflect writes that happen later to the superclass's static private field.

This alternative is currently not selected because it would be pretty complicated, and lead to a complicated mental model. It would still not make public and private static fields completely parallel, as writes from subclasses are not allowed.

Here's an example of code which would be enabled by this alternative (based on code by Justin Ridgewell):

```js
class Base {
  static #field = 'hello';

  static get() {
    return this.#field;
  }

  static set(value) {
    return this.#field = value;
  }
}

class Sub extends Base {}

Base.get(); // => 'hello'
Base.set('xyz');

Sub.get();  // => 'xyz' in this alternative, TypeError in the main proposal
Sub.set('abc');  // TypeError
```

### Restricting static private field access to `static.#foo`

Jordan Harband [suggested](https://github.com/tc39/proposal-class-fields/issues/43#issuecomment-328874041) that we make `static.` a syntax for referring to a property of the immediately enclosing class. If we add this feature, we could say that private static fields and methods may *only* be accessed that way. This has the disadvantage that, in the case of nested classes, there is no way to access the outer class's private static methods. However, as a mitigation, programmers may copy that method into another local variable before entering into another nested class, making it still available.

With this alternative, the above code sample could be written as follows:

```js
class Base {
  static #field = 'hello';

  static get() {
    return static.#field;
  }

  static set(value) {
    return static.#field = value;
  }
}

class Sub extends Base {}

Base.get(); // => 'hello'
Base.set('xyz');

Sub.get();  // => 'xyz'
Sub.set('abc');

Base.set();  // 'abc'
```

Here, `static` refers to the base class, not the subclass, so issues about access on subclass instances do not occur.

### Restricting static field private access to `ClassName.#foo`

The syntax for accessing private static fields and methods would be restricted to using the class name textually as the receiver. This would make it a SyntaxError to use `this.#privateMethod()` within a static method, for example. However, this would be a somewhat new kind of way to use scopes for early errors. Unlike var/let conflict early errors, this is much more speculative--the class name might actually be shadowed locally, which the early error would not catch, leading to a TypeError. In the championed semantics, such checks would be expected to be part of a linter or type system instead.

With this alternative, the above code sample could be written as follows:

```js
class Base {
  static #field = 'hello';

  static get() {
    return Base.#field;
  }

  static set(value) {
    return Base.#field = value;
  }
}

class Sub extends Base {}

Base.get(); // => 'hello'
Base.set('xyz');

Sub.get();  // => 'xyz'
Sub.set('abc');

Base.set();  // 'abc'
```

Here, explicit references the base class, not the subclass, so issues about access on subclass instances do not occur. A reference like `this.#field` would be an early error, helping to avoid errors by programmers.

## Alternate proposals not selected

Several alternatives have been discussed within TC39. This repository is not pursuing these ideas further. Some may be feasible, but are not selected by the champion for reasons described below.

### Initializing fields on subclasses

Kevin Gibbons has proposed that class fields have their initialisers re-run on subclasses. This would address the static private subclassing issue by adding those to subclasses as well, leading to no TypeError on use.

With this alternate, the initial counter example would have the following semantics:

```js
static Counter {
  static count = 0;
  static inc() { this.count++; }
}
class SubCounter extends Counter { }

Counter.hasOwnProperty("count");  // true
SubCounter.hasOwnProperty("count");  // true

Counter.count; // 0, own property
SubCounter.count; // 0, own property

Counter.inc();  // undefined
Counter.count;  // 1, own property
SubCounter.count;  // 0, own property

// ++ is just dealing iwth own properties the whole time
SubCounter.inc();

Counter.hasOwnProperty("count");  // true
SubCounter.hasOwnProperty("count");  // true

Counter.count;  // 1, own property
SubCounter.count;  // 1, own property

Counter.inc(); Counter.inc();
Counter.count;  // 3, own property
SubCounter.count;  // 1, own property
```

However, this proposal has certain disadvantages:
- Subclassing in JS has always been "declarative" so far, not actually executing anything from the superclass. It's really not clear this is the kind of hook we want to add to suddenly execute code here.
- The use cases that have been presented so far for expecting the reinitialization semantics seem to use subclassing as a sort of way to create a new stateful class (e.g., with its own cache or counter, or copy of some other object). These could be accomplished with a factory function which returns a class, without requiring that this is how static fields work in general for cases that are not asking for this behavior.

### Prototype chain walk for private fields and methods

Ron Buckton [has](https://github.com/tc39/proposal-class-fields/issues/43#issuecomment-348045445) [proposed](https://github.com/tc39/proposal-private-methods/issues/18) that private field and method access could reach up the prototype chain. There are two ways that this could be specified, both of which have significant issues:
- **Go up the normal prototype chain with \[\[GetPrototypeOf]]**. This would be observable and interceptible by proxies, violating a design goal of private fields that they not be manipulated that way.
- **Use a separate, parallel, immutable prototype chain**. This alternative would add extra complexity, and break the way that other use of classes consistently works together with runtime mutations in the prototype chain.

A positive aspect of Ron's proposal is that it preserves identical behavior with respect to subclassing as public static fields.

### Lexically scoped variables and function declarations in classes

Allen Wirfs-Brock has proposed lexically scoped functions and variables within class bodies. The syntax that Allen proposed was an ordinary function declaration or let/const declaration in the top level of a class body:

```js
class Foo {
  function bar() { }  // lexically scoped to the class body
  static method() { bar() }  // method on the constructor
}
```

However, that  proposed syntax may be unintuitive: keywords like function and let don't exactly scream, "unlike the other things in this list, this declaration is not exported and it's also not specific to the instance". For that reason, this repository proposes that static class features have a shape analogous to a field or method declaration. ([see discussion in bug](https://github.com/tc39/proposal-static-class-features/issues/4#issuecomment-354515761)).

### Omitting static private fields and methods

This alternative was previously proposed in this repository. The alternative is dispreferred due to [the use cases in Issue #4](https://github.com/tc39/proposal-static-class-features/issues/4).

A related alternative is moving ahead with static public fields while continuing development on static private fields. This repository is devoted to a full exploration of the problem space and is currently at Stage 2 which has been requested by committee members to proceed; it is not currently proposed to advance some parts and not others.

### Switching all fields to being based on accessors

The idea here is to avoid the subclassing overwriting hazard by changing the semantics of all field declarations: Rather than being own properties that are shadowed by a Set on a subclass, they are accessors (getter/setter pairs). In the case of instance fields, the accessor would read or write on the receiver. In the case of static fields, presumably, the read and write would happen on the superclass where they are defined, ignoring the receiver (otherwise, the TypeError issue from private static fields is then ported to public static fields as well, as the subclass constructor would not have its own value!). Private static fields would also follow this accessor pattern. In all cases, the getter would throw a TypeError when the receiver is an object which does not "have" the private field (e.g., its initializer has not run yet), an effectively new TDZ, which could reduce programmer errors.

Some downsides of this proposal:
- **Public fields would no longer be own properties**. This may be rather confusing for programmers, who may expect features like object spread to include public instance fields.
- **Does not help implementations and might hurt startup time (maybe)**. This idea was initially proposed as part of a concept for having 'static shape', which could provide more predictability for implementers and programmers. At least on the implementation side, however, there would either have to be checks on each access to see if the field was initialized, or the initialization state would have to show up in its "hidden class". Either way, there's no efficiency gain if the fields are "already there, just in TDZ". In some implementations, startup time could be even worse than with own properties, until the system learns to optimize out the accessors.
- **Loses the object model--we'd have to start again**. Data properties have an object model permitting non-writable, non-enumerable and non-configurable properties, including its use by `Object.freeze`. If we want to provide these sorts of capabilities to public fields, they would have to be built again separately.

For these reasons, the public fields proposal has been based on own properties.

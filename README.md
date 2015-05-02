# Summary

Decorators make it possible to annotate and modify classes and properties at
design time.

While ES5 object literals support arbitrary expressions in the value position,
ES6 classes only support literal functions as values. Decorators restore the
ability to run code at design time, while maintaining a declarative syntax.

# Detailed Design

A decorator is:

* an expression
* that evaluates to a constructor function
* that takes the target, name, and property descriptor as arguments
* that is able to intercept the engine while defining properties
* and due to its nature of being a constructor function, can make use of inheritance with super calls.

Consider a simple class definition:

```js
class Person {
  name() { return `${this.first} ${this.last}` }
}
```

Evaluating this class results in installing the `name` function onto
`Person.prototype`, roughly like this:

```js
Object.defineProperty(Person.prototype, 'name', {
  value: specifiedFunction,
  enumerable: false,
  configurable: true,
  writable: true
});
```

A decorator precedes the syntax that defines a property:

```js
class Person {
  @Readonly
  name() { return `${this.first} ${this.last}` }
}
```

Now, before installing the descriptor onto `Person.prototype`, the engine first
invokes the decorator:

```js
let descriptor = {
  value: specifiedFunction,
  enumerable: false,
  configurable: true,
  writable: true
};

new Readonly(Person.prototype, 'name', descriptor);
Object.defineProperty(Person.prototype, 'name', descriptor);
```

As the object references to the prototype and the descriptor can be directly modified in the decorator constructor, there's no need to create a chain of responsibility. The order of decorator constructor invocations is important.

The decorator has the same signature as `Object.defineProperty`, and has an
opportunity to intercede before the relevant `defineProperty` actually occurs.

A decorator that precedes syntactic getters and/or setters operates on the
accessor descriptor:

```js
class Person {
  @NonEnumerable
  get kidCount() { return this.children.length; }
}

function NonEnumerable(target, name, descriptor) {
  descriptor.enumerable = false;
}
```

A more detailed example illustrating a simple decorator that memoizes an
accessor.

```js
class Person {
  @Memoize
  get name() { return `${this.first} ${this.last}` }
  set name(val) {
    let [first, last] = val.split(' ');
    this.first = first;
    this.last = last;
  }
}

let memoized = new WeakMap();
function Memoize(target, name, descriptor) {
  let getter = descriptor.get, setter = descriptor.set;

  descriptor.get = function() {
    let table = memoizationFor(this);
    if (name in table) { return table[name]; }
    return table[name] = getter.call(this);
  }

  descriptor.set = function(val) {
    let table = memoizationFor(this);
    setter.call(this, val);
    table[name] = val;
  }
}

function memoizationFor(obj) {
  let table = memoized.get(obj);
  if (!table) { table = Object.create(null); memoized.set(obj, table); }
  return table;
}
```

It is also possible to decorate the class itself. In this case, the decorator
takes the target constructor.

```js
// A simple decorator
@Annotation
class MyClass { }

function Annotation(target) {
   // Add a property on target
   target.annotated = true;
}
```

Decorators can take arguments that will be available in the Decorator constructor as parameters after the regular `Object.defineProperty` parameters (on object properties) or after the `target` parameter on classes.

```js
@IsTestable(true)
class MyClass { }

function IsTestable(target, value) {
  target.isTestable = value;
}
```

The same technique could be used on property decorators:

```js
class C {
  @Enumerable(false)
  method() { }
}

function Enumerable(target, key, descriptor, value) {
 descriptor.enumerable = value;
 return descriptor;
}
```

As decorators evaluate to a constructor function, inheritance can be used to define complex decoration patterns and gain reusability. As ES6 classes desugar to constuctor functions, it's likely that complex behavior is written as classes.

```js
class BaseClassAnnotation {
  constructor(target, ...args) {
    this.args = args;
    (target.annotations = target.annotations || []).push(this);
  }
}

class View extends BaseClassAnnotation {
  constructor(target, name, template) {
    super(target, name, template);
  }
}


@View('sampleView', `
  <h1>Hello World!</h1>
`)
class Component {
  method() { }
}
```

Because descriptor decorators operate on targets, they also naturally work on
static methods. The only difference is that the first argument to the decorator
will be the class itself (the constructor) rather than the prototype, because
that is the target of the original `Object.defineProperty`.

For the same reason, descriptor decorators work on object literals, and pass
the object being created to the decorator.

# Desugaring

## Class Declaration

### Syntax

```js
@F("color")
@G
class Foo {
}
```

### Desugaring (ES6)

```js
class Foo {
}

new G(Foo);
new F(Foo, "color");
```

### Desugaring (ES5)

```js
function Foo {
}

new G(Foo);
new F(Foo, "color");
```

## Class Method Declaration

### Syntax

```js
class Foo {
  @F("color")
  @G
  bar() { }
}
```

### Desugaring (ES6)

```js
class Foo {
  bar() { }
}

var _desc = Object.getOwnPropertyDescriptor(Foo.prototype, "bar");
new G(Foo.prototype, "bar", _desc);
new F(Foo.prototype, "bar", _desc, "color");
Object.defineProperty(Foo.prototype, "bar", _desc);
```

### Desugaring (ES5)

```js
function Foo {
  bar() { }
}

var _desc = Object.getOwnPropertyDescriptor(Foo.prototype, "bar");
new G(Foo.prototype, "bar", _desc);
new F(Foo.prototype, "bar", _desc, "color");
Object.defineProperty(Foo.prototype, "bar", _desc);
```

## Class Accessor Declaration

### Syntax

```js
class Foo {
  @F("color")
  @G
  get bar() { }
  set bar(value) { }
}
```

### Desugaring (ES6)

```js
class Foo {
  get bar() { }
  set bar(value) { }
}

var _desc = Object.getOwnPropertyDescriptor(Foo.prototype, "bar");
new G(Foo.prototype, "bar", _desc);
new F(Foo.prototype, "bar", _desc, "color");
Object.defineProperty(Foo.prototype, "bar", _desc);
```

### Desugaring (ES5)

```js
function Foo() {
}
Object.defineProperty(Foo.prototype, "bar", {
  get: function () { },
  set: function (value) { },
  enumerable: true, configurable: true
});

var _desc = Object.getOwnPropertyDescriptor(Foo.prototype, "bar");
new G(Foo.prototype, "bar", _desc);
new F(Foo.prototype, "bar", _desc, "color");
Object.defineProperty(Foo.prototype, "bar", _desc);
```

## TODO: FOLLOWING TOPICS HAVE NOT BEEN CHANGED YET

## Object Literal Method Declaration

### Syntax

```js
var o = {
  @F("color")
  @G
  bar() { }
}
```

### Desugaring (ES6)

```js
var o = (function () {
  var _obj = {
    bar() { }
  }

  var _temp;
  _temp = F("color")(_obj, "bar",
    _temp = G(_obj, "bar",
      _temp = void 0) || _temp) || _temp;
  if (_temp) Object.defineProperty(_obj, "bar", _temp);
  return _obj;
})();
```

### Desugaring (ES5)

```js
var o = (function () {
    var _obj = {
        bar: function () { }
    }

    var _temp;
    _temp = F("color")(_obj, "bar",
        _temp = G(_obj, "bar",
            _temp = void 0) || _temp) || _temp;
    if (_temp) Object.defineProperty(_obj, "bar", _temp);
    return _obj;
})();
```

## Object Literal Accessor Declaration

### Syntax

```js
var o = {
  @F("color")
  @G
  get bar() { }
  set bar(value) { }
}
```

### Desugaring (ES6)

```js
var o = (function () {
  var _obj = {
    get bar() { }
    set bar(value) { }
  }

  var _temp;
  _temp = F("color")(_obj, "bar",
    _temp = G(_obj, "bar",
      _temp = void 0) || _temp) || _temp;
  if (_temp) Object.defineProperty(_obj, "bar", _temp);
  return _obj;
})();
```

### Desugaring (ES5)

```js
var o = (function () {
  var _obj = {
  }
  Object.defineProperty(_obj, "bar", {
    get: function () { },
    set: function (value) { },
    enumerable: true, configurable: true
  });

  var _temp;
  _temp = F("color")(_obj, "bar",
    _temp = G(_obj, "bar",
      _temp = void 0) || _temp) || _temp;
  if (_temp) Object.defineProperty(_obj, "bar", _temp);
  return _obj;
})();
```

# Grammar

&emsp;&emsp;*DecoratorList*<sub> [Yield]</sub>&emsp;:  
&emsp;&emsp;&emsp;*DecoratorList*<sub> [?Yield]opt</sub>&emsp; *Decorator*<sub> [?Yield]</sub>

&emsp;&emsp;*Decorator*<sub> [Yield]</sub>&emsp;:  
&emsp;&emsp;&emsp;`@`&emsp;*AssignmentExpression*<sub> [?Yield]</sub>

&emsp;&emsp;*PropertyDefinition*<sub> [Yield]</sub>&emsp;:  
&emsp;&emsp;&emsp;*IdentifierReference*<sub> [?Yield]</sub>  
&emsp;&emsp;&emsp;*CoverInitializedName*<sub> [?Yield]</sub>  
&emsp;&emsp;&emsp;*PropertyName*<sub> [?Yield]</sub>&emsp; `:`&emsp;*AssignmentExpression*<sub> [In, ?Yield]</sub>  
&emsp;&emsp;&emsp;*DecoratorList*<sub> [?Yield]opt</sub>&emsp;*MethodDefinition*<sub> [?Yield]</sub>

&emsp;&emsp;*CoverMemberExpressionSquareBracketsAndComputedPropertyName*<sub> [Yield]</sub>&emsp;:  
&emsp;&emsp;&emsp;`[`&emsp;*Expression*<sub> [In, ?Yield]</sub>&emsp;`]`

NOTE	The production *CoverMemberExpressionSquareBracketsAndComputedPropertyName* is used to cover parsing a *MemberExpression* that is part of a *Decorator* inside of an *ObjectLiteral* or *ClassBody*, to avoid lookahead when parsing a decorator against a *ComputedPropertyName*. 

&emsp;&emsp;*PropertyName*<sub> [Yield, GeneratorParameter]</sub>&emsp;:  
&emsp;&emsp;&emsp;*LiteralPropertyName*  
&emsp;&emsp;&emsp;[+GeneratorParameter] *CoverMemberExpressionSquareBracketsAndComputedPropertyName*  
&emsp;&emsp;&emsp;[~GeneratorParameter] *CoverMemberExpressionSquareBracketsAndComputedPropertyName*<sub> [?Yield]</sub>

&emsp;&emsp;*MemberExpression*<sub> [Yield]</sub>&emsp; :  
&emsp;&emsp;&emsp;[Lexical goal *InputElementRegExp*] *PrimaryExpression*<sub> [?Yield]</sub>  
&emsp;&emsp;&emsp;*MemberExpression*<sub> [?Yield]</sub>&emsp;*CoverMemberExpressionSquareBracketsAndComputedPropertyName*<sub> [?Yield]</sub>  
&emsp;&emsp;&emsp;*MemberExpression*<sub> [?Yield]</sub>&emsp;`.`&emsp;*IdentifierName*  
&emsp;&emsp;&emsp;*MemberExpression*<sub> [?Yield]</sub>&emsp;*TemplateLiteral*<sub> [?Yield]</sub>  
&emsp;&emsp;&emsp;*SuperProperty*<sub> [?Yield]</sub>  
&emsp;&emsp;&emsp;*NewSuper*&emsp;*Arguments*<sub> [?Yield]</sub>  
&emsp;&emsp;&emsp;`new`&emsp;*MemberExpression*<sub> [?Yield]</sub>&emsp;*Arguments*<sub> [?Yield]</sub>

&emsp;&emsp;*SuperProperty*<sub> [Yield]</sub>&emsp;:  
&emsp;&emsp;&emsp;`super`&emsp;*CoverMemberExpressionSquareBracketsAndComputedPropertyName*<sub> [?Yield]</sub>  
&emsp;&emsp;&emsp;`super`&emsp;`.`&emsp;*IdentifierName*

&emsp;&emsp;*ClassDeclaration*<sub> [Yield, Default]</sub>&emsp;:  
&emsp;&emsp;&emsp;*DecoratorList*<sub> [?Yield]opt</sub>&emsp;`class`&emsp;*BindingIdentifier*<sub> [?Yield]</sub>&emsp;*ClassTail*<sub> [?Yield]</sub>  
&emsp;&emsp;&emsp;[+Default] *DecoratorList*<sub> [?Yield]opt</sub>&emsp;`class`&emsp;*ClassTail*<sub> [?Yield]</sub>

&emsp;&emsp;*ClassExpression*<sub> [Yield, GeneratorParameter]</sub>&emsp;:  
&emsp;&emsp;&emsp;*DecoratorList*<sub> [?Yield]opt</sub>&emsp;`class`&emsp;*BindingIdentifier*<sub> [?Yield]opt</sub>&emsp;*ClassTail*<sub> [?Yield, ?GeneratorParameter]</sub>

&emsp;&emsp;*ClassElement*<sub> [Yield]</sub>&emsp;:  
&emsp;&emsp;&emsp;*DecoratorList*<sub> [?Yield]opt</sub>&emsp;*MethodDefinition*<sub> [?Yield]</sub>  
&emsp;&emsp;&emsp;*DecoratorList*<sub> [?Yield]opt</sub>&emsp;`static`&emsp;*MethodDefinition*<sub> [?Yield]</sub>

# Notes

In order to more directly support metadata-only decorators, a desired feature
for static analysis, the TypeScript project has made it possible for its users
to define [ambient decorators](https://github.com/jonathandturner/brainstorming/blob/master/README.md#c6-ambient-decorators)
that support a restricted syntax that can be properly analyzed without evaluation.

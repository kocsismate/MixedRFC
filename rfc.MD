# Introduction

With the addition of scalar types in PHP 7, nullables in 7.1 and object in 7.2, people writing PHP code can explicitly declare type information for most function parameters and function returns.

When a parameter or function return has no type information it is ambiguous what that means. It could mean one of the following:

* the type is a specific type, but the programmer forgot to write that code.
* the type is a specific type, but the programmer omitted to write that code to keep compatibility with an older PHP version
* the type is not currently expressible in PHPs type system, and so no type could be specified.
* for return types, it is not clear if the function will or will not return a value.

## Proposal

This RFC proposes to add the mixed type to be used in PHP's type system.

This will allow people to add types to parameters, class properties and function returns to indicate that the type information wasn't forgotten, it just can't be expressed accurately in the type system PHP has.

Mixed is used widely in the PHP manual.

 
```
var_dump ( mixed $expression [, mixed $... ] ) : void
```

as it is shorter to write that, then it is to write the equivalent using union types:

```
var_dump ( array|bool|callable|int|float|null|object|resource|string $expression [, array|bool|callable|int|float|null|object|resource|string $... ] ) : void

```


## Review of LSP, Covariance and Contravariance!

To hopefully avoid confusion about choice made in this RFC, a quick review of [Liskov Substituion Priniciple](https://en.wikipedia.org/wiki/Liskov_substitution_principle), Covariance and contravariance.


### Liskov Substituion Priniciple

Liskov's notion of a behavioural subtype defines a notion of substitutability for objects; that is, if S is a subtype of T, then objects of type T in a program may be replaced with objects of type S without altering any of the desirable properties of that program (e.g. correctness).

PHP follows LSP when performing signature checks for inheritance.


### Contravariance aka type widening aka 

PHP allows contravariance for parameter types to obey the LSP principle. A subclass may use a 'wider' aka less specific type in place of the inherited type for a parameter.

```
// Valid example

class Animal {}
class Cat extends Animal {}

class A {
    function foo(Cat $x){}
}

class B extends A {
   function foo(Animal $x){}
}

if (rand(0, 1) {
    $object = new A;
}
else {
    $object = new B;
}

$object->foo(new Cat);

```
This code works regardless of whether class A or B was created.


A subclass may not use a 'narrower' aka more specific type in place of the inherited type for a parameter.

```
// Invalid example

class Animal {
    
}
class Cat extends Animal {
    public function meow();
}

class A {
    function foo(Animal $x){}
}

class B extends A {
   function foo(Cat $x){}
}

if (rand(0, 1) {
    $object = new A;
}
else {
    $object = new B;
}

$object->foo(1);


```
This code fails %50 of the time as the int value isn't acceptable to B.


### Covariance aka type narrowing

PHP allows covariance for return types to obey the LSP principle.

A subclass may use a narrower aka more specific type as a return type.

```
// Valid example

class Animal {
    public function eat();
}

class Cat extends Animal {
    public function meow();
}

class A {
    function foo(): Animal
    {...}
}

class B extends A {
   function foo(): Cat
   {...}
}


if (rand(0, 1) {
    $object = new A;
}
else {
    $object = new B;
}

$animal = $object->foo();
$animal->eat();

```

This code works regardless of whether class A or B was created.

A subclass may not use a wider aka less specific type as a return type.

```
// Invalid example

class Animal {
    public function eat();
}

class Cat extends Animal {
    public function meow();
}

class A {
    function foo(): Animal
    {...}
}

class B extends A {
   function foo(): Cat
   {...}
}


if (rand(0, 1) {
    $object = new A;
}
else {
    $object = new B;
}

$animal = $object->foo();
$animal->meow();

```

This code fails %50 of the time as not all animals can meow.


# Mixed type checking

This RFC proposes that `mixed` would be equivalent to `array|bool|callable|int|float|null|object|resource|string` as this is the correct behaviour to conform to LSP using PHP's implementation of type checking for inheritance.

## Parameter types are contravariant

A parameter type may be widened in a subclass from a specific value type to the mixed type as this is contravariant and so is allowed in LSP.


```
// Valid example
class A
{
    public function foo(int $value){
    }
}
 
class B extends A
{
    // Parameter type was widened from int to mixed, this is allowed
    public function foo(mixed $value) {
    }
}
```


A parameter type may not be narrowed in a subclass to a more specific type as this is not contravariant and so violates LSP.
```
// Invalid example
class A
{
    public function foo(mixed $value){
    }
}
 
class B extends A
{
    // Parameter type cannot be narrowed from mixed to int
    // Fatal error thrown
    public function foo(int $value) {
    }
}

```


### Return types are covariant


Specific return type could be narrowed in a subclass by using the mixed type as this is covariant and is allowed in LSP

```
// Valid example
class A
{
    public function bar() : mixed
    {}
}
 
class B extends A
{
 
    // return type was narrowed from mixed to int, this is allowed
    public function bar() : int 
    {}
}
```


Specific return type may not be widened as this is not covariant and so breaks LSP.
```
// Invalid example
class C
{
    public function bar() : int
    {}
}

class D extends C
{
    // return type cannot be widened from int to mixed
    // Fatal error thrown
    public function bar() : mixed
    {}
}

```


## Void return type 

Currently, the signature checking done in PHP for functions that return void does not allow covariance even though this appears to be conformant to LSP. 

```
class A {
    public function bar() : void {}
}

class B extends A {
    public function bar() : int {}
}

// Fatal error: Declaration of B::bar(): int must be compatible with A::bar(): void

```

The position of this RFC is to follow that existing behaviour. i.e. you can't widen the type from void to mixed, when inheriting.


### Signature checking of function when no parameter type present 

When no native type is present for a function parameter, the signature checks for inheritance are done as if the parameter had a 'mixed' type.

```
class A
{
    // no type is specified, mixed type is assumed
    public function foo($value) {
    }
}
 
class B extends A
{
    // mixed type is explicitly specified, and is invariant to
    // type in parent class
    public function foo(mixed $value) 
    {}
}
 
class C extends B
{
    // no type is specified, mixed type is assumed which is
    // invariant to type in parent class
    public function foo($value) 
    {}
}


class D extends B
{
    // Hmm, this is correct right?
    public function foo(mixed $value = null) 
    {}
}

```

Currently this only affects inheritance in classes. 

If/when PHP gains the abilties to declare [function signatures as types](https://github.com/Danack/RfcCodex/blob/master/typedef_callables.md), rather than just the generic 'callable' type this signature checking should work for those signature checks also.


## Signature checking of function that when no return type present

Since return types may be either mixed or void, the behaviour is slightly more complicated compared to parameters. When no type is specified, subclass must either also declare no type, declare void or declare mixed (or any other value type which is subtype of mixed). Additionally neither mixed nor void return types could be changed back to no type since this would widen the resulting type.

When no native type is present for a function return, the signature checks for inheritance are done as if the parameter had a 'mixed|void' type.

```
class A
{
    // no return type is specified, mixed|void is assumed
    public function foo(){
    }
}
 
class B extends A
{
    // mixed type is explicitly specified. The type 'mixed' is
    // covariant to 'mixed|void' and so is allowed to be declared
    // for this function.
    public function foo(): mixed {
    }
}

class C extends B
{
    // INVALID - no type is specified, mixed|void is assumed. 
    // 'mixed|void' is not covariant to 'mixed' and so this breaks LSP.
    // Fatal error is thrown
    public function foo() 
    {}
}


class D extends B
{
    // INVALID - as void is not subtype of mixed, Fatal error is thrown
    public function foo() : void 
    {}
}

```

### Mixed void union type

The position of this RFC is that supporting a union of `mixed|void` is not needed and so proposes not allowing that type declaration to be used. This limitation could be lifted at a later date if a use-case was found.


## Nullability

The mixed is a union type that accepts any value type, including null. Allowing the mixed type declaration to be nullable would be duplication of information i.e. `?mixed` would be always be equivalent to be `mixed`.

The position of this RFC is to not support nullability of mixed type. This could always be added at a later date if a reason for adding it was discovered, but allowing it now seems to serve no purpose. 

This behaviour also conforms to the current usage in documentation.

```
//INVALID - Fatal error: Mixed types cannot be nullable, null is already part of the mixed type.
function foo(?mixed $arg) {} 
 

//INVALID - Fatal error: Mixed types cannot be nullable, null is already part of the mixed type.
function bar() : ?mixed {} 
```


## Backward Incompatible Changes

None, mixed is a reserved keyword since PHP 7.0.

## Proposed PHP Version(s)
7.4

## RFC Impact


###  To SAPIs
None known.

## To Existing Extensions
None known.

## To Opcache

Not analyzed.


## Proposed Voting Choices

Add mixed as a type to be used as parameter, return and class property types - yes/no.




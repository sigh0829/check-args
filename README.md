check-args
==========

Argument type checking with __no performance penalty__.

This module is intended to be used in libraries and applications during development.
You will mark functions with type checking instructions (i.e. signatures)
and in development they will be wrapped inside an arguments checker.
When a wrapped method is called, arguments will be checked and
error is thrown if there is no acceptable signature for the call.

## No penalty?

Well, '*no penalty*' means no penalty in **production**.
And in development the penalty is very low,
practically the same as custom type checking code.
And if we want to be precise,
in production there will also be some clock cycles more due to couple of more code lines to parse, but it will not be measurable.

This is accomplished by a clever trick, the type checking library is just a noop placeholder.
In development you can turn it on by making sure that `node_modules` contains the actual check implementation.

### Why just in development?

In reality, we don't need type checking in production,
but during development it is a real benefit.
You need sophisticated error messages, if you call a library method with unsuitable arguments.
And even better if the error message tells you how you should have called the method.

Of course, if you really need type checkings in production, *and* you
don't mind few extra fractions of milliseconds, you can keep it on also in production.

## Design goals

- Can be omitted in production
- Very fast, also in development
- Intuitive to use
- Informative error messages
- Stack trace should show correct location at the top

## How to use it

This is very simple.
If you just want to check that you correctly use the api, then

```shell
$ npm install check-args
```
Remove the library when you are done with debugging, by issuing command:

```shell
$ npm prune
```
If you want to keep checking enabled always when you are developing:

```shell
$ npm install check-args --save-dev
```
If you make `check-args` a dependency, the checkings will be done also in production.
I recommend not to.

## Example code

We assume that in your application you use `the-library`, that has something along the lines below...

```javascript
var accept = require("check-args-lib")

var fn = accept(Number).accept(String).for(function(dyna) {
  // do something with dyna...
  // If the type checking is enabled, you can now be sure it is
  // defined and it is a number or a string
});

module.exports = fn
```

...and if you then have something like the following lines in your app/lib **and**
you have installed the _check-args_ library (with `$ npm i check-args`)...
```javascript
// import the library
var fn = require("the-library");

fn("a string"); // this works and reaches the actual function

fn(); // this fails with an error which tells how fn() should have been called

fn(new Date()); // this fails with an error too
```

... you will have your library calls checked.




## API

### `accept([type_definition], ...)`

Describes one signature for the function.

#### Arguments

| Param | Type | Details
|-------|------|---------
| type_definition | [a type specification](#type-specification) | Describes the type of one argument in a signature

#### Returns

Object :: An object that allows you to define more signatures (`accept()`)
or wrap a function (`for()`).


### `for(wrapped_function)`

Wraps the given function with argument type checks.

#### Arguments

| Param | Type | Details
|-------|------|---------
| wrapped_function | function | The function that will be wrapped with argument type checks

#### Returns

Function :: A function that is called after argument check succeeds.
If there was no matching signature, the function will throw an error
which lists allowed signatures and the provided one. This functionality
will be enabled only when `check_args` is available. Otherwise
for returns the `wrapped_function` as it is.

### Type specification

This can be a javascript builtin type (String, Number, Boolean, ...),
a modifer object (explained later), custom type or a check-args builtin
type specification.

| Type spec type  | Explanation
|-----------------|-------------
| builtin type    | ...
| modifier object | ...
| custom type     | ...


## Wrapping a function with a checker

```javascript
var fn = accept([arguments..])
          .accept([arguments..])
          // ... any number of accept calls
          .for(function(arguments...) { });
```
See API


## Writing argument checks

It sounds awkward, but you should require the placeholder library when using
the actual api. When you utilize a library that has type checking you will
install the actual api implementation.

The first line...

```javascript
var accept = require("check-args-lib")
```

You can chain `accept()` calls as many as you need.
Each accept describes an acceptable signature to the function.

The returned object has also a method `for()`.
It takes one parameter and it must be a function.

When the type checking module is present `for()` will return
a function that checks its arguments and throws well formatted
exception if no match in given signatures is found.
If a match is found then the function is called with passed arguments.

When the type checking module is not present, all methods are stubs
and `for()` returns the given functions without wrapping it.


### Arguments to `accept()`

Each argument corresponds to acceptable argument in the target function at the same position.
Argument in the simplest form are types that we want as arguments to the function.

Example:
```javascript
var fn = accept(String, Array, Date).accept(Object, Number, String)...etc.
```
You can use builtin types or custom types. For custom types `instanceof` operator is used.

Example:
```javascript
var fn = accept(MyType1, MyType2)...etc.
```
Builtin types can found in javascript documentation, e.g.
[MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Data_structures).

#### Special meanings

| Argument    | Meaning
|-------------|----
| `Object`    | any type of argument except `null` and `undefined`
| `null`      | any type of argument that can also be `null`
| `undefined` | any type of argument that can also be `undefined` or `null`. Argument must be present, even the last.


### Modifiers

You often need to check that argument is String or null or undefined.
For this you need a new way to express things. Let's introduce
modifiers.

Example is often more than a few words...
```javascript
var fn = accept( { nullable: String } )...
```
Here the argument type is denoted with an object. Explanation:

> An argument to `accept()` can also be an object which has exactly one property.
> We call such argument `modifier object` and the name of the property is called `modifier`

> `modifier` tells how the value of the property is interpreted.

> The value of the `modifier object` follows the same rules as an `accept()` argument.
> It can even be a new `modifier object`.

| Modifier    | Meaning
|-------------|---------
| nullable    | type is **value** or `undefined` or `null`
| array       | an array where tye type of each item is **value**
| regex       | argument must be a string that matches regex in the **value**
| custom      | **value** is a function that is called with argument and returns a boolean which indicates whether the argument is valid.
| rest        | zero or more **value**, there can be only one rest modifier per `accept()`.

You can nest modifiers...
```javascript
var fn = accept( { nullable: { regex: /\S+\w+\(/ } } )...

// accept array of strings that can also be null
var fn2 = accept( { array: { nullable: String } } )...

var fn3 = accept( Number, { rest: { nullable: String } }, undefined )...
// valid calls...
fn3(3, {});
fn3(3, 'asdf', 'dfsdf', 23);
fn3(3, 'dfsdf', null);
fn3(3, 'dfsdf', null, null);
fn3(3, undefined);
```


## Usage in browser

TODO


## Modules using `check-arg`

None yet that I know

### Blacklist

Here will be listed all published modules (I can find) that have `check-args` in dependencies.
Note that having it in devDependencies is ok.
This is **evil** because if *any* of the dependent modules has it in its dependencies, type checking will always be on in the whole application.

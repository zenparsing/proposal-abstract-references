# Virtual Properties for JavaScript #

## Overview and Motivation ##

Despite its functional programming facilities, JavaScript exhibits a strong preference for object-oriented "left-to-right" composition using instance methods. Unfortunately, composition by means of instance methods requires that extensibility be achieved by adding function-valued properties to the object or a prototype ancestor of the object.  This leads to the following difficulties:

- The API burden for an abstraction is placed upon the abstraction itself, either directly or by the introduction of inheritance hierarchies. This leads to an undesirable centralization of API surface.
- Adding API surface to an existing abstraction introduces the possibility of breakage which can be difficult to predict.
- By adding API surface directly to the object or one of its prototype ancestors, it is impossible to treat the object and the API as different capabilities. If a user has access to the object, then the user necessarily has access to the full API (part of which we might want to hide).

This proposal adds support for "left-to-right" syntactic composition using a new binary operator, which does not require adding properties and methods to the object itself. It accomplishes this by introducing the concept of "virtual properties". Virtual properties extend the ECMAScript Reference specification type by allowing the *referenced name* component to be an arbitrary object.

This proposal also provides a solution to the "private fields" problem.

This work was inspired by [relationships](https://web.archive.org/web/20160804042554/http://wiki.ecmascript.org/doku.php?id=strawman:relationships) and is intended to supersede it.


## Specification ##

We introduce three new built-in symbols:

- **@@referenceGet** (Symbol.referenceGet)
- **@@referenceSet** (Symbol.referenceSet)
- **@@referenceDelete** (Symbol.referenceDelete)

We introduce a new abstract operation:

- **IsVirtualPropertyReference(V)**.  Returns **true** if the *referenced name* component of the reference *V* is neither a primitive String nor a Symbol.

The abstract operation **GetValue** becomes:

- ReturnIfAbrupt(*V*).
- If Type(*V*) is not Reference, return *V*.
- Let *base* be GetBase(*V*).
- If IsUnresolvableReference(*V*), throw a **ReferenceError** exception.
- If IsPropertyReference(*V*), then
  - If HasPrimitiveBase(*V*) is true, then
    - Assert: In this case, *base* will never be null or undefined.
    - Set *base* to ! ToObject(*base*).
  - If IsVirtualPropertyReference(*V*) is **true**, then
    - Let *nameObject* be GetReferencedName(*V*)
    - Return ? Invoke(*nameObject*, **@@referenceGet**, (*base*))
  - Else
    - Return ? base.\[\[Get\]\](GetReferencedName(*V*), GetThisValue(*V*)).
- Else *base* must be an environment record,
  - Return ? *base*.GetBindingValue(GetReferencedName(*V*), IsStrictReference(*V*)).

The abstract operation **PutValue** becomes:

- ReturnIfAbrupt(*V*).
- ReturnIfAbrupt(*W*).
- If Type(*V*) is not Reference, throw a **ReferenceError** exception.
- Let *base* be GetBase(*V*).
- If IsUnresolvableReference(*V*), then
  - If IsStrictReference(*V*) is true, then
    - Throw **ReferenceError** exception.
  - Let *globalObj* be the result of the abstract operation GetGlobalObject.
  - Return Put(*globalObj*, GetReferencedName(*V*), *W*, **false**).
- Else if IsPropertyReference(*V*), then
  - If HasPrimitiveBase(*V*) is true, then
    - Assert: In this case, *base* will never be **null** or **undefined**.
    - Set *base* to ToObject(*base*).
  - If IsVirtualPropertyReference(*V*), then
    - Let *nameObject* be GetReferencedName(*V*)
    - Let *result* be Invoke(*nameObject*, **@@referenceSet**, (*base*, *W*))
    - ReturnIfAbrupt(*result*)
  - Else
    - Let *succeeded* be the result of calling the [[Set]] internal method of *base* passing GetReferencedName(*V*), *W*, and GetThisValue(*V*) as arguments.
    - ReturnIfAbrupt(*succeeded*).
    - If *succeeded* is **false** and IsStrictReference(*V*) is **true**, then throw a **TypeError** exception.
  - Return.
- Else *base* must be a Reference whose base is an environment record.
  - Return the result of calling the SetMutableBinding concrete method of *base*, passing GetReferencedName(*V*), *W*, and IsStrictReference(*V*) as arguments.


The runtime semantics of the delete operator is modified as follows:

```
UnaryExpression : delete UnaryExpression
```

- Let *ref* be the result of evaluating *UnaryExpression*.
- ReturnIfAbrupt(*ref*).
- If Type(*ref*) is not Reference, return **true**.
- If IsUnresolvableReference(*ref*) is **true**, then
  - Assert: IsStrictReference(*ref*) is **false**.
  - Return **true**.
- If IsPropertyReference(*ref*) is **true**, then
    - If IsSuperReference(*ref*), then throw a **ReferenceError** exception.
    - Let *base* be ToObject(GetBase(*ref*))
    - If IsVirtualPropertyReference(*ref*) is **true**, then
      - Let *nameObject* be GetReferencedName(*V*)
      - Let *result* be Invoke(*nameObject*, **@@referenceDelete**, (*base*))
      - Return **true**.
    - Else
      - Let *deleteStatus* be the result of calling the [[Delete]] internal method on *base*, providing GetReferencedName(*ref*) as the argument.
      - ReturnIfAbrupt(*deleteStatus*).
      - If *deleteStatus* is **false** and IsStrictReference(*ref*) is **true**, then throw a **TypeError** exception.
      - Return *deleteStatus*.
- Else *ref* is a Reference to an Environment Record binding,
  - Let *bindings* be GetBase(*ref*).
  - Return the result of calling the DeleteBinding concrete method of *bindings*, providing GetReferencedName(*ref*) as the argument.

The only way to create a reference whose *referenced name* is neither a String nor a Symbol is by using the "virtual property" operator:

```
MemberExpression[Yield] :
    MemberExpression[?Yield] :: IdentifierReference[?Yield]
```

- Let *baseReference* be the result of evaluating *MemberExpression*.
- Let *baseValue* be GetValue(*baseReference*).
- ReturnIfAbrupt(*baseValue*).
- Let *nameValue* be the result of evaluating *IdentifierReference*.
- ReturnIfAbrupt(*nameValue*).
- Let *bv* be RequireObjectCoercible(*baseValue*).
- ReturnIfAbrupt(*bv*).
- If the code matched by the syntactic production that is being evaluated is strict mode code, let *strict* be **true**, else let *strict* be **false**.
- Return a value of type Reference whose base value is *bv* and whose referenced name is *nameValue*, and whose strict reference flag is *strict*.

## Extensions to Built-In Types ##

The built-in types are extended as follows:

```js
Function.prototype[Symbol.referenceGet] = function(base) { return this.bind(base); };

Map.prototype[Symbol.referenceGet] = Map.prototype.get;
Map.prototype[Symbol.referenceSet] = Map.prototype.set;
Map.prototype[Symbol.referenceDelete] = Map.prototype.delete;

WeakMap.prototype[Symbol.referenceGet] = WeakMap.prototype.get;
WeakMap.prototype[Symbol.referenceSet] = WeakMap.prototype.set;
WeakMap.prototype[Symbol.referenceDelete] = WeakMap.prototype.delete;
```

## Examples ##

Using an iterator library implemented as a module:

```js
import { map, takeWhile, forEach } from "iterlib";

getPlayers()
  ::map(x => x.character())
  ::takeWhile(x => x.strength > 100)
  ::forEach(x => console.log(x));
```

Using **WeakMap** objects to easily implement private object state:


```js
const data = new WeakMap();

class Point {

  constructor(x, y) {
    this::data = { x, y };
  }

  toString() {
    return `[${ this::data.x },${ this::data.y }]`;
  }

}
```

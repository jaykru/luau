# Metatable keyword in type annotations

## Summary

Introduce a way to ascribe the type of a metatable for a table via the syntax `{ metatable :: T }`.

## Motivation

Today, the only way to attach a metatable to a table, one has to write `typeof(setmetatable(t, mt))` in type annotation context. While supported, it's not the most obvious path for everyone and leaves a pretty critical gap between the syntax and the type checker for an important part of Lua.

## Design

We already have the concept of metatables in the type inference engine, so this RFC will not get into the subtyping relationship or anything of the sort. The scope of this RFC is specifically to expose syntax to produce a type variable whose table has this metatable, and no more than that.

We want to introduce a new contextual keyword `metatable` where we can write `metatable :: <type>` inside table type annotations. It can show up anywhere in a table type next to properties or indexers. Just like indexers, only one metatable is permitted per table and it will be enforced at parse time.

By being able to write that a table has some metatable, we would be able to say that our data type has overloaded some operators. For example, our `Vec3` where the `+` operator takes two `Vec3` and produces a new `Vec3`:

```
type Vec3 = {
    x: number,
    y: number,
    z: number,
    metatable :: {
        __add: (self: Vec3, other: Vec3) -> Vec3
    }
}
```

Furthermore, the metatable syntax can also help with writing the type definitions for idiomatic OO code.

```
type Person = {
    name: string,
    metatable :: PersonImpl,
}

type PersonImpl = {
    __index: PersonImpl,
    new: (name: string) -> Person,
    say: (self: Person, message: string) -> ()
}

local Person = {} :: PersonImpl
Person.__index = Person

function Person.new(name)
    local person = {}
    person.name = name
    return setmetatable(person, Person)
end

function Person:say(message)
    print(self.name .. ": " .. message)
end
```

The reason why two type synonyms are required here is because every instance created by `Person.new` is of type `Person`, but the table `Person` is what implements the methods for each instance of `Person`. The `self` in each methods of the table `Person` is _not_ `PersonImpl` because usually we want to call the methods of `PersonImpl` where a `Person` is passed as `self`.

We would also be able to define the `getmetatable` function without using C++.

```
declare function getmetatable<T>(tab: { metatable :: { __metatable: T } }): T
declare function getmetatable<MT>(tab: { metatable :: MT }): MT
declare function getmetatable(tab: string): typeof(string) -- to get the global string table
declare function getmetatable(tab: any): nil
```

For RFC completeness:

Just like indexers, it is not legal syntax when the parser context is not a table type annotation. `type Foo = metatable :: Bar` and `{f: (metatable :: Bar) -> ()}` and other variants are illegal.

It is also not legal to ascribe a metatable multiple times in a single table type annotation, just like indexers. 

Formally, the grammar change is:

```diff
  TableIndexer = '[' Type ']' ':' Type
+ TableMetatable = 'metatable' '::' Type
  TableProp = NAME ':' Type
- TablePropOrIndexer = TableProp | TableIndexer
- PropList = TablePropOrIndexer {fieldsep TablePropOrIndexer} [fieldsep]
- TableType = '{' PropList '}
+ TableEntry = TableProp | TableIndexer | TableMetatable
+ TableEntries = TableEntry {fieldsep TableEntry} [fieldsep]
+ TableType = '{' TableEntries '}'
```

... ignoring that we can't describe in EBNF that some syntax can only show up once but anywhere within a list.

## Drawbacks


## Alternatives

A few other alternative designs have been proposed in the past, but ultimately were decided against for various reasons.

### `{metatable T}`

```lua
type Vec3 = { x: number, y: number, z: number, metatable { __add: (Vec3, Vec3) -> Vec3 } }
```

This was rejected due to similarity with existing property syntax with only one character difference. This can actually turn out to be very annoying in practice due to muscle memory.

Attempting to try to reconcile this is futile because we do not want to retroactively say that certain property names are "magic," i.e. disallowing `metatable` as a property name unless it is written as `["metatable"]`.

Linting does not help because it might be the case that the linter was turned off for the file. In this case, trying to use the linter is a hint of a bad design anyway.

### `setmetatable()`

```lua
type Vec3 = setmetatable({ x: number, y: number, z: number }, { __add: (Vec3, Vec3) -> Vec3 })
```

One option is to copy the design of `typeof` for this use case: `setmetatable(<type>, <type>)`. However, we decided against it because it is inconsistent with `typeof` where it uses `()` to access the value namespace, whereas `setmetatable` would use `()` to access the type namespace.

Another issue is in the name itself, *set*metatable, which reads like it will mutate the type variable to have the metatable. There are cases where it is incorrect to mutate the type variable, so we would also have to introduce `withmetatable(<type>, <type>)` along with `setmetatable(<type>, <type>)`.

This option also does not allow a way to return the metatable for any given type variable. The only way to do so is to add _yet another_ syntax, `getmetatable(<type>)`.

That means we need 3 new syntax in total to support different use cases:

1. `setmetatable(T, MT)` where it is a side-effecting function that mutably adds the metatable `MT` to `T`.
2. `withmetatable(T, MT)` where it is a pure function that returns a copy of `T` with `MT` attached.
3. `getmetatable(T)` where it returns the metatable type of `T` if it exists, the type of `__metatable` field if it exists, or `nil` type otherwise.

One more issue is about formatting. This syntax can be readable if the data structure is relatively small, but in nontrivial data structures, it might be very hard to read it. To try to keep it cleaner, you would need to define two or three type aliases just for formatting, but it does mean that this can affect readability, especially when these types are stringified from `Luau::toString` and unions are involved.

```lua
setmetatable({
    some,
    really,
    long,
    type,
    definition,
    for,
    this,
    data,
    structure
}, {
    with,
    some,
    metamethods
})
```

Or when we have a table whose metatable's `__index` points to another table that has metatables, and so on:

```
setmetatable({
    ...
}, {
    __index: setmetatable({
        ...
    }, {
        ...
    })
})
```

Another issue is the unfortunate wart in syntax with bounded quantification, for example if we want a generic to only range over types that defines a `__add` metamethod: (using invented notation) `<a: setmetatable({}, {__add: (a, a) -> a})>(a) -> a`.

This syntax also exposes an opportunity to write things like `setmetatable(Instance, { ... })` where `Instance` is a class type variable, forcing us to write in more code that produces a type error on nonsensical use of `setmetatable`.

### `setmetatable<>`

```lua
type Vec3 = setmetatable<{ x: number, y: number, z: number }, { __add: (Vec3, Vec3) -> Vec3 }>
```

Another option is to define a `setmetatable<T, MT>` type function from the C++ side, which would indeed bridge the gap, but all type functions are pure, so the name `setmetatable` is thus inaccurate. It is more accurately called `withmetatable`. If we were to opt for this option, we would rather name it `withmetatable`.

It still does not grant us any syntax to return the metatable of the type, or `nil` otherwise. We would need to define a `getmetatable<T>` type function from C++ again, but even that would be inaccurate in some edge cases (metatable has `__metatable` or is missing/not a table, so returns `nil`) and requires either conditional types or making this an instrinsic type.

This option would mean `withmetatable<>` and `getmetatable<>` are the first exported types in Luau's prelude, which is not something that we want to do at this time.

This option has the same issue as `setmetatable()` on formatting, unfortunate quirk in syntax with bounded quantification, and requires dealing with edge cases where `T` is not a table.

### `{ @metatable {} }` (status quo when performing `Luau::toString`)

```lua
type Vec3 = { x: number, y: number, z: number, @metatable { __add: (Vec3, Vec3) -> Vec3) } }
```

This option may clash with attributes/decorators for one use case, so we don't think this deserves any further thought.

### `{ [metatable]: T }`/`{ [__metatable]: T }`

```lua
type Vec3 = { x: number, y: number, z: number, [metatable]: { __add: (Vec3, Vec3) -> Vec3 } }
```

This option has the same exact syntax as indexers. While it's possible to implement this, it doesn't seem to solve the issue where one might accidentally forget to type in brackets surrounding `metatable` to ascribe a metatable.

### `{ metatable = {} }`

```lua
type Vec3 = { x: number, y: number, z: number, metatable = { __add: (Vec3, Vec3) -> Vec3 } }
```

This option is inconsistent with type annotation grammar, so we don't think this deserves any further thought.

# libcpp3k

This package contains library support for the cpp3k project, including
the language support classes for reflection and ... probably some useful
metaclasses once we can actually use those.

libcpp3k is (currently) a header-only library. It provides the following
headers:

    - cpp3k/meta -- library support for language reflection



# Configuration and Building

This is currently a header only library. It just needs to be copied into
a location where it can be found by the compiler.

You don't even need to run `cmake` or `make`. Just copy and use (see below).

# Using libcpp3k

In order to use libccp3k, you will need a C++ compiler that supports the
`-freflection` flag (currently limited to a private branch of Clang).

Simply add `libcpp3k/include` to your include search path. Header files should
be included as:

```C++
#include <cpp3k/meta>
```

Note that that "lib" is not used in the include path for the header files.

With both Clang and cpp3k, you can use reflection to query declarations:
variables, functions, values, types, templates, and namespaces. Currently,
only the reflection of variables, functions, and values is supported.

TODO: Document how reflection can be used.

```C++
int main() {
  constexpr meta::function fn = $main;
  std::cout << fn.name << '\n';              // prints main
  std::cout << fn.parameters.size() << '\n'; // prints 0
  ...
}
```


# Implementation

The library is a small set of classes that provide access to reflected
AST nodes. These can be used to query, and in some cases, manipulate
existing declarations. These classes are written in terms of a small
implementation-defined intrinsic/trait interface that provides glue
between the language and the library.

That interface consists of the following intrinsics:

```C++
__get_attribute(ref, n)
__set_attribute(ref, n, val)
__get_array_element(ref, n)
__get_tuple_element(ref, n)
```

The design of the interface is essentially a compile-time version of socket 
options (`getsockopt` and `setsockopt`). For each option `ref` is a reflected
object (e.g., `$main`).

NOTE: The current implementation requires ref to be an intptr_t whose value
is node bound by reflecting e.g., $main.

For the `get/set` traits, the `n` is an integer constant expression that
determines the property being selected. The type of the expression depends
entirely upon the value of n. See the table below for the list of supported 
attributes, what they apply to and their corresponding types.

The `__get_attribute(ref, n)` trait returns a prvalue of the corresponding
type. The `__set_attribute(ref, n, val)` modifies the property of a
declaration. Currently, modifications are supported only within a metaclass
definition.

NOTE: We probably don't want programmers arbitrarily changing properties of
e.g. `std`, like making it inline. That seems dangerous.


The `__get_array_element` and `__get_tuple_element` are used for array-like
access and tuple-like access into sequences of declarations; in each case
`n` is an integer expression denoting the requested element. In the 
`__get_tuple_element` case, `n` must be a constant expression.

The interface supports both forms of access because some sequence attributes
may yield objects of different types. For example, the members of a namespace
include variables, functions, types, and other namespaces. Returning an
tuple-like object allows us to return specific kinds of instances for each
of those things.



### The Reflection concept

We can actually define a concept that determines what types are metatypes.
This is used entirely by the implementation and is not meaningful for users.

```C++
template<typename T>
concept bool Reflection()
{
  return requires(T t) {
    { t.__node } -> std::intptr_t;
  }
}
```


### Constant and bound reflections

Some properties of reflected entities are constant; they can't ever be changed,
so it doesn't make sense to bind the property to an AST node. We can simply
extract the property and add it to the reflected class. 

We definitely want to do this with string properties (e.g., an entity's name).
This is because we can rewrite the expression `$x` into `R { __magic__, "x" }`.
As an added benefit, we don't have to think about storage for `"x"` because
it gets the usual string processing rules.

Language linkage is probably another such attribute (saved as a string, not
in an integer value?).

These extracted attributes must be passed as arguments to the constructor
of the reflection type.


### The constexpr boundary

The constexpr is not sufficient to guarantee that an expression is evaluated
at compile time.

Consider:

```C++
cout << $x.linkage() << '\n';
```

Here, linkage is an attribute that must ultimately evaluate to a literal
of some kind.  However, it is not invoked in a constexpr context and so
the underlying reflection query is passed along to code gen. That's bad,
because we don't interpret those instructions.

What we really want is to make the expression as constexpr as possible --
eagerly constexpr, I suppose. That is, whenever an eager function is called
with literal objects, it must be evaluated.

So, we should declare it like this:

```C++
struct variable
{
  __eager linkage_t linkage() const { return __get_attribute(node, 0); }
};
```

Here, __eager is a declaration specifier (implying inline and constexpr),
that requires the function to be evaluated if its arguments can be evaluated.
This needs to be conditional because we may use the function internally in
contexts where the function cannot be eagerly evaluated:

struct variable
{
  __eager bool has_external_linkage() const 
  { 
    return linkage() == 0; 
  }
};

This function is also eager, meaning that it will also be opportunistically
evaluated.


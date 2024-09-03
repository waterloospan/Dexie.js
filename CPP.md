Peaks C++ Style Guide
======================

It's based on Google's C++ Style Guide, simplified to suit our own needs. If anything is missing, we can discuss and adopt more from the original Google guide. \
This style guide only govern our own code, there's no need to refactor 3rd party codes included in our project.

C++ Version
-----------

Target C++20.\
Do not use C++23 features.\
Do not use non-standard extensions.

Header Files
------------

Regular C++: in general, every `.cc` file should have an associated `.h` file. \
CUDA: every `.cu` file should have an assoicated `.cuh` file. \
Unit tests may not need header files.

Header files should be self-contained (compile on their own) and end in `.h` or `.cuh`. \
Users and refactoring tools should not have to adhere to special conditions to include the header. \
Specifically, a header should have `header guards` and include all other headers it needs.

### The #define Guard

All header files should have `#define` guards to prevent multiple inclusion. \
The format of the symbol name should be `_PEAKS_<PATH>_<FILE>_H_`. \
To guarantee uniqueness, they should be based on the full path in the source tree. \
For example, the file `foo/bar.h`: 

```
#ifndef _PEAKS_FOO_BAR_H_
#define _PEAKS_FOO_BAR_H_ 

...

#endif  // _PEAKS_FOO_BAR_H_
```

### Include What You Use

Do not rely on transitive inclusion. If a source or header file refers to a symbol defined elsewhere, the file should directly include a header file which properly intends to provide a declaration or definition of that symbol. It should not include header files for any other reason.

### Forward Declarations

Avoid using forward declarations where possible.

### Inline Functions

Define functions inline only when they are small and simple: 10 lines or fewer, also without loops or recursive calls.

### Names and Order of Includes

Include headers in the following order: Related header, C system headers, C++ standard library headers, other libraries' headers, our own headers. \
Do not use aliases `.` (the current directory) or `..` (the parent directory). \
Headers should only be included using an angle-bracketed path if the library requires you to do so.

Scoping
-------

### Namespaces

With few exceptions, always place code in a namespace. \
Our root namespace is peaks, and everything in directory possibly with its path. e.g. peaks::ml \
Avoid _using-directives_ (e.g., `using namespace foo`). \
Avoid using inline namespaces. \
Terminate a namespace with comments referring its name. \
Single-line nested namespace declarations are preferred but not required.
    

### Internal Linkage

When definitions in a `.cc` file do not need to be referenced outside that file, give them internal linkage by placing them in an unnamed namespace or declaring them as `static`. Do not use either of these constructs in `.h` files.

### Nonmember, Static Member, and Global Functions

Prefer placing nonmember functions in a namespace; use completely global functions rarely. \
Do not use a class simply to group static members. Static methods of a class should generally be closely related to instances of the class or the class's static data.

Nonmember functions should not depend on external variables.

### Local Variables

Place a function's variables in the narrowest scope possible, and initialize variables in the declaration.

C++ allows you to declare variables anywhere in a function. We encourage you to declare them in a scope as local as possible, and as close to the first use as possible. This makes it easier for the reader to find the declaration and see what type the variable is and what it was initialized to. In particular, initialization should be used instead of declaration and assignment, e.g.,:

```
int i;
i = f();      // Bad -- initialization separate from declaration.

int i = f();  // Good -- declaration has initialization.

int jobs = NumJobs();
// More code...
f(jobs);      // Bad -- declaration separate from use.

int jobs = NumJobs();
f(jobs);      // Good -- declaration immediately (or closely) followed by use.

std::vector<int> v;
v.push\_back(1);  // Prefer initializing using brace initialization.
v.push\_back(2);

std::vector<int> v = {1, 2};  // Good -- v starts initialized.
```

Variables needed for `if`, `while` and `for` statements should normally be declared within those statements, so that such variables are confined to those scopes. E.g.:

while (const char\* p = strchr(str, '/')) str = p + 1;

There is one caveat: if the variable is an object, its constructor is invoked every time it enters scope and is created, and its destructor is invoked every time it goes out of scope.

// Inefficient implementation:
for (int i = 0; i < 1000000; ++i) {
  Foo f;  // My ctor and dtor get called 1000000 times each.
  f.DoSomething(i);
}

It may be more efficient to declare such a variable used in a loop outside that loop:

Foo f;  // My ctor and dtor get called once each.
for (int i = 0; i < 1000000; ++i) {
  f.DoSomething(i);
}

### Static and Global Variables

Objects with [static storage duration](http://en.cppreference.com/w/cpp/language/storage_duration#Storage_duration) are forbidden unless they are [trivially destructible](http://en.cppreference.com/w/cpp/types/is_destructible). Informally this means that the destructor does not do anything, even taking member and base destructors into account. More formally it means that the type has no user-defined or virtual destructor and that all bases and non-static members are trivially destructible. 

Static function-local variables may use dynamic initialization. Use of dynamic initialization for static class member variables or variables at namespace scope is discouraged.

#### Common patterns

*   Global strings: if you require a named global or static string constant, consider using a `constexpr` variable of `string_view`, character array, or character pointer, pointing to a string literal. String literals have static storage duration already and are usually sufficient. See [TotW #140.](https://abseil.io/tips/140)
*   Maps, sets, and other dynamic containers: if you require a static, fixed collection, such as a set to search against or a lookup table, you cannot use the dynamic containers from the standard library as a static variable, since they have non-trivial destructors. Instead, consider a simple array of trivial types, e.g., an array of arrays of ints (for a "map from int to int"), or an array of pairs (e.g., pairs of `int` and `const char*`). For small collections, linear search is entirely sufficient (and efficient, due to memory locality); consider using the facilities from [absl/algorithm/container.h](https://github.com/abseil/abseil-cpp/blob/master/absl/algorithm/container.h) for the standard operations. If necessary, keep the collection in sorted order and use a binary search algorithm. If you do really prefer a dynamic container from the standard library, consider using a function-local static pointer, as described below .
*   Smart pointers (`std::unique_ptr`, `std::shared_ptr`): smart pointers execute cleanup during destruction and are therefore forbidden. Consider whether your use case fits into one of the other patterns described in this section. One simple solution is to use a plain pointer to a dynamically allocated object and never delete it (see last item).
*   Static variables of custom types: if you require static, constant data of a type that you need to define yourself, give the type a trivial destructor and a `constexpr` constructor.
*   If all else fails, you can create an object dynamically and never delete it by using a function-local static pointer or reference (e.g., `static const auto& impl = *new T(args...);`).

### thread\_local Variables

`thread_local` variables that aren't declared inside a function must be initialized with a true compile-time constant, and this must be enforced by using the [`constinit`](https://en.cppreference.com/w/cpp/language/constinit) attribute. Prefer `thread_local` over other ways of defining thread-local data.

Variables can be declared with the `thread_local` specifier:

thread\_local Foo foo = ...;

Such a variable is actually a collection of objects, so that when different threads access it, they are actually accessing different objects. `thread_local` variables are much like [static storage duration variables](#Static_and_Global_Variables) in many respects. For instance, they can be declared at namespace scope, inside functions, or as static class members, but not as ordinary class members.

`thread_local` variable instances are initialized much like static variables, except that they must be initialized separately for each thread, rather than once at program startup. This means that `thread_local` variables declared within a function are safe, but other `thread_local` variables are subject to the same initialization-order issues as static variables (and more besides).

`thread_local` variables have a subtle destruction-order issue: during thread shutdown, `thread_local` variables will be destroyed in the opposite order of their initialization (as is generally true in C++). If code triggered by the destructor of any `thread_local` variable refers to any already-destroyed `thread_local` on that thread, we will get a particularly hard to diagnose use-after-free.

*   Thread-local data is inherently safe from races (because only one thread can ordinarily access it), which makes `thread_local` useful for concurrent programming.
*   `thread_local` is the only standard-supported way of creating thread-local data.

*   Accessing a `thread_local` variable may trigger execution of an unpredictable and uncontrollable amount of other code during thread-start or first use on a given thread.
*   `thread_local` variables are effectively global variables, and have all the drawbacks of global variables other than lack of thread-safety.
*   The memory consumed by a `thread_local` variable scales with the number of running threads (in the worst case), which can be quite large in a program.
*   Data members cannot be `thread_local` unless they are also `static`.
*   We may suffer from use-after-free bugs if `thread_local` variables have complex destructors. In particular, the destructor of any such variable must not call any code (transitively) that refers to any potentially-destroyed `thread_local`. This property is hard to enforce.
*   Approaches for avoiding use-after-free in global/static contexts do not work for `thread_local`s. Specifically, skipping destructors for globals and static variables is allowable because their lifetimes end at program shutdown. Thus, any "leak" is managed immediately by the OS cleaning up our memory and other resources. By contrast, skipping destructors for `thread_local` variables leads to resource leaks proportional to the total number of threads that terminate during the lifetime of the program.

`thread_local` variables at class or namespace scope must be initialized with a true compile-time constant (i.e., they must have no dynamic initialization). To enforce this, `thread_local` variables at class or namespace scope must be annotated with [`constinit`](https://en.cppreference.com/w/cpp/language/constinit) (or `constexpr`, but that should be rare):

   constinit thread\_local Foo foo = ...;
  

`thread_local` variables inside a function have no initialization concerns, but still risk use-after-free during thread exit. Note that you can use a function-scope `thread_local` to simulate a class- or namespace-scope `thread_local` by defining a function or static method that exposes it:

Foo& MyThreadLocalFoo() {
  thread\_local Foo result = ComplicatedInitialization();
  return result;
}

Note that `thread_local` variables will be destroyed whenever a thread exits. If the destructor of any such variable refers to any other (potentially-destroyed) `thread_local` we will suffer from hard to diagnose use-after-free bugs. Prefer trivial types, or types that provably run no user-provided code at destruction to minimize the potential of accessing any other `thread_local`.

`thread_local` should be preferred over other mechanisms for defining thread-local data.

Classes
-------

Classes are the fundamental unit of code in C++. Naturally, we use them extensively. This section lists the main dos and don'ts you should follow when writing a class.

### Doing Work in Constructors

Avoid virtual method calls in constructors, and avoid initialization that can fail if you can't signal an error.

It is possible to perform arbitrary initialization in the body of the constructor.

*   No need to worry about whether the class has been initialized or not.
*   Objects that are fully initialized by constructor call can be `const` and may also be easier to use with standard containers or algorithms.

*   If the work calls virtual functions, these calls will not get dispatched to the subclass implementations. Future modification to your class can quietly introduce this problem even if your class is not currently subclassed, causing much confusion.
*   There is no easy way for constructors to signal errors, short of crashing the program (not always appropriate) or using exceptions (which are [forbidden](#Exceptions)).
*   If the work fails, we now have an object whose initialization code failed, so it may be an unusual state requiring a `bool IsValid()` state checking mechanism (or similar) which is easy to forget to call.
*   You cannot take the address of a constructor, so whatever work is done in the constructor cannot easily be handed off to, for example, another thread.

Constructors should never call virtual functions. If appropriate for your code , terminating the program may be an appropriate error handling response. Otherwise, consider a factory function or `Init()` method as described in [TotW #42](https://abseil.io/tips/42). Avoid `Init()` methods on objects with no other states that affect which public methods may be called (semi-constructed objects of this form are particularly hard to work with correctly).

### Implicit Conversions

Do not define implicit conversions. Use the `explicit` keyword for conversion operators and single-argument constructors.

Implicit conversions allow an object of one type (called the source type) to be used where a different type (called the destination type) is expected, such as when passing an `int` argument to a function that takes a `double` parameter.

In addition to the implicit conversions defined by the language, users can define their own, by adding appropriate members to the class definition of the source or destination type. An implicit conversion in the source type is defined by a type conversion operator named after the destination type (e.g., `operator bool()`). An implicit conversion in the destination type is defined by a constructor that can take the source type as its only argument (or only argument with no default value).

The `explicit` keyword can be applied to a constructor or a conversion operator, to ensure that it can only be used when the destination type is explicit at the point of use, e.g., with a cast. This applies not only to implicit conversions, but to list initialization syntax:

class Foo {
  explicit Foo(int x, double y);
  ...
};

void Func(Foo f);

Func({42, 3.14});  // Error

This kind of code isn't technically an implicit conversion, but the language treats it as one as far as `explicit` is concerned.

*   Implicit conversions can make a type more usable and expressive by eliminating the need to explicitly name a type when it's obvious.
*   Implicit conversions can be a simpler alternative to overloading, such as when a single function with a `string_view` parameter takes the place of separate overloads for `std::string` and `const char*`.
*   List initialization syntax is a concise and expressive way of initializing objects.

*   Implicit conversions can hide type-mismatch bugs, where the destination type does not match the user's expectation, or the user is unaware that any conversion will take place.
*   Implicit conversions can make code harder to read, particularly in the presence of overloading, by making it less obvious what code is actually getting called.
*   Constructors that take a single argument may accidentally be usable as implicit type conversions, even if they are not intended to do so.
*   When a single-argument constructor is not marked `explicit`, there's no reliable way to tell whether it's intended to define an implicit conversion, or the author simply forgot to mark it.
*   Implicit conversions can lead to call-site ambiguities, especially when there are bidirectional implicit conversions. This can be caused either by having two types that both provide an implicit conversion, or by a single type that has both an implicit constructor and an implicit type conversion operator.
*   List initialization can suffer from the same problems if the destination type is implicit, particularly if the list has only a single element.

Type conversion operators, and constructors that are callable with a single argument, must be marked `explicit` in the class definition. As an exception, copy and move constructors should not be `explicit`, since they do not perform type conversion.

Implicit conversions can sometimes be necessary and appropriate for types that are designed to be interchangeable, for example when objects of two types are just different representations of the same underlying value. In that case, contact your project leads to request a waiver of this rule.

Constructors that cannot be called with a single argument may omit `explicit`. Constructors that take a single `std::initializer_list` parameter should also omit `explicit`, in order to support copy-initialization (e.g., `MyType m = {1, 2};`).

### Copyable and Movable Types

A class's public API must make clear whether the class is copyable, move-only, or neither copyable nor movable. Support copying and/or moving if these operations are clear and meaningful for your type.

A movable type is one that can be initialized and assigned from temporaries.

A copyable type is one that can be initialized or assigned from any other object of the same type (so is also movable by definition), with the stipulation that the value of the source does not change. `std::unique_ptr<int>` is an example of a movable but not copyable type (since the value of the source `std::unique_ptr<int>` must be modified during assignment to the destination). `int` and `std::string` are examples of movable types that are also copyable. (For `int`, the move and copy operations are the same; for `std::string`, there exists a move operation that is less expensive than a copy.)

For user-defined types, the copy behavior is defined by the copy constructor and the copy-assignment operator. Move behavior is defined by the move constructor and the move-assignment operator, if they exist, or by the copy constructor and the copy-assignment operator otherwise.

The copy/move constructors can be implicitly invoked by the compiler in some situations, e.g., when passing objects by value.

Objects of copyable and movable types can be passed and returned by value, which makes APIs simpler, safer, and more general. Unlike when passing objects by pointer or reference, there's no risk of confusion over ownership, lifetime, mutability, and similar issues, and no need to specify them in the contract. It also prevents non-local interactions between the client and the implementation, which makes them easier to understand, maintain, and optimize by the compiler. Further, such objects can be used with generic APIs that require pass-by-value, such as most containers, and they allow for additional flexibility in e.g., type composition.

Copy/move constructors and assignment operators are usually easier to define correctly than alternatives like `Clone()`, `CopyFrom()` or `Swap()`, because they can be generated by the compiler, either implicitly or with `= default`. They are concise, and ensure that all data members are copied. Copy and move constructors are also generally more efficient, because they don't require heap allocation or separate initialization and assignment steps, and they're eligible for optimizations such as [copy elision](http://en.cppreference.com/w/cpp/language/copy_elision).

Move operations allow the implicit and efficient transfer of resources out of rvalue objects. This allows a plainer coding style in some cases.

Some types do not need to be copyable, and providing copy operations for such types can be confusing, nonsensical, or outright incorrect. Types representing singleton objects (`Registerer`), objects tied to a specific scope (`Cleanup`), or closely coupled to object identity (`Mutex`) cannot be copied meaningfully. Copy operations for base class types that are to be used polymorphically are hazardous, because use of them can lead to [object slicing](https://en.wikipedia.org/wiki/Object_slicing). Defaulted or carelessly-implemented copy operations can be incorrect, and the resulting bugs can be confusing and difficult to diagnose.

Copy constructors are invoked implicitly, which makes the invocation easy to miss. This may cause confusion for programmers used to languages where pass-by-reference is conventional or mandatory. It may also encourage excessive copying, which can cause performance problems.

Every class's public interface must make clear which copy and move operations the class supports. This should usually take the form of explicitly declaring and/or deleting the appropriate operations in the `public` section of the declaration.

Specifically, a copyable class should explicitly declare the copy operations, a move-only class should explicitly declare the move operations, and a non-copyable/movable class should explicitly delete the copy operations. A copyable class may also declare move operations in order to support efficient moves. Explicitly declaring or deleting all four copy/move operations is permitted, but not required. If you provide a copy or move assignment operator, you must also provide the corresponding constructor.

class Copyable {
 public:
  Copyable(const Copyable& other) = default;
  Copyable& operator=(const Copyable& other) = default;

  // The implicit move operations are suppressed by the declarations above.
  // You may explicitly declare move operations to support efficient moves.
};

class MoveOnly {
 public:
  MoveOnly(MoveOnly&& other) = default;
  MoveOnly& operator=(MoveOnly&& other) = default;

  // The copy operations are implicitly deleted, but you can
  // spell that out explicitly if you want:
  MoveOnly(const MoveOnly&) = delete;
  MoveOnly& operator=(const MoveOnly&) = delete;
};

class NotCopyableOrMovable {
 public:
  // Not copyable or movable
  NotCopyableOrMovable(const NotCopyableOrMovable&) = delete;
  NotCopyableOrMovable& operator=(const NotCopyableOrMovable&)
      = delete;

  // The move operations are implicitly disabled, but you can
  // spell that out explicitly if you want:
  NotCopyableOrMovable(NotCopyableOrMovable&&) = delete;
  NotCopyableOrMovable& operator=(NotCopyableOrMovable&&)
      = delete;
};

These declarations/deletions can be omitted only if they are obvious:

*   If the class has no `private` section, like a [struct](#Structs_vs._Classes) or an interface-only base class, then the copyability/movability can be determined by the copyability/movability of any public data members.
*   If a base class clearly isn't copyable or movable, derived classes naturally won't be either. An interface-only base class that leaves these operations implicit is not sufficient to make concrete subclasses clear.
*   Note that if you explicitly declare or delete either the constructor or assignment operation for copy, the other copy operation is not obvious and must be declared or deleted. Likewise for move operations.

A type should not be copyable/movable if the meaning of copying/moving is unclear to a casual user, or if it incurs unexpected costs. Move operations for copyable types are strictly a performance optimization and are a potential source of bugs and complexity, so avoid defining them unless they are significantly more efficient than the corresponding copy operations. If your type provides copy operations, it is recommended that you design your class so that the default implementation of those operations is correct. Remember to review the correctness of any defaulted operations as you would any other code.

To eliminate the risk of slicing, prefer to make base classes abstract, by making their constructors protected, by declaring their destructors protected, or by giving them one or more pure virtual member functions. Prefer to avoid deriving from concrete classes.

### Structs vs. Classes

Use a `struct` only for passive objects that carry data; everything else is a `class`.

The `struct` and `class` keywords behave almost identically in C++. We add our own semantic meanings to each keyword, so you should use the appropriate keyword for the data-type you're defining.

`structs` should be used for passive objects that carry data, and may have associated constants. All fields must be public. The struct must not have invariants that imply relationships between different fields, since direct user access to those fields may break those invariants. Constructors, destructors, and helper methods may be present; however, these methods must not require or enforce any invariants.

If more functionality or invariants are required, or struct has wide visibility and expected to evolve, then a `class` is more appropriate. If in doubt, make it a `class`.

For consistency with STL, you can use `struct` instead of `class` for stateless types, such as traits, [template metafunctions](#Template_metaprogramming), and some functors.

Note that member variables in structs and classes have [different naming rules](#Variable_Names).

### Structs vs. Pairs and Tuples

Prefer to use a `struct` instead of a pair or a tuple whenever the elements can have meaningful names.

While using pairs and tuples can avoid the need to define a custom type, potentially saving work when _writing_ code, a meaningful field name will almost always be much clearer when _reading_ code than `.first`, `.second`, or `std::get<X>`. While C++14's introduction of `std::get<Type>` to access a tuple element by type rather than index (when the type is unique) can sometimes partially mitigate this, a field name is usually substantially clearer and more informative than a type.

Pairs and tuples may be appropriate in generic code where there are not specific meanings for the elements of the pair or tuple. Their use may also be required in order to interoperate with existing code or APIs.

### Inheritance

Composition is often more appropriate than inheritance. When using inheritance, make it `public`.

When a sub-class inherits from a base class, it includes the definitions of all the data and operations that the base class defines. "Interface inheritance" is inheritance from a pure abstract base class (one with no state or defined methods); all other inheritance is "implementation inheritance".

Implementation inheritance reduces code size by re-using the base class code as it specializes an existing type. Because inheritance is a compile-time declaration, you and the compiler can understand the operation and detect errors. Interface inheritance can be used to programmatically enforce that a class expose a particular API. Again, the compiler can detect errors, in this case, when a class does not define a necessary method of the API.

For implementation inheritance, because the code implementing a sub-class is spread between the base and the sub-class, it can be more difficult to understand an implementation. The sub-class cannot override functions that are not virtual, so the sub-class cannot change implementation.

Multiple inheritance is especially problematic, because it often imposes a higher performance overhead (in fact, the performance drop from single inheritance to multiple inheritance can often be greater than the performance drop from ordinary to virtual dispatch), and because it risks leading to "diamond" inheritance patterns, which are prone to ambiguity, confusion, and outright bugs.

All inheritance should be `public`. If you want to do private inheritance, you should be including an instance of the base class as a member instead. You may use `final` on classes when you don't intend to support using them as base classes.

Do not overuse implementation inheritance. Composition is often more appropriate. Try to restrict use of inheritance to the "is-a" case: `Bar` subclasses `Foo` if it can reasonably be said that `Bar` "is a kind of" `Foo`.

Limit the use of `protected` to those member functions that might need to be accessed from subclasses. Note that [data members should be `private`](#Access_Control).

Explicitly annotate overrides of virtual functions or virtual destructors with exactly one of an `override` or (less frequently) `final` specifier. Do not use `virtual` when declaring an override. Rationale: A function or destructor marked `override` or `final` that is not an override of a base class virtual function will not compile, and this helps catch common errors. The specifiers serve as documentation; if no specifier is present, the reader has to check all ancestors of the class in question to determine if the function or destructor is virtual or not.

Multiple inheritance is permitted, but multiple _implementation_ inheritance is strongly discouraged.

### Operator Overloading

Overload operators judiciously. Do not use user-defined literals.

C++ permits user code to [declare overloaded versions of the built-in operators](http://en.cppreference.com/w/cpp/language/operators) using the `operator` keyword, so long as one of the parameters is a user-defined type. The `operator` keyword also permits user code to define new kinds of literals using `operator""`, and to define type-conversion functions such as `operator bool()`.

Operator overloading can make code more concise and intuitive by enabling user-defined types to behave the same as built-in types. Overloaded operators are the idiomatic names for certain operations (e.g., `==`, `<`, `=`, and `<<`), and adhering to those conventions can make user-defined types more readable and enable them to interoperate with libraries that expect those names.

User-defined literals are a very concise notation for creating objects of user-defined types.

*   Providing a correct, consistent, and unsurprising set of operator overloads requires some care, and failure to do so can lead to confusion and bugs.
*   Overuse of operators can lead to obfuscated code, particularly if the overloaded operator's semantics don't follow convention.
*   The hazards of function overloading apply just as much to operator overloading, if not more so.
*   Operator overloads can fool our intuition into thinking that expensive operations are cheap, built-in operations.
*   Finding the call sites for overloaded operators may require a search tool that's aware of C++ syntax, rather than e.g., grep.
*   If you get the argument type of an overloaded operator wrong, you may get a different overload rather than a compiler error. For example, `foo < bar` may do one thing, while `&foo < &bar` does something totally different.
*   Certain operator overloads are inherently hazardous. Overloading unary `&` can cause the same code to have different meanings depending on whether the overload declaration is visible. Overloads of `&&`, `||`, and `,` (comma) cannot match the evaluation-order semantics of the built-in operators.
*   Operators are often defined outside the class, so there's a risk of different files introducing different definitions of the same operator. If both definitions are linked into the same binary, this results in undefined behavior, which can manifest as subtle run-time bugs.
*   User-defined literals (UDLs) allow the creation of new syntactic forms that are unfamiliar even to experienced C++ programmers, such as `"Hello World"sv` as a shorthand for `std::string_view("Hello World")`. Existing notations are clearer, though less terse.
*   Because they can't be namespace-qualified, uses of UDLs also require use of either using-directives (which [we ban](#Namespaces)) or using-declarations (which [we ban in header files](#Aliases) except when the imported names are part of the interface exposed by the header file in question). Given that header files would have to avoid UDL suffixes, we prefer to avoid having conventions for literals differ between header files and source files.

Define overloaded operators only if their meaning is obvious, unsurprising, and consistent with the corresponding built-in operators. For example, use `|` as a bitwise- or logical-or, not as a shell-style pipe.

Define operators only on your own types. More precisely, define them in the same headers, `.cc` files, and namespaces as the types they operate on. That way, the operators are available wherever the type is, minimizing the risk of multiple definitions. If possible, avoid defining operators as templates, because they must satisfy this rule for any possible template arguments. If you define an operator, also define any related operators that make sense, and make sure they are defined consistently.

Prefer to define non-modifying binary operators as non-member functions. If a binary operator is defined as a class member, implicit conversions will apply to the right-hand argument, but not the left-hand one. It will confuse your users if `a + b` compiles but `b + a` doesn't.

For a type `T` whose values can be compared for equality, define a non-member `operator==` and document when two values of type `T` are considered equal. If there is a single obvious notion of when a value `t1` of type `T` is less than another such value `t2` then you may also define `operator<=>`, which should be consistent with `operator==`. Prefer not to overload the other comparison and ordering operators.

Don't go out of your way to avoid defining operator overloads. For example, prefer to define `==`, `=`, and `<<`, rather than `Equals()`, `CopyFrom()`, and `PrintTo()`. Conversely, don't define operator overloads just because other libraries expect them. For example, if your type doesn't have a natural ordering, but you want to store it in a `std::set`, use a custom comparator rather than overloading `<`.

Do not overload `&&`, `||`, `,` (comma), or unary `&`. Do not overload `operator""`, i.e., do not introduce user-defined literals. Do not use any such literals provided by others (including the standard library).

Type conversion operators are covered in the section on [implicit conversions](#Implicit_Conversions). The `=` operator is covered in the section on [copy constructors](#Copy_Constructors). Overloading `<<` for use with streams is covered in the section on [streams](#Streams). See also the rules on [function overloading](#Function_Overloading), which apply to operator overloading as well.

### Access Control

Make classes' data members `private`, unless they are [constants](#Constant_Names). This simplifies reasoning about invariants, at the cost of some easy boilerplate in the form of accessors (usually `const`) if necessary.

For technical reasons, we allow data members of a test fixture class defined in a `.cc` file to be `protected` when using [Google Test](https://github.com/google/googletest). If a test fixture class is defined outside of the `.cc` file it is used in, for example in a `.h` file, make data members `private`.

### Declaration Order

Group similar declarations together, placing `public` parts earlier.

A class definition should usually start with a `public:` section, followed by `protected:`, then `private:`. Omit sections that would be empty.

Within each section, prefer grouping similar kinds of declarations together, and prefer the following order:

1.  Types and type aliases (`typedef`, `using`, `enum`, nested structs and classes, and `friend` types)
2.  (Optionally, for structs only) non-`static` data members
3.  Static constants
4.  Factory functions
5.  Constructors and assignment operators
6.  Destructor
7.  All other functions (`static` and non-`static` member functions, and `friend` functions)
8.  All other data members (static and non-static)

Do not put large method definitions inline in the class definition. Usually, only trivial or performance-critical, and very short, methods may be defined inline. See [Inline Functions](#Inline_Functions) for more details.

Functions
---------

### Inputs and Outputs

The output of a C++ function is naturally provided via a return value and sometimes via output parameters (or in/out parameters).

Prefer using return values over output parameters: they improve readability, and often provide the same or better performance.

Prefer to return by value or, failing that, return by reference. Avoid returning a raw pointer unless it can be null.

Parameters are either inputs to the function, outputs from the function, or both. Non-optional input parameters should usually be values or `const` references, while non-optional output and input/output parameters should usually be references (which cannot be null). Generally, use `std::optional` to represent optional by-value inputs, and use a `const` pointer when the non-optional form would have used a reference. Use non-`const` pointers to represent optional outputs and optional input/output parameters.

Avoid defining functions that require a reference parameter to outlive the call. In some cases reference parameters can bind to temporaries, leading to lifetime bugs. Instead, find a way to eliminate the lifetime requirement (for example, by copying the parameter), or pass retained parameters by pointer and document the lifetime and non-null requirements.

When ordering function parameters, put all input-only parameters before any output parameters. In particular, do not add new parameters to the end of the function just because they are new; place new input-only parameters before the output parameters. This is not a hard-and-fast rule. Parameters that are both input and output muddy the waters, and, as always, consistency with related functions may require you to bend the rule. Variadic functions may also require unusual parameter ordering.

### Write Short Functions

Prefer small and focused functions.

We recognize that long functions are sometimes appropriate, so no hard limit is placed on functions length. If a function exceeds about 40 lines, think about whether it can be broken up without harming the structure of the program.

Even if your long function works perfectly now, someone modifying it in a few months may add new behavior. This could result in bugs that are hard to find. Keeping your functions short and simple makes it easier for other people to read and modify your code. Small functions are also easier to test.

You could find long and complicated functions when working with some code. Do not be intimidated by modifying existing code: if working with such a function proves to be difficult, you find that errors are hard to debug, or you want to use a piece of it in several different contexts, consider breaking up the function into smaller and more manageable pieces.

### Function Overloading

Use overloaded functions (including constructors) only if a reader looking at a call site can get a good idea of what is happening without having to first figure out exactly which overload is being called.

You may write a function that takes a `const std::string&` and overload it with another that takes `const char*`. However, in this case consider `std::string_view` instead.

class MyClass {
 public:
  void Analyze(const std::string &text);
  void Analyze(const char \*text, size\_t textlen);
};

Overloading can make code more intuitive by allowing an identically-named function to take different arguments. It may be necessary for templatized code, and it can be convenient for Visitors.

Overloading based on `const` or ref qualification may make utility code more usable, more efficient, or both. (See [TotW 148](http://abseil.io/tips/148) for more.)

If a function is overloaded by the argument types alone, a reader may have to understand C++'s complex matching rules in order to tell what's going on. Also many people are confused by the semantics of inheritance if a derived class overrides only some of the variants of a function.

You may overload a function when there are no semantic differences between variants. These overloads may vary in types, qualifiers, or argument count. However, a reader of such a call must not need to know which member of the overload set is chosen, only that **something** from the set is being called. If you can document all entries in the overload set with a single comment in the header, that is a good sign that it is a well-designed overload set.

### Default Arguments

Default arguments are allowed on non-virtual functions when the default is guaranteed to always have the same value. Follow the same restrictions as for [function overloading](#Function_Overloading), and prefer overloaded functions if the readability gained with default arguments doesn't outweigh the downsides below.

Often you have a function that uses default values, but occasionally you want to override the defaults. Default parameters allow an easy way to do this without having to define many functions for the rare exceptions. Compared to overloading the function, default arguments have a cleaner syntax, with less boilerplate and a clearer distinction between 'required' and 'optional' arguments.

Defaulted arguments are another way to achieve the semantics of overloaded functions, so all the [reasons not to overload functions](#Function_Overloading) apply.

The defaults for arguments in a virtual function call are determined by the static type of the target object, and there's no guarantee that all overrides of a given function declare the same defaults.

Default parameters are re-evaluated at each call site, which can bloat the generated code. Readers may also expect the default's value to be fixed at the declaration instead of varying at each call.

Function pointers are confusing in the presence of default arguments, since the function signature often doesn't match the call signature. Adding function overloads avoids these problems.

Default arguments are banned on virtual functions, where they don't work properly, and in cases where the specified default might not evaluate to the same value depending on when it was evaluated. (For example, don't write `void f(int n = counter++);`.)

In some other cases, default arguments can improve the readability of their function declarations enough to overcome the downsides above, so they are allowed. When in doubt, use overloads.

### Trailing Return Type Syntax

Use trailing return types only where using the ordinary syntax (leading return types) is impractical or much less readable.

C++ allows two different forms of function declarations. In the older form, the return type appears before the function name. For example:

int foo(int x);

The newer form uses the `auto` keyword before the function name and a trailing return type after the argument list. For example, the declaration above could equivalently be written:

auto foo(int x) -> int;

The trailing return type is in the function's scope. This doesn't make a difference for a simple case like `int` but it matters for more complicated cases, like types declared in class scope or types written in terms of the function parameters.

Trailing return types are the only way to explicitly specify the return type of a [lambda expression](#Lambda_expressions). In some cases the compiler is able to deduce a lambda's return type, but not in all cases. Even when the compiler can deduce it automatically, sometimes specifying it explicitly would be clearer for readers.

Sometimes it's easier and more readable to specify a return type after the function's parameter list has already appeared. This is particularly true when the return type depends on template parameters. For example:

    template <typename T, typename U>
    auto add(T t, U u) -> decltype(t + u);
  

versus

    template <typename T, typename U>
    decltype(declval<T&>() + declval<U&>()) add(T t, U u);
  

Trailing return type syntax is relatively new and it has no analogue in C++-like languages such as C and Java, so some readers may find it unfamiliar.

Existing code bases have an enormous number of function declarations that aren't going to get changed to use the new syntax, so the realistic choices are using the old syntax only or using a mixture of the two. Using a single version is better for uniformity of style.

In most cases, continue to use the older style of function declaration where the return type goes before the function name. Use the new trailing-return-type form only in cases where it's required (such as lambdas) or where, by putting the type after the function's parameter list, it allows you to write the type in a much more readable way. The latter case should be rare; it's mostly an issue in fairly complicated template code, which is [discouraged in most cases](#Template_metaprogramming).

Google-Specific Magic
---------------------

There are various tricks and utilities that we use to make C++ code more robust, and various ways we use C++ that may differ from what you see elsewhere.

### Ownership and Smart Pointers

Prefer to have single, fixed owners for dynamically allocated objects. Prefer to transfer ownership with smart pointers.

"Ownership" is a bookkeeping technique for managing dynamically allocated memory (and other resources). The owner of a dynamically allocated object is an object or function that is responsible for ensuring that it is deleted when no longer needed. Ownership can sometimes be shared, in which case the last owner is typically responsible for deleting it. Even when ownership is not shared, it can be transferred from one piece of code to another.

"Smart" pointers are classes that act like pointers, e.g., by overloading the `*` and `->` operators. Some smart pointer types can be used to automate ownership bookkeeping, to ensure these responsibilities are met. [`std::unique_ptr`](http://en.cppreference.com/w/cpp/memory/unique_ptr) is a smart pointer type which expresses exclusive ownership of a dynamically allocated object; the object is deleted when the `std::unique_ptr` goes out of scope. It cannot be copied, but can be _moved_ to represent ownership transfer. [`std::shared_ptr`](http://en.cppreference.com/w/cpp/memory/shared_ptr) is a smart pointer type that expresses shared ownership of a dynamically allocated object. `std::shared_ptr`s can be copied; ownership of the object is shared among all copies, and the object is deleted when the last `std::shared_ptr` is destroyed.

*   It's virtually impossible to manage dynamically allocated memory without some sort of ownership logic.
*   Transferring ownership of an object can be cheaper than copying it (if copying it is even possible).
*   Transferring ownership can be simpler than 'borrowing' a pointer or reference, because it reduces the need to coordinate the lifetime of the object between the two users.
*   Smart pointers can improve readability by making ownership logic explicit, self-documenting, and unambiguous.
*   Smart pointers can eliminate manual ownership bookkeeping, simplifying the code and ruling out large classes of errors.
*   For `const` objects, shared ownership can be a simple and efficient alternative to deep copying.

*   Ownership must be represented and transferred via pointers (whether smart or plain). Pointer semantics are more complicated than value semantics, especially in APIs: you have to worry not just about ownership, but also aliasing, lifetime, and mutability, among other issues.
*   The performance costs of value semantics are often overestimated, so the performance benefits of ownership transfer might not justify the readability and complexity costs.
*   APIs that transfer ownership force their clients into a single memory management model.
*   Code using smart pointers is less explicit about where the resource releases take place.
*   `std::unique_ptr` expresses ownership transfer using move semantics, which are relatively new and may confuse some programmers.
*   Shared ownership can be a tempting alternative to careful ownership design, obfuscating the design of a system.
*   Shared ownership requires explicit bookkeeping at run-time, which can be costly.
*   In some cases (e.g., cyclic references), objects with shared ownership may never be deleted.
*   Smart pointers are not perfect substitutes for plain pointers.

If dynamic allocation is necessary, prefer to keep ownership with the code that allocated it. If other code needs access to the object, consider passing it a copy, or passing a pointer or reference without transferring ownership. Prefer to use `std::unique_ptr` to make ownership transfer explicit. For example:

std::unique\_ptr<Foo> FooFactory();
void FooConsumer(std::unique\_ptr<Foo> ptr);

Do not design your code to use shared ownership without a very good reason. One such reason is to avoid expensive copy operations, but you should only do this if the performance benefits are significant, and the underlying object is immutable (i.e., `std::shared_ptr<const Foo>`). If you do use shared ownership, prefer to use `std::shared_ptr`.

Never use `std::auto_ptr`. Instead, use `std::unique_ptr`.

### cpplint

Use `cpplint.py` to detect style errors.

`cpplint.py` is a tool that reads a source file and identifies many style errors. It is not perfect, and has both false positives and false negatives, but it is still a valuable tool.

Some projects have instructions on how to run `cpplint.py` from their project tools. If the project you are contributing to does not, you can download [`cpplint.py`](https://raw.githubusercontent.com/google/styleguide/gh-pages/cpplint/cpplint.py) separately.

Other C++ Features
------------------

### Rvalue References

Use rvalue references only in certain special cases listed below.

Rvalue references are a type of reference that can only bind to temporary objects. The syntax is similar to traditional reference syntax. For example, `void f(std::string&& s);` declares a function whose argument is an rvalue reference to a `std::string`.

When the token '&&' is applied to an unqualified template argument in a function parameter, special template argument deduction rules apply. Such a reference is called a forwarding reference.

*   Defining a move constructor (a constructor taking an rvalue reference to the class type) makes it possible to move a value instead of copying it. If `v1` is a `std::vector<std::string>`, for example, then `auto v2(std::move(v1))` will probably just result in some simple pointer manipulation instead of copying a large amount of data. In many cases this can result in a major performance improvement.
*   Rvalue references make it possible to implement types that are movable but not copyable, which can be useful for types that have no sensible definition of copying but where you might still want to pass them as function arguments, put them in containers, etc.
*   `std::move` is necessary to make effective use of some standard-library types, such as `std::unique_ptr`.
*   [Forwarding references](#Forwarding_references) which use the rvalue reference token, make it possible to write a generic function wrapper that forwards its arguments to another function, and works whether or not its arguments are temporary objects and/or const. This is called 'perfect forwarding'.

*   Rvalue references are not yet widely understood. Rules like reference collapsing and the special deduction rule for forwarding references are somewhat obscure.
*   Rvalue references are often misused. Using rvalue references is counter-intuitive in signatures where the argument is expected to have a valid specified state after the function call, or where no move operation is performed.

Do not use rvalue references (or apply the `&&` qualifier to methods), except as follows:

*   You may use them to define move constructors and move assignment operators (as described in [Copyable and Movable Types](#Copyable_Movable_Types)).
*   You may use them to define `&&`\-qualified methods that logically "consume" `*this`, leaving it in an unusable or empty state. Note that this applies only to method qualifiers (which come after the closing parenthesis of the function signature); if you want to "consume" an ordinary function parameter, prefer to pass it by value.
*   You may use forwarding references in conjunction with `[std::forward](http://en.cppreference.com/w/cpp/utility/forward)`, to support perfect forwarding.
*   You may use them to define pairs of overloads, such as one taking `Foo&&` and the other taking `const Foo&`. Usually the preferred solution is just to pass by value, but an overloaded pair of functions sometimes yields better performance, for example if the functions sometimes don't consume the input. As always: if you're writing more complicated code for the sake of performance, make sure you have evidence that it actually helps.

### Friends

We allow use of `friend` classes and functions, within reason.

Friends should usually be defined in the same file so that the reader does not have to look in another file to find uses of the private members of a class. A common use of `friend` is to have a `FooBuilder` class be a friend of `Foo` so that it can construct the inner state of `Foo` correctly, without exposing this state to the world. In some cases it may be useful to make a unittest class a friend of the class it tests.

Friends extend, but do not break, the encapsulation boundary of a class. In some cases this is better than making a member `public` when you want to give only one other class access to it. However, most classes should interact with other classes solely through their public members.

### Exceptions

We do not use C++ exceptions.

*   Exceptions allow higher levels of an application to decide how to handle "can't happen" failures in deeply nested functions, without the obscuring and error-prone bookkeeping of error codes.

*   Exceptions are used by most other modern languages. Using them in C++ would make it more consistent with Python, Java, and the C++ that others are familiar with.

*   Some third-party C++ libraries use exceptions, and turning them off internally makes it harder to integrate with those libraries.
*   Exceptions are the only way for a constructor to fail. We can simulate this with a factory function or an `Init()` method, but these require heap allocation or a new "invalid" state, respectively.
*   Exceptions are really handy in testing frameworks.

*   When you add a `throw` statement to an existing function, you must examine all of its transitive callers. Either they must make at least the basic exception safety guarantee, or they must never catch the exception and be happy with the program terminating as a result. For instance, if `f()` calls `g()` calls `h()`, and `h` throws an exception that `f` catches, `g` has to be careful or it may not clean up properly.
*   More generally, exceptions make the control flow of programs difficult to evaluate by looking at code: functions may return in places you don't expect. This causes maintainability and debugging difficulties. You can minimize this cost via some rules on how and where exceptions can be used, but at the cost of more that a developer needs to know and understand.
*   Exception safety requires both RAII and different coding practices. Lots of supporting machinery is needed to make writing correct exception-safe code easy. Further, to avoid requiring readers to understand the entire call graph, exception-safe code must isolate logic that writes to persistent state into a "commit" phase. This will have both benefits and costs (perhaps where you're forced to obfuscate code to isolate the commit). Allowing exceptions would force us to always pay those costs even when they're not worth it.
*   Turning on exceptions adds data to each binary produced, increasing compile time (probably slightly) and possibly increasing address space pressure.
*   The availability of exceptions may encourage developers to throw them when they are not appropriate or recover from them when it's not safe to do so. For example, invalid user input should not cause exceptions to be thrown. We would need to make the style guide even longer to document these restrictions!

On their face, the benefits of using exceptions outweigh the costs, especially in new projects. However, for existing code, the introduction of exceptions has implications on all dependent code. If exceptions can be propagated beyond a new project, it also becomes problematic to integrate the new project into existing exception-free code. Because most existing C++ code at Google is not prepared to deal with exceptions, it is comparatively difficult to adopt new code that generates exceptions.

Given that Google's existing code is not exception-tolerant, the costs of using exceptions are somewhat greater than the costs in a new project. The conversion process would be slow and error-prone. We don't believe that the available alternatives to exceptions, such as error codes and assertions, introduce a significant burden.

Our advice against using exceptions is not predicated on philosophical or moral grounds, but practical ones. Because we'd like to use our open-source projects at Google and it's difficult to do so if those projects use exceptions, we need to advise against exceptions in Google open-source projects as well. Things would probably be different if we had to do it all over again from scratch.

This prohibition also applies to exception handling related features such as `std::exception_ptr` and `std::nested_exception`.

There is an [exception](#Windows_Code) to this rule (no pun intended) for Windows code.

### `noexcept`

Specify `noexcept` when it is useful and correct.

The `noexcept` specifier is used to specify whether a function will throw exceptions or not. If an exception escapes from a function marked `noexcept`, the program crashes via `std::terminate`.

The `noexcept` operator performs a compile-time check that returns true if an expression is declared to not throw any exceptions.

*   Specifying move constructors as `noexcept` improves performance in some cases, e.g., `std::vector<T>::resize()` moves rather than copies the objects if T's move constructor is `noexcept`.
*   Specifying `noexcept` on a function can trigger compiler optimizations in environments where exceptions are enabled, e.g., compiler does not have to generate extra code for stack-unwinding, if it knows that no exceptions can be thrown due to a `noexcept` specifier.

*   In projects following this guide that have exceptions disabled it is hard to ensure that `noexcept` specifiers are correct, and hard to define what correctness even means.
*   It's hard, if not impossible, to undo `noexcept` because it eliminates a guarantee that callers may be relying on, in ways that are hard to detect.

You may use `noexcept` when it is useful for performance if it accurately reflects the intended semantics of your function, i.e., that if an exception is somehow thrown from within the function body then it represents a fatal error. You can assume that `noexcept` on move constructors has a meaningful performance benefit. If you think there is significant performance benefit from specifying `noexcept` on some other function, please discuss it with your project leads.

Prefer unconditional `noexcept` if exceptions are completely disabled (i.e., most Google C++ environments). Otherwise, use conditional `noexcept` specifiers with simple conditions, in ways that evaluate false only in the few cases where the function could potentially throw. The tests might include type traits check on whether the involved operation might throw (e.g., `std::is_nothrow_move_constructible` for move-constructing objects), or on whether allocation can throw (e.g., `absl::default_allocator_is_nothrow` for standard default allocation). Note in many cases the only possible cause for an exception is allocation failure (we believe move constructors should not throw except due to allocation failure), and there are many applications where it’s appropriate to treat memory exhaustion as a fatal error rather than an exceptional condition that your program should attempt to recover from. Even for other potential failures you should prioritize interface simplicity over supporting all possible exception throwing scenarios: instead of writing a complicated `noexcept` clause that depends on whether a hash function can throw, for example, simply document that your component doesn’t support hash functions throwing and make it unconditionally `noexcept`.

### Run-Time Type Information (RTTI)

Avoid using run-time type information (RTTI).

RTTI allows a programmer to query the C++ class of an object at run-time. This is done by use of `typeid` or `dynamic_cast`.

The standard alternatives to RTTI (described below) require modification or redesign of the class hierarchy in question. Sometimes such modifications are infeasible or undesirable, particularly in widely-used or mature code.

RTTI can be useful in some unit tests. For example, it is useful in tests of factory classes where the test has to verify that a newly created object has the expected dynamic type. It is also useful in managing the relationship between objects and their mocks.

RTTI is useful when considering multiple abstract objects. Consider

bool Base::Equal(Base\* other) = 0;
bool Derived::Equal(Base\* other) {
  Derived\* that = dynamic\_cast<Derived\*>(other);
  if (that == nullptr)
    return false;
  ...
}

Querying the type of an object at run-time frequently means a design problem. Needing to know the type of an object at runtime is often an indication that the design of your class hierarchy is flawed.

Undisciplined use of RTTI makes code hard to maintain. It can lead to type-based decision trees or switch statements scattered throughout the code, all of which must be examined when making further changes.

RTTI has legitimate uses but is prone to abuse, so you must be careful when using it. You may use it freely in unittests, but avoid it when possible in other code. In particular, think twice before using RTTI in new code. If you find yourself needing to write code that behaves differently based on the class of an object, consider one of the following alternatives to querying the type:

*   Virtual methods are the preferred way of executing different code paths depending on a specific subclass type. This puts the work within the object itself.
*   If the work belongs outside the object and instead in some processing code, consider a double-dispatch solution, such as the Visitor design pattern. This allows a facility outside the object itself to determine the type of class using the built-in type system.

When the logic of a program guarantees that a given instance of a base class is in fact an instance of a particular derived class, then a `dynamic_cast` may be used freely on the object. Usually one can use a `static_cast` as an alternative in such situations.

Decision trees based on type are a strong indication that your code is on the wrong track.

if (typeid(\*data) == typeid(D1)) {
  ...
} else if (typeid(\*data) == typeid(D2)) {
  ...
} else if (typeid(\*data) == typeid(D3)) {
...

Code such as this usually breaks when additional subclasses are added to the class hierarchy. Moreover, when properties of a subclass change, it is difficult to find and modify all the affected code segments.

Do not hand-implement an RTTI-like workaround. The arguments against RTTI apply just as much to workarounds like class hierarchies with type tags. Moreover, workarounds disguise your true intent.

### Casting

Use C++-style casts like `static_cast<float>(double_value)`, or brace initialization for conversion of arithmetic types like `int64_t y = int64_t{1} << 42`. Do not use cast formats like `(int)x` unless the cast is to `void`. You may use cast formats like `T(x)` only when `T` is a class type.

C++ introduced a different cast system from C that distinguishes the types of cast operations.

The problem with C casts is the ambiguity of the operation; sometimes you are doing a _conversion_ (e.g., `(int)3.5`) and sometimes you are doing a _cast_ (e.g., `(int)"hello"`). Brace initialization and C++ casts can often help avoid this ambiguity. Additionally, C++ casts are more visible when searching for them.

The C++-style cast syntax is verbose and cumbersome.

In general, do not use C-style casts. Instead, use these C++-style casts when explicit type conversion is necessary.

*   Use brace initialization to convert arithmetic types (e.g., `int64_t{x}`). This is the safest approach because code will not compile if conversion can result in information loss. The syntax is also concise.
*   Use `absl::implicit_cast` to safely cast up a type hierarchy, e.g., casting a `Foo*` to a `SuperclassOfFoo*` or casting a `Foo*` to a `const Foo*`. C++ usually does this automatically but some situations need an explicit up-cast, such as use of the `?:` operator.
*   Use `static_cast` as the equivalent of a C-style cast that does value conversion, when you need to explicitly up-cast a pointer from a class to its superclass, or when you need to explicitly cast a pointer from a superclass to a subclass. In this last case, you must be sure your object is actually an instance of the subclass.
*   Use `const_cast` to remove the `const` qualifier (see [const](#Use_of_const)).
*   Use `reinterpret_cast` to do unsafe conversions of pointer types to and from integer and other pointer types, including `void*`. Use this only if you know what you are doing and you understand the aliasing issues. Also, consider dereferencing the pointer (without a cast) and using `std::bit_cast` to cast the resulting value.
*   Use `std::bit_cast` to interpret the raw bits of a value using a different type of the same size (a type pun), such as interpreting the bits of a `double` as `int64_t`.

See the [RTTI section](#Run-Time_Type_Information__RTTI_) for guidance on the use of `dynamic_cast`.

### Streams

Use streams where appropriate, and stick to "simple" usages. Overload `<<` for streaming only for types representing values, and write only the user-visible value, not any implementation details.

Streams are the standard I/O abstraction in C++, as exemplified by the standard header `<iostream>`. They are widely used in Google code, mostly for debug logging and test diagnostics.

The `<<` and `>>` stream operators provide an API for formatted I/O that is easily learned, portable, reusable, and extensible. `printf`, by contrast, doesn't even support `std::string`, to say nothing of user-defined types, and is very difficult to use portably. `printf` also obliges you to choose among the numerous slightly different versions of that function, and navigate the dozens of conversion specifiers.

Streams provide first-class support for console I/O via `std::cin`, `std::cout`, `std::cerr`, and `std::clog`. The C APIs do as well, but are hampered by the need to manually buffer the input.

*   Stream formatting can be configured by mutating the state of the stream. Such mutations are persistent, so the behavior of your code can be affected by the entire previous history of the stream, unless you go out of your way to restore it to a known state every time other code might have touched it. User code can not only modify the built-in state, it can add new state variables and behaviors through a registration system.
*   It is difficult to precisely control stream output, due to the above issues, the way code and data are mixed in streaming code, and the use of operator overloading (which may select a different overload than you expect).
*   The practice of building up output through chains of `<<` operators interferes with internationalization, because it bakes word order into the code, and streams' support for localization is [flawed](http://www.boost.org/doc/libs/1_48_0/libs/locale/doc/html/rationale.html#rationale_why).
*   The streams API is subtle and complex, so programmers must develop experience with it in order to use it effectively.
*   Resolving the many overloads of `<<` is extremely costly for the compiler. When used pervasively in a large code base, it can consume as much as 20% of the parsing and semantic analysis time.

Use streams only when they are the best tool for the job. This is typically the case when the I/O is ad-hoc, local, human-readable, and targeted at other developers rather than end-users. Be consistent with the code around you, and with the codebase as a whole; if there's an established tool for your problem, use that tool instead. In particular, logging libraries are usually a better choice than `std::cerr` or `std::clog` for diagnostic output, and the libraries in `absl/strings` or the equivalent are usually a better choice than `std::stringstream`.

Avoid using streams for I/O that faces external users or handles untrusted data. Instead, find and use the appropriate templating libraries to handle issues like internationalization, localization, and security hardening.

If you do use streams, avoid the stateful parts of the streams API (other than error state), such as `imbue()`, `xalloc()`, and `register_callback()`. Use explicit formatting functions (such as `absl::StreamFormat()`) rather than stream manipulators or formatting flags to control formatting details such as number base, precision, or padding.

Overload `<<` as a streaming operator for your type only if your type represents a value, and `<<` writes out a human-readable string representation of that value. Avoid exposing implementation details in the output of `<<`; if you need to print object internals for debugging, use named functions instead (a method named `DebugString()` is the most common convention).

### Preincrement and Predecrement

Use the prefix form (`++i`) of the increment and decrement operators unless you need postfix semantics.

When a variable is incremented (`++i` or `i++`) or decremented (`--i` or `i--`) and the value of the expression is not used, one must decide whether to preincrement (decrement) or postincrement (decrement).

A postfix increment/decrement expression evaluates to the value _as it was before it was modified_. This can result in code that is more compact but harder to read. The prefix form is generally more readable, is never less efficient, and can be more efficient because it doesn't need to make a copy of the value as it was before the operation.

The tradition developed, in C, of using post-increment, even when the expression value is not used, especially in `for` loops.

Use prefix increment/decrement, unless the code explicitly needs the result of the postfix increment/decrement expression.

### Use of const

In APIs, use `const` whenever it makes sense. `constexpr` is a better choice for some uses of const.

Declared variables and parameters can be preceded by the keyword `const` to indicate the variables are not changed (e.g., `const int foo`). Class functions can have the `const` qualifier to indicate the function does not change the state of the class member variables (e.g., `class Foo { int Bar(char c) const; };`).

Easier for people to understand how variables are being used. Allows the compiler to do better type checking, and, conceivably, generate better code. Helps people convince themselves of program correctness because they know the functions they call are limited in how they can modify your variables. Helps people know what functions are safe to use without locks in multi-threaded programs.

`const` is viral: if you pass a `const` variable to a function, that function must have `const` in its prototype (or the variable will need a `const_cast`). This can be a particular problem when calling library functions.

We strongly recommend using `const` in APIs (i.e., on function parameters, methods, and non-local variables) wherever it is meaningful and accurate. This provides consistent, mostly compiler-verified documentation of what objects an operation can mutate. Having a consistent and reliable way to distinguish reads from writes is critical to writing thread-safe code, and is useful in many other contexts as well. In particular:

*   If a function guarantees that it will not modify an argument passed by reference or by pointer, the corresponding function parameter should be a reference-to-const (`const T&`) or pointer-to-const (`const T*`), respectively.
*   For a function parameter passed by value, `const` has no effect on the caller, thus is not recommended in function declarations. See [TotW #109](https://abseil.io/tips/109).
*   Declare methods to be `const` unless they alter the logical state of the object (or enable the user to modify that state, e.g., by returning a non-`const` reference, but that's rare), or they can't safely be invoked concurrently.

Using `const` on local variables is neither encouraged nor discouraged.

All of a class's `const` operations should be safe to invoke concurrently with each other. If that's not feasible, the class must be clearly documented as "thread-unsafe".

#### Where to put the const

Some people favor the form `int const *foo` to `const int* foo`. They argue that this is more readable because it's more consistent: it keeps the rule that `const` always follows the object it's describing. However, this consistency argument doesn't apply in codebases with few deeply-nested pointer expressions since most `const` expressions have only one `const`, and it applies to the underlying value. In such cases, there's no consistency to maintain. Putting the `const` first is arguably more readable, since it follows English in putting the "adjective" (`const`) before the "noun" (`int`).

That said, while we encourage putting `const` first, we do not require it. But be consistent with the code around you!

### Use of constexpr, constinit, and consteval

Use `constexpr` to define true constants or to ensure constant initialization. Use `constinit` to ensure constant initialization for non-constant variables.

Some variables can be declared `constexpr` to indicate the variables are true constants, i.e., fixed at compilation/link time. Some functions and constructors can be declared `constexpr` which enables them to be used in defining a `constexpr` variable. Functions can be declared `consteval` to restrict their use to compile time.

Use of `constexpr` enables definition of constants with floating-point expressions rather than just literals; definition of constants of user-defined types; and definition of constants with function calls.

Prematurely marking something as `constexpr` may cause migration problems if later on it has to be downgraded. Current restrictions on what is allowed in `constexpr` functions and constructors may invite obscure workarounds in these definitions.

`constexpr` definitions enable a more robust specification of the constant parts of an interface. Use `constexpr` to specify true constants and the functions that support their definitions. `consteval` may be used for code that must not be invoked at runtime. Avoid complexifying function definitions to enable their use with `constexpr`. Do not use `constexpr` or `consteval` to force inlining.

### Integer Types

Of the built-in C++ integer types, the only one used is `int`. If a program needs an integer type of a different size, use an exact-width integer type from `<cstdint>`, such as `int16_t`. If you have a value that could ever be greater than or equal to 2^31, use a 64-bit type such as `int64_t`. Keep in mind that even if your value won't ever be too large for an `int`, it may be used in intermediate calculations which may require a larger type. When in doubt, choose a larger type.

C++ does not specify exact sizes for the integer types like `int`. Common sizes on contemporary architectures are 16 bits for `short`, 32 bits for `int`, 32 or 64 bits for `long`, and 64 bits for `long long`, but different platforms make different choices, in particular for `long`.

Uniformity of declaration.

The sizes of integral types in C++ can vary based on compiler and architecture.

The standard library header `<cstdint>` defines types like `int16_t`, `uint32_t`, `int64_t`, etc. You should always use those in preference to `short`, `unsigned long long` and the like, when you need a guarantee on the size of an integer. Prefer to omit the `std::` prefix for these types, as the extra 5 characters do not merit the added clutter. Of the built-in integer types, only `int` should be used. When appropriate, you are welcome to use standard type aliases like `size_t` and `ptrdiff_t`.

We use `int` very often, for integers we know are not going to be too big, e.g., loop counters. Use plain old `int` for such things. You should assume that an `int` is at least 32 bits, but don't assume that it has more than 32 bits. If you need a 64-bit integer type, use `int64_t` or `uint64_t`.

For integers we know can be "big", use `int64_t`.

You should not use the unsigned integer types such as `uint32_t`, unless there is a valid reason such as representing a bit pattern rather than a number, or you need defined overflow modulo 2^N. In particular, do not use unsigned types to say a number will never be negative. Instead, use assertions for this.

If your code is a container that returns a size, be sure to use a type that will accommodate any possible usage of your container. When in doubt, use a larger type rather than a smaller type.

Use care when converting integer types. Integer conversions and promotions can cause undefined behavior, leading to security bugs and other problems.

#### On Unsigned Integers

Unsigned integers are good for representing bitfields and modular arithmetic. Because of historical accident, the C++ standard also uses unsigned integers to represent the size of containers - many members of the standards body believe this to be a mistake, but it is effectively impossible to fix at this point. The fact that unsigned arithmetic doesn't model the behavior of a simple integer, but is instead defined by the standard to model modular arithmetic (wrapping around on overflow/underflow), means that a significant class of bugs cannot be diagnosed by the compiler. In other cases, the defined behavior impedes optimization.

That said, mixing signedness of integer types is responsible for an equally large class of problems. The best advice we can provide: try to use iterators and containers rather than pointers and sizes, try not to mix signedness, and try to avoid unsigned types (except for representing bitfields or modular arithmetic). Do not use an unsigned type merely to assert that a variable is non-negative.

### 64-bit Portability

Code should be 64-bit and 32-bit friendly. Bear in mind problems of printing, comparisons, and structure alignment.

*   Correct portable `printf()` conversion specifiers for some integral typedefs rely on macro expansions that we find unpleasant to use and impractical to require (the `PRI` macros from `<cinttypes>`). Unless there is no reasonable alternative for your particular case, try to avoid or even upgrade APIs that rely on the `printf` family. Instead use a library supporting typesafe numeric formatting, such as [`StrCat`](https://github.com/abseil/abseil-cpp/blob/master/absl/strings/str_cat.h) or [`Substitute`](https://github.com/abseil/abseil-cpp/blob/master/absl/strings/substitute.h) for fast simple conversions, or [`std::ostream`](#Streams).
    
    Unfortunately, the `PRI` macros are the only portable way to specify a conversion for the standard bitwidth typedefs (e.g., `int64_t`, `uint64_t`, `int32_t`, `uint32_t`, etc). Where possible, avoid passing arguments of types specified by bitwidth typedefs to `printf`\-based APIs. Note that it is acceptable to use typedefs for which printf has dedicated length modifiers, such as `size_t` (`z`), `ptrdiff_t` (`t`), and `maxint_t` (`j`).
    
*   Remember that `sizeof(void *)` != `sizeof(int)`. Use `intptr_t` if you want a pointer-sized integer.
*   You may need to be careful with structure alignments, particularly for structures being stored on disk. Any class/structure with a `int64_t`/`uint64_t` member will by default end up being 8-byte aligned on a 64-bit system. If you have such structures being shared on disk between 32-bit and 64-bit code, you will need to ensure that they are packed the same on both architectures. Most compilers offer a way to alter structure alignment. For gcc, you can use `__attribute__((packed))`. MSVC offers `#pragma pack()` and `__declspec(align())`.
*   Use [braced-initialization](#Casting) as needed to create 64-bit constants. For example:
    
    int64\_t my\_value{0x123456789};
    uint64\_t my\_mask{uint64\_t{3} << 48};
    

### Preprocessor Macros

Avoid defining macros, especially in headers; prefer inline functions, enums, and `const` variables. Name macros with a project-specific prefix. Do not use macros to define pieces of a C++ API.

Macros mean that the code you see is not the same as the code the compiler sees. This can introduce unexpected behavior, especially since macros have global scope.

The problems introduced by macros are especially severe when they are used to define pieces of a C++ API, and still more so for public APIs. Every error message from the compiler when developers incorrectly use that interface now must explain how the macros formed the interface. Refactoring and analysis tools have a dramatically harder time updating the interface. As a consequence, we specifically disallow using macros in this way. For example, avoid patterns like:

class WOMBAT\_TYPE(Foo) {
  // ...

 public:
  EXPAND\_PUBLIC\_WOMBAT\_API(Foo)

  EXPAND\_WOMBAT\_COMPARISONS(Foo, ==, <)
};

Luckily, macros are not nearly as necessary in C++ as they are in C. Instead of using a macro to inline performance-critical code, use an inline function. Instead of using a macro to store a constant, use a `const` variable. Instead of using a macro to "abbreviate" a long variable name, use a reference. Instead of using a macro to conditionally compile code ... well, don't do that at all (except, of course, for the `#define` guards to prevent double inclusion of header files). It makes testing much more difficult.

Macros can do things these other techniques cannot, and you do see them in the codebase, especially in the lower-level libraries. And some of their special features (like stringifying, concatenation, and so forth) are not available through the language proper. But before using a macro, consider carefully whether there's a non-macro way to achieve the same result. If you need to use a macro to define an interface, contact your project leads to request a waiver of this rule.

The following usage pattern will avoid many problems with macros; if you use macros, follow it whenever possible:

*   Don't define macros in a `.h` file.
*   `#define` macros right before you use them, and `#undef` them right after.
*   Do not just `#undef` an existing macro before replacing it with your own; instead, pick a name that's likely to be unique.
*   Try not to use macros that expand to unbalanced C++ constructs, or at least document that behavior well.
*   Prefer not using `##` to generate function/class/variable names.

Exporting macros from headers (i.e., defining them in a header without `#undef`ing them before the end of the header) is extremely strongly discouraged. If you do export a macro from a header, it must have a globally unique name. To achieve this, it must be named with a prefix consisting of your project's namespace name (but upper case).

### 0 and nullptr/NULL

Use `nullptr` for pointers, and `'\0'` for chars (and not the `0` literal).

For pointers (address values), use `nullptr`, as this provides type-safety.

Use `'\0'` for the null character. Using the correct type makes the code more readable.

### sizeof

Prefer `sizeof(varname)` to `sizeof(type)`.

Use `sizeof(varname)` when you take the size of a particular variable. `sizeof(varname)` will update appropriately if someone changes the variable type either now or later. You may use `sizeof(type)` for code unrelated to any particular variable, such as code that manages an external or internal data format where a variable of an appropriate C++ type is not convenient.

MyStruct data;
memset(&data, 0, sizeof(data));

memset(&data, 0, sizeof(MyStruct));

if (raw\_size < sizeof(int)) {
  LOG(ERROR) << "compressed record not big enough for count: " << raw\_size;
  return false;
}

### Type Deduction (including auto)

Use type deduction only if it makes the code clearer to readers who aren't familiar with the project, or if it makes the code safer. Do not use it merely to avoid the inconvenience of writing an explicit type.

There are several contexts in which C++ allows (or even requires) types to be deduced by the compiler, rather than spelled out explicitly in the code:

[Function template argument deduction](https://en.cppreference.com/w/cpp/language/template_argument_deduction)

A function template can be invoked without explicit template arguments. The compiler deduces those arguments from the types of the function arguments:

template <typename T>
void f(T t);

f(0);  // Invokes f<int>(0)

[`auto` variable declarations](https://en.cppreference.com/w/cpp/language/auto)

A variable declaration can use the `auto` keyword in place of the type. The compiler deduces the type from the variable's initializer, following the same rules as function template argument deduction with the same initializer (so long as you don't use curly braces instead of parentheses).

auto a = 42;  // a is an int
auto& b = a;  // b is an int&
auto c = b;   // c is an int
auto d{42};   // d is an int, not a std::initializer\_list<int>

`auto` can be qualified with `const`, and can be used as part of a pointer or reference type, and (since C++17) as a non-type template argument. A rare variant of this syntax uses `decltype(auto)` instead of `auto`, in which case the deduced type is the result of applying [`decltype`](https://en.cppreference.com/w/cpp/language/decltype) to the initializer.

[Function return type deduction](https://en.cppreference.com/w/cpp/language/function#Return_type_deduction)

`auto` (and `decltype(auto)`) can also be used in place of a function return type. The compiler deduces the return type from the `return` statements in the function body, following the same rules as for variable declarations:

auto f() { return 0; }  // The return type of f is int

[Lambda expression](#Lambda_expressions) return types can be deduced in the same way, but this is triggered by omitting the return type, rather than by an explicit `auto`. Confusingly, [trailing return type](#trailing_return) syntax for functions also uses `auto` in the return-type position, but that doesn't rely on type deduction; it's just an alternate syntax for an explicit return type.

[Generic lambdas](https://isocpp.org/wiki/faq/cpp14-language#generic-lambdas)

A lambda expression can use the `auto` keyword in place of one or more of its parameter types. This causes the lambda's call operator to be a function template instead of an ordinary function, with a separate template parameter for each `auto` function parameter:

// Sort \`vec\` in decreasing order
std::sort(vec.begin(), vec.end(), \[\](auto lhs, auto rhs) { return lhs > rhs; });

[Lambda init captures](https://isocpp.org/wiki/faq/cpp14-language#lambda-captures)

Lambda captures can have explicit initializers, which can be used to declare wholly new variables rather than only capturing existing ones:

\[x = 42, y = "foo"\] { ... }  // x is an int, and y is a const char\*

This syntax doesn't allow the type to be specified; instead, it's deduced using the rules for `auto` variables.

[Class template argument deduction](https://en.cppreference.com/w/cpp/language/class_template_argument_deduction)

See [below](#CTAD).

[Structured bindings](https://en.cppreference.com/w/cpp/language/structured_binding)

When declaring a tuple, struct, or array using `auto`, you can specify names for the individual elements instead of a name for the whole object; these names are called "structured bindings", and the whole declaration is called a "structured binding declaration". This syntax provides no way of specifying the type of either the enclosing object or the individual names:

auto \[iter, success\] = my\_map.insert({key, value});
if (!success) {
  iter->second = value;
}

The `auto` can also be qualified with `const`, `&`, and `&&`, but note that these qualifiers technically apply to the anonymous tuple/struct/array, rather than the individual bindings. The rules that determine the types of the bindings are quite complex; the results tend to be unsurprising, except that the binding types typically won't be references even if the declaration declares a reference (but they will usually behave like references anyway).

(These summaries omit many details and caveats; see the links for further information.)

*   C++ type names can be long and cumbersome, especially when they involve templates or namespaces.
*   When a C++ type name is repeated within a single declaration or a small code region, the repetition may not be aiding readability.
*   It is sometimes safer to let the type be deduced, since that avoids the possibility of unintended copies or type conversions.

C++ code is usually clearer when types are explicit, especially when type deduction would depend on information from distant parts of the code. In expressions like:

auto foo = x.add\_foo();
auto i = y.Find(key);

it may not be obvious what the resulting types are if the type of `y` isn't very well known, or if `y` was declared many lines earlier.

Programmers have to understand when type deduction will or won't produce a reference type, or they'll get copies when they didn't mean to.

If a deduced type is used as part of an interface, then a programmer might change its type while only intending to change its value, leading to a more radical API change than intended.

The fundamental rule is: use type deduction only to make the code clearer or safer, and do not use it merely to avoid the inconvenience of writing an explicit type. When judging whether the code is clearer, keep in mind that your readers are not necessarily on your team, or familiar with your project, so types that you and your reviewer experience as unnecessary clutter will very often provide useful information to others. For example, you can assume that the return type of `make_unique<Foo>()` is obvious, but the return type of `MyWidgetFactory()` probably isn't.

These principles apply to all forms of type deduction, but the details vary, as described in the following sections.

#### Function template argument deduction

Function template argument deduction is almost always OK. Type deduction is the expected default way of interacting with function templates, because it allows function templates to act like infinite sets of ordinary function overloads. Consequently, function templates are almost always designed so that template argument deduction is clear and safe, or doesn't compile.

#### Local variable type deduction

For local variables, you can use type deduction to make the code clearer by eliminating type information that is obvious or irrelevant, so that the reader can focus on the meaningful parts of the code:

std::unique\_ptr<WidgetWithBellsAndWhistles> widget =
    std::make\_unique<WidgetWithBellsAndWhistles>(arg1, arg2);
absl::flat\_hash\_map<std::string,
                    std::unique\_ptr<WidgetWithBellsAndWhistles>>::const\_iterator
    it = my\_map\_.find(key);
std::array<int, 6> numbers = {4, 8, 15, 16, 23, 42};

auto widget = std::make\_unique<WidgetWithBellsAndWhistles>(arg1, arg2);
auto it = my\_map\_.find(key);
std::array numbers = {4, 8, 15, 16, 23, 42};

Types sometimes contain a mixture of useful information and boilerplate, such as `it` in the example above: it's obvious that the type is an iterator, and in many contexts the container type and even the key type aren't relevant, but the type of the values is probably useful. In such situations, it's often possible to define local variables with explicit types that convey the relevant information:

if (auto it = my\_map\_.find(key); it != my\_map\_.end()) {
  WidgetWithBellsAndWhistles& widget = \*it->second;
  // Do stuff with \`widget\`
}

If the type is a template instance, and the parameters are boilerplate but the template itself is informative, you can use class template argument deduction to suppress the boilerplate. However, cases where this actually provides a meaningful benefit are quite rare. Note that class template argument deduction is also subject to a [separate style rule](#CTAD).

Do not use `decltype(auto)` if a simpler option will work, because it's a fairly obscure feature, so it has a high cost in code clarity.

#### Return type deduction

Use return type deduction (for both functions and lambdas) only if the function body has a very small number of `return` statements, and very little other code, because otherwise the reader may not be able to tell at a glance what the return type is. Furthermore, use it only if the function or lambda has a very narrow scope, because functions with deduced return types don't define abstraction boundaries: the implementation _is_ the interface. In particular, public functions in header files should almost never have deduced return types.

#### Parameter type deduction

`auto` parameter types for lambdas should be used with caution, because the actual type is determined by the code that calls the lambda, rather than by the definition of the lambda. Consequently, an explicit type will almost always be clearer unless the lambda is explicitly called very close to where it's defined (so that the reader can easily see both), or the lambda is passed to an interface so well-known that it's obvious what arguments it will eventually be called with (e.g., the `std::sort` example above).

#### Lambda init captures

Init captures are covered by a [more specific style rule](#Lambda_expressions), which largely supersedes the general rules for type deduction.

#### Structured bindings

Unlike other forms of type deduction, structured bindings can actually give the reader additional information, by giving meaningful names to the elements of a larger object. This means that a structured binding declaration may provide a net readability improvement over an explicit type, even in cases where `auto` would not. Structured bindings are especially beneficial when the object is a pair or tuple (as in the `insert` example above), because they don't have meaningful field names to begin with, but note that you generally [shouldn't use pairs or tuples](#Structs_vs._Tuples) unless a pre-existing API like `insert` forces you to.

If the object being bound is a struct, it may sometimes be helpful to provide names that are more specific to your usage, but keep in mind that this may also mean the names are less recognizable to your reader than the field names. We recommend using a comment to indicate the name of the underlying field, if it doesn't match the name of the binding, using the same syntax as for function parameter comments:

auto \[/\*field\_name1=\*/bound\_name1, /\*field\_name2=\*/bound\_name2\] = ...

As with function parameter comments, this can enable tools to detect if you get the order of the fields wrong.

### Class Template Argument Deduction

Use class template argument deduction only with templates that have explicitly opted into supporting it.

[Class template argument deduction](https://en.cppreference.com/w/cpp/language/class_template_argument_deduction) (often abbreviated "CTAD") occurs when a variable is declared with a type that names a template, and the template argument list is not provided (not even empty angle brackets):

std::array a = {1, 2, 3};  // \`a\` is a std::array<int, 3>

The compiler deduces the arguments from the initializer using the template's "deduction guides", which can be explicit or implicit.

Explicit deduction guides look like function declarations with trailing return types, except that there's no leading `auto`, and the function name is the name of the template. For example, the above example relies on this deduction guide for `std::array`:

namespace std {
template <class T, class... U>
array(T, U...) -> std::array<T, 1 + sizeof...(U)>;
}

Constructors in a primary template (as opposed to a template specialization) also implicitly define deduction guides.

When you declare a variable that relies on CTAD, the compiler selects a deduction guide using the rules of constructor overload resolution, and that guide's return type becomes the type of the variable.

CTAD can sometimes allow you to omit boilerplate from your code.

The implicit deduction guides that are generated from constructors may have undesirable behavior, or be outright incorrect. This is particularly problematic for constructors written before CTAD was introduced in C++17, because the authors of those constructors had no way of knowing about (much less fixing) any problems that their constructors would cause for CTAD. Furthermore, adding explicit deduction guides to fix those problems might break any existing code that relies on the implicit deduction guides.

CTAD also suffers from many of the same drawbacks as `auto`, because they are both mechanisms for deducing all or part of a variable's type from its initializer. CTAD does give the reader more information than `auto`, but it also doesn't give the reader an obvious cue that information has been omitted.

Do not use CTAD with a given template unless the template's maintainers have opted into supporting use of CTAD by providing at least one explicit deduction guide (all templates in the `std` namespace are also presumed to have opted in). This should be enforced with a compiler warning if available.

Uses of CTAD must also follow the general rules on [Type deduction](#Type_deduction).

### Designated Initializers

Use designated initializers only in their C++20-compliant form.

[Designated initializers](https://en.cppreference.com/w/cpp/language/aggregate_initialization#Designated_initializers) are a syntax that allows for initializing an aggregate ("plain old struct") by naming its fields explicitly:

  struct Point {
    float x = 0.0;
    float y = 0.0;
    float z = 0.0;
  };

  Point p = {
    .x = 1.0,
    .y = 2.0,
    // z will be 0.0
  };

The explicitly listed fields will be initialized as specified, and others will be initialized in the same way they would be in a traditional aggregate initialization expression like `Point{1.0, 2.0}`.

Designated initializers can make for convenient and highly readable aggregate expressions, especially for structs with less straightforward ordering of fields than the `Point` example above.

While designated initializers have long been part of the C standard and supported by C++ compilers as an extension, they were not supported by C++ prior to C++20.

The rules in the C++ standard are stricter than in C and compiler extensions, requiring that the designated initializers appear in the same order as the fields appear in the struct definition. So in the example above, it is legal according to C++20 to initialize `x` and then `z`, but not `y` and then `x`.

Use designated initializers only in the form that is compatible with the C++20 standard: with initializers in the same order as the corresponding fields appear in the struct definition.

### Lambda Expressions

Use lambda expressions where appropriate. Prefer explicit captures when the lambda will escape the current scope.

Lambda expressions are a concise way of creating anonymous function objects. They're often useful when passing functions as arguments. For example:

std::sort(v.begin(), v.end(), \[\](int x, int y) {
  return Weight(x) < Weight(y);
});

They further allow capturing variables from the enclosing scope either explicitly by name, or implicitly using a default capture. Explicit captures require each variable to be listed, as either a value or reference capture:

int weight = 3;
int sum = 0;
// Captures \`weight\` by value and \`sum\` by reference.
std::for\_each(v.begin(), v.end(), \[weight, &sum\](int x) {
  sum += weight \* x;
});

Default captures implicitly capture any variable referenced in the lambda body, including `this` if any members are used:

const std::vector<int> lookup\_table = ...;
std::vector<int> indices = ...;
// Captures \`lookup\_table\` by reference, sorts \`indices\` by the value
// of the associated element in \`lookup\_table\`.
std::sort(indices.begin(), indices.end(), \[&\](int a, int b) {
  return lookup\_table\[a\] < lookup\_table\[b\];
});

A variable capture can also have an explicit initializer, which can be used for capturing move-only variables by value, or for other situations not handled by ordinary reference or value captures:

std::unique\_ptr<Foo> foo = ...;
\[foo = std::move(foo)\] () {
  ...
}

Such captures (often called "init captures" or "generalized lambda captures") need not actually "capture" anything from the enclosing scope, or even have a name from the enclosing scope; this syntax is a fully general way to define members of a lambda object:

\[foo = std::vector<int>({1, 2, 3})\] () {
  ...
}

The type of a capture with an initializer is deduced using the same rules as `auto`.

*   Lambdas are much more concise than other ways of defining function objects to be passed to STL algorithms, which can be a readability improvement.
*   Appropriate use of default captures can remove redundancy and highlight important exceptions from the default.
*   Lambdas, `std::function`, and `std::bind` can be used in combination as a general purpose callback mechanism; they make it easy to write functions that take bound functions as arguments.

*   Variable capture in lambdas can be a source of dangling-pointer bugs, particularly if a lambda escapes the current scope.
*   Default captures by value can be misleading because they do not prevent dangling-pointer bugs. Capturing a pointer by value doesn't cause a deep copy, so it often has the same lifetime issues as capture by reference. This is especially confusing when capturing `this` by value, since the use of `this` is often implicit.
*   Captures actually declare new variables (whether or not the captures have initializers), but they look nothing like any other variable declaration syntax in C++. In particular, there's no place for the variable's type, or even an `auto` placeholder (although init captures can indicate it indirectly, e.g., with a cast). This can make it difficult to even recognize them as declarations.
*   Init captures inherently rely on [type deduction](#Type_deduction), and suffer from many of the same drawbacks as `auto`, with the additional problem that the syntax doesn't even cue the reader that deduction is taking place.
*   It's possible for use of lambdas to get out of hand; very long nested anonymous functions can make code harder to understand.

*   Use lambda expressions where appropriate, with formatting as described [below](#Formatting_Lambda_Expressions).
*   Prefer explicit captures if the lambda may escape the current scope. For example, instead of:
    
    {
      Foo foo;
      ...
      executor->Schedule(\[&\] { Frobnicate(foo); })
      ...
    }
    // BAD! The fact that the lambda makes use of a reference to \`foo\` and
    // possibly \`this\` (if \`Frobnicate\` is a member function) may not be
    // apparent on a cursory inspection. If the lambda is invoked after
    // the function returns, that would be bad, because both \`foo\`
    // and the enclosing object could have been destroyed.
    
    prefer to write:
    
    {
      Foo foo;
      ...
      executor->Schedule(\[&foo\] { Frobnicate(foo); })
      ...
    }
    // BETTER - The compile will fail if \`Frobnicate\` is a member
    // function, and it's clearer that \`foo\` is dangerously captured by
    // reference.
    
*   Use default capture by reference (`[&]`) only when the lifetime of the lambda is obviously shorter than any potential captures.
*   Use default capture by value (`[=]`) only as a means of binding a few variables for a short lambda, where the set of captured variables is obvious at a glance, and which does not result in capturing `this` implicitly. (That means that a lambda that appears in a non-static class member function and refers to non-static class members in its body must capture `this` explicitly or via `[&]`.) Prefer not to write long or complex lambdas with default capture by value.
*   Use captures only to actually capture variables from the enclosing scope. Do not use captures with initializers to introduce new names, or to substantially change the meaning of an existing name. Instead, declare a new variable in the conventional way and then capture it, or avoid the lambda shorthand and define a function object explicitly.
*   See the section on [type deduction](#Type_deduction) for guidance on specifying the parameter and return types.

### Template Metaprogramming

Avoid complicated template programming.

Template metaprogramming refers to a family of techniques that exploit the fact that the C++ template instantiation mechanism is Turing complete and can be used to perform arbitrary compile-time computation in the type domain.

Template metaprogramming allows extremely flexible interfaces that are type safe and high performance. Facilities like [GoogleTest](https://github.com/google/googletest), `std::tuple`, `std::function`, and Boost.Spirit would be impossible without it.

The techniques used in template metaprogramming are often obscure to anyone but language experts. Code that uses templates in complicated ways is often unreadable, and is hard to debug or maintain.

Template metaprogramming often leads to extremely poor compile time error messages: even if an interface is simple, the complicated implementation details become visible when the user does something wrong.

Template metaprogramming interferes with large scale refactoring by making the job of refactoring tools harder. First, the template code is expanded in multiple contexts, and it's hard to verify that the transformation makes sense in all of them. Second, some refactoring tools work with an AST that only represents the structure of the code after template expansion. It can be difficult to automatically work back to the original source construct that needs to be rewritten.

Template metaprogramming sometimes allows cleaner and easier-to-use interfaces than would be possible without it, but it's also often a temptation to be overly clever. It's best used in a small number of low level components where the extra maintenance burden is spread out over a large number of uses.

Think twice before using template metaprogramming or other complicated template techniques; think about whether the average member of your team will be able to understand your code well enough to maintain it after you switch to another project, or whether a non-C++ programmer or someone casually browsing the code base will be able to understand the error messages or trace the flow of a function they want to call. If you're using recursive template instantiations or type lists or metafunctions or expression templates, or relying on SFINAE or on the `sizeof` trick for detecting function overload resolution, then there's a good chance you've gone too far.

If you use template metaprogramming, you should expect to put considerable effort into minimizing and isolating the complexity. You should hide metaprogramming as an implementation detail whenever possible, so that user-facing headers are readable, and you should make sure that tricky code is especially well commented. You should carefully document how the code is used, and you should say something about what the "generated" code looks like. Pay extra attention to the error messages that the compiler emits when users make mistakes. The error messages are part of your user interface, and your code should be tweaked as necessary so that the error messages are understandable and actionable from a user point of view.

### Concepts and Constraints

Use concepts sparingly. In general, concepts and constraints should only be used in cases where templates would have been used prior to C++20. Avoid introducing new concepts in headers, unless the headers are marked as internal to the library. Do not define concepts that are not enforced by the compiler. Prefer constraints over template metaprogramming, and avoid the `template<_Concept_ T>` syntax; instead, use the `requires(_Concept<T>_)` syntax.

The `concept` keyword is a new mechanism for defining requirements (such as type traits or interface specifications) for a template parameter. The `requires` keyword provides mechanisms for placing anonymous constraints on templates and verifying that constraints are satisfied at compile time. Concepts and constraints are often used together, but can be also used independently.

*   Concepts allow the compiler to generate much better error messages when templates are involved, which can reduce confusion and significantly improve the development experience.
*   Concepts can reduce the boilerplate necessary for defining and using compile-time constraints, often increasing the clarity of the resulting code.
*   Constraints provide some capabilities that are difficult to achieve with templates and SFINAE techniques.

*   As with templates, concepts can make code significantly more complex and difficult to understand.
*   Concept syntax can be confusing to readers, as concepts appear similar to class types at their usage sites.
*   Concepts, especially at API boundaries, increase code coupling, rigidity, and ossification.
*   Concepts and constraints can replicate logic from a function body, resulting in code duplication and increased maintenance costs.
*   Concepts muddy the source of truth for their underlying contracts, as they are standalone named entities that can be utilized in multiple locations, all of which evolve separately from each other. This can cause the stated and implied requirements to diverge over time.
*   Concepts and constraints affect overload resolution in novel and non-obvious ways.
*   As with SFINAE, constraints make it harder to refactor code at scale.

Predefined concepts in the standard library should be preferred to type traits, when equivalent ones exist. (e.g., if `std::is_integral_v` would have been used before C++20, then `std::integral` should be used in C++20 code.) Similarly, prefer modern constraint syntax (via `requires(_Condition_)`). Avoid legacy template metaprogramming constructs (such as `std::enable_if<_Condition_>`) as well as the `template<_Concept_ T>` syntax.

Do not manually re-implement any existing concepts or traits. For example, use `requires(std::default_initializable<T>)` instead of `requires(requires { T v; })` or the like.

New `concept` declarations should be rare, and only defined internally within a library, such that they are not exposed at API boundaries. More generally, do not use concepts or constraints in cases where you wouldn't use their legacy template equivalents in C++17.

Do not define concepts that duplicate the function body, or impose requirements that would be insignificant or obvious from reading the body of the code or the resulting error messages. For example, avoid the following:

template <typename T>     // Bad - redundant with negligible benefit
concept Addable = std::copyable<T> && requires(T a, T b) { a + b; };
template <Addable T>
T Add(T x, T y, T z) { return x + y + z; }

Instead, prefer to leave code as an ordinary template unless you can demonstrate that concepts result in significant improvement for that particular case, such as in the resulting error messages for a deeply nested or non-obvious requirement.

Concepts should be statically verifiable by the compiler. Do not use any concept whose primary benefits would come from a semantic (or otherwise unenforced) constraint. Requirements that are unenforced at compile time should instead be imposed via other mechanisms such as comments, assertions, or tests.

### Boost

Use only approved libraries from the Boost library collection.

The [Boost library collection](https://www.boost.org/) is a popular collection of peer-reviewed, free, open-source C++ libraries.

Boost code is generally very high-quality, is widely portable, and fills many important gaps in the C++ standard library, such as type traits and better binders.

Some Boost libraries encourage coding practices which can hamper readability, such as metaprogramming and other advanced template techniques, and an excessively "functional" style of programming.

In order to maintain a high level of readability for all contributors who might read and maintain code, we only allow an approved subset of Boost features. Currently, the following libraries are permitted:

*   [Call Traits](https://www.boost.org/libs/utility/call_traits.htm) from `boost/call_traits.hpp`
*   [Compressed Pair](https://www.boost.org/libs/utility/compressed_pair.htm) from `boost/compressed_pair.hpp`
*   [The Boost Graph Library (BGL)](https://www.boost.org/libs/graph/) from `boost/graph`, except serialization (`adj_list_serialize.hpp`) and parallel/distributed algorithms and data structures (`boost/graph/parallel/*` and `boost/graph/distributed/*`).
*   [Property Map](https://www.boost.org/libs/property_map/) from `boost/property_map`, except parallel/distributed property maps (`boost/property_map/parallel/*`).
*   [Iterator](https://www.boost.org/libs/iterator/) from `boost/iterator`
*   The part of [Polygon](https://www.boost.org/libs/polygon/) that deals with Voronoi diagram construction and doesn't depend on the rest of Polygon: `boost/polygon/voronoi_builder.hpp`, `boost/polygon/voronoi_diagram.hpp`, and `boost/polygon/voronoi_geometry_type.hpp`
*   [Bimap](https://www.boost.org/libs/bimap/) from `boost/bimap`
*   [Statistical Distributions and Functions](https://www.boost.org/libs/math/doc/html/dist.html) from `boost/math/distributions`
*   [Special Functions](https://www.boost.org/libs/math/doc/html/special.html) from `boost/math/special_functions`
*   [Root Finding & Minimization Functions](https://www.boost.org/libs/math/doc/html/root_finding.html) from `boost/math/tools`
*   [Multi-index](https://www.boost.org/libs/multi_index/) from `boost/multi_index`
*   [Heap](https://www.boost.org/libs/heap/) from `boost/heap`
*   The flat containers from [Container](https://www.boost.org/libs/container/): `boost/container/flat_map`, and `boost/container/flat_set`
*   [Intrusive](https://www.boost.org/libs/intrusive/) from `boost/intrusive`.
*   [The `boost/sort` library](https://www.boost.org/libs/sort/).
*   [Preprocessor](https://www.boost.org/libs/preprocessor/) from `boost/preprocessor`.

We are actively considering adding other Boost features to the list, so this list may be expanded in the future.

### Other C++ Features

As with [Boost](#Boost), some modern C++ extensions encourage coding practices that hamper readability—for example by removing checked redundancy (such as type names) that may be helpful to readers, or by encouraging template metaprogramming. Other extensions duplicate functionality available through existing mechanisms, which may lead to confusion and conversion costs.

In addition to what's described in the rest of the style guide, the following C++ features may not be used:

*   Compile-time rational numbers (`<ratio>`), because of concerns that it's tied to a more template-heavy interface style.
*   The `<cfenv>` and `<fenv.h>` headers, because many compilers do not support those features reliably.
*   The `<filesystem>` header, which does not have sufficient support for testing, and suffers from inherent security vulnerabilities.

### Nonstandard Extensions

Nonstandard extensions to C++ may not be used unless otherwise specified.

Compilers support various extensions that are not part of standard C++. Such extensions include GCC's `__attribute__`, intrinsic functions such as `__builtin_prefetch` or SIMD, `#pragma`, inline assembly, `__COUNTER__`, `__PRETTY_FUNCTION__`, compound statement expressions (e.g., `foo = ({ int x; Bar(&x); x })`, variable-length arrays and `alloca()`, and the "[Elvis Operator](https://en.wikipedia.org/wiki/Elvis_operator)" `a?:b`.

*   Nonstandard extensions may provide useful features that do not exist in standard C++.
*   Important performance guidance to the compiler can only be specified using extensions.

*   Nonstandard extensions do not work in all compilers. Use of nonstandard extensions reduces portability of code.
*   Even if they are supported in all targeted compilers, the extensions are often not well-specified, and there may be subtle behavior differences between compilers.
*   Nonstandard extensions add to the language features that a reader must know to understand the code.
*   Nonstandard extensions require additional work to port across architectures.

Do not use nonstandard extensions. You may use portability wrappers that are implemented using nonstandard extensions, so long as those wrappers are provided by a designated project-wide portability header.

### Aliases

Public aliases are for the benefit of an API's user, and should be clearly documented.

There are several ways to create names that are aliases of other entities:

using Bar = Foo;
typedef Foo Bar;  // But prefer \`using\` in C++ code.
using ::other\_namespace::Foo;
using enum MyEnumType;  // Creates aliases for all enumerators in MyEnumType.

In new code, `using` is preferable to `typedef`, because it provides a more consistent syntax with the rest of C++ and works with templates.

Like other declarations, aliases declared in a header file are part of that header's public API unless they're in a function definition, in the private portion of a class, or in an explicitly-marked internal namespace. Aliases in such areas or in `.cc` files are implementation details (because client code can't refer to them), and are not restricted by this rule.

*   Aliases can improve readability by simplifying a long or complicated name.
*   Aliases can reduce duplication by naming in one place a type used repeatedly in an API, which _might_ make it easier to change the type later.

*   When placed in a header where client code can refer to them, aliases increase the number of entities in that header's API, increasing its complexity.
*   Clients can easily rely on unintended details of public aliases, making changes difficult.
*   It can be tempting to create a public alias that is only intended for use in the implementation, without considering its impact on the API, or on maintainability.
*   Aliases can create risk of name collisions
*   Aliases can reduce readability by giving a familiar construct an unfamiliar name
*   Type aliases can create an unclear API contract: it is unclear whether the alias is guaranteed to be identical to the type it aliases, to have the same API, or only to be usable in specified narrow ways

Don't put an alias in your public API just to save typing in the implementation; do so only if you intend it to be used by your clients.

When defining a public alias, document the intent of the new name, including whether it is guaranteed to always be the same as the type it's currently aliased to, or whether a more limited compatibility is intended. This lets the user know whether they can treat the types as substitutable or whether more specific rules must be followed, and can help the implementation retain some degree of freedom to change the alias.

Don't put namespace aliases in your public API. (See also [Namespaces](#Namespaces)).

For example, these aliases document how they are intended to be used in client code:

namespace mynamespace {
// Used to store field measurements. DataPoint may change from Bar\* to some internal type.
// Client code should treat it as an opaque pointer.
using DataPoint = ::foo::Bar\*;

// A set of measurements. Just an alias for user convenience.
using TimeSeries = std::unordered\_set<DataPoint, std::hash<DataPoint>, DataPointComparator>;
}  // namespace mynamespace

These aliases don't document intended use, and half of them aren't meant for client use:

namespace mynamespace {
// Bad: none of these say how they should be used.
using DataPoint = ::foo::Bar\*;
using ::std::unordered\_set;  // Bad: just for local convenience
using ::std::hash;           // Bad: just for local convenience
typedef unordered\_set<DataPoint, hash<DataPoint>, DataPointComparator> TimeSeries;
}  // namespace mynamespace

However, local convenience aliases are fine in function definitions, `private` sections of classes, explicitly marked internal namespaces, and in `.cc` files:

// In a .cc file
using ::foo::Bar;

### Switch Statements

If not conditional on an enumerated value, switch statements should always have a `default` case (in the case of an enumerated value, the compiler will warn you if any values are not handled). If the default case should never execute, treat this as an error. For example:

switch (var) {
  case 0: {
    ...
    break;
  }
  case 1: {
    ...
    break;
  }
  default: {
    LOG(FATAL) << "Invalid value in switch statement: " << var;
  }
}

Fall-through from one case label to another must be annotated using the `[[fallthrough]];` attribute. `[[fallthrough]];` should be placed at a point of execution where a fall-through to the next case label occurs. A common exception is consecutive case labels without intervening code, in which case no annotation is needed.

switch (x) {
  case 41:  // No annotation needed here.
  case 43:
    if (dont\_be\_picky) {
      // Use this instead of or along with annotations in comments.
      \[\[fallthrough\]\];
    } else {
      CloseButNoCigar();
      break;
    }
  case 42:
    DoSomethingSpecial();
    \[\[fallthrough\]\];
  default:
    DoSomethingGeneric();
    break;
}

Inclusive Language
------------------

In all code, including naming and comments, use inclusive language and avoid terms that other programmers might find disrespectful or offensive (such as "master" and "slave", "blacklist" and "whitelist", or "redline"), even if the terms also have an ostensibly neutral meaning. Similarly, use gender-neutral language unless you're referring to a specific person (and using their pronouns). For example, use "they"/"them"/"their" for people of unspecified gender ([even when singular](https://apastyle.apa.org/style-grammar-guidelines/grammar/singular-they)), and "it"/"its" for software, computers, and other things that aren't people.

Naming
------

The most important consistency rules are those that govern naming. The style of a name immediately informs us what sort of thing the named entity is: a type, a variable, a function, a constant, a macro, etc., without requiring us to search for the declaration of that entity. The pattern-matching engine in our brains relies a great deal on these naming rules.

Naming rules are pretty arbitrary, but we feel that consistency is more important than individual preferences in this area, so regardless of whether you find them sensible or not, the rules are the rules.

### General Naming Rules

Optimize for readability using names that would be clear even to people on a different team.

Use names that describe the purpose or intent of the object. Do not worry about saving horizontal space as it is far more important to make your code immediately understandable by a new reader. Minimize the use of abbreviations that would likely be unknown to someone outside your project (especially acronyms and initialisms). Do not abbreviate by deleting letters within a word. As a rule of thumb, an abbreviation is probably OK if it's listed in Wikipedia. Generally speaking, descriptiveness should be proportional to the name's scope of visibility. For example, `n` may be a fine name within a 5-line function, but within the scope of a class, it's likely too vague.

class MyClass {
 public:
  int CountFooErrors(const std::vector<Foo>& foos) {
    int n = 0;  // Clear meaning given limited scope and context
    for (const auto& foo : foos) {
      ...
      ++n;
    }
    return n;
  }
  void DoSomethingImportant() {
    std::string fqdn = ...;  // Well-known abbreviation for Fully Qualified Domain Name
  }
 private:
  const int kMaxAllowedConnections = ...;  // Clear meaning within context
};

class MyClass {
 public:
  int CountFooErrors(const std::vector<Foo>& foos) {
    int total\_number\_of\_foo\_errors = 0;  // Overly verbose given limited scope and context
    for (int foo\_index = 0; foo\_index < foos.size(); ++foo\_index) {  // Use idiomatic \`i\`
      ...
      ++total\_number\_of\_foo\_errors;
    }
    return total\_number\_of\_foo\_errors;
  }
  void DoSomethingImportant() {
    int cstmr\_id = ...;  // Deletes internal letters
  }
 private:
  const int kNum = ...;  // Unclear meaning within broad scope
};

Note that certain universally-known abbreviations are OK, such as `i` for an iteration variable and `T` for a template parameter.

For the purposes of the naming rules below, a "word" is anything that you would write in English without internal spaces. This includes abbreviations, such as acronyms and initialisms. For names written in mixed case (also sometimes referred to as "[camel case](https://en.wikipedia.org/wiki/Camel_case)" or "[Pascal case](https://en.wiktionary.org/wiki/Pascal_case)"), in which the first letter of each word is capitalized, prefer to capitalize abbreviations as single words, e.g., `StartRpc()` rather than `StartRPC()`.

Template parameters should follow the naming style for their category: type template parameters should follow the rules for [type names](#Type_Names), and non-type template parameters should follow the rules for [variable names](#Variable_Names).

### File Names

Filenames should be all lowercase and can include underscores (`_`) or dashes (`-`). Follow the convention that your project uses. If there is no consistent local pattern to follow, prefer "`_`".

Examples of acceptable file names:

*   `my_useful_class.cc`
*   `my-useful-class.cc`
*   `myusefulclass.cc`
*   `myusefulclass_test.cc // _unittest and _regtest are deprecated.`

C++ files should end in `.cc` and header files should end in `.h`. Files that rely on being textually included at specific points should end in `.inc` (see also the section on [self-contained headers](#Self_contained_Headers)).

Do not use filenames that already exist in `/usr/include`, such as `db.h`.

In general, make your filenames very specific. For example, use `http_server_logs.h` rather than `logs.h`. A very common case is to have a pair of files called, e.g., `foo_bar.h` and `foo_bar.cc`, defining a class called `FooBar`.

### Type Names

Type names start with a capital letter and have a capital letter for each new word, with no underscores: `MyExcitingClass`, `MyExcitingEnum`.

The names of all types — classes, structs, type aliases, enums, and type template parameters — have the same naming convention. Type names should start with a capital letter and have a capital letter for each new word. No underscores. For example:

// classes and structs
class UrlTable { ...
class UrlTableTester { ...
struct UrlTableProperties { ...

// typedefs
typedef hash\_map<UrlTableProperties \*, std::string> PropertiesMap;

// using aliases
using PropertiesMap = hash\_map<UrlTableProperties \*, std::string>;

// enums
enum class UrlTableError { ...

### Concept Names

Concept names follow the same rules as [type names](#Type_Names).

### Variable Names

The names of variables (including function parameters) and data members are `snake_case` (all lowercase, with underscores between words). Data members of classes (but not structs) additionally have trailing underscores. For instance: `a_local_variable`, `a_struct_data_member`, `a_class_data_member_`.

#### Common Variable names

For example:

std::string table\_name;  // OK - snake\_case.

std::string tableName;   // Bad - mixed case.

#### Class Data Members

Data members of classes, both static and non-static, are named like ordinary nonmember variables, but with a trailing underscore.

class TableInfo {
  ...
 private:
  std::string table\_name\_;  // OK - underscore at end.
  static Pool<TableInfo>\* pool\_;  // OK.
};

#### Struct Data Members

Data members of structs, both static and non-static, are named like ordinary nonmember variables. They do not have the trailing underscores that data members in classes have.

struct UrlTableProperties {
  std::string name;
  int num\_entries;
  static Pool<UrlTableProperties>\* pool;
};

See [Structs vs. Classes](#Structs_vs._Classes) for a discussion of when to use a struct versus a class.

### Constant Names

Variables declared `constexpr` or `const`, and whose value is fixed for the duration of the program, are named with a leading "k" followed by mixed case. Underscores can be used as separators in the rare cases where capitalization cannot be used for separation. For example:

const int kDaysInAWeek = 7;
const int kAndroid8\_0\_0 = 24;  // Android 8.0.0

All such variables with static storage duration (i.e., statics and globals, see [Storage Duration](http://en.cppreference.com/w/cpp/language/storage_duration#Storage_duration) for details) should be named this way, including those in templates where different instantiations of the template may have different values. This convention is optional for variables of other storage classes, e.g., automatic variables; otherwise the usual variable naming rules apply. For example:

void ComputeFoo(absl::string\_view suffix) {
  // Either of these is acceptable.
  const absl::string\_view kPrefix = "prefix";
  const absl::string\_view prefix = "prefix";
  ...
}

void ComputeFoo(absl::string\_view suffix) {
  // Bad - different invocations of ComputeFoo give kCombined different values.
  const std::string kCombined = absl::StrCat(kPrefix, suffix);
  ...
}

### Function Names

Regular functions have mixed case; accessors and mutators may be named like variables.

Ordinarily, functions should start with a capital letter and have a capital letter for each new word.

AddTableEntry()
DeleteUrl()
OpenFileOrDie()

(The same naming rule applies to class- and namespace-scope constants that are exposed as part of an API and that are intended to look like functions, because the fact that they're objects rather than functions is an unimportant implementation detail.)

Accessors and mutators (get and set functions) may be named like variables. These often correspond to actual member variables, but this is not required. For example, `int count()` and `void set_count(int count)`.

### Namespace Names

Namespace names are all lower-case, with words separated by underscores. Top-level namespace names are based on the project name . Avoid collisions between nested namespaces and well-known top-level namespaces.

The name of a top-level namespace should usually be the name of the project or team whose code is contained in that namespace. The code in that namespace should usually be in a directory whose basename matches the namespace name (or in subdirectories thereof).

Keep in mind that the [rule against abbreviated names](#General_Naming_Rules) applies to namespaces just as much as variable names. Code inside the namespace seldom needs to mention the namespace name, so there's usually no particular need for abbreviation anyway.

Avoid nested namespaces that match well-known top-level namespaces. Collisions between namespace names can lead to surprising build breaks because of name lookup rules. In particular, do not create any nested `std` namespaces. Prefer unique project identifiers (`websearch::index`, `websearch::index_util`) over collision-prone names like `websearch::util`. Also avoid overly deep nesting namespaces ([TotW #130](https://abseil.io/tips/130)).

For `internal` namespaces, be wary of other code being added to the same `internal` namespace causing a collision (internal helpers within a team tend to be related and may lead to collisions). In such a situation, using the filename to make a unique internal name is helpful (`websearch::index::frobber_internal` for use in `frobber.h`).

### Enumerator Names

Enumerators (for both scoped and unscoped enums) should be named like [constants](#Constant_Names), not like [macros](#Macro_Names). That is, use `kEnumName` not `ENUM_NAME`.

enum class UrlTableError {
  kOk = 0,
  kOutOfMemory,
  kMalformedInput,
};

enum class AlternateUrlTableError {
  OK = 0,
  OUT\_OF\_MEMORY = 1,
  MALFORMED\_INPUT = 2,
};

Until January 2009, the style was to name enum values like [macros](#Macro_Names). This caused problems with name collisions between enum values and macros. Hence, the change to prefer constant-style naming was put in place. New code should use constant-style naming.

### Macro Names

You're not really going to [define a macro](#Preprocessor_Macros), are you? If you do, they're like this: `MY_MACRO_THAT_SCARES_SMALL_CHILDREN_AND_ADULTS_ALIKE`.

Please see the [description of macros](#Preprocessor_Macros); in general macros should _not_ be used. However, if they are absolutely needed, then they should be named with all capitals and underscores, and with a project-specific prefix.

#define MYPROJECT\_ROUND(x) ...

### Exceptions to Naming Rules

If you are naming something that is analogous to an existing C or C++ entity then you can follow the existing naming convention scheme.

`bigopen()`

function name, follows form of `open()`

`uint`

`typedef`

`bigpos`

`struct` or `class`, follows form of `pos`

`sparse_hash_map`

STL-like entity; follows STL naming conventions

`LONGLONG_MAX`

a constant, as in `INT_MAX`

Comments
--------

Comments are absolutely vital to keeping our code readable. The following rules describe what you should comment and where. But remember: while comments are very important, the best code is self-documenting. Giving sensible names to types and variables is much better than using obscure names that you must then explain through comments.

When writing your comments, write for your audience: the next contributor who will need to understand your code. Be generous — the next one may be you!

### Comment Style

Use either the `//` or `/* */` syntax, as long as you are consistent.

You can use either the `//` or the `/* */` syntax; however, `//` is _much_ more common. Be consistent with how you comment and what style you use where.

### File Comments

Start each file with license boilerplate.

If a source file (such as a `.h` file) declares multiple user-facing abstractions (common functions, related classes, etc.), include a comment describing the collection of those abstractions. Include enough detail for future authors to know what does not fit there. However, the detailed documentation about individual abstractions belongs with those abstractions, not at the file level.

For instance, if you write a file comment for `frobber.h`, you do not need to include a file comment in `frobber.cc` or `frobber_test.cc`. On the other hand, if you write a collection of classes in `registered_objects.cc` that has no associated header file, you must include a file comment in `registered_objects.cc`.

#### Legal Notice and Author Line

Every file should contain license boilerplate. Choose the appropriate boilerplate for the license used by the project (for example, Apache 2.0, BSD, LGPL, GPL).

If you make significant changes to a file with an author line, consider deleting the author line. New files should usually not contain copyright notice or author line.

### Struct and Class Comments

Every non-obvious class or struct declaration should have an accompanying comment that describes what it is for and how it should be used.

// Iterates over the contents of a GargantuanTable.
// Example:
//    std::unique\_ptr<GargantuanTableIterator> iter = table->NewIterator();
//    for (iter->Seek("foo"); !iter->done(); iter->Next()) {
//      process(iter->key(), iter->value());
//    }
class GargantuanTableIterator {
  ...
};

#### Class Comments

The class comment should provide the reader with enough information to know how and when to use the class, as well as any additional considerations necessary to correctly use the class. Document the synchronization assumptions the class makes, if any. If an instance of the class can be accessed by multiple threads, take extra care to document the rules and invariants surrounding multithreaded use.

The class comment is often a good place for a small example code snippet demonstrating a simple and focused usage of the class.

When sufficiently separated (e.g., `.h` and `.cc` files), comments describing the use of the class should go together with its interface definition; comments about the class operation and implementation should accompany the implementation of the class's methods.

### Function Comments

Declaration comments describe use of the function (when it is non-obvious); comments at the definition of a function describe operation.

#### Function Declarations

Almost every function declaration should have comments immediately preceding it that describe what the function does and how to use it. These comments may be omitted only if the function is simple and obvious (e.g., simple accessors for obvious properties of the class). Private methods and functions declared in `.cc` files are not exempt. Function comments should be written with an implied subject of _This function_ and should start with the verb phrase; for example, "Opens the file", rather than "Open the file". In general, these comments do not describe how the function performs its task. Instead, that should be left to comments in the function definition.

Types of things to mention in comments at the function declaration:

*   What the inputs and outputs are. If function argument names are provided in \`backticks\`, then code-indexing tools may be able to present the documentation better.
*   For class member functions: whether the object remembers reference or pointer arguments beyond the duration of the method call. This is quite common for pointer/reference arguments to constructors.
*   For each pointer argument, whether it is allowed to be null and what happens if it is.
*   For each output or input/output argument, what happens to any state that argument is in. (E.g. is the state appended to or overwritten?).
*   If there are any performance implications of how a function is used.

Here is an example:

// Returns an iterator for this table, positioned at the first entry
// lexically greater than or equal to \`start\_word\`. If there is no
// such entry, returns a null pointer. The client must not use the
// iterator after the underlying GargantuanTable has been destroyed.
//
// This method is equivalent to:
//    std::unique\_ptr<Iterator> iter = table->NewIterator();
//    iter->Seek(start\_word);
//    return iter;
std::unique\_ptr<Iterator> GetIterator(absl::string\_view start\_word) const;

However, do not be unnecessarily verbose or state the completely obvious.

When documenting function overrides, focus on the specifics of the override itself, rather than repeating the comment from the overridden function. In many of these cases, the override needs no additional documentation and thus no comment is required.

When commenting constructors and destructors, remember that the person reading your code knows what constructors and destructors are for, so comments that just say something like "destroys this object" are not useful. Document what constructors do with their arguments (for example, if they take ownership of pointers), and what cleanup the destructor does. If this is trivial, just skip the comment. It is quite common for destructors not to have a header comment.

#### Function Definitions

If there is anything tricky about how a function does its job, the function definition should have an explanatory comment. For example, in the definition comment you might describe any coding tricks you use, give an overview of the steps you go through, or explain why you chose to implement the function in the way you did rather than using a viable alternative. For instance, you might mention why it must acquire a lock for the first half of the function but why it is not needed for the second half.

Note you should _not_ just repeat the comments given with the function declaration, in the `.h` file or wherever. It's okay to recapitulate briefly what the function does, but the focus of the comments should be on how it does it.

### Variable Comments

In general the actual name of the variable should be descriptive enough to give a good idea of what the variable is used for. In certain cases, more comments are required.

#### Class Data Members

The purpose of each class data member (also called an instance variable or member variable) must be clear. If there are any invariants (special values, relationships between members, lifetime requirements) not clearly expressed by the type and name, they must be commented. However, if the type and name suffice (`int num_events_;`), no comment is needed.

In particular, add comments to describe the existence and meaning of sentinel values, such as nullptr or -1, when they are not obvious. For example:

private:
 // Used to bounds-check table accesses. -1 means
 // that we don't yet know how many entries the table has.
 int num\_total\_entries\_;

#### Global Variables

All global variables should have a comment describing what they are, what they are used for, and (if unclear) why they need to be global. For example:

// The total number of test cases that we run through in this regression test.
const int kNumTestCases = 6;

### Implementation Comments

In your implementation you should have comments in tricky, non-obvious, interesting, or important parts of your code.

#### Explanatory Comments

Tricky or complicated code blocks should have comments before them.

#### Function Argument Comments

When the meaning of a function argument is nonobvious, consider one of the following remedies:

*   If the argument is a literal constant, and the same constant is used in multiple function calls in a way that tacitly assumes they're the same, you should use a named constant to make that constraint explicit, and to guarantee that it holds.
*   Consider changing the function signature to replace a `bool` argument with an `enum` argument. This will make the argument values self-describing.
*   For functions that have several configuration options, consider defining a single class or struct to hold all the options , and pass an instance of that. This approach has several advantages. Options are referenced by name at the call site, which clarifies their meaning. It also reduces function argument count, which makes function calls easier to read and write. As an added benefit, you don't have to change call sites when you add another option.
*   Replace large or complex nested expressions with named variables.
*   As a last resort, use comments to clarify argument meanings at the call site.

Consider the following example:

// What are these arguments?
const DecimalNumber product = CalculateProduct(values, 7, false, nullptr);

versus:

ProductOptions options;
options.set\_precision\_decimals(7);
options.set\_use\_cache(ProductOptions::kDontUseCache);
const DecimalNumber product =
    CalculateProduct(values, options, /\*completion\_callback=\*/nullptr);

#### Don'ts

Do not state the obvious. In particular, don't literally describe what code does, unless the behavior is nonobvious to a reader who understands C++ well. Instead, provide higher level comments that describe _why_ the code does what it does, or make the code self describing.

Compare this:

// Find the element in the vector.  <-- Bad: obvious!
if (std::find(v.begin(), v.end(), element) != v.end()) {
  Process(element);
}

To this:

// Process "element" unless it was already processed.
if (std::find(v.begin(), v.end(), element) != v.end()) {
  Process(element);
}

Self-describing code doesn't need a comment. The comment from the example above would be obvious:

if (!IsAlreadyProcessed(element)) {
  Process(element);
}

### Punctuation, Spelling, and Grammar

Pay attention to punctuation, spelling, and grammar; it is easier to read well-written comments than badly written ones.

Comments should be as readable as narrative text, with proper capitalization and punctuation. In many cases, complete sentences are more readable than sentence fragments. Shorter comments, such as comments at the end of a line of code, can sometimes be less formal, but you should be consistent with your style.

Although it can be frustrating to have a code reviewer point out that you are using a comma when you should be using a semicolon, it is very important that source code maintain a high level of clarity and readability. Proper punctuation, spelling, and grammar help with that goal.

### TODO Comments

Use `TODO` comments for code that is temporary, a short-term solution, or good-enough but not perfect.

`TODO`s should include the string `TODO` in all caps, followed by the bug ID, name, e-mail address, or other identifier of the person or issue with the best context about the problem referenced by the `TODO`.

// TODO: bug 12345678 - Remove this after the 2047q4 compatibility window expires.
// TODO: example.com/my-design-doc - Manually fix up this code the next time it's touched.
// TODO(bug 12345678): Update this list after the Foo service is turned down.
// TODO(John): Use a "\\\*" here for concatenation operator.

If your `TODO` is of the form "At a future date do something" make sure that you either include a very specific date ("Fix by November 2005") or a very specific event ("Remove this code when all clients can handle XML responses.").

Formatting
----------

Coding style and formatting are pretty arbitrary, but a project is much easier to follow if everyone uses the same style. Individuals may not agree with every aspect of the formatting rules, and some of the rules may take some getting used to, but it is important that all project contributors follow the style rules so that they can all read and understand everyone's code easily.

To help you format code correctly, we've created a [settings file for emacs](https://raw.githubusercontent.com/google/styleguide/gh-pages/google-c-style.el).

### Line Length

Each line of text in your code should be at most 80 characters long.

We recognize that this rule is controversial, but so much existing code already adheres to it, and we feel that consistency is important.

Those who favor this rule argue that it is rude to force them to resize their windows and there is no need for anything longer. Some folks are used to having several code windows side-by-side, and thus don't have room to widen their windows in any case. People set up their work environment assuming a particular maximum window width, and 80 columns has been the traditional standard. Why change it?

Proponents of change argue that a wider line can make code more readable. The 80-column limit is an hidebound throwback to 1960s mainframes; modern equipment has wide screens that can easily show longer lines.

80 characters is the maximum.

A line may exceed 80 characters if it is

*   a comment line which is not feasible to split without harming readability, ease of cut and paste or auto-linking -- e.g., if a line contains an example command or a literal URL longer than 80 characters.
*   a string literal that cannot easily be wrapped at 80 columns. This may be because it contains URIs or other semantically-critical pieces, or because the literal contains an embedded language, or a multiline literal whose newlines are significant like help messages. In these cases, breaking up the literal would reduce readability, searchability, ability to click links, etc. Except for test code, such literals should appear at namespace scope near the top of a file. If a tool like Clang-Format doesn't recognize the unsplittable content, [disable the tool](https://clang.llvm.org/docs/ClangFormatStyleOptions.html#disabling-formatting-on-a-piece-of-code) around the content as necessary.  
      
    (We must balance between usability/searchability of such literals and the readability of the code around them.)
*   an include statement.
*   a [header guard](#The__define_Guard)
*   a using-declaration

### Non-ASCII Characters

Non-ASCII characters should be rare, and must use UTF-8 formatting.

You shouldn't hard-code user-facing text in source, even English, so use of non-ASCII characters should be rare. However, in certain cases it is appropriate to include such words in your code. For example, if your code parses data files from foreign sources, it may be appropriate to hard-code the non-ASCII string(s) used in those data files as delimiters. More commonly, unittest code (which does not need to be localized) might contain non-ASCII strings. In such cases, you should use UTF-8, since that is an encoding understood by most tools able to handle more than just ASCII.

Hex encoding is also OK, and encouraged where it enhances readability — for example, `"\xEF\xBB\xBF"`, or, even more simply, `"\uFEFF"`, is the Unicode zero-width no-break space character, which would be invisible if included in the source as straight UTF-8.

When possible, avoid the `u8` prefix. It has significantly different semantics starting in C++20 than in C++17, producing arrays of `char8_t` rather than `char`, and will change again in C++23.

You shouldn't use `char16_t` and `char32_t` character types, since they're for non-UTF-8 text. For similar reasons you also shouldn't use `wchar_t` (unless you're writing code that interacts with the Windows API, which uses `wchar_t` extensively).

### Spaces vs. Tabs

Use only spaces, and indent 2 spaces at a time.

We use spaces for indentation. Do not use tabs in your code. You should set your editor to emit spaces when you hit the tab key.

### Function Declarations and Definitions

Return type on the same line as function name, parameters on the same line if they fit. Wrap parameter lists which do not fit on a single line as you would wrap arguments in a [function call](#Function_Calls).

Functions look like this:

ReturnType ClassName::FunctionName(Type par\_name1, Type par\_name2) {
  DoSomething();
  ...
}

If you have too much text to fit on one line:

ReturnType ClassName::ReallyLongFunctionName(Type par\_name1, Type par\_name2,
                                             Type par\_name3) {
  DoSomething();
  ...
}

or if you cannot fit even the first parameter:

ReturnType LongClassName::ReallyReallyReallyLongFunctionName(
    Type par\_name1,  // 4 space indent
    Type par\_name2,
    Type par\_name3) {
  DoSomething();  // 2 space indent
  ...
}

Some points to note:

*   Choose good parameter names.
*   A parameter name may be omitted only if the parameter is not used in the function's definition.
*   If you cannot fit the return type and the function name on a single line, break between them.
*   If you break after the return type of a function declaration or definition, do not indent.
*   The open parenthesis is always on the same line as the function name.
*   There is never a space between the function name and the open parenthesis.
*   There is never a space between the parentheses and the parameters.
*   The open curly brace is always on the end of the last line of the function declaration, not the start of the next line.
*   The close curly brace is either on the last line by itself or on the same line as the open curly brace.
*   There should be a space between the close parenthesis and the open curly brace.
*   All parameters should be aligned if possible.
*   Default indentation is 2 spaces.
*   Wrapped parameters have a 4 space indent.

Unused parameters that are obvious from context may be omitted:

class Foo {
 public:
  Foo(const Foo&) = delete;
  Foo& operator=(const Foo&) = delete;
};

Unused parameters that might not be obvious should comment out the variable name in the function definition:

class Shape {
 public:
  virtual void Rotate(double radians) = 0;
};

class Circle : public Shape {
 public:
  void Rotate(double radians) override;
};

void Circle::Rotate(double /\*radians\*/) {}

// Bad - if someone wants to implement later, it's not clear what the
// variable means.
void Circle::Rotate(double) {}

Attributes, and macros that expand to attributes, appear at the very beginning of the function declaration or definition, before the return type:

  ABSL\_ATTRIBUTE\_NOINLINE void ExpensiveFunction();
  \[\[nodiscard\]\] bool IsOk();

### Lambda Expressions

Format parameters and bodies as for any other function, and capture lists like other comma-separated lists.

For by-reference captures, do not leave a space between the ampersand (`&`) and the variable name.

int x = 0;
auto x\_plus\_n = \[&x\](int n) -> int { return x + n; }

Short lambdas may be written inline as function arguments.

absl::flat\_hash\_set<int> to\_remove = {7, 8, 9};
std::vector<int> digits = {3, 9, 1, 8, 4, 7, 1};
digits.erase(std::remove\_if(digits.begin(), digits.end(), \[&to\_remove\](int i) {
               return to\_remove.contains(i);
             }),
             digits.end());

### Floating-point Literals

Floating-point literals should always have a radix point, with digits on both sides, even if they use exponential notation. Readability is improved if all floating-point literals take this familiar form, as this helps ensure that they are not mistaken for integer literals, and that the `E`/`e` of the exponential notation is not mistaken for a hexadecimal digit. It is fine to initialize a floating-point variable with an integer literal (assuming the variable type can exactly represent that integer), but note that a number in exponential notation is never an integer literal.

float f = 1.f;
long double ld = -.5L;
double d = 1248e6;

float f = 1.0f;
float f2 = 1.0;  // Also OK
float f3 = 1;    // Also OK
long double ld = -0.5L;
double d = 1248.0e6;

### Function Calls

Either write the call all on a single line, wrap the arguments at the parenthesis, or start the arguments on a new line indented by four spaces and continue at that 4 space indent. In the absence of other considerations, use the minimum number of lines, including placing multiple arguments on each line where appropriate.

Function calls have the following format:

bool result = DoSomething(argument1, argument2, argument3);

If the arguments do not all fit on one line, they should be broken up onto multiple lines, with each subsequent line aligned with the first argument. Do not add spaces after the open paren or before the close paren:

bool result = DoSomething(averyveryveryverylongargument1,
                          argument2, argument3);

Arguments may optionally all be placed on subsequent lines with a four space indent:

if (...) {
  ...
  ...
  if (...) {
    bool result = DoSomething(
        argument1, argument2,  // 4 space indent
        argument3, argument4);
    ...
  }

Put multiple arguments on a single line to reduce the number of lines necessary for calling a function unless there is a specific readability problem. Some find that formatting with strictly one argument on each line is more readable and simplifies editing of the arguments. However, we prioritize for the reader over the ease of editing arguments, and most readability problems are better addressed with the following techniques.

If having multiple arguments in a single line decreases readability due to the complexity or confusing nature of the expressions that make up some arguments, try creating variables that capture those arguments in a descriptive name:

int my\_heuristic = scores\[x\] \* y + bases\[x\];
bool result = DoSomething(my\_heuristic, x, y, z);

Or put the confusing argument on its own line with an explanatory comment:

bool result = DoSomething(scores\[x\] \* y + bases\[x\],  // Score heuristic.
                          x, y, z);

If there is still a case where one argument is significantly more readable on its own line, then put it on its own line. The decision should be specific to the argument which is made more readable rather than a general policy.

Sometimes arguments form a structure that is important for readability. In those cases, feel free to format the arguments according to that structure:

// Transform the widget by a 3x3 matrix.
my\_widget.Transform(x1, x2, x3,
                    y1, y2, y3,
                    z1, z2, z3);

### Braced Initializer List Format

Format a braced initializer list exactly like you would format a function call in its place.

If the braced list follows a name (e.g., a type or variable name), format as if the `{}` were the parentheses of a function call with that name. If there is no name, assume a zero-length name.

// Examples of braced init list on a single line.
return {foo, bar};
functioncall({foo, bar});
std::pair<int, int> p{foo, bar};

// When you have to wrap.
SomeFunction(
    {"assume a zero-length name before {"},
    some\_other\_function\_parameter);
SomeType variable{
    some, other, values,
    {"assume a zero-length name before {"},
    SomeOtherType{
        "Very long string requiring the surrounding breaks.",
        some, other, values},
    SomeOtherType{"Slightly shorter string",
                  some, other, values}};
SomeType variable{
    "This is too long to fit all in one line"};
MyType m = {  // Here, you could also break before {.
    superlongvariablename1,
    superlongvariablename2,
    {short, interior, list},
    {interiorwrappinglist,
     interiorwrappinglist2}};

### Looping and branching statements

At a high level, looping or branching statements consist of the following **components**:

*   One or more **statement keywords** (e.g. `if`, `else`, `switch`, `while`, `do`, or `for`).
*   One **condition or iteration specifier**, inside parentheses.
*   One or more **controlled statements**, or blocks of controlled statements.

For these statements:

*   The components of the statement should be separated by single spaces (not line breaks).
*   Inside the condition or iteration specifier, put one space (or a line break) between each semicolon and the next token, except if the token is a closing parenthesis or another semicolon.
*   Inside the condition or iteration specifier, do not put a space after the opening parenthesis or before the closing parenthesis.
*   Put any controlled statements inside blocks (i.e. use curly braces).
*   Inside the controlled blocks, put one line break immediately after the opening brace, and one line break immediately before the closing brace.

if (condition) {                   // Good - no spaces inside parentheses, space before brace.
  DoOneThing();                    // Good - two-space indent.
  DoAnotherThing();
} else if (int a = f(); a != 3) {  // Good - closing brace on new line, else on same line.
  DoAThirdThing(a);
} else {
  DoNothing();
}

// Good - the same rules apply to loops.
while (condition) {
  RepeatAThing();
}

// Good - the same rules apply to loops.
do {
  RepeatAThing();
} while (condition);

// Good - the same rules apply to loops.
for (int i = 0; i < 10; ++i) {
  RepeatAThing();
}

if(condition) {}                   // Bad - space missing after \`if\`.
else if ( condition ) {}           // Bad - space between the parentheses and the condition.
else if (condition){}              // Bad - space missing before \`{\`.
else if(condition){}               // Bad - multiple spaces missing.

for (int a = f();a == 10) {}       // Bad - space missing after the semicolon.

// Bad - \`if ... else\` statement does not have braces everywhere.
if (condition)
  foo;
else {
  bar;
}

// Bad - \`if\` statement too long to omit braces.
if (condition)
  // Comment
  DoSomething();

// Bad - \`if\` statement too long to omit braces.
if (condition1 &&
    condition2)
  DoSomething();

For historical reasons, we allow one exception to the above rules: the curly braces for the controlled statement or the line breaks inside the curly braces may be omitted if as a result the entire statement appears on either a single line (in which case there is a space between the closing parenthesis and the controlled statement) or on two lines (in which case there is a line break after the closing parenthesis and there are no braces).

// OK - fits on one line.
if (x == kFoo) { return new Foo(); }

// OK - braces are optional in this case.
if (x == kFoo) return new Foo();

// OK - condition fits on one line, body fits on another.
if (x == kBar)
  Bar(arg1, arg2, arg3);

This exception does not apply to multi-keyword statements like `if ... else` or `do ... while`.

// Bad - \`if ... else\` statement is missing braces.
if (x) DoThis();
else DoThat();

// Bad - \`do ... while\` statement is missing braces.
do DoThis();
while (x);

Use this style only when the statement is brief, and consider that loops and branching statements with complex conditions or controlled statements may be more readable with curly braces. Some projects require curly braces always.

`case` blocks in `switch` statements can have curly braces or not, depending on your preference. If you do include curly braces, they should be placed as shown below.

switch (var) {
  case 0: {  // 2 space indent
    Foo();   // 4 space indent
    break;
  }
  default: {
    Bar();
  }
}

Empty loop bodies should use either an empty pair of braces or `continue` with no braces, rather than a single semicolon.

while (condition) {}  // Good - \`{}\` indicates no logic.
while (condition) {
  // Comments are okay, too
}
while (condition) continue;  // Good - \`continue\` indicates no logic.

while (condition);  // Bad - looks like part of \`do-while\` loop.

### Pointer and Reference Expressions

No spaces around period or arrow. Pointer operators do not have trailing spaces.

The following are examples of correctly-formatted pointer and reference expressions:

x = \*p;
p = &x;
x = r.y;
x = r->y;

Note that:

*   There are no spaces around the period or arrow when accessing a member.
*   Pointer operators have no space after the `*` or `&`.

When referring to a pointer or reference (variable declarations or definitions, arguments, return types, template parameters, etc), you may place the space before or after the asterisk/ampersand. In the trailing-space style, the space is elided in some cases (template parameters, etc).

// These are fine, space preceding.
char \*c;
const std::string &str;
int \*GetPointer();
std::vector<char \*>

// These are fine, space following (or elided).
char\* c;
const std::string& str;
int\* GetPointer();
std::vector<char\*>  // Note no space between '\*' and '>'

You should do this consistently within a single file. When modifying an existing file, use the style in that file.

It is allowed (if unusual) to declare multiple variables in the same declaration, but it is disallowed if any of those have pointer or reference decorations. Such declarations are easily misread.

// Fine if helpful for readability.
int x, y;

int x, \*y;  // Disallowed - no & or \* in multiple declaration
int\* x, \*y;  // Disallowed - no & or \* in multiple declaration; inconsistent spacing
char \* c;  // Bad - spaces on both sides of \*
const std::string & str;  // Bad - spaces on both sides of &

### Boolean Expressions

When you have a boolean expression that is longer than the [standard line length](#Line_Length), be consistent in how you break up the lines.

In this example, the logical AND operator is always at the end of the lines:

if (this\_one\_thing > this\_other\_thing &&
    a\_third\_thing == a\_fourth\_thing &&
    yet\_another && last\_one) {
  ...
}

Note that when the code wraps in this example, both of the `&&` logical AND operators are at the end of the line. This is more common in Google code, though wrapping all operators at the beginning of the line is also allowed. Feel free to insert extra parentheses judiciously because they can be very helpful in increasing readability when used appropriately, but be careful about overuse. Also note that you should always use the punctuation operators, such as `&&` and `~`, rather than the word operators, such as `and` and `compl`.

### Return Values

Do not needlessly surround the `return` expression with parentheses.

Use parentheses in `return expr;` only where you would use them in `x = expr;`.

return result;                  // No parentheses in the simple case.
// Parentheses OK to make a complex expression more readable.
return (some\_long\_condition &&
        another\_condition);

return (value);                // You wouldn't write var = (value);
return(result);                // return is not a function!

### Variable and Array Initialization

You may choose between `=`, `()`, and `{}`; the following are all correct:

int x = 3;
int x(3);
int x{3};
std::string name = "Some Name";
std::string name("Some Name");
std::string name{"Some Name"};

Be careful when using a braced initialization list `{...}` on a type with an `std::initializer_list` constructor. A nonempty _braced-init-list_ prefers the `std::initializer_list` constructor whenever possible. Note that empty braces `{}` are special, and will call a default constructor if available. To force the non-`std::initializer_list` constructor, use parentheses instead of braces.

std::vector<int> v(100, 1);  // A vector containing 100 items: All 1s.
std::vector<int> v{100, 1};  // A vector containing 2 items: 100 and 1.

Also, the brace form prevents narrowing of integral types. This can prevent some types of programming errors.

int pi(3.14);  // OK -- pi == 3.
int pi{3.14};  // Compile error: narrowing conversion.

### Preprocessor Directives

The hash mark that starts a preprocessor directive should always be at the beginning of the line.

Even when preprocessor directives are within the body of indented code, the directives should start at the beginning of the line.

// Good - directives at beginning of line
  if (lopsided\_score) {
#if DISASTER\_PENDING      // Correct -- Starts at beginning of line
    DropEverything();
# if NOTIFY               // OK but not required -- Spaces after #
    NotifyClient();
# endif
#endif
    BackToNormal();
  }

// Bad - indented directives
  if (lopsided\_score) {
    #if DISASTER\_PENDING  // Wrong!  The "#if" should be at beginning of line
    DropEverything();
    #endif                // Wrong!  Do not indent "#endif"
    BackToNormal();
  }

### Class Format

Sections in `public`, `protected` and `private` order, each indented one space.

The basic format for a class definition (lacking the comments, see [Class Comments](#Class_Comments) for a discussion of what comments are needed) is:

class MyClass : public OtherClass {
 public:      // Note the 1 space indent!
  MyClass();  // Regular 2 space indent.
  explicit MyClass(int var);
  ~MyClass() {}

  void SomeFunction();
  void SomeFunctionThatDoesNothing() {
  }

  void set\_some\_var(int var) { some\_var\_ = var; }
  int some\_var() const { return some\_var\_; }

 private:
  bool SomeInternalFunction();

  int some\_var\_;
  int some\_other\_var\_;
};

Things to note:

*   Any base class name should be on the same line as the subclass name, subject to the 80-column limit.
*   The `public:`, `protected:`, and `private:` keywords should be indented one space.
*   Except for the first instance, these keywords should be preceded by a blank line. This rule is optional in small classes.
*   Do not leave a blank line after these keywords.
*   The `public` section should be first, followed by the `protected` and finally the `private` section.
*   See [Declaration Order](#Declaration_Order) for rules on ordering declarations within each of these sections.

### Constructor Initializer Lists

Constructor initializer lists can be all on one line or with subsequent lines indented four spaces.

The acceptable formats for initializer lists are:

// When everything fits on one line:
MyClass::MyClass(int var) : some\_var\_(var) {
  DoSomething();
}

// If the signature and initializer list are not all on one line,
// you must wrap before the colon and indent 4 spaces:
MyClass::MyClass(int var)
    : some\_var\_(var), some\_other\_var\_(var + 1) {
  DoSomething();
}

// When the list spans multiple lines, put each member on its own line
// and align them:
MyClass::MyClass(int var)
    : some\_var\_(var),             // 4 space indent
      some\_other\_var\_(var + 1) {  // lined up
  DoSomething();
}

// As with any other code block, the close curly can be on the same
// line as the open curly, if it fits.
MyClass::MyClass(int var)
    : some\_var\_(var) {}

### Namespace Formatting

The contents of namespaces are not indented.

[Namespaces](#Namespaces) do not add an extra level of indentation. For example, use:

namespace {

void foo() {  // Correct.  No extra indentation within namespace.
  ...
}

}  // namespace

Do not indent within a namespace:

namespace {

  // Wrong!  Indented when it should not be.
  void foo() {
    ...
  }

}  // namespace

### Horizontal Whitespace

Use of horizontal whitespace depends on location. Never put trailing whitespace at the end of a line.

#### General

int i = 0;  // Two spaces before end-of-line comments.

void f(bool b) {  // Open braces should always have a space before them.
  ...
int i = 0;  // Semicolons usually have no space before them.
// Spaces inside braces for braced-init-list are optional.  If you use them,
// put them on both sides!
int x\[\] = { 0 };
int x\[\] = {0};

// Spaces around the colon in inheritance and initializer lists.
class Foo : public Bar {
 public:
  // For inline function implementations, put spaces between the braces
  // and the implementation itself.
  Foo(int b) : Bar(), baz\_(b) {}  // No spaces inside empty braces.
  void Reset() { baz\_ = 0; }  // Spaces separating braces from implementation.
  ...

Adding trailing whitespace can cause extra work for others editing the same file, when they merge, as can removing existing trailing whitespace. So: Don't introduce trailing whitespace. Remove it if you're already changing that line, or do it in a separate clean-up operation (preferably when no-one else is working on the file).

#### Loops and Conditionals

if (b) {          // Space after the keyword in conditions and loops.
} else {          // Spaces around else.
}
while (test) {}   // There is usually no space inside parentheses.
switch (i) {
for (int i = 0; i < 5; ++i) {
// Loops and conditions may have spaces inside parentheses, but this
// is rare.  Be consistent.
switch ( i ) {
if ( test ) {
for ( int i = 0; i < 5; ++i ) {
// For loops always have a space after the semicolon.  They may have a space
// before the semicolon, but this is rare.
for ( ; i < 5 ; ++i) {
  ...

// Range-based for loops always have a space before and after the colon.
for (auto x : counts) {
  ...
}
switch (i) {
  case 1:         // No space before colon in a switch case.
    ...
  case 2: break;  // Use a space after a colon if there's code after it.

#### Operators

// Assignment operators always have spaces around them.
x = 0;

// Other binary operators usually have spaces around them, but it's
// OK to remove spaces around factors.  Parentheses should have no
// internal padding.
v = w \* x + y / z;
v = w\*x + y/z;
v = w \* (x + z);

// No spaces separating unary operators and their arguments.
x = -5;
++x;
if (x && !y)
  ...

#### Templates and Casts

// No spaces inside the angle brackets (< and >), before
// <, or between >( in a cast
std::vector<std::string> x;
y = static\_cast<char\*>(x);

// Spaces between type and pointer are OK, but be consistent.
std::vector<char \*> x;

### Vertical Whitespace

Minimize use of vertical whitespace.

This is more a principle than a rule: don't use blank lines when you don't have to. In particular, don't put more than one or two blank lines between functions, resist starting functions with a blank line, don't end functions with a blank line, and be sparing with your use of blank lines. A blank line within a block of code serves like a paragraph break in prose: visually separating two thoughts.

The basic principle is: The more code that fits on one screen, the easier it is to follow and understand the control flow of the program. Use whitespace purposefully to provide separation in that flow.

Some rules of thumb to help when blank lines may be useful:

*   Blank lines at the beginning or end of a function do not help readability.
*   Blank lines inside a chain of if-else blocks may well help readability.
*   A blank line before a comment line usually helps readability — the introduction of a new comment suggests the start of a new thought, and the blank line makes it clear that the comment goes with the following thing instead of the preceding.
*   Blank lines immediately inside a declaration of a namespace or block of namespaces may help readability by visually separating the load-bearing content from the (largely non-semantic) organizational wrapper. Especially when the first declaration inside the namespace(s) is preceded by a comment, this becomes a special case of the previous rule, helping the comment to "attach" to the subsequent declaration.

Exceptions to the Rules
-----------------------

The coding conventions described above are mandatory. However, like all good rules, these sometimes have exceptions, which we discuss here.

### Existing Non-conformant Code

You may diverge from the rules when dealing with code that does not conform to this style guide.

If you find yourself modifying code that was written to specifications other than those presented by this guide, you may have to diverge from these rules in order to stay consistent with the local conventions in that code. If you are in doubt about how to do this, ask the original author or the person currently responsible for the code. Remember that _consistency_ includes local consistency, too.

### Windows Code

Windows programmers have developed their own set of coding conventions, mainly derived from the conventions in Windows headers and other Microsoft code. We want to make it easy for anyone to understand your code, so we have a single set of guidelines for everyone writing C++ on any platform.

It is worth reiterating a few of the guidelines that you might forget if you are used to the prevalent Windows style:

*   Do not use Hungarian notation (for example, naming an integer `iNum`). Use the Google naming conventions, including the `.cc` extension for source files.
*   Windows defines many of its own synonyms for primitive types, such as `DWORD`, `HANDLE`, etc. It is perfectly acceptable, and encouraged, that you use these types when calling Windows API functions. Even so, keep as close as you can to the underlying C++ types. For example, use `const TCHAR *` instead of `LPCTSTR`.
*   When compiling with Microsoft Visual C++, set the compiler to warning level 3 or higher, and treat all warnings as errors.
*   Do not use `#pragma once`; instead use the standard Google include guards. The path in the include guards should be relative to the top of your project tree.
*   In fact, do not use any nonstandard extensions, like `#pragma` and `__declspec`, unless you absolutely must. Using `__declspec(dllimport)` and `__declspec(dllexport)` is allowed; however, you must use them through macros such as `DLLIMPORT` and `DLLEXPORT`, so that someone can easily disable the extensions if they share the code.

However, there are just a few rules that we occasionally need to break on Windows:

*   Normally we [strongly discourage the use of multiple implementation inheritance](#Multiple_Inheritance); however, it is required when using COM and some ATL/WTL classes. You may use multiple implementation inheritance to implement COM or ATL/WTL classes and interfaces.
*   Although you should not use exceptions in your own code, they are used extensively in the ATL and some STLs, including the one that comes with Visual C++. When using the ATL, you should define `_ATL_NO_EXCEPTIONS` to disable exceptions. You should investigate whether you can also disable exceptions in your STL, but if not, it is OK to turn on exceptions in the compiler. (Note that this is only to get the STL to compile. You should still not write exception handling code yourself.)
*   The usual way of working with precompiled headers is to include a header file at the top of each source file, typically with a name like `StdAfx.h` or `precompile.h`. To make your code easier to share with other projects, avoid including this file explicitly (except in `precompile.cc`), and use the `/FI` compiler option to include the file automatically.
*   Resource headers, which are usually named `resource.h` and contain only macros, do not need to conform to these style guidelines.

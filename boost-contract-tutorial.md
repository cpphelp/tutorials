# Boost.Contract: A Complete Tutorial and Instruction Manual

### Introduction

This tutorial is a comprehensive guide to contract programming in C++ using the [Boost.Contract](https://www.boost.org/doc/libs/release/libs/contract/doc/html/index.html) library. You will start with no prior knowledge of contract programming and finish with the ability to use every feature the library offers, including subcontracting across class hierarchies, custom failure handlers, assertion levels, and compile-time configuration.

Contract programming is one of the most powerful techniques available for writing correct software. By the time you reach the end of this document, you will have written contracts for free functions, constructors, destructors, public member functions, virtual functions with full subcontracting, and more. You will understand when and why to use each feature, and you will know how to tune contract checking for both development and production builds.

The tutorial is divided into four parts. Part I introduces the concept of contract programming itself and gets you up and running with Boost.Contract. Part II covers the core day-to-day features you will use in every project. Part III tackles advanced topics like inheritance, virtual functions, and move semantics. Part IV addresses expert-level concerns such as assertion levels, failure handlers, selective disabling, and real-world design patterns.

## Prerequisites

Before you begin, make sure you have the following in place:

- A C++11-capable compiler. GCC 5 or later, Clang 3.4 or later, and MSVC 2015 or later are all supported.
- A working Boost installation. If you have not yet installed Boost, follow the [Boost Getting Started Guide](https://www.boost.org/doc/libs/release/more/getting_started/index.html).
- Familiarity with C++ classes, inheritance, virtual functions, and lambda expressions. You do not need to be an expert, but you should be comfortable reading code that uses these features.

---

# Part I - Foundations

## Step 1 - Understanding Contract Programming

Before you write a single line of Boost.Contract code, you need to understand what contract programming is and why it exists. This section teaches you the concept from scratch.

### The Metaphor

Think about a legal contract between two parties. A homeowner hires a contractor to build a deck. The contract states that the homeowner will provide the lumber and a clear site (the homeowner's obligations), and the contractor will build a structurally sound deck by a certain date (the contractor's guarantee). If the homeowner does not provide lumber, the contractor is not at fault for missing the deadline. If the homeowner provides everything promised and the deck collapses, the contractor is at fault.

Software contract programming works the same way. Every function has a *caller* (the client) and a *callee* (the supplier). The contract between them specifies what the caller must provide and what the callee guarantees in return. When both sides honor their obligations, the software works correctly. When a contract is violated, the violation points directly to the party at fault, making bugs dramatically easier to find.

### The Three Pillars

Contract programming rests on three core concepts:

**Preconditions** are conditions that must be true when a function is called. They are the caller's obligations. For example, a square root function might require that its argument is non-negative. If the caller passes a negative number, the caller has violated the contract, and the callee is not responsible for the result.

**Postconditions** are conditions that must be true when a function returns (assuming no exception was thrown). They are the callee's guarantees. For example, the square root function guarantees that its result, when squared, equals the original argument (within floating-point tolerance). If this condition is false after the function returns, the callee has a bug.

**Class invariants** are conditions that must hold for every object of a class whenever that object is visible to client code. They are checked after constructors complete, before and after every public member function call, and before destructors run. For example, a `vector` class might maintain the invariant that its size is always less than or equal to its capacity.

### Obligations and Benefits

The relationship between caller and callee can be summarized as follows:

| | Caller (Client) | Callee (Supplier) |
|---|---|---|
| **Obligation** | Satisfy preconditions | Satisfy postconditions and maintain invariants |
| **Benefit** | Can rely on postconditions and invariants | Can assume preconditions are met |

This table captures the essence of Design by Contract. The caller benefits from the callee's guarantees, and the callee benefits from the caller's obligations. Neither side needs to check what the other side is responsible for, which eliminates redundant defensive checks and makes the code cleaner and more focused.

### How Contracts Differ from Defensive Programming

In defensive programming, every function validates its inputs regardless of who is responsible for them. A square root function might check for negative arguments and return an error code or throw an exception. This approach is safe, but it has costs: the checks are redundant when the caller already ensures correctness, they obscure the function's core logic, and they make it unclear who is responsible for the error.

Contract programming takes a different stance. The square root function declares a precondition that the argument must be non-negative. If the caller violates this precondition, the program has a bug, and the contract violation is detected immediately rather than being masked by a graceful error return. Contracts are about *correctness* (finding bugs), not about *robustness* (handling expected errors). Both techniques have their place, and Boost.Contract does not prevent you from using exceptions and error codes for expected runtime conditions. Contracts address a different concern: ensuring that the assumptions your code depends on are actually true.

### Origins

Contract programming was formalized by Bertrand Meyer in the Eiffel programming language in 1986. Meyer coined the term *Design by Contract* and built contract support directly into Eiffel's syntax with `require` (preconditions), `ensure` (postconditions), and `invariant` (class invariants) keywords. His book *Object-Oriented Software Construction* (Prentice Hall, 1997) remains the definitive reference on the topic.

The ideas behind contracts are even older, rooted in C.A.R. Hoare's work on program verification from the 1960s. Hoare logic provides the formal foundation: a *Hoare triple* `{P} S {Q}` states that if precondition `P` holds before statement `S` executes, then postcondition `Q` holds afterward. Contracts are the practical, programmer-friendly expression of this formal concept.

In the C++ world, contracts have been a long-sought language feature. The C++26 standard introduces contract assertions with `[[pre:]]`, `[[post:]]`, and `[[assert:]]` attributes. However, the standard facility does not include class invariants, old values, or subcontracting. Boost.Contract provides all of these features today, on any C++11 compiler.

### References

- Bertrand Meyer, *Object-Oriented Software Construction*, 2nd edition, Prentice Hall, 1997
- [Design by Contract - Wikipedia](https://en.wikipedia.org/wiki/Design_by_contract)
- [Building Bug-Free O-O Software - Eiffel.com](https://www.eiffel.com/values/design-by-contract/introduction/)
- [C++26 Contract Assertions - cppreference](https://en.cppreference.com/w/cpp/language/contracts)

Now that you understand what contract programming is and why it matters, you are ready to start using it in C++.

## Step 2 - Getting Started with Boost.Contract

In this section you will set up Boost.Contract and write your first contract-checked function.

### Including the Library

The easiest way to use Boost.Contract is to include the convenience header:

```cpp
#include <boost/contract.hpp>
```

This single header pulls in everything you need. If you prefer finer-grained includes, individual headers are available under `boost/contract/` (such as `boost/contract/function.hpp`, `boost/contract/constructor.hpp`, and so on), but the convenience header is recommended for most uses.

### Linking

Boost.Contract is a compiled library by default. You need to link against `libboost_contract` (or `boost_contract` depending on your build system). There are three linking modes:

- **Dynamic linking** (default): Define `BOOST_CONTRACT_DYN_LINK` when building your project. This is the recommended mode.
- **Static linking**: Define `BOOST_CONTRACT_STATIC_LINK` to link the library statically.
- **Header-only**: Define `BOOST_CONTRACT_HEADER_ONLY` to avoid linking entirely. This is convenient for small projects but is not recommended for larger codebases because it increases compilation time and binary size.

### Your First Contract

Create a file called `first_contract.cpp` and add the following code:

```cpp
#include <boost/contract.hpp>
#include <limits>
#include <cassert>

void inc(int& x) {
    boost::contract::old_ptr<int> old_x = BOOST_CONTRACT_OLDOF(x);
    boost::contract::check c = boost::contract::function()
        .precondition([&] {
            BOOST_CONTRACT_ASSERT(x < std::numeric_limits<int>::max());
        })
        .postcondition([&] {
            BOOST_CONTRACT_ASSERT(x == *old_x + 1);
        })
    ;

    ++x; // Function body.
}

int main() {
    int x = 10;
    inc(x);
    assert(x == 11);
    return 0;
}
```

This function increments an integer. The precondition states that `x` must not already be at its maximum value (otherwise the increment would overflow). The postcondition states that after the function returns, `x` is one greater than its old value. The function body is the single line `++x;` that appears after the contract declaration.

### Anatomy of the Pattern

Every Boost.Contract contract follows the same structural pattern. Understanding it now will make the rest of this tutorial flow naturally.

**`boost::contract::function()`** creates a contract specification object for a non-member function (or a private/protected member function). Other entry points exist for constructors (`boost::contract::constructor`), destructors (`boost::contract::destructor`), and public member functions (`boost::contract::public_function`), but they all follow the same fluent API.

**`.precondition([&] { ... })`** takes a lambda that contains your precondition checks. The lambda is called at function entry.

**`.postcondition([&] { ... })`** takes a lambda containing postcondition checks. The lambda is called when the function returns normally (not when it throws).

**`BOOST_CONTRACT_ASSERT(condition)`** is a macro that checks a boolean condition. If the condition is false, a contract violation occurs. By default, this calls `std::terminate`, but you can customize this behavior (covered in a later section).

**`boost::contract::check c = ...`** is the crucial RAII variable. You must assign the result of the contract specification to a variable of type `boost::contract::check`, declared with an explicit type (not `auto`), and placed immediately before the function body. The `check` object's constructor runs the preconditions, and its destructor runs the postconditions. If you forget this variable or declare it incorrectly, the library will detect it and generate a runtime error.

**`boost::contract::old_ptr<T>`** stores a copy of a value taken before the function body executes, so that postconditions can compare the "after" state to the "before" state. The `BOOST_CONTRACT_OLDOF(expr)` macro creates the copy.

With this foundation in place, you are ready to explore each contract feature in depth.

---

# Part II - Core Concepts

## Step 3 - Non-Member Functions

In this section you will write complete contracts for free (non-member) functions, using all four contract clauses: preconditions, postconditions, old values, and exception guarantees.

The pattern for a non-member function looks like this:

```cpp
#include <boost/contract.hpp>
#include <limits>

int inc(int& x) {
    int result;
    boost::contract::old_ptr<int> old_x = BOOST_CONTRACT_OLDOF(x);
    boost::contract::check c = boost::contract::function()
        .precondition([&] {
            BOOST_CONTRACT_ASSERT(x < std::numeric_limits<int>::max());
        })
        .postcondition([&] {
            BOOST_CONTRACT_ASSERT(x == *old_x + 1);
            BOOST_CONTRACT_ASSERT(result == *old_x);
        })
        .except([&] {
            BOOST_CONTRACT_ASSERT(x == *old_x);
        })
    ;

    return result = x++; // Function body.
}
```

This version of `inc` returns the original value while incrementing `x`. The postcondition verifies both behaviors. The exception guarantee (`.except()`) states that if the function throws, `x` is unchanged. For a trivial increment operation, an exception is unlikely, but this clause becomes essential for functions that perform I/O, memory allocation, or other operations that can fail.

### The Fluent API Chain

The contract specification methods must be called in a specific order:

1. `.precondition(...)` - checked first, at function entry
2. `.old(...)` - old values copied after preconditions pass
3. `.postcondition(...)` - checked when the function returns normally
4. `.except(...)` - checked when the function throws an exception

Every clause is optional. You can specify any subset in the required order. If you only need preconditions and postconditions, omit `.old()` and `.except()`. If you only need postconditions, omit the rest. The fluent API enforces the ordering at compile time: calling `.precondition()` after `.postcondition()` will not compile.

You use the same `boost::contract::function()` entry point for non-member functions, private member functions, protected member functions, lambda functions, and even loop bodies. Any code that does not check class invariants uses this entry point.

With non-member function contracts covered, you are ready to look at old values more closely.

## Step 4 - Old Values

Postconditions frequently need to compare the state of the world after a function executes to the state before it executed. Old values make this possible. In this section you will learn the different ways to capture and use old values.

### Basic Old Values

The most common pattern declares an `old_ptr` and initializes it with `BOOST_CONTRACT_OLDOF`:

```cpp
boost::contract::old_ptr<int> old_x = BOOST_CONTRACT_OLDOF(x);
```

This copies the current value of `x` before the function body runs. In postconditions, you dereference the pointer with `*old_x` to access the saved value.

### Deferred Old Value Copies

Sometimes the expression you want to save as an old value depends on a precondition being satisfied. For example, if a precondition checks that an index is within bounds, and the old value expression accesses the element at that index, you do not want the old value copy to execute before the precondition is verified.

The `.old()` clause handles this case:

```cpp
char replace(std::string& s, unsigned index, char x) {
    char result;
    boost::contract::old_ptr<char> old_char; // Declared null.
    boost::contract::check c = boost::contract::function()
        .precondition([&] {
            BOOST_CONTRACT_ASSERT(index < s.size());
        })
        .old([&] { // Executed after preconditions pass.
            old_char = BOOST_CONTRACT_OLDOF(s[index]);
        })
        .postcondition([&] {
            BOOST_CONTRACT_ASSERT(s[index] == x);
            BOOST_CONTRACT_ASSERT(result == *old_char);
        })
    ;

    result = s[index];
    s[index] = x;
    return result;
}
```

Here `old_char` is declared without initialization (it starts as null). The `.old()` lambda runs after the precondition verifies that `index` is valid, so the `s[index]` expression is safe.

### Non-Copyable Types

In generic code, you might write contracts for types that are not copyable. The `BOOST_CONTRACT_OLDOF` macro requires the type to be copyable, and attempting to use it with a non-copyable type will produce a compile error. For this situation, Boost.Contract provides `old_ptr_if_copyable<T>`:

```cpp
boost::contract::old_ptr_if_copyable<T> old_val = BOOST_CONTRACT_OLDOF(val);
```

If `T` is copyable, `old_val` works like a normal `old_ptr`. If `T` is not copyable, `old_val` is always null, and postconditions that dereference it are silently skipped. This allows you to write generic contracts that work with any type, providing stronger checking when the type supports it and gracefully degrading when it does not.

Now that you understand how to capture past state, you are ready to learn about class invariants - the contracts that govern objects over their entire lifetime.

## Step 5 - Class Invariants

A class invariant is a condition that every object of a class must satisfy whenever that object is visible to client code. Unlike preconditions and postconditions, which apply to individual function calls, invariants apply to the object as a whole. They are the beating heart of contract programming for classes.

### When Invariants Are Checked

Boost.Contract checks class invariants at these points:

- **After a constructor completes**: the object has just been created and must already satisfy its invariants.
- **Before and after every public member function call**: the object must be consistent when the function starts and when it finishes.
- **Before a destructor begins**: the object must be in a valid state before it is destroyed.

Note that invariants are *not* checked for private or protected member functions. These functions are internal implementation details that may temporarily break invariants as part of a larger operation. Only the public interface, which client code can observe, enforces invariant checking.

### Writing an Invariant

You define an invariant by adding a `void invariant() const` member function to your class:

```cpp
class unique_identifiers :
    private boost::contract::constructor_precondition<unique_identifiers>
{
public:
    void invariant() const {
        BOOST_CONTRACT_ASSERT(size() >= 0);
    }

    // ... rest of the class
};
```

The `invariant()` function must be `const` because checking the invariant should not modify the object. You can put any number of `BOOST_CONTRACT_ASSERT` calls inside it.

### Static Invariants

Some invariants apply to the class as a whole rather than to any particular instance. For example, a class that tracks how many instances exist might assert that the count is non-negative. You express this with a static member function:

```cpp
static void static_invariant() {
    BOOST_CONTRACT_ASSERT(instances() >= 0);
}
```

Static invariants are checked for static public functions, and they are also checked (along with non-static invariants) for constructors, destructors, and non-static public functions.

### Making Invariants Private

If you want to keep your `invariant()` function private (which is generally good practice, since it is an implementation detail), you need to grant the library access to it:

```cpp
class my_class {
    friend class boost::contract::access;

    void invariant() const {
        BOOST_CONTRACT_ASSERT(/* ... */);
    }

public:
    // ... public interface
};
```

The `friend` declaration lets Boost.Contract call the private `invariant()` function. Without it, the library cannot find the invariant and will not check it.

With class invariants understood, you are ready to learn how constructors and destructors interact with the contract system.

## Step 6 - Constructors

Constructor contracts are special because the object does not yet exist when the constructor begins executing. The invariant cannot be checked at entry (the object is not yet formed), and preconditions cannot be specified in the usual way (there is no `this` pointer to pass to the fluent API at the point where preconditions would be checked).

### The Constructor Contract Pattern

```cpp
class unique_identifiers :
    private boost::contract::constructor_precondition<unique_identifiers>
{
public:
    void invariant() const {
        BOOST_CONTRACT_ASSERT(size() >= 0);
    }

    unique_identifiers(int from, int to) :
        boost::contract::constructor_precondition<unique_identifiers>([&] {
            BOOST_CONTRACT_ASSERT(from >= 0);
            BOOST_CONTRACT_ASSERT(to >= from);
        })
    {
        boost::contract::check c = boost::contract::constructor(this)
            .postcondition([&] {
                BOOST_CONTRACT_ASSERT(size() == (to - from + 1));
            })
        ;

        // Constructor body.
        for (int id = from; id <= to; ++id) vect_.push_back(id);
    }

private:
    std::vector<int> vect_;
};
```

### Constructor Preconditions

Because the object is not yet constructed when preconditions need to be checked, constructor preconditions are specified via the base class `boost::contract::constructor_precondition<Class>`. Your class inherits privately from this base, and you pass a lambda to it in the member initializer list. This lambda runs before the constructor body and before any member initialization that depends on the preconditions being satisfied.

### Constructor Postconditions and Invariants

Inside the constructor body, you create the `boost::contract::check` object by calling `boost::contract::constructor(this)`. This sets up postcondition checking (run when the constructor completes) and invariant checking (also run when the constructor completes, to verify the newly constructed object is valid). You cannot specify `.precondition()` here - use the base class mechanism instead.

The interplay is: preconditions run first (via the base class initializer), then the constructor body executes, then postconditions and invariants are checked. This ordering ensures the object is fully formed before any postcondition or invariant accesses it.

Now you understand how objects are born with contracts. The next section covers how they are destroyed.

## Step 7 - Destructors

Destructor contracts are simpler than constructor contracts because the object is fully formed when the destructor begins. Invariants are checked at entry (the object must be valid before destruction begins), and postconditions can verify any cleanup guarantees.

### The Destructor Contract Pattern

```cpp
virtual ~unique_identifiers() {
    boost::contract::check c = boost::contract::destructor(this);

    // Destructor body.
}
```

Destructors cannot have preconditions. The C++ language guarantees that destructors are always callable (you cannot prevent an object from being destroyed), so there is no meaningful precondition to check. The `boost::contract::destructor(this)` call sets up invariant checking at entry and postcondition checking at exit.

If your destructor does not need postconditions and your class has no invariants, you can omit the contract entirely. But if your class defines an `invariant()` function, including the contract in the destructor ensures that the invariant is verified one last time before the object disappears.

### Exception Safety in Destructors

Be careful with contracts in destructors. If an invariant or postcondition fails and the failure handler throws an exception, and the destructor was called during stack unwinding from another exception, the program will call `std::terminate`. This is the same issue that applies to any throwing destructor in C++. The section on custom failure handlers later in this tutorial covers how to handle this safely.

You now know how to write contracts for every lifecycle event of an object. The next section brings it all together for ordinary public member functions.

## Step 8 - Public Member Functions

Public member functions are the most common place you will write contracts. They check class invariants at entry and exit, and they support preconditions, postconditions, old values, and exception guarantees.

### The Basic Pattern

```cpp
bool find(int id) const {
    bool result;
    boost::contract::check c = boost::contract::public_function(this)
        .precondition([&] {
            BOOST_CONTRACT_ASSERT(id >= 0);
        })
        .postcondition([&] {
            if (size() == 0) BOOST_CONTRACT_ASSERT(!result);
        })
    ;

    // Function body.
    return result = std::find(vect_.begin(), vect_.end(), id) != vect_.end();
}
```

The key difference from `boost::contract::function()` is that `boost::contract::public_function(this)` receives the `this` pointer. This tells the library to check the class invariant before and after the function executes. For const member functions, only `const` invariants are checked.

### Accessor Functions

Even simple accessor functions benefit from contracts when the class has invariants:

```cpp
int size() const {
    boost::contract::check c = boost::contract::public_function(this);
    return vect_.size();
}
```

This function has no preconditions or postconditions of its own, but the `public_function(this)` call ensures invariants are checked. If your class has no invariants and the accessor has no pre/postconditions, you can omit the contract entirely for efficiency.

You have now covered all the fundamental contract types. In the next part, you will learn how contracts interact with inheritance, virtual functions, and other advanced C++ features.

---

# Part III - Advanced Topics

## Step 9 - Virtual Functions and Subcontracting

Subcontracting is one of the most powerful features of Boost.Contract, and it is something that C++26 contracts do not provide. When a derived class overrides a virtual function, the base class's contracts are automatically combined with the derived class's contracts according to precise rules rooted in the Liskov Substitution Principle.

### Why Subcontracting Matters

The Liskov Substitution Principle states that if code works correctly with a base class reference, it must also work correctly with a derived class reference. Contracts enforce this principle: a derived class can accept *more* inputs than the base class (weaker preconditions), and it must provide *at least as strong* guarantees as the base class (stronger postconditions and invariants). This is not merely a convention - Boost.Contract checks it automatically at runtime.

### The Subcontracting Rules

- **Preconditions are OR'd** (weakened in derived classes). A derived class's precondition succeeds if *either* the base class's precondition *or* the derived class's own precondition is satisfied. This means the derived class can accept a wider range of inputs.
- **Postconditions are AND'd** (strengthened in derived classes). Both the base class's postcondition *and* the derived class's postcondition must hold. The derived class can make additional promises but cannot break the base class's promises.
- **Invariants are AND'd** (strengthened in derived classes). The base class invariant and the derived class invariant must both hold.

### Setting Up a Virtual Function

To enable subcontracting, you need several pieces of infrastructure:

```cpp
class unique_identifiers {
public:
    void invariant() const {
        BOOST_CONTRACT_ASSERT(size() >= 0);
    }

    virtual int push_back(int id, boost::contract::virtual_* v = 0) {
        int result;
        boost::contract::old_ptr<bool> old_find =
                BOOST_CONTRACT_OLDOF(v, find(id));
        boost::contract::old_ptr<int> old_size =
                BOOST_CONTRACT_OLDOF(v, size());
        boost::contract::check c = boost::contract::public_function(
                v, result, this)
            .precondition([&] {
                BOOST_CONTRACT_ASSERT(id >= 0);
                BOOST_CONTRACT_ASSERT(!find(id));
            })
            .postcondition([&] (int const result) {
                if (!*old_find) {
                    BOOST_CONTRACT_ASSERT(find(id));
                    BOOST_CONTRACT_ASSERT(size() == *old_size + 1);
                }
                BOOST_CONTRACT_ASSERT(result == id);
            })
        ;

        vect_.push_back(id);
        return result = id;
    }

private:
    std::vector<int> vect_;
};
```

There are several important details here:

**The `virtual_*` parameter.** Every virtual function that participates in subcontracting must have an extra `boost::contract::virtual_* v = 0` parameter at the end. This parameter is used internally by the library to coordinate contract checking between base and derived classes. Callers never pass a value for it (the default is `0`).

**Passing `v` to `BOOST_CONTRACT_OLDOF`.** When inside a virtual function, old value copies must receive the `v` parameter: `BOOST_CONTRACT_OLDOF(v, expr)`. This ensures old values are copied at the right time during subcontracting.

**Passing `v` and `result` to `public_function`.** The call `boost::contract::public_function(v, result, this)` passes the virtual parameter and the result variable. The library uses these to coordinate postcondition checking across the class hierarchy.

**Postcondition with `result` parameter.** When a virtual function returns a value, the postcondition lambda takes `result` as a parameter rather than capturing it. This allows the library to pass the correct result value during subcontracting.

### Writing the Override

```cpp
class identifiers
    #define BASES public unique_identifiers
    : BASES
{
public:
    typedef BOOST_CONTRACT_BASE_TYPES(BASES) base_types;
    #undef BASES

    void invariant() const {
        BOOST_CONTRACT_ASSERT(empty() == (size() == 0));
    }

    int push_back(int id, boost::contract::virtual_* v = 0) /* override */ {
        int result;
        boost::contract::old_ptr<bool> old_find =
                BOOST_CONTRACT_OLDOF(v, find(id));
        boost::contract::old_ptr<int> old_size =
                BOOST_CONTRACT_OLDOF(v, size());
        boost::contract::check c = boost::contract::public_function<
            override_push_back
        >(v, result, &identifiers::push_back, this, id)
            .precondition([&] {
                BOOST_CONTRACT_ASSERT(id >= 0);
                BOOST_CONTRACT_ASSERT(find(id)); // Can push duplicates.
            })
            .postcondition([&] (int const result) {
                if (*old_find) BOOST_CONTRACT_ASSERT(size() == *old_size);
            })
        ;

        if (!find(id)) unique_identifiers::push_back(id);
        return result = id;
    }
    BOOST_CONTRACT_OVERRIDE(push_back)

    // ... rest of derived class
};
```

Several new elements appear here:

**`BOOST_CONTRACT_BASE_TYPES(BASES)`** declares a `base_types` typedef that tells the library about the class hierarchy. The `#define BASES` / `#undef BASES` pattern is a convention that keeps the base class list in one place.

**`boost::contract::public_function<override_push_back>(...)`** uses the override type trait generated by `BOOST_CONTRACT_OVERRIDE(push_back)`. This tells the library which function is being overridden so it can look up the base class contracts.

**The override function receives additional arguments**: the `v` parameter, the `result`, a pointer to the member function (`&identifiers::push_back`), the `this` pointer, and all function arguments (`id`).

**`BOOST_CONTRACT_OVERRIDE(push_back)`** must appear in the class body (typically after the function definition). It generates a type `override_push_back` that the library uses for subcontracting. If you have multiple overrides, you can use `BOOST_CONTRACT_OVERRIDES(func1, func2, func3)` to declare them all at once.

### Pure Virtual Functions

Pure virtual functions can also have contracts. You write the contract in the out-of-line default implementation:

```cpp
template<typename Iterator>
class range {
public:
    virtual Iterator begin(boost::contract::virtual_* v = 0) = 0;
    virtual Iterator end() = 0;
    virtual bool empty() const = 0;
};

template<typename Iterator>
Iterator range<Iterator>::begin(boost::contract::virtual_* v) {
    Iterator result;
    boost::contract::check c = boost::contract::public_function(v, result, this)
        .postcondition([&] (Iterator const& result) {
            if (empty()) BOOST_CONTRACT_ASSERT(result == end());
        })
    ;
    assert(false); // Pure virtual - body never executes.
    return result;
}
```

The body of a pure virtual function's default implementation is never actually executed by the library. Only its contracts are used for subcontracting. The `assert(false)` is a safety net in case the body is accidentally called.

Now that you understand subcontracting, the next section addresses the specific mechanics of return values in virtual functions.

## Step 10 - Return Values in Virtual Functions

When a virtual function returns a value, the postcondition needs access to that return value. Boost.Contract handles this through a specific pattern that differs slightly depending on whether the return type is a value or a reference.

### Value Return Types

For functions that return by value, you declare a local result variable and pass it to `public_function`:

```cpp
virtual int get(boost::contract::virtual_* v = 0) const {
    int result;
    boost::contract::check c = boost::contract::public_function(
            v, result, this)
        .postcondition([&] (int const result) {
            BOOST_CONTRACT_ASSERT(result <= 0);
        })
    ;
    return result = n_;
}
```

The postcondition lambda takes `result` as a *parameter* rather than capturing it. This is essential for subcontracting: the library passes the actual return value through this parameter when checking base class postconditions.

### Reference Return Types

When the return type is a reference, you cannot use a plain local variable because references must be initialized at declaration. Instead, use `boost::optional`:

```cpp
virtual T& at(unsigned index, boost::contract::virtual_* v = 0) {
    boost::optional<T&> result;
    boost::contract::check c = boost::contract::public_function(v, result, this)
        .precondition([&] {
            BOOST_CONTRACT_ASSERT(index < size());
        })
        .postcondition([&] (boost::optional<T const&> const& result) {
            BOOST_CONTRACT_ASSERT(*result == operator[](index));
        })
    ;
    return *(result = vect_[index]);
}
```

The `boost::optional` wraps the reference, allowing it to start uninitialized. The postcondition lambda receives `boost::optional<T const&> const&` and dereferences it with `*result` to access the actual return value.

You now understand the full machinery for virtual function contracts. The next section covers static public functions.

## Step 11 - Static Public Functions

Static public functions do not operate on a specific object, so they cannot check non-static invariants. However, they can and do check static invariants.

```cpp
template<class C>
class make {
public:
    static void static_invariant() {
        BOOST_CONTRACT_ASSERT(instances() >= 0);
    }

    static int instances() {
        boost::contract::check c = boost::contract::public_function<make>();
        return instances_;
    }

    make() : object() {
        boost::contract::old_ptr<int> old_instances =
                BOOST_CONTRACT_OLDOF(instances());
        boost::contract::check c = boost::contract::constructor(this)
            .postcondition([&] {
                BOOST_CONTRACT_ASSERT(instances() == *old_instances + 1);
            })
        ;
        ++instances_;
    }

    ~make() {
        boost::contract::old_ptr<int> old_instances =
                BOOST_CONTRACT_OLDOF(instances());
        boost::contract::check c = boost::contract::destructor(this)
            .postcondition([&] {
                BOOST_CONTRACT_ASSERT(instances() == *old_instances - 1);
            })
        ;
        --instances_;
    }

    C object;

private:
    static int instances_;
};

template<class C>
int make<C>::instances_ = 0;
```

The key difference is the call `boost::contract::public_function<make>()` with an explicit template parameter and no `this` pointer. This tells the library to check the static invariant of the `make` class. Non-static invariants are not checked because there is no object to check them on.

The constructor and destructor in this example also have contracts with postconditions that verify the instance count is maintained correctly. Notice how old values capture the count before the operation, and postconditions verify the count changed by exactly one.

With static functions covered, you are ready to handle overloaded functions.

## Step 12 - Overloaded Functions

When a class has multiple overloads of the same virtual function, the `BOOST_CONTRACT_OVERRIDES` macro needs to be invoked only once for all overloads of that name. The challenge is resolving the ambiguous function pointer when passing it to `public_function`. You handle this with `static_cast`:

```cpp
class string_lines
    #define BASES public lines
    : BASES
{
public:
    typedef BOOST_CONTRACT_BASE_TYPES(BASES) base_types;
    #undef BASES

    BOOST_CONTRACT_OVERRIDES(put)

    void put(std::string const& x,
            boost::contract::virtual_* v = 0) /* override */ {
        boost::contract::check c = boost::contract::public_function<
                override_put>(
            v,
            static_cast<void (string_lines::*)(std::string const&,
                    boost::contract::virtual_*)>(&string_lines::put),
            this, x
        )
            .postcondition([&] { /* ... */ })
        ;
        // Function body.
    }

    void put(char x, boost::contract::virtual_* v = 0) /* override */ {
        boost::contract::check c = boost::contract::public_function<
                override_put>(
            v,
            static_cast<void (string_lines::*)(char,
                    boost::contract::virtual_*)>(&string_lines::put),
            this, x
        )
            .postcondition([&] { /* ... */ })
        ;
        // Function body.
    }
};
```

The `static_cast` tells the compiler exactly which overload you mean. Each overload has its own contract specification with its own preconditions and postconditions, but they all share the single `override_put` type generated by `BOOST_CONTRACT_OVERRIDES(put)`.

The same pattern applies to const/non-const overloads, overloads with different arities, and overloads with default parameters. In each case, `static_cast` resolves the ambiguity.

You are now equipped to handle any function signature. The next section shows that contracts are not limited to named functions.

## Step 13 - Contracts for Lambdas, Loops, and Code Blocks

One of the elegant aspects of Boost.Contract is that `boost::contract::function()` is not restricted to named functions. You can use it anywhere you want to specify contracts for a block of code.

### Lambda Functions

```cpp
int total = 0;
std::for_each(v.cbegin(), v.cend(),
    [&total] (int const x) {
        boost::contract::old_ptr<int> old_total =
                BOOST_CONTRACT_OLDOF(total);
        boost::contract::check c = boost::contract::function()
            .precondition([&] {
                BOOST_CONTRACT_ASSERT(
                        total < std::numeric_limits<int>::max() - x);
            })
            .postcondition([&] {
                BOOST_CONTRACT_ASSERT(total == *old_total + x);
            })
        ;

        total += x;
    }
);
```

This lambda accumulates a total. The precondition guards against integer overflow, and the postcondition verifies each addition is correct. The contract is checked on every invocation of the lambda.

### Loop Bodies

```cpp
int total = 0;
for (std::vector<int>::const_iterator i = v.begin(); i != v.end(); ++i) {
    boost::contract::old_ptr<int> old_total = BOOST_CONTRACT_OLDOF(total);
    boost::contract::check c = boost::contract::function()
        .precondition([&] {
            BOOST_CONTRACT_ASSERT(
                    total < std::numeric_limits<int>::max() - *i);
        })
        .postcondition([&] {
            BOOST_CONTRACT_ASSERT(total == *old_total + *i);
        })
    ;

    total += *i;
}
```

This pattern turns each loop iteration into a contract-checked operation. The precondition acts as a loop guard, and the postcondition verifies the loop body's effect on each iteration. This is the closest you can get to formal loop invariants in C++ without language-level support.

### Implementation Checks

For simple inline checks that do not need the full precondition/postcondition ceremony, you can assign a nullary lambda directly to a `check` variable:

```cpp
boost::contract::check c = [] {
    BOOST_CONTRACT_ASSERT(gcd(12, 28) == 4);
    BOOST_CONTRACT_ASSERT(gcd(4, 14) == 2);
};
```

The lambda runs immediately when `c` is constructed. This is useful for self-tests, sanity checks, or verifying program state at a particular point in your code.

Now you will learn how contracts interact with one of modern C++'s most important features: move semantics.

## Step 14 - Move Semantics and Contracts

Move constructors and move assignment operators present a unique challenge for contracts. After a move, the source object is in a valid but unspecified state - it must still be destructible, but its invariants may be weakened. Boost.Contract handles this gracefully.

### The Pattern

The key idea is to use a `moved()` flag to distinguish between normal and moved-from objects:

```cpp
class circular_buffer :
        private boost::contract::constructor_precondition<circular_buffer> {
public:
    void invariant() const {
        if (!moved()) {
            BOOST_CONTRACT_ASSERT(index() < size());
        }
        // Invariants that must hold even for moved-from objects
        // (e.g., those needed for safe destruction) go here.
    }

    circular_buffer(circular_buffer&& other) :
        boost::contract::constructor_precondition<circular_buffer>([&] {
            BOOST_CONTRACT_ASSERT(!other.moved());
        })
    {
        boost::contract::check c = boost::contract::constructor(this)
            .postcondition([&] {
                BOOST_CONTRACT_ASSERT(!moved());
                BOOST_CONTRACT_ASSERT(other.moved());
            })
        ;
        move(std::forward<circular_buffer>(other));
    }

    circular_buffer& operator=(circular_buffer&& other) {
        boost::contract::check c = boost::contract::public_function(this)
            .precondition([&] {
                BOOST_CONTRACT_ASSERT(!other.moved());
            })
            .postcondition([&] {
                BOOST_CONTRACT_ASSERT(!moved());
                BOOST_CONTRACT_ASSERT(other.moved());
            })
        ;
        return move(std::forward<circular_buffer>(other));
    }

    // Moved-from objects can always be destroyed.
    ~circular_buffer() {
        boost::contract::check c = boost::contract::destructor(this);
    }

    bool moved() const {
        boost::contract::check c = boost::contract::public_function(this);
        return moved_;
    }

private:
    bool moved_;
    // ... other members
};
```

The invariant is conditioned on `!moved()`, so that full invariant checking only applies to live (non-moved-from) objects. The move constructor's precondition ensures you do not move from an already-moved-from object. The postconditions verify the transfer was complete: the destination is live and the source is marked as moved-from.

Notice that the move assignment does not require `!moved()` as a precondition on `this`. A moved-from object can be the target of a new assignment (either copy or move), which restores it to a valid state. Only the *source* must not already be moved-from.

This pattern works naturally with the C++ object model and gives you strong contract coverage for move operations without false positives from invariant checks on moved-from objects.

---

# Part IV - Expert Topics

## Step 15 - Assertion Levels

Not all assertions are equally expensive to check. Boost.Contract provides three assertion levels that let you control the cost-benefit tradeoff:

### Default Assertions

```cpp
BOOST_CONTRACT_ASSERT(x > 0);
```

Default assertions are always checked when contracts are enabled. Use them for checks that are cheap relative to the function's own cost.

### Audit Assertions

```cpp
BOOST_CONTRACT_ASSERT_AUDIT(std::is_sorted(first, last));
```

Audit assertions are only checked when the macro `BOOST_CONTRACT_AUDITS` is defined. Use them for expensive checks that you want during thorough testing but not in normal debug builds. The `std::is_sorted` check on a range, for example, is O(n) and might be too expensive for a function that is otherwise O(log n).

You can also guard expensive old value copies with the same macro:

```cpp
boost::contract::old_ptr<vector> old_me, old_other;
#ifdef BOOST_CONTRACT_AUDITS
    old_me = BOOST_CONTRACT_OLDOF(*this);
    old_other = BOOST_CONTRACT_OLDOF(other);
#endif
```

### Axiom Assertions

```cpp
BOOST_CONTRACT_ASSERT_AXIOM(!valid(begin(), end()));
```

Axiom assertions are *never* checked at runtime. They serve as executable documentation for properties that are true but cannot be efficiently verified (or cannot be verified at all in C++). For example, "this iterator range is valid" is a meaningful property, but there is no general way to check it. Axiom assertions state the intended truth for the benefit of human readers and static analysis tools.

The three levels give you a graduated approach: always-on for cheap checks, opt-in for expensive checks, and documentation-only for uncheckable properties. A recommended strategy is to enable all three levels in your CI test suite (`BOOST_CONTRACT_AUDITS` defined), use default-only in regular debug builds, and disable all contracts in release builds (see the section on disabling contracts).

## Step 16 - Custom Failure Handlers

By default, a contract violation calls `std::terminate`. This is a safe default - if a contract is violated, the program has a bug, and continuing might cause further damage. However, you may want different behavior: logging the violation, throwing an exception to let the program recover, or customizing behavior differently for different contexts.

### Setting Handlers

Boost.Contract provides setter functions for each type of contract failure:

```cpp
int main() {
    boost::contract::set_precondition_failure(
    boost::contract::set_postcondition_failure(
    boost::contract::set_invariant_failure(
    boost::contract::set_old_failure(
        [] (boost::contract::from where) {
            if (where == boost::contract::from_destructor) {
                // Never throw from destructors.
                std::clog << "ignored destructor contract failure" << std::endl;
            } else {
                throw; // Re-throw the assertion_failure exception.
            }
        }
    ))));

    boost::contract::set_except_failure(
        [] (boost::contract::from) {
            // Already an active exception, so do not throw another.
            std::clog << "ignored exception guarantee failure" << std::endl;
        }
    );

    boost::contract::set_check_failure(
        [] {
            throw; // Re-throw for implementation checks.
        }
    );
}
```

### The `from` Enum

The `boost::contract::from` parameter tells you which context the failure occurred in:

- `boost::contract::from_constructor` - a constructor contract failed
- `boost::contract::from_destructor` - a destructor contract failed
- `boost::contract::from_function` - any other function contract failed

This is essential for destructor safety. If a contract fails inside a destructor and the failure handler throws, and the destructor was called during stack unwinding, the program will `std::terminate`. The pattern above handles this by logging rather than throwing when `where == from_destructor`.

### Throwing Custom Exceptions

Inside your contract lambdas, you can throw your own exceptions instead of using `BOOST_CONTRACT_ASSERT`:

```cpp
explicit cstring(char const* chars) :
    boost::contract::constructor_precondition<cstring>([&] {
        BOOST_CONTRACT_ASSERT(chars); // Throws assertion_failure.
        if (std::strlen(chars) > MaxSize) throw too_large_error(); // Custom.
    })
{
    // ...
}
```

When the failure handler uses `throw;` (re-throw), it propagates whatever exception the contract lambda originally threw - whether that is `boost::contract::assertion_failure` from `BOOST_CONTRACT_ASSERT` or your own custom exception type.

## Step 17 - Separate Body Implementation

For larger projects, you may want to separate the contract declarations (which go in headers) from the function body implementations (which go in source files). This improves compilation time because changes to the function body do not require recompiling files that only depend on the contract specification.

### Header File

```cpp
// separate_body.hpp
class iarray :
        private boost::contract::constructor_precondition<iarray> {
public:
    void invariant() const {
        BOOST_CONTRACT_ASSERT(size() <= capacity());
    }

    explicit iarray(unsigned max, unsigned count = 0) :
        boost::contract::constructor_precondition<iarray>([&] {
            BOOST_CONTRACT_ASSERT(count <= max);
        }),
        values_(new int[max]),
        capacity_(max)
    {
        boost::contract::check c = boost::contract::constructor(this)
            .postcondition([&] {
                BOOST_CONTRACT_ASSERT(capacity() == max);
                BOOST_CONTRACT_ASSERT(size() == count);
            })
        ;
        constructor_body(max, count); // Delegate to source file.
    }

    virtual void push_back(int value, boost::contract::virtual_* v = 0) {
        boost::contract::old_ptr<unsigned> old_size =
                BOOST_CONTRACT_OLDOF(v, size());
        boost::contract::check c = boost::contract::public_function(v, this)
            .precondition([&] {
                BOOST_CONTRACT_ASSERT(size() < capacity());
            })
            .postcondition([&] {
                BOOST_CONTRACT_ASSERT(size() == *old_size + 1);
            })
        ;
        push_back_body(value); // Delegate to source file.
    }

private:
    void constructor_body(unsigned max, unsigned count);
    void push_back_body(int value);
    // ...
};
```

### Source File

```cpp
// separate_body.cpp
#include "separate_body.hpp"

void iarray::constructor_body(unsigned max, unsigned count) {
    for (unsigned i = 0; i < count; ++i) values_[i] = int();
    size_ = count;
}

void iarray::push_back_body(int value) {
    values_[size_++] = value;
}
```

The contracts are fully visible in the header (which serves as the function's specification), while the implementation details are hidden in the source file. Changing the body does not invalidate the header, so dependent translation units do not need recompilation.

## Step 18 - Selectively Disabling Contracts

In production builds, you may want to disable some or all contract checking for performance. Boost.Contract provides a comprehensive set of macros for this purpose.

### Disabling by Contract Type

Define these macros before including any Boost.Contract header:

| Macro | Effect |
|---|---|
| `BOOST_CONTRACT_NO_PRECONDITIONS` | Disable all precondition checks |
| `BOOST_CONTRACT_NO_POSTCONDITIONS` | Disable all postcondition checks |
| `BOOST_CONTRACT_NO_INVARIANTS` | Disable all invariant checks |
| `BOOST_CONTRACT_NO_EXCEPTS` | Disable all exception guarantee checks |
| `BOOST_CONTRACT_NO_OLDS` | Disable all old value copies |
| `BOOST_CONTRACT_NO_ALL` | Disable everything |

### Disabling by Operation Type

| Macro | Effect |
|---|---|
| `BOOST_CONTRACT_NO_CONSTRUCTORS` | Disable contracts for constructors |
| `BOOST_CONTRACT_NO_DESTRUCTORS` | Disable contracts for destructors |
| `BOOST_CONTRACT_NO_PUBLIC_FUNCTIONS` | Disable contracts for public member functions |
| `BOOST_CONTRACT_NO_FUNCTIONS` | Disable contracts for non-member/private/protected functions |

### Conditional Compilation for Zero Overhead

When contracts are disabled via these macros, the library eliminates the contract checking code. However, the contract *declarations* (lambdas, old pointers, etc.) may still have a small overhead. For truly zero overhead, you can use `#ifdef` guards:

```cpp
#include <boost/contract/core/config.hpp>
#ifndef BOOST_CONTRACT_NO_ALL
    #include <boost/contract.hpp>
#endif

int inc(int& x) {
    int result;
    #ifndef BOOST_CONTRACT_NO_OLDS
        boost::contract::old_ptr<int> old_x = BOOST_CONTRACT_OLDOF(x);
    #endif
    #ifndef BOOST_CONTRACT_NO_FUNCTIONS
        boost::contract::check c = boost::contract::function()
            #ifndef BOOST_CONTRACT_NO_PRECONDITIONS
                .precondition([&] {
                    BOOST_CONTRACT_ASSERT(
                            x < std::numeric_limits<int>::max());
                })
            #endif
            #ifndef BOOST_CONTRACT_NO_POSTCONDITIONS
                .postcondition([&] {
                    BOOST_CONTRACT_ASSERT(x == *old_x + 1);
                    BOOST_CONTRACT_ASSERT(result == *old_x);
                })
            #endif
        ;
    #endif

    return result = x++;
}
```

This approach is verbose, but it guarantees that no contract-related code is compiled when contracts are disabled. The recommended strategy for most projects is:

- **Debug and test builds**: Enable all contracts (no disabling macros).
- **CI builds**: Enable all contracts plus `BOOST_CONTRACT_AUDITS` for thorough checking.
- **Release builds**: Define `BOOST_CONTRACT_NO_ALL` for zero overhead, or selectively keep preconditions enabled as a safety net.

## Step 19 - Configuration Reference

Boost.Contract offers several configuration macros beyond the disabling macros covered above.

### `BOOST_CONTRACT_MAX_ARGS`

Default: 10. The maximum number of arguments that `public_function` can accept for overriding functions. If your overriding functions have more than 10 parameters (which is rare), increase this value. Higher values increase compilation time.

### Name Customization

| Macro | Default | Purpose |
|---|---|---|
| `BOOST_CONTRACT_BASES_TYPEDEF` | `base_types` | Name of the base types typedef |
| `BOOST_CONTRACT_INVARIANT_FUNC` | `invariant` | Name of the invariant function |
| `BOOST_CONTRACT_STATIC_INVARIANT_FUNC` | `static_invariant` | Name of the static invariant function |

These macros let you avoid name collisions if your code already uses `invariant` or `base_types` for other purposes.

### `BOOST_CONTRACT_PERMISSIVE`

When defined, the library is more lenient about certain usage errors. For example, it will not generate a runtime error if the `check` variable is declared with the wrong type. This is useful during migration but should not be used in production.

### `BOOST_CONTRACT_ON_MISSING_CHECK_DECL`

Controls what happens when a contract specification result is not assigned to a `boost::contract::check` variable. By default, this is a runtime error. Defining this macro lets you change the behavior (e.g., to a warning or silent ignore).

### `BOOST_CONTRACT_DISABLE_THREADS`

Boost.Contract uses a global mutex to prevent recursive contract checking (which could otherwise cause infinite loops when an invariant calls a public function that checks the invariant again). If your program is single-threaded, defining `BOOST_CONTRACT_DISABLE_THREADS` eliminates the mutex overhead.

## Step 20 - Real-World Patterns

This section presents complete, realistic examples drawn from the Boost.Contract example suite.

### The Stack (Meyer's Classic)

This is a direct translation of Bertrand Meyer's iconic stack example from *Object-Oriented Software Construction*. It demonstrates how a complete data structure is specified with contracts:

```cpp
#include <boost/contract.hpp>

template<typename T>
class stack
    #define BASES private boost::contract::constructor_precondition<stack<T>>
    : BASES
{
    friend boost::contract::access;
    typedef BOOST_CONTRACT_BASE_TYPES(BASES) base_types;
    #undef BASES

    void invariant() const {
        BOOST_CONTRACT_ASSERT(count() >= 0);
        BOOST_CONTRACT_ASSERT(count() <= capacity());
        BOOST_CONTRACT_ASSERT(empty() == (count() == 0));
    }

public:
    explicit stack(int n) :
        boost::contract::constructor_precondition<stack>([&] {
            BOOST_CONTRACT_ASSERT(n >= 0);
        })
    {
        boost::contract::check c = boost::contract::constructor(this)
            .postcondition([&] {
                BOOST_CONTRACT_ASSERT(capacity() == n);
            })
        ;
        capacity_ = n;
        count_ = 0;
        array_ = new T[n];
    }

    virtual ~stack() {
        boost::contract::check c = boost::contract::destructor(this);
        delete[] array_;
    }

    int capacity() const {
        boost::contract::check c = boost::contract::public_function(this);
        return capacity_;
    }

    int count() const {
        boost::contract::check c = boost::contract::public_function(this);
        return count_;
    }

    T const& item() const {
        boost::contract::check c = boost::contract::public_function(this)
            .precondition([&] {
                BOOST_CONTRACT_ASSERT(!empty());
            })
        ;
        return array_[count_ - 1];
    }

    bool empty() const {
        bool result;
        boost::contract::check c = boost::contract::public_function(this)
            .postcondition([&] {
                BOOST_CONTRACT_ASSERT(result == (count() == 0));
            })
        ;
        return result = (count_ == 0);
    }

    bool full() const {
        bool result;
        boost::contract::check c = boost::contract::public_function(this)
            .postcondition([&] {
                BOOST_CONTRACT_ASSERT(result == (count() == capacity()));
            })
        ;
        return result = (count_ == capacity_);
    }

    void put(T const& x) {
        boost::contract::old_ptr<int> old_count =
                BOOST_CONTRACT_OLDOF(count());
        boost::contract::check c = boost::contract::public_function(this)
            .precondition([&] {
                BOOST_CONTRACT_ASSERT(!full());
            })
            .postcondition([&] {
                BOOST_CONTRACT_ASSERT(!empty());
                BOOST_CONTRACT_ASSERT(item() == x);
                BOOST_CONTRACT_ASSERT(count() == *old_count + 1);
            })
        ;
        array_[count_++] = x;
    }

    void remove() {
        boost::contract::old_ptr<int> old_count =
                BOOST_CONTRACT_OLDOF(count());
        boost::contract::check c = boost::contract::public_function(this)
            .precondition([&] {
                BOOST_CONTRACT_ASSERT(!empty());
            })
            .postcondition([&] {
                BOOST_CONTRACT_ASSERT(!full());
                BOOST_CONTRACT_ASSERT(count() == *old_count - 1);
            })
        ;
        --count_;
    }

private:
    int capacity_;
    int count_;
    T* array_;
};
```

Study this example carefully. Every public function either has explicit pre/postconditions or at minimum checks the class invariant. The invariant captures the fundamental properties of a stack: the count is bounded, and emptiness is consistent with the count. The `put` and `remove` functions have preconditions that prevent overflow and underflow, and their postconditions verify the expected state changes using old values.

### When to Use Contracts vs. Exceptions vs. Assertions

Contracts, exceptions, and assertions serve different purposes and are not interchangeable:

**Contracts** express correctness requirements. A violated contract means there is a bug in the program. Contracts are checked during development and testing and may be disabled in production. They answer the question: "Is this code correct?"

**Exceptions** handle expected runtime failures. Running out of memory, failing to open a file, or receiving malformed network input are not bugs - they are anticipated conditions that the program must handle gracefully. Exceptions should never be disabled. They answer the question: "Can this operation succeed right now?"

**Assertions** (`assert()`) are a lightweight form of contract for internal implementation checks. They are useful but lack the structure and expressiveness of Boost.Contract (no postconditions, no old values, no invariants, no subcontracting).

A good rule of thumb: if the condition being checked is the caller's responsibility, it is a precondition. If it is the callee's guarantee, it is a postcondition. If it is a property of the object, it is an invariant. If it is none of these and just a sanity check on internal state, `assert()` or `BOOST_CONTRACT_CHECK` is appropriate. If it is an expected runtime condition that could legitimately occur, use an exception or error code.

## Step 21 - Boost.Contract and C++26 Contracts

C++26 introduces language-level contract support with three new attributes:

```cpp
int sqrt(int x)
    [[pre: x >= 0]]
    [[post r: r >= 0]]
{
    // ...
}
```

This is a significant milestone for C++, but the language facility and Boost.Contract address different scopes:

| Feature | Boost.Contract | C++26 Contracts |
|---|---|---|
| Preconditions | Yes | Yes |
| Postconditions | Yes (with old values) | Yes (limited) |
| Class invariants | Yes (automatic checking) | No |
| Old values | Yes (`old_ptr<T>`) | No |
| Subcontracting | Yes (automatic OR/AND) | No |
| Exception guarantees | Yes (`.except()`) | No |
| Assertion levels | Yes (default/audit/axiom) | Yes (default/audit) |
| Works on C++11 | Yes | No (C++26 only) |

Boost.Contract provides a substantially richer contract programming model. Class invariants, old values, and subcontracting are core features of the Design by Contract methodology that C++26 does not address.

If you are starting a new project on a C++26 compiler, you might use `[[pre:]]` and `[[post:]]` for basic function contracts and Boost.Contract for class invariants, old values, and subcontracting. The two approaches can coexist: language-level contracts on the function declaration and Boost.Contract inside the function body for the features the language does not provide.

If you are working with C++11 through C++23, Boost.Contract is your only option for library-level contract support, and it remains a powerful and complete solution.

## Conclusion

You have now covered the full breadth of Boost.Contract, from the foundational concepts of contract programming to the expert-level mechanics of assertion levels, failure handlers, and compile-time configuration.

Here is a quick-reference summary of the patterns you have learned:

| Context | Entry Point |
|---|---|
| Non-member / private / protected function | `boost::contract::function()` |
| Constructor | `boost::contract::constructor(this)` |
| Constructor precondition | `boost::contract::constructor_precondition<Class>` (base class) |
| Destructor | `boost::contract::destructor(this)` |
| Public member function (non-virtual) | `boost::contract::public_function(this)` |
| Public member function (virtual, base) | `boost::contract::public_function(v, result, this)` |
| Public member function (override) | `boost::contract::public_function<Override>(v, result, &C::f, this, args...)` |
| Static public function | `boost::contract::public_function<Class>()` |

And the fluent API chain:

```
.precondition([&] { ... })
.old([&] { ... })
.postcondition([&] { ... })  // or  .postcondition([&] (auto const& result) { ... })
.except([&] { ... })
```

For further reading:

- [Boost.Contract Official Documentation](https://www.boost.org/doc/libs/release/libs/contract/doc/html/index.html)
- [Boost.Contract API Reference](https://www.boost.org/doc/libs/release/libs/contract/doc/html/reference.html)
- [Boost.Contract Source and Examples on GitHub](https://github.com/boostorg/contract)
- [Boost Mailing List (tag: contract)](https://www.boost.org/community/groups.html#main)

Contract programming catches bugs at their source, documents your code's assumptions in a machine-checkable form, and makes your software more reliable. Now that you have the tools, put them to work.

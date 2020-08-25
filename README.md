## Introduction

<a href="https://dlang.org/articles/ctarguments.html" target="_blank">Compile time sequences</a> in D are insane. Not insane "crazy" but insanely interesting. One of the selling points of D is that it takes compile time programming in C++ to another level; where templates in C++ were a 'happy?' accident, in D they were purposely built into the language along with other generic and meta-programming tools. To highlight this aspect of the D language, we will implement a selection of templates to manipulate compile time sequences of types. Other than the `text` function from the standard library in the <a href="https://dlang.org/phobos/std_conv.html" target="_blank">`std.conv`</a> module and two one-liners from the <a href="https://dlang.org/phobos/std_meta.html" target="_blank">`std.meta`</a> module, we will build everything else ourselves.

This article builds on the concepts of a <a href="../reading-idx-files-in-d/" target="_blank">previous article</a> that is about reading IDX files at compile time in D. The focus of this article is compile time sequences, as well as introducing new tools and concepts. Some desirable features of compile time sequences are:

1. Indexing - selecting items using a (numerical) index. This is built-in; we don't need to do anything to get this functionality.
2. Append, Prepend, and Concatenate - we will see that these are implicit in the definition of compile time sequences.
3. Removing item(s).
4. Inserting item(s).

Good sources for further information of the concepts discussed here are given in:

* D's <a href="https://dlang.org/spec/spec.html" target="_blank">language reference.</a>
* <a href="http://ddili.org/ders/d.en/index.html" target="_blank">"Programming in D"</a> book by Ali Çehreli
* Documentation of the <a href="https://dlang.org/phobos/std_meta.html" target="_blank">`std.meta`</a> package.
* The <a href="https://github.com/dlang/phobos/blob/master/std/meta.d" target="_blank"> `std.meta`</a> package on GitHub. I don't routinely recommend reading source code of existing libraries and maybe I should do that more, but you can learn a lot about how to write compile time code in D by reading `std.meta`.

## Template abbreviations and preliminaries

There are lots of templatable data types and commands in D and it's easy to overlook some. These include:

* Functions.
* Structs, classes, and interfaces.
* Enumerations.
* Alias - used for type aliasing.
* Template expressions. Templates in D are an important part of compile time programming, as are `static if`, ` static foreach` loops (covered in the <a href="../reading-idx-files-in-d/" target="_blank">previous article</a>), and recursion. You could say C++ templates are also powerful, but remember that templates in D were **not incidental**, they were designed into the language from the beginning, they are there on purpose, very powerful, and easy to use. They are also prettier than C++ templates.

### Function template abbreviations

We won't be creating any function templates in these compile time sequences, however for completeness the example below shows a function template for the inverse square law for <a href="https://en.wikipedia.org/wiki/Newton%27s_law_of_universal_gravitation" target="_blank">gravitational attraction force between two bodies</a>:

```d
// Longhand
template GravitationalAttraction(T)
{
  auto GravitationalAttraction(T G, T m1, T m2, T r)
  {
    return (G*m1*m2)/(r^^2); // the two carets are not a mistake
  }
}

// Shorthand
auto GravitationalAttraction(T)(T G, T m1, T m2, T r)
{
  return (G*m1*m2)/(r^^2);
}
```

*These two forms are treated exactly the same by the compiler. Therefore, you can only include one of them in your script.*

An example call for the template would be:<br>`auto f = GravitationalAttraction!(float)(G, m1, m2, r);`

*Note that D has type inference for templates but this topic will not be covered here.*

### Structs, classes, and interfaces

Structs, classes, and interface template abbreviations work in the same way, so I'll only show an example for a `struct`:

```d
// Longhand
template Planet(T)
{
  struct Planet
  {
    T mass;
    T volume;
    T surfaceArea;
    T meanSurfaceTemp;
  }
}
// Shorthand
struct Planet(T)
{
  T mass;
  T volume;
  T surfaceArea;
  T meanSurfaceTemp;
}
```

The template struct can be called using:<br>`auto planet = Planet!(double)(mass, vol, surfA, meanTemp);`.

### Enumerations

<a href="https://dlang.org/phobos/std_traits.html" target="_blank">Traits</a> in D are templates that *"extract information about types and symbols at compile time"*, frequently used as compile time predicates. Enumeration templates are an opportunity to show how simple versions of these can be constructed. Suppose we want to restrict the template type `T` in the functions and struts shown previously to floating point types `float`, `double`, and `real`. A compile time predicate is exactly what you need:

```d
// Longhand
template isFloat(T)
{
  enum bool isFloat = is(T == float) || is(T == double) || is(T == real);
}
// Shorthand
enum bool isFloat(T) = is(T == float) || is(T == double) || is(T == real);
```

Notice that you can not simply write `T == float` for types; you need to use the `is()` expression to obtain a `bool`. This enum can be used with the previous templates. For functions:

```d
// Longhand
template GravitationalAttraction(T)
if(isFloat!T)
{
  auto GravitationalAttraction(T G, T m1, T m2, T r)
  {
    return (G*m1*m2)/(r^^2);
  }
}
// Shorthand
auto GravitationalAttraction(T)(T G, T m1, T m2, T r)
if(isFloat!T)
{
  return (G*m1*m2)/(r^^2);
}
```

structs:

```d
// Longhand
template Planet(T)
if(isFloat!T)
{
  struct Planet
  {
    T mass;
    T volume;
    T surfaceArea;
    T meanSurfaceTemp;
  }
}
// Shorthand
struct Planet(T)
if(isFloat!T)
{
  T mass;
  T volume;
  T surfaceArea;
  T meanSurfaceTemp;
}
```

Restricting templates in this way is how we create **template constraints** in D.

### The alias keyword

A common use for the `alias` keyword is aliasing a type. For example, here is an alias for a pointer type:

```
// Longhand
template P(T)
{
  alias P = T*;
}
// Shorthand
alias P(T) = T*;
```

to use it do `P!(real) x;`

## AliasSeq!(T) compile time sequences


`AliasSeq` is implemented in the <a href="https://dlang.org/phobos/std_meta.html#Alias">std.meta</a> library of Phobos (D's standard library). The implementation is **very** simple:

```d
alias AliasSeq(T...) = T;
```

We will also borrow the implementation of `Nothing` from the same module as it is not available for being imported (because it has the `package` visibility):

```d
alias Nothing = AliasSeq!();
```

That's it!

## Indexing compile time sequences

To create a sequence of types we simply do:

```d
alias tList = AliasSeq!(bool, string, ubyte, short, ushort);
```

`pragma`s interact directly with the compiler; for instance, we can print a message at compile time by using `pragma(msg, "The message, ", "then another message ...")`:

```d
pragma(msg, "Input types: ", tList);
```

outputs:

```d
Input types: (bool, string, ubyte, short, ushort)
```

We can access individual elements of a sequence for instance by `tList[1]` and carry out slice operations for instance by `tList[1..3]`:

```d
pragma(msg, "Single index selection: ", tList[1]);
pragma(msg, "Slice selection: ", tList[1..3]);
```

outputs:

```d
Single index selection: string
Slice selection: (string, ubyte)
```

## Appending, prepending, and concatenating type lists

One important aspect of `AliasSeq(T...)` typelists is that they are not a traditional "container"; when they are input into templates they kind of just "expand" and become like separate arguments of a function as if they were not contained but inputted in separately. We'll see what that means later but one consequence is there is no need to define operations for Append, Prepend, and Concatenate because it's already implied in the definition. Just put typelists together and that's it.


### Append

Here, a new typelist is defined by appending `ulong` to the previously defined typelist:

```d
alias appended = AliasSeq!(tList, ulong);
pragma(msg, "Appended list: ", appended);
```

Output:

```d
Appended list: (bool, string, ubyte, short, ushort, ulong)
```

### Prepend

Here, a new typelist is defined by prepending `ulong` to the previously defined typelist:

```d
alias prepended = AliasSeq!(ulong, tList);
pragma(msg, "Prepended list: ", prepended, "\n");
```
Output:
```d
Prepended list: (ulong, bool, string, ubyte, short, ushort)
```

### Concatenate

Here, two typelists are concatenated:

```d
alias concat = AliasSeq!(tList, AliasSeq!(int, long, ulong));
pragma(msg, "Concatenated list: ", concat);
```

Output:
```d
Concatenated list: (bool, string, ubyte, short, ushort, int, long, ulong)
```

Notice how they are now one typelist.

## Replacing items

### Replacing a single item by index

The code below shows the implementation for the internals of the `Replace` template that replaces a single item in an `AliasSeq` compile time sequence. It is marked `private` which limits its <a href="https://dlang.org/spec/attribute.html#visibility_attributes" target="_blank">visibility</a> to the module being defined; its interface is defined later.


```d
private template Replace(long i, long r, S, Args...)
{
  static if(Args.length == 1)
  {
    static if(i == r)
    {
      alias Replace = S;
    }else{
      alias Replace = Args[0];
    }
  }else static if(Args.length > 1)
  {
    static if(i == r)
    {
      alias Replace = AliasSeq!(Replace!(i - 1, r, S, 
                                          Args[0..($ - 1)]), S);
    }else
    {
      alias Replace = AliasSeq!(Replace!(i - 1, r, S, 
                                          Args[0..($ - 1)]), Args[$ - 1]);
    }
  }else static if(Args.length == 0)
  {
    alias Replace = Args;
  }else{
    static assert(false, "Error from Replace template something went wrong");
  }
}
```

Let's break down what's going on here. Firstly, the declaration:

```d
private template Replace(long i, long r, S, Args...){/*... Code ...*/}
```

Templates can have typed arguments; in this case `long i` and `long r`. Here, `r` is the index location to the item we want replaced, `i` is a candidate index. We check if `i == r` and if it is, we do the replacement.

We have an individual type `S` and then a variable number of types `Args...`. `S` is what we replace whatever is in `r` with and `Args` is the type sequence where the replacement will be done. Note that you can have variable number of template arguments only at the end. In the code above, `S` will always be assumed to be a single element by the compiler. What is happening is essentially pattern matching. For example, if we compile this script:

```d
alias AliasSeq(T...) = T;

alias lhs = AliasSeq!(byte, ubyte);
alias rhs = AliasSeq!(short, ushort);

template Join(L, R...)
{
  pragma(msg, "L: ", L);
  pragma(msg, "R: ", R);
  alias Join = AliasSeq!(L, R);
}

alias joined = Join!(lhs, rhs);
pragma(msg, "Join!(lhs, rhs): ", joined);

void main(){}
```

we get the output:

```d
L: byte
R: (ubyte, short, ushort)
Join!(lhs, rhs): (byte, ubyte, short, ushort)
```

**Compiler-interpreter:**
*If you are "weirded out" by the `void main(){}` at the end, it is needed because although all our code is compile time, a `main` function is needed for run time. We could put our code in `main` but we don't have to. To compile the script in a "standard way" we need a `main` function, but we can get rid of this by using the `-o-` flag to suppress the creation of an object (executable) file. If we do this we can delete the `main` function and compile using `dmd script.d -o-` (for dmd). There's nothing to "run" because we don't create an object file and everything we do happens at compile time a bit like an interpreted script.*

If we try to use this template instead:

```d
template Join(L, R)
{
  alias Join = AliasSeq!(L, R);
}
```

we will get a compilation error:

```d
Error: template instance Join!(byte, ubyte, short, ushort) 
            does not match template declaration Join(L, R)
```

and if we try

```d
template Join(L..., R...)
{
  alias Join = AliasSeq!(L, R);
}
```

we will get... `Error: variadic template parameter must be last`.

The `Replace` template is recursive, and there are two conditions we care about. Firstly the termination, when `Args.length == 1` and then the continuation when `Args.length > 1`. When `Args.length == 1` we terminate with:

```d
static if(i == r)
{
  alias Replace = S;
}else{
  alias Replace = Args[0];
}
```

When `Args.length > 1` we do the following recursive call:

```d
static if(i == r)
{
  alias Replace = AliasSeq!(Replace!(i - 1, 
                       r, S, Args[0..($ - 1)]), S);
}else
{
  alias Replace = AliasSeq!(Replace!(i - 1, 
                       r, S, Args[0..($ - 1)]), Args[$ - 1]);
}
```

so we start at the end of the sequence and work our way backwards to the first element.

There are two other conditions: The first of these is if we ever get a case when `Args.length == 0` i.e. `Args == AliasSeq!()` argument, and the other is if we get something else we don't expect (which may well be overkill) but is an example of using a `static assert` to trigger an error in a specific case. Before going further into the first of these cases let's look at the interface - the template that is meant to be called using a **template overload**:

```d
alias Replace(long r, S, Args...) = Replace!(Args.length - 1, r, S, Args);
```

... meaning that we *should* never hit the error case. If we wanted, we could move the `Args.length == 0` case to another template entirely using **template constraints**:

```d
private template Replace(long i, long r, S, Args...)
if(Args.length == 0)
{
  alias Replace = Args;
}
```
and the declaration for the first template becomes:

```d
private template Replace(long i, long r, S, Args...)
if(Args.length > 0)
{/*... Code ... */}
```

Let's see if `Replace` template works:

```d
// Type list
alias tList = AliasSeq!(bool, string, ubyte, short, ushort);
// Replace examples ...
alias replace0 = Replace!(0, int, tList);
pragma(msg, "Modify (bool @ 0) => int: ", replace0);
alias replace1 = Replace!(1, byte, tList);
pragma(msg, "Modify (string @ 1) => byte: ", replace1);
alias replace2 = Replace!(tList.length - 1, uint, tList);
pragma(msg, "Modify (ushort @ end) => uint: ", replace2);
// Replace in Nothing for Args.length == 0
alias replace3 = Replace!(0, int, Nothing);
pragma(msg, "Replace in Nothing: ", replace3);
```

Output:

```d
Modify (bool @ 0) => int: (int, string, ubyte, short, ushort)
Modify (string @ 1) => byte: (bool, byte, ubyte, short, ushort)
Modify (ushort @ end) => uint: (bool, string, ubyte, short, uint)
Replace in Nothing: ()
```

### Replacing multiple items with an individual type

In this section we introduce the concept of <a href="https://dlang.org/articles/mixin.html" target="_blank">string mixins</a>. String mixins are a little like macros in C. They allow the user to use strings to construct code and include them in scripts. Unlike macros in C, *"\[string mixins\] in text must form complete declarations, statements, or expressions"*, and they have additional protections that make them inherently safer than those in C. However, in this case we are not creating runtime code from mixins but code for templates.

In D the usual method for concatenating strings is using `'~'` operator; however, we have to be careful with this: `'~'` will interpret the integer `0` as a null byte, so we use the `text` function to do concatenation \[<a href="https://forum.dlang.org/post/cwdkvwxyziunxwphvsqi@forum.dlang.org" target="_blank">reference</a>\]. The template for doing a `Replace` of multiple items with a single one is given below:

```d
import std.conv: text;
template Replace(alias indices, S, Args...)
if(Args.length > 0)
{
  enum N = indices.length;
  static foreach(i; 0..N)
  {
    static if(i == 0)
    {
      debug pragma(msg, "Compiled string 1: ", 
          text(`alias x`, i, ` = Replace!(indices[i], S, Args);`));
      mixin(text(`alias x`, i, ` = Replace!(indices[i], S, Args);`));
    }else{
      debug pragma(msg, "Compiled string 2: ",
          text(`alias x`, i, ` = Replace!(indices[i], r, S, x`, (i - 1), `);`));
      mixin(text(`alias x`, i, ` = Replace!(indices[i], S, x`, (i - 1), `);`));
    }
  }
  debug pragma(msg, "Compiled string 3: ", 
        text(`alias Replace = x`, N - 1, `;`));
  mixin(text(`alias Replace = x`, N - 1, `;`));
}
```

Let's break this down. As we discussed in a <a href="../reading-idx-files-in-d/" target="_blank">previous article</a> all compile time variables are constants so they can not be changed once they are defined. In this case we iterate over all the numbers in `indices` and call the `Replace` template for single items on each of the indices. Since we can not store each iteration into the same (immutable) variable, we generate a new compile time sequence each time: `x0, x1, ...xN`. And we do this by referring to the previous sequence: `Args, x0, ..., x(N-1)`. `static foreach` loop bodies in D are *not scoped*; it is the equivalent of pasting the code over and over for each iteration. (The <a href="../reading-idx-files-in-d/" target="_blank">previous article</a> shows how double curly brackets are used to introduce scope for each iteration when desired.) So we are in fact creating a series of sequences with different names, effectively returning the final sequence from the template:

```d
mixin(text(`alias Replace = x`, N - 1, `;`));
```

The code also shows `debug` conditional compilation directives before each `pragma` statement which allows us to include these only when the code is compiled with the `-debug` dmd compiler switch. As you may have noticed, we introduced an `alias` keyword in our template declaration `template Replace(alias indices, S, Args...){/*... Code ...*/}`, this allows us to specify anything that is not a template and we don't have to provide the type.

Using this `Replace` overload for the multiple replacement by a single item:

```d
alias replace4 = Replace!([0, 2, 4], ulong, tList);
pragma(msg, "Replace ([0, 2, 4]) => ulong: ", replace4);
```

output:

```d
Replace ([0, 2, 4]) => ulong: (ulong, string, ulong, short, ulong)
```


### Replacing multiple items by a tuple of items

Things will now get a little more interesting. We've already shown that there is no way of passing more than one `AliasSeq!(T)` sequence as template parameters while retaining containment, you always end up with all but one (or however extra) of the template parameters in the variable length. To remedy this we can use a tuple typelist, there are tuple types and functions in <a href="https://dlang.org/phobos/std_typecons.html" target="_blank">std.typecons</a> but we will be building our own:

```d
struct Tuple(T...)
{
  enum long length = T.length;
  alias get(long i) = T[i];
  alias getTypes = T;
}
```

This `struct` is special: It has one member (an `enum long` constant), no methods, and some template instance aliases, perfect for our compile time needs. We will be able to use it as a *"container"* for types that can be passed along with `AliasSeq` without merging typelists. The `length` constant gives us the length of the typelist, and the `get(long i)` template allows up to get the ith template parameter. Usage for a tuple `x` is `x.get!(i)`, and the `getTypes` alias provides the original typelist `T`.

Next, we create a predicate trait that tells us whether something is a `Tuple` or not:

```d
enum bool isTuple(Tup...) = false;
enum bool isTuple(Tup: Tuple!T, T...) = true;
```

You might notice that this is slightly different from our previous predicate. Here we define a kind of "catchall" for types that are not `Tuple` as `false`, and then provide a definition for Tuple types as `true`. Now let's see if this works *(D has unit testing capabilities but we will not be covering them here)*:

```d
alias tupleConcat = AliasSeq!(real, Tuple!(tList));
pragma(msg, "Concatenation with Tuple: ", tupleConcat);
```

output:

```d
Concatenation with Tuple: (real, Tuple!(bool, string, ubyte, short, ushort))
```

Now the functionality of `Tuple`:

```d
pragma(msg, "\nTuple length: ", Tuple!(tList).length);
pragma(msg, "Tuple types: ", Tuple!(tList).getTypes);
pragma(msg, "Tuple!(tList).get!(0): ", Tuple!(tList).get!(0));
pragma(msg, "Tuple!(tList).get!(1): ", Tuple!(tList).get!(1));
pragma(msg, "Tuple!(tList).get!(2): ", Tuple!(tList).get!(2), "\n");
```

```d
Tuple length: 5L
Tuple types: (bool, string, ubyte, short, ushort)
Tuple!(tList).get!(0): bool
Tuple!(tList).get!(1): string
Tuple!(tList).get!(2): ubyte
```

and `isTuple`:

```d
pragma(msg, "\nTesting isTuple (false): ", isTuple!(real));
pragma(msg, "Testing isTuple (false): ", 
                          isTuple!(AliasSeq!(long, ulong, real)));
pragma(msg, "Testing isTuple (true): ", 
                          isTuple!(Tuple!(long, ulong, real)), "\n");
```

```d
Testing isTuple (false): false
Testing isTuple (false): false
Testing isTuple (true): true
```

Now we are ready for the template that replaces multiple items in `AliasSeq` by the types in a tuple:

```d
template Replace(alias indices, S, Args...)
if((Args.length > 0) && isTuple!(S) && (indices.length == S.length))
{
  enum N = indices.length;
  static foreach(i; 0..N)
  {
    static if(i == 0)
    {
      mixin(text(`alias x`, i, ` = Replace!(indices[i], S.get!(i), Args);`));
    }else{
      mixin(text(`alias x`, i, 
             ` = Replace!(indices[i], S.get!(i), x`, (i - 1), `);`));
    }
  }
  mixin(text(`alias Replace = x`, N - 1, `;`));
}
```

Note that the template constraint includes `isTuple`, we should go back to our previous implementation of `Replace` and amend it's constraints to ensure that they are distinguishable:

```d
// Replacing multiple items with an individual type
template Replace(alias indices, S, Args...)
if(Args.length > 0 && !isTuple!(S)){/*... Code ...*/}
```

Notice that in addition to the `isTuple!(S)` constraint in the new `Replace` template there is the aditional constraint `indices.length == S.length`. We could have used a `static assert(indices.length == S.length, "... message ...");` in the template body but it's a design choice where cases with `indices.length != S.length` don't enter the template body. Instead, we could create yet another overload for that case:

```
template Replace(alias indices, S, Args...)
if((Args.length > 0) && isTuple!(S) && (indices.length != S.length))
{
  static assert(false, "tuple length is not equal to length of replacement indices");
}
```

Testable with:

```d
//alias replace6 = Replace!([0, 1, 2, 4], Tuple!(long, ulong, real), tList);
```

## Removing items from a compile time sequence

### Removing a single item from a compile time sequence

In this case we would like to remove the rth item from `Args`. The internal function in this case is very similar to the recursive internal function in the `Replace` case. Of course by now you know that this can be written in different ways, for example by breaking up the cases for different lengths of `Args` to different templates by using template constraints, note the use of `Nothing` here.

```d
private template Remove(long i, long r, Args...)
{
  static if(Args.length == 1)
  {
    static if(i == r)
    {
      alias Remove = Nothing;
    }else{
      alias Remove = Args[0];
    }
  }else static if(Args.length > 1)
  {
    static if(i == r)
    {
      alias Remove = Remove!(i - 1, r, Args[0..($ - 1)]);
    }else
    {
      alias Remove = AliasSeq!(Remove!(i - 1, r, Args[0..($ - 1)]), Args[$ - 1]);
    }
  }else static if(Args.length == 0)
  {
    alias Remove = Args;
  }else{
    static assert(0, "Error from Remove template something went wrong");
  }
}
```

the programmer interface is:

```d
alias Remove(long r, Args...) = Remove!(Args.length - 1, r, Args);
```

We can check that it all works with:

```d
pragma(msg, "\nSequence: ", tList);
alias remove0 = Remove!(0, tList);
pragma(msg, "Removed item @ 0 (bool) => Nothing: ", remove0);
alias remove1 = Remove!(2, tList);
pragma(msg, "Removed item @ 2 (ubyte) => Nothing: ", remove1);
alias remove2 = Remove!(tList.length - 1, tList);
pragma(msg, "Removed item @ end (ushort) => Nothing: ", remove2);
```

output:

```d
Sequence: (bool, string, ubyte, short, ushort)
Removed item @ 0 (bool) => Nothing: (string, ubyte, short, ushort)
Removed item @ 2 (ubyte) => Nothing: (bool, string, short, ushort)
Removed item @ end (ushort) => Nothing: (bool, string, ubyte, short)
```

### Removing multiple items from a compile time sequence

That's gravy, we built it directly from what we've learnt. Using `alias` for specifying multiple items could be a bit too "general" for selecting multiple items. So, let's see if we can use a `Tuple` to specify the indices this time. First, we add a template method for getting enums rather than types to our previous `Tuple` definition:

```d
struct Tuple(T...)
{
  enum long length = T.length;
  alias get(long i) = T[i];
  enum getEnum(long i) = T[i]; //added
  alias getTypes = T;
}
```

We can now specify the multiple `Remove` template:

```d
template Remove(Indices, Args...)
if(isTuple!(Indices) && (Indices.length <= Args.length))
{
  enum N = Indices.length;
  static foreach(i; 0..N)
  {
    static if(i == 0)
    {
      mixin(text(`alias x`, i, ` = Remove!(Indices.getEnum!(i), Args);`));
    }else{
      mixin(text(`alias x`, i, ` = Remove!(Indices.getEnum!(i) - i, x`, 
                                                           (i - 1), `);`));
    }
  }
  mixin(text(`alias Remove = x`, N - 1, `;`));
}
```

Observe that for indices greater than zero we amend the index using `Indices.getEnum!(i) - i` because the sequence has decreased in length by one. Let's see if it works:


```d
alias indices = Tuple!(0L, 2L, 4L);
alias remove3 = Remove!(indices, tList);
pragma(msg, "\nRemove multiple items ", "@", indices, ": ", remove3);
```

output:

```d
Remove multiple items @Tuple!(0L, 2L, 4L): (string, short)
```

## Inserting types into compile time sequences

Inserting a single type or multiple types is simple. The case for inserting a single type `E` at position `i` can be achieved by concatenating all items before `i`, the new item(s), and all items after `i` (including `i` itself):

```d
template Insert(long i, E, Args...)
if((i < Args.length) && (i > 0)  && !isTuple!E)
{
  alias lhs = Args[0..(i)];
  alias rhs = Args[(i)..$];
  alias Insert = AliasSeq!(lhs, E, rhs);
}
```

The code for inserting multiple items `E` defined as a `Tuple` the first of which is at position `i` is given as:

```d
template Insert(long i, E, Args...)
if((i < Args.length) && (i > 0) && isTuple!E)
{
  alias lhs = Args[0..(i)];
  alias rhs = Args[(i)..$];
  alias Insert = AliasSeq!(lhs, E.getTypes, rhs);
}
```

Let's see if it works:

```d
pragma(msg, "\nInitial sequence: ", tList);
alias insertSeq = AliasSeq!(int, long, ulong);
pragma(msg, "Sequence to insert: ", insertSeq);
alias insert0 = Insert!(1, Tuple!(insertSeq), tList);
pragma(msg, "Inserted sequence @ 1: ", insert0);
alias insert1 = Insert!(tList.length - 1, Tuple!(insertSeq), tList);
pragma(msg, "Inserted sequence @ ($ - 1): ", insert1);
```

output:

```d
Initial sequence: (bool, string, ubyte, short, ushort)
Sequence to insert: (int, long, ulong)
Inserted sequence @ 1: (bool, int, long, ulong, string, ubyte, short, ushort)
Inserted sequence @ ($ - 1): (bool, string, ubyte, short, int, long, ulong, ushort)
```

That's it! I leave the final case as an exercise: here, `I` is a `Tuple!(long[]...)` and `E` is a `Tuple!(T...)`.

```d
template Insert(I, E, Args...)
if(isTuple!I && isTuple!E && (I.length == E.length))
{/* Code ...*/}
```

## Coda

You've reached the end of the article, **well done!** It might seem as if this is a rabbit hole of compile time programming madness but I assure you it is not, it does actually have uses one of which I plan to write about at a later date.

**Thank you!**

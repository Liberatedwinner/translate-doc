# [변수의 범위](@id scope-of-variables)

변수의 *범위 (또는 변수 영역; scope)* 는 그 안에서 변수가 보이는 (visible) 영역이다. 
변수의 범위 설정은 변수의 이름이 서로 충돌하는 것을 막는데 도움이 된다.
이 개념은 직관적이다: 두 함수 모두는 두 `x`가 같은 것을 참조하지 않아도 `x`라는 전달인자를 가질 수 있다. 
마찬가지로, 코드의 다른 블록에서 같은 것을 참조하지 않아도 동일한 이름을 사용할 수 있는 수많은 경우가 있다.
동일한 변수가 같은 것을 참조하거나 참조하지 않을 때의 규칙을 범위 규칙 (scope rule)이라고 하며, 이 절에서는 이를 세세히 살펴볼 것이다.

언어의 특정 구조 (construct)는 *범위 블록* 을 도입하는데, 이는 변수의 몇몇 집합의 범위로 적합한, 코드의 영역들이다.
변수의 범위는 소스 줄의 임의 집합이 될 수 없다. 그 대신에, 변수의 범위는 항상 이들 블록 중 한 곳에 해당한다. 
Julia에는 *전역 범위 (global scope)* 와 *지역 범위 (local scope)* 의 두 주된 유형의 범위가 있다.
후자는 중첩이 가능하다. 범위 블록을 도입한 구조들은 다음과 같다.

### [범위 구조](@id man-scope-table)

구조 (Construct) | 범위 type | 중첩가능한 범위 블록 (Scope blocks it may be nested in)
------------ | -------------  |---------------------------
[`module`](@ref), [`baremodule`](@ref)            | global | global
interactive prompt (REPL)                         | global | global
(mutable) [`struct`](@ref), [`macro`](@ref)       | local  | global
[`for`](@ref), [`while`](@ref), [`try-catch-finally`](@ref try), [`let`](@ref) | local | global or local
functions (either syntax, anonymous & do-blocks) | local | global or local
comprehensions, broadcast-fusing                 | local | global or local

주목할 것은, 이 표에서 [begin 블록](@ref man-compound-expressions)과 [if 블록](@ref man-conditional-evaluation)이 빠졌다는 것인데, 이 둘은 새로운 범위를 도입하지 *않았다*.
두 범위의 type은 아래에 설명할 조금 다른 규칙을 따르고 있다.

Julia는 정적 범위 ([lexical scoping](https://en.wikipedia.org/wiki/Scope_%28computer_science%29#Lexical_scoping_vs._dynamic_scoping))를 사용하는데, 이는 함수의 범위가 그 호출자 (caller) 의 범위에서 상속되는 것이 아니라 함수가 정의된 범위에서 상속됨을 의미한다. 예를 들어서, 다음의 코드에서 `foo` 내의 `x`는 모듈 `bar`의 전역 범위에 있는 `x`를 참조한다.

```jldoctest moduleBar
julia> module Bar
           x = 1
           foo() = x
       end;
```

그리고 `foo`가 사용된 범위의 `x`를 참조하는 것이 아니다.

```jldoctest moduleBar
julia> import .Bar

julia> x = -1;

julia> Bar.foo()
1
```

그러므로 *정적 범위* 는 변수의 범위를 소스 코드만으로 유추할 수 있음을 의미한다.

## 전역 범위

Each module introduces a new global scope, separate from the global scope of all other modules;
there is no all-encompassing global scope. Modules can introduce variables of other modules into
their scope through the [using or import](@ref modules) statements or through qualified access using the
dot-notation, i.e. each module is a so-called *namespace*. Note that variable bindings can only
be changed within their global scope and not from an outside module.

```jldoctest
julia> module A
           a = 1 # a global in A's scope
       end;

julia> module B
           module C
               c = 2
           end
           b = C.c    # can access the namespace of a nested global scope
                      # through a qualified access
           import ..A # makes module A available
           d = A.a
       end;

julia> module D
           b = a # errors as D's global scope is separate from A's
       end;
ERROR: UndefVarError: a not defined

julia> module E
           import ..A # make module A available
           A.a = 2    # throws below error
       end;
ERROR: cannot assign variables in other modules
```

Note that the interactive prompt (aka REPL) is in the global scope of the module `Main`.

## Local Scope

A new local scope is introduced by most code blocks (see above
[table](@ref man-scope-table) for a complete list).
A local scope inherits all the variables from a parent local scope,
both for reading and writing.
Unlike global scopes, local scopes are not namespaces,
thus variables in an inner scope cannot be retrieved from the parent scope through some sort of
qualified access.

The following rules and examples pertain to local scopes.
A newly introduced variable in a local scope cannot be referenced by a parent scope.
For example, here the ``z`` is not introduced into the top-level scope:

```jldoctest
julia> for i = 1:10
           z = i
       end

julia> z
ERROR: UndefVarError: z not defined
```

!!! note
    In this and all following examples it is assumed that their top-level is a global scope
    with a clean workspace, for instance a newly started REPL.

Inner local scopes can, however, update variables in their parent scopes:

```jldoctest
julia> for i = 1:1
           z = i
           for j = 1:1
               z = 0
           end
           println(z)
       end
0
```

Inside a local scope a variable can be forced to be a new local variable using the [`local`](@ref) keyword:

```jldoctest
julia> for i = 1:1
           x = i + 1
           for j = 1:1
               local x = 0
           end
           println(x)
       end
2
```

Inside a local scope a global variable can be assigned to by using the keyword [`global`](@ref):

```jldoctest
julia> for i = 1:10
           global z
           z = i
       end

julia> z
10
```

The location of both the `local` and `global` keywords within the scope block is irrelevant.
The following is equivalent to the last example (although stylistically worse):

```jldoctest
julia> for i = 1:10
           z = i
           global z
       end

julia> z
10
```

The `local` and `global` keywords can also be applied to destructuring assignments, e.g.
`local x, y = 1, 2`. In this case the keyword affects all listed variables.

In a local scope, all variables are inherited from its parent
global scope block unless:

  * an assignment would result in a modified *global* variable, or
  * a variable is specifically marked with the keyword `local`.

Thus global variables are only inherited for reading, not for writing:

```jldoctest
julia> x, y = 1, 2;

julia> function foo()
           x = 2        # assignment introduces a new local
           return x + y # y refers to the global
       end;

julia> foo()
4

julia> x
1
```

An explicit `global` is needed to assign to a global variable:

!!! sidebar "Avoiding globals"
    Avoiding changing the value of global variables is considered by many
    to be a programming best-practice.
    Changing the value of a global variable can cause "action at a distance",
    making the behavior of a program harder to reason about.
    This is why the scope blocks that introduce local scope require the `global`
    keyword to declare the intent to modify a global variable.

```jldoctest
julia> x = 1;

julia> function foobar()
           global x = 2
       end;

julia> foobar();

julia> x
2
```

Note that *nested functions* can modify their parent scope's *local* variables:

```jldoctest
julia> x, y = 1, 2;

julia> function baz()
           x = 2 # introduces a new local
           function bar()
               x = 10       # modifies the parent's x
               return x + y # y is global
           end
           return bar() + x # 12 + 10 (x is modified in call of bar())
       end;

julia> baz()
22

julia> x, y # verify that global x and y are unchanged
(1, 2)
```

The reason to allow modifying local variables of parent scopes in
nested functions is to allow constructing [`closures`](https://en.wikipedia.org/wiki/Closure_%28computer_programming%29)
which have private state, for instance the `state` variable in the
following example:

```jldoctest
julia> let state = 0
           global counter() = (state += 1)
       end;

julia> counter()
1

julia> counter()
2
```

See also the closures in the examples in the next two sections. A variable,
such as `x` in the first example and `state` in the second, that is inherited
from the enclosing scope by the inner function is sometimes called a
*captured* variable. Captured variables can present performance challenges
discussed in [performance tips](@ref man-performance-tips).

The distinction between inheriting global scope and nesting local scope
can lead to some slight differences between functions
defined in local versus global scopes for variable assignments.
Consider the modification of the last example by moving `bar` to the global scope:

```jldoctest
julia> x, y = 1, 2;

julia> function bar()
           x = 10 # local, no longer a closure variable
           return x + y
       end;

julia> function quz()
           x = 2 # local
           return bar() + x # 12 + 2 (x is not modified)
       end;

julia> quz()
14

julia> x, y # verify that global x and y are unchanged
(1, 2)
```

Note that the above nesting rules do not pertain to type and macro definitions as they can only appear
at the global scope. There are special scoping rules concerning the evaluation of default and
keyword function arguments which are described in the [Function section](@ref man-functions).

An assignment introducing a variable used inside a function, type or macro definition need not
come before its inner usage:

```jldoctest
julia> f = y -> y + a;

julia> f(3)
ERROR: UndefVarError: a not defined
Stacktrace:
[...]

julia> a = 1
1

julia> f(3)
4
```

This behavior may seem slightly odd for a normal variable, but allows for named functions -- which
are just normal variables holding function objects -- to be used before they are defined. This
allows functions to be defined in whatever order is intuitive and convenient, rather than forcing
bottom up ordering or requiring forward declarations, as long as they are defined by the time
they are actually called. As an example, here is an inefficient, mutually recursive way to test
if positive integers are even or odd:

```jldoctest
julia> even(n) = (n == 0) ? true : odd(n - 1);

julia> odd(n) = (n == 0) ? false : even(n - 1);

julia> even(3)
false

julia> odd(3)
true
```

Julia provides built-in, efficient functions to test for oddness and evenness called [`iseven`](@ref)
and [`isodd`](@ref) so the above definitions should only be considered to be examples of scope,
not efficient design.

### Let Blocks

Unlike assignments to local variables, `let` statements allocate new variable bindings each time
they run. An assignment modifies an existing value location, and `let` creates new locations.
This difference is usually not important, and is only detectable in the case of variables that
outlive their scope via closures. The `let` syntax accepts a comma-separated series of assignments
and variable names:

```jldoctest
julia> x, y, z = -1, -1, -1;

julia> let x = 1, z
           println("x: $x, y: $y") # x is local variable, y the global
           println("z: $z") # errors as z has not been assigned yet but is local
       end
x: 1, y: -1
ERROR: UndefVarError: z not defined
```

The assignments are evaluated in order, with each right-hand side evaluated in the scope before
the new variable on the left-hand side has been introduced. Therefore it makes sense to write
something like `let x = x` since the two `x` variables are distinct and have separate storage.
Here is an example where the behavior of `let` is needed:

```jldoctest
julia> Fs = Vector{Any}(undef, 2); i = 1;

julia> while i <= 2
           Fs[i] = ()->i
           global i += 1
       end

julia> Fs[1]()
3

julia> Fs[2]()
3
```

Here we create and store two closures that return variable `i`. However, it is always the same
variable `i`, so the two closures behave identically. We can use `let` to create a new binding
for `i`:

```jldoctest
julia> Fs = Vector{Any}(undef, 2); i = 1;

julia> while i <= 2
           let i = i
               Fs[i] = ()->i
           end
           global i += 1
       end

julia> Fs[1]()
1

julia> Fs[2]()
2
```

Since the `begin` construct does not introduce a new scope, it can be useful to use a zero-argument
`let` to just introduce a new scope block without creating any new bindings:

```jldoctest
julia> let
           local x = 1
           let
               local x = 2
           end
           x
       end
1
```

Since `let` introduces a new scope block, the inner local `x` is a different variable than the
outer local `x`.

### For Loops and Comprehensions

`for` loops, `while` loops, and [Comprehensions](@ref) have the following behavior: any new variables
introduced in their body scopes are freshly allocated for each loop iteration, as if the loop body
were surrounded by a `let` block:

```jldoctest
julia> Fs = Vector{Any}(undef, 2);

julia> for j = 1:2
           Fs[j] = ()->j
       end

julia> Fs[1]()
1

julia> Fs[2]()
2
```

A `for` loop or comprehension iteration variable is always a new variable:

```julia-repl enable_doctest_when_deprecation_warning_is_removed
julia> function f()
           i = 0
           for i = 1:3
           end
           return i
       end;

julia> f()
0
```

However, it is occasionally useful to reuse an existing local variable as the iteration variable.
This can be done conveniently by adding the keyword `outer`:

```jldoctest
julia> function f()
           i = 0
           for outer i = 1:3
           end
           return i
       end;

julia> f()
3
```

## Constants

A common use of variables is giving names to specific, unchanging values. Such variables are only
assigned once. This intent can be conveyed to the compiler using the [`const`](@ref) keyword:

```jldoctest
julia> const e  = 2.71828182845904523536;

julia> const pi = 3.14159265358979323846;
```

Multiple variables can be declared in a single `const` statement:
```jldoctest
julia> const a, b = 1, 2
(1, 2)
```

The `const` declaration should only be used in global scope on globals.
It is difficult for the compiler to optimize code involving global variables, since
their values (or even their types) might change at almost any time. If a global variable will
not change, adding a `const` declaration solves this performance problem.

Local constants are quite different. The compiler is able to determine automatically when a local
variable is constant, so local constant declarations are not necessary, and in fact are currently
not supported.

Special top-level assignments, such as those performed by the `function` and `struct` keywords,
are constant by default.

Note that `const` only affects the variable binding; the variable may be bound to a mutable
object (such as an array), and that object may still be modified. Additionally when one tries
to assign a value to a variable that is declared constant the following scenarios are possible:

* if a new value has a different type than the type of the constant then an error is thrown:
```jldoctest
julia> const x = 1.0
1.0

julia> x = 1
ERROR: invalid redefinition of constant x
```
* if a new value has the same type as the constant then a warning is printed:
```jldoctest
julia> const y = 1.0
1.0

julia> y = 2.0
WARNING: redefining constant y
2.0
```
* if an assignment would not result in the change of variable value no message is given:
```jldoctest
julia> const z = 100
100

julia> z = 100
100
```
The last rule applies for immutable objects even if the variable binding would change, e.g.:
```julia-repl
julia> const s1 = "1"
"1"

julia> s2 = "1"
"1"

julia> pointer.([s1, s2], 1)
2-element Array{Ptr{UInt8},1}:
 Ptr{UInt8} @0x00000000132c9638
 Ptr{UInt8} @0x0000000013dd3d18

julia> s1 = s2
"1"

julia> pointer.([s1, s2], 1)
2-element Array{Ptr{UInt8},1}:
 Ptr{UInt8} @0x0000000013dd3d18
 Ptr{UInt8} @0x0000000013dd3d18
```
However, for mutable objects the warning is printed as expected:
```jldoctest
julia> const a = [1]
1-element Array{Int64,1}:
 1

julia> a = [1]
WARNING: redefining constant a
1-element Array{Int64,1}:
 1
```

Note that although sometimes possible, changing the value of a `const` variable
is strongly discouraged, and is intended only for convenience during
interactive use.
Changing constants can cause various problems or unexpected behaviors.
For instance, if a method references a constant and is already
compiled before the constant is changed then it might keep using the old value:
```jldoctest
julia> const x = 1
1

julia> f() = x
f (generic function with 1 method)

julia> f()
1

julia> x = 2
WARNING: redefining constant x
2

julia> f()
1
```

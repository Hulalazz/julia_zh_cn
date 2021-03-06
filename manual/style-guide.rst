.. _man-style-guide:

**********
 代码样式
**********

The following sections explain a few aspects of idiomatic Julia coding style.
None of these rules are absolute; they are only suggestions to help familiarize
you with the language and to help you choose among alternative designs.

写成函数，别写成脚本
--------------------

Writing code as a series of steps at the top level is a quick way to get
started solving a problem, but you should try to divide a program into
functions as soon as possible. Functions are more reusable and testable,
and clarify what steps are being done and what their inputs and outputs are.
Furthermore, code inside functions tends to run much faster than top level
code, due to how Julia's compiler works.

It is also worth emphasizing that functions should take arguments, instead
of operating directly on global variables (aside from constants like ``pi``).

避免类型过于严格
----------------

Code should be as generic as possible. Instead of writing::

    convert(Complex{Float64}, x)

it's better to use available generic functions::

    complex(float(x))

The second version will convert ``x`` to an appropriate type, instead of
always the same type.

This style point is especially relevant to function arguments. For
example, don't declare an argument to be of type ``Int`` or ``Int32``
if it really could be any integer, expressed with the abstract type
``Integer``.  In fact, in many cases you can omit the argument type
altogether, unless it is needed to disambiguate from other method
definitions, since a ``MethodError`` will be thrown anyway if a type
is passed that does not support any of the requisite operations.
(This is known as `duck typing <http://en.wikipedia.org/wiki/Duck_typing>`_.)

For example, consider the following definitions of a function
``addone`` that returns one plus its argument::

    addone(x::Int) = x + 1             # works only for Int
    addone(x::Integer) = x + one(x)    # any integer type
    addone(x::Number) = x + one(x)     # any numeric type
    addone(x) = x + one(x)             # any type supporting + and one

The last definition of ``addone`` handles any type supporting the
``one`` function (which returns 1 in the same type as ``x``, which
avoids unwanted type promotion) and the ``+`` function with those
arguments.  The key thing to realize is that there is *no performance
penalty* to defining *only* the general ``addone(x) = x + one(x)``,
because Julia will automatically compile specialized versions as
needed.  For example, the first time you call ``addone(12)``, Julia
will automatically compile a specialized ``addone`` function for
``x::Int`` arguments, with the call to ``one`` replaced by its inlined
value ``1``.  Therefore, the first three definitions of ``addone``
above are completely redundant.

Handle excess argument diversity in the caller
----------------------------------------------

Instead of::

    function foo(x, y)
        x = int(x); y = int(y)
        ...
    end
    foo(x, y)

use::

    function foo(x::Int, y::Int)
        ...
    end
    foo(int(x), int(y))

This is better style because ``foo`` does not really accept numbers of all
types; it really needs ``Int`` s.

One issue here is that if a function inherently requires integers, it
might be better to force the caller to decide how non-integers should
be converted (e.g. floor or ceiling). Another issue is that declaring
more specific types leaves more "space" for future method definitions.

如果函数修改了它的参数，在函数名后加 `!`
----------------------------------------

Instead of::

    function double{T<:Number}(a::AbstractArray{T})
        for i = 1:endof(a); a[i] *= 2; end
	a
    end

use::

    function double!{T<:Number}(a::AbstractArray{T})
        for i = 1:endof(a); a[i] *= 2; end
	a
    end

The Julia standard library uses this convention throughout and
contains examples of functions with both copying and modifying forms
(e.g., ``sort`` and ``sort!``), and others which are just modifying
(e.g., ``push!``, ``pop!``, ``splice!``).  It is typical for
such functions to also return the modified array for convenience.

避免奇葩的类型集合
------------------

像 ``Union(Function,String)`` 这样的类型，说明你的设计有问题。

Try to avoid nullable fields
----------------------------

When using ``x::Union(Nothing,T)``, ask whether the option for ``x`` to be
``nothing`` is really necessary. Here are some alternatives to consider:

- Find a safe default value to initialize ``x`` with
- Introduce another type that lacks ``x``
- If there are many fields like ``x``, store them in a dictionary
- Determine whether there is a simple rule for when ``x`` is ``nothing``.
  For example, often the field will start as ``nothing`` but get initialized at
  some well-defined point. In that case, consider leaving it undefined at first.

Avoid elaborate container types
-------------------------------

It is usually not much help to construct arrays like the following::

    a = Array(Union(Int,String,Tuple,Array), n)

In this case ``cell(n)`` is better. It is also more helpful to the compiler
to annotate specific uses (e.g. ``a[i]::Int``) than to try to pack many
alternatives into one type.

使用和 Julia ``base/`` 相同的命名传统
-------------------------------------

- 模块和类型名称以大写开头, 并且使用驼峰形式: ``module SparseMatrix``,
  ``immutable UnitRange``.
- 函数名称使用小写 (``maximum``, ``convert``). 在容易读懂的情况下把几
  个单词连在一起写 (``isequal``, ``haskey``). 在必要的情况下, 使用下划
  线作为单词的分隔符. 下划线也可以用来表示多个概念的组合
  (``remotecall_fetch`` 相比 ``remotecall(fetch(...))`` 是一种更有效的
  实现), 或者是为了区分 (``sum_kbn``). 简洁是提倡的, 但是要避免缩写
  (``indexin`` 而不是 ``indxin``) 因为很难记住某些单词是否缩写或者怎么
  缩写的.

如果一个函数需要多个单词来描述, 想一下这个函数是否包含了多个概念, 这样
的情况下最好分拆成多个部分.


不要滥用 try-catch
------------------

It is better to avoid errors than to rely on catching them.

不要把条件表达式用圆括号括起来
------------------------------

Julia doesn't require parens around conditions in ``if`` and ``while``.
Write::

    if a == b

instead of::

    if (a == b)

不要滥用 ...
------------

Splicing function arguments can be addictive. Instead of ``[a..., b...]``,
use simply ``[a, b]``, which already concatenates arrays.
``collect(a)`` is better than ``[a...]``, but since ``a`` is already iterable
it is often even better to leave it alone, and not convert it to an array.

Don't use unnecessary static parameters
---------------------------------------

A function signature::

    foo{T<:Real}(x::T) = ...

should be written as::

    foo(x::Real) = ...

instead, especially if ``T`` is not used in the function body.
Even if ``T`` is used, it can be replaced with ``typeof(x)`` if convenient.
There is no performance difference.
Note that this is not a general caution against static parameters, just
against uses where they are not needed.

Note also that container types, specifically may need type parameters in
function calls. See the FAQ :ref:`man-abstract-container-type`
for more information.

Avoid confusion about whether something is an instance or a type
----------------------------------------------------------------

Sets of definitions like the following are confusing::

    foo(::Type{MyType}) = ...
    foo(::MyType) = foo(MyType)

Decide whether the concept in question will be written as ``MyType`` or
``MyType()``, and stick to it.

The preferred style is to use instances by default, and only add
methods involving ``Type{MyType}`` later if they become necessary
to solve some problem.

If a type is effectively an enumeration, it should be defined as a single
(ideally ``immutable``) type, with the enumeration values being instances
of it. Constructors and conversions can check whether values are valid.
This design is preferred over making the enumeration an abstract type,
with the "values" as subtypes.

不要滥用 macros
---------------

Be aware of when a macro could really be a function instead.

Calling ``eval`` inside a macro is a particularly dangerous warning sign;
it means the macro will only work when called at the top level. If such
a macro is written as a function instead, it will naturally have access
to the run-time values it needs.

Don't expose unsafe operations at the interface level
-----------------------------------------------------

If you have a type that uses a native pointer::

    type NativeType
        p::Ptr{Uint8}
        ...
    end

don't write definitions like the following::

    getindex(x::NativeType, i) = unsafe_load(x.p, i)

The problem is that users of this type can write ``x[i]`` without realizing
that the operation is unsafe, and then be susceptible to memory bugs.

Such a function should either check the operation to ensure it is safe, or
have ``unsafe`` somewhere in its name to alert callers.

Don't overload methods of base container types
----------------------------------------------

It is possible to write definitions like the following::

    show(io::IO, v::Vector{MyType}) = ...

This would provide custom showing of vectors with a specific new element type.
While tempting, this should be avoided. The trouble is that users will expect
a well-known type like ``Vector`` to behave in a certain way, and overly
customizing its behavior can make it harder to work with.

Be careful with type equality
-----------------------------

You generally want to use ``isa`` and ``<:`` (``issubtype``) for testing types,
not ``==``. Checking types for exact equality typically only makes sense
when comparing to a known concrete type (e.g. ``T == Float64``), or if you
*really, really* know what you're doing.

不要写 ``x->f(x)``
------------------

高阶函数经常被用作匿名函数来调用，虽然这样很方便，但是尽量少这么写。例如，尽量把 ``map(x->f(x), a)`` 写成 ``map(f, a)`` 。

.. Since higher-order functions are often called with anonymous functions, it
.. is easy to conclude that this is desirable or even necessary.
.. But any function can be passed directly, without being "wrapped" in an
.. anonymous function. Instead of writing ``map(x->f(x), a)``, write
.. ``map(f, a)``.

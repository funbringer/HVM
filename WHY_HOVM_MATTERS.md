What is HOVM, and why it matters?
=================================

In essence, HOVM is just a machine that takes, as its input, a functional
program that looks like untyped Haskell, and outputs its evaluated result. For
example, given the following input:

```javascript
// Doubles every number in the [1, 2, 3] list
(Fn (Nil))       = (Nil)
(Fn (Cons x xs)) = (Cons (* x 2) (Fn xs))
(Main)           = (Fn (Cons 1 (Cons 2 (Cons 3 Nil))))
```

HOVM outputs `(Cons 2 (Cons 4 (Cons 6 Nil)))`. That's it. What makes it special,
though, is **how** it does that.

What makes HOVM special?
========================

HOVM is based on a new, mathematically beautiful model of computation, the
Interaction Net, which is like the perfert child of the Lambda Calculus with the
Turing Machine. In a way, it is very similar to Haskell's STG, but with key
differences that give it outstanding characteristics, such as:

- Being **beta-optimal**: it is exponentially faster for many inputs.

- Being **inherently parallel**: it can be evaluated in thousands of cores.

- Being **memory-efficient**: no garbage collection, pure mutable datatypes.

- Being **strongly confluent**: it has a solid "gas cost" model.

In other words, thanks to this elegant underlying model, a language compiled to
HOVM can be as expressive as Haskell, as memory-efficient as Rust, all while
having the potential to run in thousands of cores, like CUDA.

The idea for HOVM existed before (Formality's old runtime), but, up to a few
months ago, the real-world efficiency was still 20-30x behind GHC, mostly due to
pointer overload, which negated its theoretical advantages. Thanks to a recent
breakthrough in the memory layout, we were able to completely redesign the
runtime, and reach a peak speed of **2.5 billion rewrites per second** on my
machine. That's **50x** more than the previous implementation, and enough to
compete with GHC today. Given its efficiency, and the naturally superior
properties of the Interaction Net, I firmly believe HOVM's current design is
ready to scale and become the undisputed fastest runtime in the world.

Benchmarks
==========

Before we get technical, let's see some benchmarks against Haskell's GHC. Note
that HOVM's current implementation is a proof-of-concept implemented in about 1
month by 4 part-time devs. It obviously won't always beat the most mature
functional runtime in the world. In some of the tests below, HOVM obliterates
GHC due to better asymptotics; but that's not new, due to asymptotics (after
all, Python QuickSort > C BubbleSort). What is notable, though, is that even in
cases where optimality plays no role, HOVM still does fairly well.

Haskell was measured with:

```bash
ghc -O2 main.hs -o main
time ./main
```

HOVM was measured with:

```bash
hovm main.hovm
clang -O2 main.c -o main
time ./main
```

Composition of Identity
-----------------------

**Applies the identity function `2^N` times to 0.**

<table>
<tr> <td>HOVM</td> <td>Haskell</td> </tr>
<tr>
<td>

```javascript
(Comp 0 f x) = (f x)
(Comp n f x) = (Comp (- n 1) λk(f (f k)) x)
(Main)       = (Comp N λx(x) 0)
```

</td>
<td>

```haskell
comp 0 f x = f x
comp n f x = comp (n - 1) (\x -> f (f x)) x
main       = print$ comp n (\x->x) (0::Int)
```

</td>
</tr>
</table>

// TODO: CHART HERE

#### Comment

Function composition is one of cornerstones of functional programming, and,
amazingly, it is one of the cases where GHC performs poorly. In general, if the
composition of a function `f` has a constant-size normal form, then `f^N(x)` is
constant-time (`O(L)`) on HOVM, and exponential-time (`O(2^L)`) on GHC, where
`L` is the bit-size of `N`. Since the normal form of `id . id` is just `id`,
this program is exponentially faster in HOVM.

Composition of Increment
------------------------

**Applies the `λx -> x + 1` function `2^N` times to 0.**

<table>
<tr> <td>HOVM</td> <td>Haskell</td> </tr>
<tr>
<td>

```javascript
(Pow 0 f x) = (f x)
(Pow n f x) = (Pow (- n 1) λk(f (f k)) x)
(Main)      = (Pow N λx(+ x 1) 0)
  ```

</td>
<td>

```haskell
pow 0 f x = f x
pow n f x = pow (n - 1) (\x -> f (f x)) x
main      = print $ pow n (\x -> x + 1) (0 :: Int)
```

</td>
</tr>
</table>

// TODO: CHART HERE

#### Comment

The composition of `u32_inc = λx. x + 1` does NOT have a constant-size normal
form. For example, `u32_inc^4(x) = λx. x + 1 + 1 + 1 + 1`, and the size grows as
`N` grows. Because of that, both HOVM and GHC have the same asymptotics here.
To my surprise, though, HOVM is about 2x faster, even single-threaded.

Composition of Increment (using datatypes)
------------------------------------------

**Applies the `Inc` function `2^N` times to BitString datatype.**

<table>
<tr> <td>HOVM</td> <td>Haskell</td> </tr>
<tr>
<td>

```javascript
(N)           = 26
(Comp 0 f x)  = (f x)
(Comp n f x)  = (Comp (- n 1) λk(f (f k)) x)
(Inc (E))     = (E)
(Inc (O bs))  = (I bs)
(Inc (I bs))  = (O (Inc bs))
(Zero 0)      = (E)
(Zero n)      = (O (Zero (- n 1)))
(ToInt (E))   = 0
(ToInt (O n)) = (* (ToInt n) 2)
(ToInt (I n)) = (+ (* (ToInt n) 2) 1)
(Main)        = (ToInt (Comp N λx(Inc x) (Zero 32)))
```

</td>
<td>

```haskell
data Bits   = E | O Bits | I Bits deriving Show
comp 0 f x  = f x
comp n f x  = comp (n - 1) (\k -> f (f k)) x
inc E       = E
inc (O bs)  = I bs
inc (I bs)  = O (inc bs)
zero 0      = E
zero n      = O (zero (n - 1))
toInt (E)   = 0
toInt (O n) = toInt n * 2
toInt (I n) = toInt n * 2 + 1
main        = print $ toInt (comp n inc (zero 32))
```

</td>
</tr>
</table>

// TODO: CHART HERE

#### Comment

This benchmark is similar to the previous, except that, instead of incrementing
a machine integer `2^N` times, we increment a BitString represented as an
algebraic datatype. The purpose of this benchmark is to stress-test how fast the
runtime can perform pattern-matching and recursion. There is no asymptotical
gain on the HOVM side, yet it is faster here (again, to my surprise).

Composition of Increment (using λ-encoded datatypes)
----------------------------------------------------

**Applies the `Inc` function `2^N` times to λ-Encoded BitString datatype.**

<table>
<tr> <td>HOVM</td> <td>Haskell</td> </tr>
<tr>
<td>

```javascript
(N)          = 24
(Comp 0 f x) = (f x)
(Comp n f x) = (Comp (- n 1) λk(f (f k)) x)
(E)          = λe λo λi e
(O pred)     = λe λo λi (o pred)
(I pred)     = λe λo λi (i pred)
(Inc bs)     = λe λo λi (bs e i λpred(o (Inc pred)))
(Zero 0)     = (E)
(Zero n)     = (O (Zero (- n 1)))
(ToInt bs)   = (bs 0 λn(* (ToInt n) 2) λn(+ (* (ToInt n) 2) 1))
(Main)       = (ToInt (Comp N λx(Inc x) (Zero 64)))
```

</td>
<td>

```haskell
newtype BS = BS { get :: forall a. a -> (BS -> a) -> (BS -> a) -> a }
comp 0 f x = f x
comp n f x = comp (n - 1) (\k -> f (f k)) x
e          = BS (\e -> \o -> \i -> e)
o pred     = BS (\e -> \o -> \i -> o pred)
i pred     = BS (\e -> \o -> \i -> i pred)
inc bs     = BS (\e -> \o -> \i -> get bs e i (\pred -> o (inc pred)))
zero 0     = e
zero n     = o (zero (n - 1))
toInt bs   = get bs 0 (\n -> toInt n * 2) (\n -> toInt n * 2 + 1)
main       = print $ toInt (comp n (\x -> inc x) (zero 64))
```

</td>
</tr>
</table>

// TODO: CHART HERE

#### Comment

This is again similar to the previous benchmark, except that, this time, instead
of using built-in datatypes to represent the BitString, we're using the [Scott
Encoding](https://kseo.github.io/posts/2016-12-13-scott-encoding.html), which is
a way to represent data using lambdas. Doing so causes HOVM to be exponentially
faster than GHC. This result looks wrong; after all, we shouldn't be able to
call `inc` a quadrillion times instantaneously. But that's exactly what happens.
What is going on is that, since the composition of *inc* has a small normal
form, this causes it to be slowly morphed into an "add with carry function" as
the program executes. This effect is what I call "runtime fusion".

Haskell programmers aren't unfamiliar with λ-encodings being used for
optimization. For example, the widely used [Free Monads
library](https://hackage.haskell.org/package/free-5.1.7/docs/Control-Monad-Free-Church.html)
has an entire version that replaces native datatypes by λ-encodings, and it is
much faster that way. But, since GHC isn't optimal, the application of this
technique is very limited in Haskell. HOVM opens doors for an entire field of
unexplored algorithms based on runtime fusion. I'd not be surprised if
there are solutions to hard problems lying there.



TODOS
=====

Before we get started, let's dive into some benchmarks.

**[TODO]** benchmarks where HOVM greatly outperforms GHC due to beta-optimality:
function exponentiation, map fusion, lambda encoding arithmetic, etc. 'HOVM is
to GHC as GHC is to C"

**[TODO]** benchmarks where HOVM greatly outperforms GHC due to parallelism:
sorting, rendering, parsing, mendelbrot set, etc.

**[TODO]** benchmarks where HOVM still underperforms GHC: numeric loops,
in-place mutation, C-like code, etc.

**[TODO]** Remark that HOVM is still on its infancy, and there are still many
optimizations and features to add (like mutable arrays) before it is generally
good; but stress that, on the long term, the underlying computation model paves
the way for it to be faster than everything that exists, specially when we
consider that 1. functional / high-order algorithms are every day more common,
2. the future is parallel and the core count will explode as soon as we start
using them properly

**[TODO]** explain how HOVM got there, starting from optimal evaluators (absal,
optlam, etc., which weren't as efficient as GHC in practice), models of
computation (Turing Machines, Lambda Calculus, Interaction Nets), and how the
new memory format plus user-defineds rewrite rules allowed HOVM to become an
order of magnitude faster and finally get close to GHC in real-world cases

**[TODO]** explain how it is for that a high-level language not to be garbage
collected. trace parallels with Rust, explaining how both languages are similar
on their basis, yet differ when it comes to duplication, with Rust going the
reference/borrow route, while HOVM goes the novel "lazy cloning" route, which
allows structures to be lazily shared/copied, including lambdas

**[TODO]** explain how it is asymptotically more efficient than Haskell in many
cases, and why. begin by explaining why lazy evaluation is, intuitivelly,
superior, since it only computes what is necessary, but is actually inferior
since it can lead to computing the same expression twice. explain how Haskell
solves this problem via sharing (memoization of redexes). explain how it isn't
capable of sharing computations inside lambda binders. and that's when HOVM is
superior to Haskell, asymptotically

**[TODO]** remember the intuitive explanation of why it doesn't need a gc
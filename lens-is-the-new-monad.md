There are my notes from watching Edward's talk on Lenses[^lens-talk]

* * *

# Lenses From Scratch

## The data-lens approach

### Introducing Lens and Store datatypes

A Lens is a pair of 'Getter' and 'Setter' into a data structure. These operations are named `view` and `set`.

```haskell
view :: s -> a
set  :: s -> a -> s
```

Where `s` is the 'whole' and `a` is the 'part'.

Combined we get -

```haskell
data Lens s a = Lens (s -> a) (s -> a -> s)
```

We can combine the common parameter `s` to get -

```haskell
data Lens s a = Lens (s -> (a, a -> s))
```

The structure `(a, a -> s)` is also known as a `Store` -

```haskell
data Store a s = Store a (a -> s)
```

So then we have -

```haskell
data Lens s a = Lens (s -> Store a s)
```

### Store is a Costate Comonad Coalgebra

The following is taken from tel's lens tutorial[^lens-tutorial]

A `Comonad w a` is defined as something with operations -

```haskell
extract   :: w a -> a
duplicate :: w a -> w (w a)
```

It turns out that `Store` is a `Comonad` -

```haskell
instance Comonad (Store a) where
  extract (Store a f) = f a
  duplicate (Store a f) = Store a (\b -> Store b f)
```

Now `Store` is also called `Costate Comonad` as it is dual to `State`. Traditionally, a co-something is something with the arrows reversed. However in this case, we reverse `(,)` and `(->)`. So calling Store as Costate is a slight misnomer.

```haskell
State == (a -> (a ,  s))
Store == (a ,  (a -> s))
```

`Store` is also a `Functor` in the same way `(->)` is a Functor -

```haskell
instance Functor (Store a) where
  fmap g (Store a f) = Store a (g . f)
```

A `Coalgebra` for a `Functor` is simply something that constructs the Functor from a 'seed' -

```haskell
type Coalg f a = a -> f a
```

So the Coalgebra for Store is the same as Lens -

```haskell
Coalg (Store a) s == s -> Store a s == Lens s a
```

So effectively Lens is a `Store Coalgebra` or a `Costate Comonad Coalgebra`.

### We need something more

While this definition of a lens works fine (it's the approach used by data-lens package), it has two problems -

1. It is not possible to write a Lens that changes the type of the data structure
2. To use a Lens, we have to import the Store data structure and associated libraries.

These problems are solved in the Lens library using something called Van Laarhoven Lenses.

## Semantic Editor Combinators

A `Semantic Editor Combinator` (first introduced by Conal Elliott), is something that 'modifies' something deep within a structure.

```haskell
type SEC s t a b = (a -> b) -> s -> t
```

[^lens-talk]: Edward's talk on lenses on [Video](http://www.youtube.com/watch?v=cefnmjtAolY)
[^lens-tutorial]: Tel's [lens tutorial](https://www.fpcomplete.com/user/tel/lenses-from-scratch)

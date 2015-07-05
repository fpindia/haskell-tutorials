This is a comprehensive lens tutorial that is distilled from watching [Edward's talk on Lenses](http://www.youtube.com/watch?v=cefnmjtAolY)
---------------

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

Combining them we get -

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

This is the approach used by the `data-lens` library, but *NOT* the `lens` library.

### Digression 1 - Lens is a "Costate Comonad Coalgebra"

The data-lens approach tells us why Lens is also a "Costate Comonad Coalgebra".

Note: The following section is taken from [tel's lens tutorial](https://www.fpcomplete.com/user/tel/lenses-from-scratch)

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

### Digression 2 - Lens Laws

The two lens operations, when adapted to work with the Lens datatype look like this -

```haskell
view :: Lens s a -> s -> a
set :: Lens s a -> s -> a -> s
```

These operations *must* follow the following lens laws -

##### 1. If we get something from `s` and then put it back, it's the same as not doing anything -

```haskell
set l (view l s) s = s
```

In other words - "A lens has no hidden effects".

##### 2. If we put something in, we can get it back out -

```haskell
view l (set l s a) = a
```

##### 3. If we set something, and then 'overwrite' it with something else, it's the same as if I only did the second setting. The first setting has no effect -

```haskell
set l (set l s a) b = set l s b
```

### We need something more

While this definition of a lens works fine (it's the approach used by data-lens package), it has two problems -

1. It is not possible to write a Lens that changes the type of the data structure
2. To use a Lens, we have to import the Store data structure and associated libraries
3. We can compose lenses using the Data.Category (.) but to use it we need to hide the one from prelude

These problems are solved in the Lens library using something called "Van Laarhoven Lenses".


## Semantic Editor Combinators

### Basic idea

Note: This section borrows from [Conal's original blog post on Semantic Editors Combinators](http://conal.net/blog/posts/semantic-editor-combinators)

A `Semantic Editor Combinator` (first introduced by Conal Elliott), is something that 'modifies' something deep within a structure.

```haskell
type SEC s t a b = (a -> b) -> s -> t
```

Where again `s` is the 'whole' and `a` is the 'part'. When `a` is replaced by `b`, the original `s` becomes a `t`. The types here are kept as general as possible to apply to all situations. In specific instances, `s` and `t` usually relate to each other in some way.

### Simple Examples

`first` and `second` from `Control.Arrow` modify the first and the second parts respectively of a pair (when operating on functions).

```haskell
first :: (a -> b) -> (a,x) -> (b,x)
second :: (a -> b) -> (x,a) -> (x,b)
```

Looking at the definition of `SEC`, these can be represented as -

```haskell
first :: SEC (a,x) (b,x) a b
second :: SEC (x,a) (x,b) a b
```

Similarly we can write a combinator that modifies the output of a function -

```haskell
result :: SEC (a -> b) (a -> c) b c
result = (.)
```

Or a combinator that modifies the argument of the function -

```haskell
argument :: SEC (a -> c) (b -> c) b a
argument = flip (.)
```

### The power of the dot

Semantic editor combinators don't necessarily act on only one 'point' in a 'structure'.

For example we can write a combinator to modify all elements of a list

```haskell
element :: (a -> b) -> [a] -> [b]
element = fmap
```

This `fmap` is a very general purpose SEC that can modify parts of *any* `Functor` instance.

For example it can also be used in place of `second` (thanks to the functor instance for tuples) or `result` (thanks to the functor instance for functions).

```haskell
fmap :: Functor f => SEC (f a) (f b) a b
```
Multiple `fmap`s can be composed -

```haskell
fmap . fmap . fmap :: (Functor f, Functor g, Functor h) => SEC (f (g (h a))) (f (g (h b))) a b
```

Turns out that all SEC compose in a similar fashion. For example (.) -

```haskell
(.) . (.) . (.) :: SEC (c -> d -> e -> a) -> (c -> d -> e -> b) a b
```

In fact any number of different SEC can be freely composed using (.) to modify deeply embedded values in a typesafe manner -

```haskell
deep :: SEC (a, b -> [c]) (a, b -> [d]) c d
deep = second . result . element
```

Where the definition can be read intuitively as - "with the second element of the pair, then with the result of the function, modify all the elements of the list"

This definition seems to have the order of the functions reversed. Intuitively whenever we compose functions together in Haskell, the operation we perform first comes at the end. However with SEC (and with lenses), the order of functions is reversed (similar to the record update syntax in procedural languages).

### Traversable

Another example of an SEC is `Traversable`.

A `Traversable` gives us a function `traverse` which can traverse the structure from left to right.

```haskell
traverse :: (Applicative m, Traversable f) => (a -> m b) -> f a -> m (f b)
```

Note: When `m` is a `Monad` and `f` is a List then `traverse` is the same as `mapM`

`Traverse` can be composed in a similar fashion to `fmap` and is an `SEC` -

```haskell
traverse :: (Traversable f, Applicative m) => SEC (f a) (f b) a (m b)
traverse . traverse :: (Traversable f, Traversable g, Applicative m) => SEC (f (g a)) (f (g b)) a (m b)
```

Note that while this is an `SEC`, unlike the other `SEC` we saw, the "modification function" passed to the SEC is of the type `a -> m b` instead of the usual `a -> b`. This means it doesn't compose well with other SEC. We solve this problem with `Setter`, which is described in the next section.


## Setters

### Solving the Traversable problem

All `Traversable` are also `Functor`, this can be easily seen when `m` is `Identity`. Hence `Traversable` has `Functor` as a superclass. To get an `fmap` for a `Traverse`, use `fmapDefault`.

```haskell
fmapDefault :: Traversable f => (a -> b) -> f a -> f b
fmapDefault = runIdentity . traverse (Identity . f)
```

This allows you to write code like -

```haskell
instance Functor Foo where
  fmap = fmapDefault -- We let the Traversable instance derive fmap

instance Traversable Foo where
  traverse = ...
```

Within `fmapDefault` the type of `traverse` is specialised to use the `Identity` functor -

```haskell
traverse :: (a -> Identity b) -> f a -> Identity (f b)
```

This is kind of like an `SEC` (with `s` == `f a` and `t` == `f b`) but with `Identity` mucking things up.

So let's generalise this type in the same manner we generalised `SEC`.

```haskell
type Setter s t a b = (a -> Identity b) -> s -> Identity t
traverse :: Setter s t a b
```

These `Setter` are used to generalise "deep modification" of data structure in the same way as `SEC` did. Let's see how.

### 'over' and 'mapped'

In `fmapDefault` if we make `traverse` an argument instead of hard coding it, we get a different function. We call this `over`.

```haskell
over :: Setter s t a b -> (a -> b) -> s -> t
over :: Setter s t a b -> SEC s t a b
over t f = runIdentity . t (Identity . f)
```

Which means that `over` can take a `Setter` and generate an `SEC` out of it. Which can then be used as before.

For example, let's see if we can generate `fmap` using `over` -

We need a `Setter` where `s` is `f a` and `t` is `f b`. Let's call this function `mapped`.

```haskell
mapped :: Functor f => Setter (f a) (f b) a b
```

Its definition can be derived by carefully canceling out the effects of `runIdentity` from the definition of `over`. 

```haskell
mapped f = Identity . fmap (runIdentity . f)
```

And

```haskell
over mapped f == runIdentity . mapped (Identity . f)
              == runIdentity . Identity . fmap (runIdentity . Identity . f)
              == fmap f -- Because runIdentity . Identity == id
```

`Setter` compose just like `SEC` -

```haskell
mapped :: Setter (f a) (f b) a b
mapped :: (a -> Identity b) -> (f a) -> (Identity (f b))
==>
mapped . mapped :: (a -> Identity b) -> (f (g a)) -> (Identity (f (g b)))
                :: Setter (f (g a)) (f (g b)) a b
```

### Using Setters

`Setters` can be used to modify data in a `Functor` -

```haskell
over mapped (+1) [1,2,3] == [2,3,4]
over (mapped.mapped) (+1) [[1,2],[3]] == [[2,3],[4]]
```

`Setter` can even change the type of the data structure, unlike the formulation in 'data-lens'

```haskell
over (mapped.mapped) length [["hello", "world"], ["iii"]] = [[5,5], [3]]
```

`Setter` can easily be created for any datatype.

For example, `Text` is not a `Functor` because it can not contain values of arbitrary types. However we can easily create a `Setter` by simply converting it to a String and back -

```haskell
chars :: (Char -> Identity Char) ‐> Text -> Identity Text
chars f = fmap pack . mapped f	
```

And then we can compose this with other setters for varied functionality -

```haskell
over chars :: (Char ‐> Char) ‐> Text ‐> Text
over (mapped.chars) :: Functor f => (Char ‐> Char) -> f Text ‐> f Text
over (traverse.chars) :: Traversable f => (Char ‐> Char) ‐> f Text ‐>	f Text
```

Just like `chars`, it's possible to have other setters that aren't functors -

```haskell
both :: Setter (a,a) (b,b) a b
both f (a,b) = (,) <$> f a <$> f b
```

```haskell
first :: Setter (a,c) (b,c) a b
first f (a,b) = (,b) <$> f a
```

And then similarly compose them -

```haskell
over both (+1) (2,3) = (3,4)
over (mapped.both) length (("hello", "world"), ("iii", "iiii")) = ((5,5), (3,4))
```


### Setter Laws

`Setter` are very similar to `Functor` and must follow similar laws -

##### 1. If you `fmap`/`over` something with `id` it's like you did nothing at all

For functors -

```haskell
fmap id = id
```

For setters -

```haskell
over l id = id
```

##### 2. If you `fmap`/`over` `f` and then `g`, it's like `fmap`/`over` `(f . g)`

For functors -

```haskell
fmap f . fmap g = fmap (f . g)
```

For setters -

```haskell
over l f . over l g = over l (f . g)
```

Note: Unlike functors, the second law of setters doesn't automatically follow from the first.


### Other Setters



### TO BE CONTINUED ##


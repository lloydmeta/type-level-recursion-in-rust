## Type-level recursion
### in Rust

[github.com/lloydmeta](https://github.com/lloydmeta)

---

## Type-level what?

* __Normal recursion__: your logic runs until an exit condition is reached (could stack overflow at runtime)
* __Type-level recursion__: your compiler runs until an exit type is found (could stack overflow at compile-time)

Note: Ok, I may be butchering/making up a term, but by “type-level recursion”, I’m referring to recursive expansions/evaluations of types at compile-time, particularly for the purpose of proving that a certain typeclass instance exists at a function call site.

This is distinct from runtime “value”-level recursion that occurs when you call a function that calls itself.

If you’re having trouble understanding the difference:

__Value-level recursion__: If it can’t find an exit condition, your program is stuck running forever.

__Type-level recursion__: If it can’t expand/find the exit-type, your compiler will either give up or never finish compiling; you won’t even have a program to run.

----

## Motivation

Transforming unaligned structs

```rust
#[derive(LabelledGeneric)]
struct NewUser<'a> {
    first_name: &'a str,
    last_name: &'a str,
    age: usize,
}

// Notice that the fields are mismatched in terms of ordering
// *and* also in terms of the number of fields.
#[derive(LabelledGeneric)]
struct ShortUser<'a> {
    last_name: &'a str,
    first_name: &'a str,
}

let n_user = NewUser {
    first_name: "Joe",
    last_name: "Blow",
    age: 30,
};

// transform_from automagically sculpts the labelled generic
// representation of the source object to that of the target type
let s_user: ShortUser = transform_from(n_user); // done
```

Note:

* Type-safe (compiler verified), performant (as fast or faster ;) than manually writing by hand)

---

## Traits

Traits are Rust's way of doing:

* Interfaces (OOP)
* Typelcasses (Haskell-style FP)

Note: Traits are not entirely necessary for writing Rust: I'm fairly sure you can write lots of useful programs in Rust without using this language feature. It might be painful, and you would have to forgo using a lot of the standard library, but it's possible.

In practical terms, Traits allows you to do:

1. Method Overloading
2. Separation of data from behaviour (nice, because unlike the OOP approach of interfaces, you don't need to edit the original type definition in order to add/change behaviour)

----

### Example Trait w/ Impl

```rust
// A trait that allows you to call "area" on something
trait HasArea {
    fn area(&self) -> f64;
}

// Our type
struct Circle {
    x: f64,
    y: f64,
    radius: f64,
}

// Our implementation of HasArea for Circle
impl HasArea for Circle {
    fn area(&self) -> f64 {
        std::f64::consts::PI * (self.radius * self.radius)
    }
}
```

---

## Dependent Trait implementations

`impl TraitX for TypeA`

depends on

`impl TraitY for TypeB`

Note:
Sometimes, you want to have the implementation of a trait for a given type be _dependent_ on the existence of an implementation of a trait (not necessarily even the same) for another type.

----

### `Add` trait from std lib

```rust
pub trait Add<RHS=Self> {
    /// The resulting type after applying the `+` operator
    #[stable(feature = "rust1", since = "1.0.0")]
    type Output;

    /// The method for the `+` operator
    #[stable(feature = "rust1", since = "1.0.0")]
    fn add(self, rhs: RHS) -> Self::Output;
}
```

Note:

This is copied straight from `core::ops`

Note that the Add trait has a right hand side (RHS) type parameter to represent the type that the implementing trait is being added

----

### Our example data type

```rust
struct Cup<A> {
    content: A,
}
```

Note:

This is our data type. It's a Cup with a single field, content. It's a struct and it's job is to hold data. In order to make it act like it has behaviour, we need to implement traits for it.

The `A` type parameter means that our Cups will be generic on the content field. Cuz we're fancy like dat.

----

### Implementing `Add` for `Cup`

```
impl<A: Add<A>> Add<Cup<A>> for Cup<A>
{
    // This is what is called an associated type.
    // Here, Output is the type that will be returned
    // from the add operation
    type Output = Cup<< A as Add<A> >::Output>;

    fn add(self, rhs: Cup<A>) -> Self::Output {
        // Here we make use of the Add trait for A to add
        // the contents from both cups together
        let added_content = self.content.add(rhs.content);
        Cup { content: added_content }
    }
}
```

Note:
Making `Cup` part of the `Add` trait will allow us to call `cup_a + cup_b`

Note that the type of Output is `Cup<< A as Add<A> >::Output>`, which means that ultimately, the output of Adding of 2 `Cup<A>`s will depend on what the Output of `Add<A>` is. The `< A as Add<A> >` part can be read as “summon the `Add<A>` implementation for the type `A`” (the compiler will do the actual lookup work here; if one doesn’t exist, your code will fail to compile), and the `::Output` following it means “retrieve the associated type, `Output`, from that implementation”. Let this sink in, because it's an important concept.

----

### Even more versatile implementation

```rust
impl<A, B> Add<Cup<B>> for Cup<A>
    // This next line means "A must have an Add<B> implementation"
    where A: Add<B>
{
    // The Output associated type now depends on the Output of <A as Add<B>>
    type Output = Cup<<A as Add<B>>::Output>;

    fn add(self, rhs: Cup<B>) -> Self::Output {
        // Notice that we can use the operator "+"
        let added_content = self.content + rhs.content;
        Cup { content: added_content }
    }
}
```

Note:

In our original implementation, we only allowed adding 2 cups together if their type parameters were the same. In this one, we open that part up and say that we allow adding 2 cups together if there exists an `Add` implementation for the left hand side cup holding `A` that allows for adding with a type `B`.

In order to support this, we add a `B` type parameter to our trait implementation.

The rest is quite similar to the original.

---

## Must knows

* Traits
* How to use implementation bounds
* How to summon an implementation for a given type
* How to write and use associated types

Note:

These are some of the things that we've covered thus far and you must know in order to understand what's coming up next.

---

## Mental model for value-level recursion

> You write a function that keeps calling itself until an exit condition is met, then returns a value.

---

## Mental model for type-level recursion

> You write implementations of your trait for exit-types and work-to-be-done types.
>
> If the compiler can't find a trait implementation for the exit-type, your program won't compile.

Note:

In order to prove an implementation of your trait exists for a concrete type at a function call site, the compiler will try to lookup/expand recursively until it can figure out a concrete implementation to use, or gives up with an error.

This is just a mental model, actual implementations may differ / change.

---

## Actual type-level recursive use-case

"Plucking" from an HList.

```
// Our HList types consist of these
pub struct HCons<H, T> {
    pub head: H,
    pub tail: T,
}
pub struct HNil;

// h has type Hlist![ {integer}, &str, f32, bool ]
let h = hlist![ 1, "Joe", 42f32, true ];

// We tell it the target type, and let the compiler infer the rest
let (target, remainder): (f32, _) = h.pluck();

assert_eq!(target, 42f32);
assert_eq!(remainder, hlist![1, "Joe", true]);
```

Note:

* An HList is a heterogenously-typed list. The length and type are known at compile time
* This may look pointless but it's an important small piece in a bigger picture.

---

## Attempt 1

```rust
// Our trait
trait Plucker<Target> {

  type Remainder;

  // Pluck should return the target type and the Remainder in a pair
  fn pluck(self) -> (Target, Self::Remainder);
}
```

Note:

Super simple first attempt. `Target` is the type we want to pluck, and we have a an associated type for our remainder to generalise on that.

----

## Exit-type trait impl

```rust
impl <Target, Tail> Plucker<Target> for HCons<Target, Tail> {

  // Target is the head element, so the Remainder type is the tail!
  type Remainder = Tail;

  fn pluck(self) -> (Target, Self::Remainder) {
    (self.head, self.tail)
  }
}
```

Note:

The “exit-type” implementation is for when the current head of the HList contains the target type.

----

## Recurse-type trait impl

```rust
impl <Target, Head, Tail> Plucker<Target> for HCons<Head, Tail>
  where Tail: Plucker<Target>

  // Target is in the tail, so we add the current head type to the remainder
  // And use the Tail's Plucker's Remainder type as the tail :)
  type Remainder = HCons<Head, <Tail as Plucker<Target>>::Remainder>;

  fn pluck(self) -> (Target, Self::Remainder) {
    let (tail_target, tail_remainder): (Target, <Tail as Plucker<Target>>::Remainder) = self.tail.pluck();
    (
      tail_target,
      HCons { head: self.head, tail: tail_remainder}
    )

  }
}
```

Note:

* This is the non-trivial part where the target type is not in Head, but in the Tail of our HList. I’ll sometimes refer to this as the “work-to-be-done” type.
* We require an implementation of `Plucker<Target>` for the `Tail` of our HCons
* Our associated `Remainder` type is now going to depend on the `Remainder` of our tail.

----

## The magic

Given these 2 implementations, the compiler will, at a given `pluck()` call-site, try to generate implementations.

----

## Sadly

```rust
error[E0119]: conflicting implementations of trait `Plucker<_>` for type `frunk_core::hlist::HCons<_, _>`:
   --> tests/example.rs:306:1
    |
296 |   impl <Target, Tail> Plucker<Target> for HCons<Target, Tail> {
    |  _- starting here...
297 | |
298 | |     // Target is the head element, so the Remainder type is the tail!
299 | |     type Remainder = Tail;
300 | |
301 | |     fn pluck(self) -> (Target, Self::Remainder) {
302 | |         (self.head, self.tail)
303 | |     }
304 | | }
    | |_- ...ending here: first implementation here
305 |
306 |   impl <Target, Head, Tail> Plucker<Target> for HCons<Head, Tail>
    |   ^ conflicting implementation for `frunk_core::hlist::HCons<_, _>`
```

Note:

The Rust compiler is helpfully is telling us that it can’t distinguish between our two implementations

----

## It's true

```rust
// exit (work done) type implementation
impl <Target, Tail>  Plucker<Target> for HCons<Target, Tail>

// work-to-be-done implementation
impl <Target, Head, Tail> Plucker<Target> for HCons<Head, Tail>
```

Note:

The `Plucker<Target>` part is exactly the same, and sure, we’ve used `Target` instead of `Head` in the for `HCons<..>` part in the first case, but simply using different type parameters isn’t enough to distinguish between the two.

---

## Back to the drawing board

Trick: Introduce a new type parameter that we will fill in with concrete types

```rust
// the new and improved Plucker trait
trait Plucker<Target, Index> {
    type Remainder;

    fn pluck(self) -> (Target, Self::Remainder);
}
```

----

## Exit-type trait impl

```rust
// This will be the type we'll use to denote that the Target is in the Head
enum Here {}

impl <Target, Tail> Plucker<Target, Here> for HCons<Target, Tail> {

  // Target type is in the Head, so the Remainder type must be the tail!
  type Remainder = Tail;

  fn pluck(self) -> (Target, Self::Remainder) {
    (self.head, self.tail)
  }
}
```

Note:

`Here` is an empty enum because we just want to have a _type_ and have no real use for a runtime value.

Our implementation uses `Here` as a way of indicating that we have the target type "here" in the head of the list and don't need to traverse further. Note that `Here` is a concrete type.

----

## Recurse-type trait impl

```rust
// Type for representing a not-here Index
struct There<T>(PhantomData<T>);

impl<Head, Tail, Target, TailIndex> Plucker<Target, There<TailIndex>> for HCons<Head, Tail>
    // This where clause can be interpreted as "the target must be pluckable from the Tail"
    where Tail: Plucker<Target, TailIndex>
{
    type Remainder = HCons<Head, <Tail as Plucker<Target, TailIndex>>::Remainder>;

    fn pluck(self) -> (Target, Self::Remainder) {
        let (target, tail_remainder): (Target, <Tail as Plucker<Target, TailIndex>>::Remainder) =
            <Tail as Plucker<Target, TailIndex>>::pluck(self.tail);
        (target,
         HCons {
             head: self.head,
             tail: tail_remainder,
         })
    }
}
```

Note:

* `PhantomData` is just there to satisfy the compiler (otherwise it complains that there is an unused and thus unbounded type parameter)
* `There<T>` is a concrete type that has a type parameter (to be filled-in/found by the compiler)
* Credit goes to an exiting HList lib

----

## Fruit of our efforts

```rust
let h: Hlist![ &str, bool, f32, i32 ] = hlist![ "Joe", false, 42f32, 9 ];
let (v, _): (f32,_) = h.pluck();
```

---

## Taking it to the next level

Sculpting HLists

```rust
// Given an HList of type Hlist![ i32, &str, f32, bool ]
let h] = hlist![9000, "joe", 41f32, true];

let (reshaped, remainder): (Hlist![ f32, i32, &str ], _) = h.sculpt();
```

Note:

We'd like to be able to "sculpt" it into another, differently shaped HList.

Of course, the types in the new HList must be a subset of the original HList,
and if not, compilation should fail.

Similar to pluck(), we'd also want the remainder of the original HList _not_
used in the final result.

----

## Left as an exercise

Hint: Use an Hlist type for the index.

---

# Thanks !

Check out frunk: https://docs.rs/frunk
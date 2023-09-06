- Feature Name:  `usagetimes`
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

**Usagetimes** feature provides concept that based on which components can be used independently.

So, the new feature allows partial borrowing based on usage scenarios provided by type itself.

Note: for convenience of reader, **usagetimes** will be **bold**, so you can use them as anchors while reading.

# Motivation
[motivation]: #motivation

In general, partial borrowing allows safety usage scenarios that can't be described with current syntax.
For reference, you can see [next issue](https://github.com/rust-lang/rfcs/issues/1215).
Plus, you can read [OpenGL API Gist](https://gist.github.com/kylewlacy/824e09131f0f3b4b9062),
which requires this or similar feature to be implemented for achieving the high performance.

Also, I want to draw attention to partial borrow problem and make a conclusion about situation at the present moment.
I hope, that section will point RFC authors on existing problems and gives them change to rethinking about their concepts.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

**Usagetimes** can be provided for `enums`, `structs` and `traits`, and specifies non overlapping usages.
That's why the named **usagetimes**.

## Example #1: Providing **usagetimes** for `struct`

Assume, you have some struct that obviously can be used independently:

```rust
trait Draw {
    fn draw(&self, surface: &mut Surface);
}

// Here we say, that Point have usagetimes `x, `y and `color
#[derive(Default)]
struct Point<`x, `y, `color> {
    x: `x f64,
    y: `y f64,
    // 8u - alpha, 8u - red, 8u - green, 8u - blue
    c: `color u32
}

impl<`x, `y, `color, `coords: `x + `y> Point<`x, `y, `color> {
    // Mut self under color usagetime
    pub fn color_mut(&`color mut self) -> &mut Color {
        self.c
    }

    pub fn coords_mut(&`coords mut self) -> (&mut f64, &mut f64) {
        (&mut self.x, &mut self.y)
    }
}

// Point under `* usagetime - required to have all fields for usage
impl Draw for Point {
    fn draw(&self, surface: &mut Surface) { /* ... */ }
}

// Same as previous, `_ exist here for trait and can be used for specifying usagetimes of Point
impl Draw for Point<`_ as `x, `_ as `y, `_ as `color> {
    fn draw(&self, surface: &mut Surface) { /* ... */ }
}

// Point under exact usagetimes
// here `_ of Trait bound to `x, `y, `color of impl
impl<`x, `y, `color> Draw for Point<`x, `y, `color> {
    fn draw(&self, surface: &mut Surface) { /* ... */ }
}
```

The first `Draw` implementation is strait, but for now it is same as other variant.
Second one is based on trait usagetime `` `_ `` (same as `'_` for lifetimes).

When you decide to add some independent field to `Point`, the first one will require all fields to be involved,
but the second one - only `` `x ``, `` `y `` and `` `color ``.

Let's add `z` field to struct as independent part.

```rust
#[derive(Default)]
struct Point</* ... */ `z> {
    /* ... */
    z: `z f64
}

impl Point {
    /* ... */
    pub fn ref_z(&`z self) -> &f64 { self.z }
    pub fn mut_z(&`z mut self) -> &mut f64 { self.z }
}

// Variant I: impl Draw for Point
// Variant II: impl Draw for Point<`_ as `x, `_ as `y, `_ as `color>

fn main() {
    let mut surface = Surface::default();
    let mut point = Point::default();

    let draw = &point as &dyn Draw;

    // We want to mutate z here
    // error (Variant I): cannot borrow point under `z usagetime since it already was moved in line
    //                    let draw = &point as &dyn Draw;
    //                               ~~~~~~~~~~~~~~~~~~~~
    //                    Draw impl requires full usagetime of Point
    //
    // fine (Variant II): Draw impl for Point takes only `x, `y (via `coords) and `color usagetimes
    *point.mut_z() = 0.0;

    draw.draw(&mut surface);
}
```

You don't actually need to specify usagetimes for public fields:

```rust
// Here usagetime `c is replaced by `color since it never used
#[derive(Default)]
struct Point<`color> {
    pub x: f64,
    pub y: f64,
    pub z: f64,
    pub c: `color u32
}

fn mutate_all(x: &mut f64, y: &mut f64, z: &mut f64) {}

fn main() {
    let mut point = Point::default();
    // before    => after point.x         => after point.y     => after point.z
    // Point<`*> => Point<`*, `!x> => Point<`*, `!x, `!y> => Point<`*, `!x, `!y, `!z>
    mutate_all(point.x, point.y, point.z);
    // here we have Point<`*>

    let mut x = point.x;
    let mut y = point.y;
    let mut z = point.z;

    // before        => after point.color_mut()
    // Point<`*, `!x, `!y, `!z> => Point<`*, `!x, `!y, `!z, `!color>
    let mut color = point.color_mut();

    // We actually can read address and this is safe
    // But note, that we can't move point or access it fields
    println!("Point address is {}", &point as *const Point as usize);
}
```

Let's say that we want have trait `Point` trait for all `Point1D`, `Point2D` and so on types.
And also we want to change color and coords independently:

```rust
#[derive(Default)]
struct Point1D {
    pub x: f64,
    pub color: u32
}

// Point2D and Point3D similar

trait Point<`coords, `color> {
    fn coords_ref(&`coords self) -> Vec<&f64>;
    fn coords_mut(&`coords mut self) -> Vec<&mut f64>;

    fn color_ref(&`color self) -> &Color;
    fn color_mut(&`color mut self) -> &mut Color;
}

// here `coords and `color provided automatically by Point trait
// `color of Point bounds automatically to Point1 `color
impl Point for Point1D<`coords as `x> {
    fn coords_ref(&`coords self) -> Vec<&f64> {
        vec![self.x]
    }

    fn coords_mut(&`coords mut self) -> Vec<&mut f64> {
        vec![&mut self.x]
    }

    fn color_ref(&`color self) -> &Color {
        self.color
    }

    fn color_mut(&`color mut self) -> &mut Color {
        &mut self.color
    }
}

// Point2D and Point3D similar
// impl Point for Point2D<`coords as `x, `coords as `y> {}
// And so on...

fn main() {
    let point1 = Point1D::default();
    println!("The `Point1` address is {}", &point1 as *const Point1 as usize);

    let mut point = Box::new(point1) as Box<dyn Point>;
    // here Point1 is no longer exist since it moved to Box

    // Safety, trait have non overlapping usagetimes `coords and `color
    let mut coords = point.coords_mut();
    let mut color = point.color_mut();
    // Box<dyn Point<`*, `!coords, `!color>>

    // Next line is error, but let's assume we can take address with the next way
    // Tracking issue (not related to RFC, just to this line): https://github.com/rust-lang/rust/issues/81513
    println!("The `dyn Point` address is {}", &*point as *const () as usize);

    // Here you can modify coords and color

    // Dropping both
    coords;
    color;

    // point is Box<dyn Point<`*>>
}
```

## Example #2: Providing **usagetimes** for `enum`

Enumerations can have its own **usagetimes** only for types in variants.
This is because we can't guarantee which variant will be at access moment.

```rust
// error: unable to define usagetimes for enum variant One(`one ...)
enum Enum<`one, `two, `three: `one + `two> {
    One(`one ...),
    Two(`two ...),
    Three(`three ...),
    Four(...)
}
```

But, what is more logically is that we can have several usagetimes for types inside:

```rust
// Here we bound usagetimes of enum to usagetimes of structs
// In case when we don't bound it, we need to full usagetime of the used type
enum PointVariants<`x, `y> {
    D1(&mut Point1D<`x>),
    D2(&mut Point2D<`x, `y>),
    D3(&mut Point3D<`x, `y>)
}

impl PointVariants {
    fn print_x(&`x self) {
        // Enum under full usage `x
        match &self {
            PointVariants::D1(point) => {
                // Here we reborrowing point under Point1D<`x> usagetime
                println!("Point1D under `x usagetime: Point1D { x: {} }", &point.x);
            }
            /* ... */
        }
    }

    fn print_y(&`y self) {
        // Enum under full usage `y
        match &self {
            PointVariants::D1(point) => {
                // point is Point1D<`!> since it doesn't have `y usagetime
                // So all is fine
                println!("Point1D {} hasn't field y", point as *const Point1D as usize);
            }
            /* ... */
        }
    }
}

fn main() {
    let mut point_3d = Point3D::default();
    let mut point = PointVariants::D3(&mut point_3d);

    // all is fine
    let z = &mut point_3d.z;
    point.print_x();
    point.print_y();

    // error: cannot take point_3d under usagetime `y since it already was taken early in line ...
    let y = &mut point_3d.y;
}
```

Here, enum says that it can mutate struct under **usagetimes** `` `x `` and `` `y ``.
And we can see, that it doesn't actually do this in scope, but we can't guarantee same in other scope.

So, **usagetimes** can't allow such unsound behavior.

Assume, we want to set `x` to `y` field:

```rust
impl PointVariants {
    // usagetimes can be provided locally for exact types
    // Here `y inherits mut from both, but x is ref only
    fn set_x_to_y<`both: ref `x + `y>(&`both mut self) {
        // Enum under `both usage
        match &self {
            PointVariants::D1(point) => {
                // point is Point1D<ref `x>, but in body we never access `y usagetime of Point1D
                // Which is not exist, so all is fine
                eprintln!("Point1D {} hasn't field y", point as *const Point1D as usize);
            }

            PointVariants::D2(point) => {
                // point is Point2D<ref `x, `y>
                // All is fine
                point.y = point.x;
            }
            /* ... */
        }
    }
}

// Here, obviously, we may have different enums
// `y inherits mut
fn mutate_y_print_x(set: &mut PointVariants<ref `x, `y>, print: &PointVariants<`x) {
    set.set_x_to_y();
    // Note: `x in print_x also inherits ref, since it's immutable borrowed
    print.print_x();
}

fn main() {
    let mut point_3d = Point3D::default();
    let mut point = PointVariants::D3(&mut point_3d);

    // all is fine
    let z = &mut point_3d.z;
    
    // All is fine, since usages doesn't overlaps in mut context
    // ref `x, mut `y \
    // ref `x         /
    // Totally safe
    mutate_y_print_x(&mut point, &point);
}
```

## Example #3: Providing usagetimes for trait

**Usagetimes** is powerful feature for advanced cases, so here is example,
how you can make trait require ineffective implementation:

```rust
trait LenIterator<`len, `next> {
    type Item;

    fn len(&`len self) -> usize;
    fn next(&`next mut self) -> Option<Self::Item>;
}
```

Here you take a role of library developer and you have some optimization for `LenIterator` objects.
But wait a second, different usagetimes requires to be non overlapping,
then you can't just modify both values at the same time!

```rust
struct LenIteratorStruct<T> {
    pub items: Vec<T>
}

// Note: here when we automatically resolve bounds, `len and `next never bounds to any of LenIteratorStruct
//       since there is no similar names `len, `items of LenIterator and `items of LenIteratorStruct
//        So we need to resolve it manually (and provide usagetimes for impl to bound trait and struct)
impl<`len, `next, T> LenIterator<`len, `next> for LenIteratorStruct<`next as `items, T> {
    type Item = T;

    // All is fine
    fn next(&`next mut self) -> Option<Self::Item> {
        self.items.pop()
    }

    // What we suppose to do here?
    fn len(&`len self) -> usize {
        // error: `len usagetime is not bounded to LenIteratorStruct
        // note: unbounded usagetimes cannot get access to type
        self.items.len()
    }
}

// Let's try to bound `len to LenIteratorStruct and `next to `items
// error: `len is bound to full usagetime of LenIteratorStruct, `next is bounded to `items already
//        so usagetimes `len and `next overlapping, which is forbidden
impl<`len, `next, T> LenIterator<`len, `next> for LenIteratorStruct<`next as `items, `len as `_, T> {
    /* ... */
}

// Let's try to not bound `next and `len to LenIteratorStruct
impl<`items, `len, `next, T> LenIterator<`len, `next> for LenIteratorStruct<`items, T> {
    type Item = T;

    fn next(&`next mut self) -> Option<Self::Item> {
        // error: `next usagetime is not bounded to LenIteratorStruct
        // note: unbounded usagetimes cannot get access to type
        self.items.pop()
    }

    // error: incorrect signature
    //        expected fn next(&`next mut self) -> Option<Self::Item>
    //        got      fn next(&`items mut self) -> Option<Self::Item>
    //
    //        Usagetime `items not found for trait LenIterator
    fn next(&`items mut self) -> Option<Self::Item> {
        unreachable!("Compile time error")
    }
}

// error: can't bound `len as `items because `next already bounded to `items
impl<`len, `next, T> LenIterator<`len, `next> for LenIteratorStruct<`next as `items, `len as `items, T> {}

// error: cannot bound `len to usage `_ of LenIteratorStruct since it already bounded to `next
impl<`len, `next, T> LenIterator<`len, `next> for LenIteratorStruct<`next as `_, `len as `_, T> {}

// UNRESOLVED QUESTION: (in thread)
//                      What is default bounds to usagetimes? Sync + Send or ?Sync + ?Send
//                      For now is ?Sync + ?Send, so we can't send type clone, or reference to another thread
//
// We need to provide usagetime since we want to implement trait with partial borrow
//   otherwise we will bound to whole struct and that is not possible
struct SpecialLenIteratorStruct<`items, `len, T> {
    items: `items Vec<T>,
    items_len: `items Rc<RefCell<usize>>,
    len: `len Rc<RefCell<usize>>
}

impl<`len, `next, T> LenIterator<`len, `next> for LenIteratorStruct<`next as `items, `len, T> {
    // `next bounded to `items - all is fine
    fn next(&`next mut self) -> Option<Self::Item> {
        if *self.items_len.borrow() > 0 {
            *self.items_len.borrow_mut() -= 1;
        }
        self.items.pop()
    }

    fn len(&`len self) -> usize {
        *self.len.borrow()
    }
}
```

All is works, but isn't better to overthinking about your trait definition?

When you use partial borrowing (via usagetime feature) is better to think about
"How user need to implement this trait" and make an implementation for some type.

Also, you can use the result for your documentation to guide someone how to use your library more effective.

## Example #4: Overlapping usagetimes

Assume, we have different line structs designed for different drawing features:

```rust
#[derive(Default)]
struct Point2D {
    x: f64,
    y: f64
}

#[derive(Default)]
struct StraitLine {
    start: Point2D,
    end: Point2D,
    color: u32
}

#[derive(Default)]
struct GradientLine {
    start: Point2D,
    end: Point2D,
    // 0.0 red, 0.5 green, 1.0 blue
    // start                    end
    // red >>>>>> green >>>>>> blue
    gradient_anchors: Vec<(f64, u32)>
}

// impl Point for Point2D
trait Point {
    fn x_mut(&mut self) -> &mut f64;
    fn y_mut(&mut self) -> &mut f64;

    fn distance(&self, other: &Self) -> &f64;
}

trait Line {
    type Point: Point;

    fn start_mut(&mut self) -> &mut Self::Point;
    fn end_mut(&mut self) -> &mut Self::Point;
    fn len(&self) -> &f64
}
```

But then you decide, that you need to mutate both start and end at the same time.
It is sound logically to calculate len with real values, so we need to forbid partial borrowing for len,
and have an ability to mutate start and end, when we don't need len.

```rust
// Assume, magic doesn't need to mutate x and y at once, and it does it in order
fn magic<P: Point>(start: &mut P, end: &mut P) {}

fn main() {
    let mut strait = StraitLine::default();
    let mut line = &mut strait as &mut dyn Line;

    // error: cannot borrow line as mut twice - standard error
    magic(line.start_mut(), line.end_mut());
}
```

Obviously, you need to have at least two usagetimes that not overlapping,
and one that overlapping with the previous both.
But usagetimes can't overlapping isn't? Fortunately, they can, but only when signature explicit says,
that one usagetimes overlaps with another one.

```rust
// Note we can put `len to fn len
trait Line<`start, `end, `len: `start + `end> {
    type Point: Point;

    fn start_mut(&`start mut self) -> &mut Self::Point;
    fn end_mut(&`end mut self) -> &mut Self::Point;
    fn len(&`len self) -> &f64;
}
```

Further implementation is trivial (we just need to provide usagetimes for `StraitPoint` and `GradientPoint`),
but you can find it at the next lines:

```rust
#[derive(Default)]
struct Point2D {
    x: f64,
    y: f64
}

#[derive(Default)]
struct StraitLine<`start, `end> {
    start: `start Point2D,
    end: `end Point2D,
    color: u32
}

#[derive(Default)]
struct GradientLine<`start, `end> {
    start: `start Point2D,
    end: `end Point2D,
    // 0.0 red, 0.5 green, 1.0 blue
    // start                    end
    // red >>>>>> green >>>>>> blue
    gradient_anchors: Vec<(f64, u32)>
}

// Note: we don't need to provide usagetimes, since `start and `end automatically bounds Line and StraitLine,
//       and `len is bounds only to Line (since there is no name for StraitLine)
impl Line for StraitLine {
    type Point = Point2D;

    fn start_mut(&`start mut self) -> &mut Self::Point {
        &mut self.start
    }

    fn end_mut(&`end mut self) -> &mut Self::Point {
        &mut self.end
    }

    // Why this is fine?
    fn len(&`len self) -> &f64 {
        // here `len stands for complex usagetime that contains `start and `end,
        // so trait signature forbid any actions under with `start or `end under `len at the same time
        // all just fine
        self.start.distance(self.other)
    }
}
```

And then you can use your magic function:

```rust
fn main() {
    let mut strait = StraitLine::default();
    let mut line = &mut strait as &mut dyn Line;

    // fine: &dyn Line<`*>
    //    => &dyn Line<`*, `!start> (first arg passed)
    //    => &dyn Line<`*, `!start, `!end> (second arg passed)
    magic(line.start_mut(), line.end_mut());
    // fine: after calling, usagetimes end, so we have &dyn Line<`*>
    println!("Len is {}", line.len());

    // Wrong usage
    {
        // &mut Point, &mut dyn Line<`end>
        let mut start = line.start_mut();
        // error: cannot borrow line under `len usagetime since
        //        it overlaps with `start usagetime which was borrowed early here ...
        let length = line.len();
    }
}
```

## Restrictions: `Sync` \ `Send` traits

UNRESOLVED QUESTION: (in thread)
                     Which bounds are default?

We can't partial borrow trait which usagetimes doesn't bounded by `Sync` and `Send` traits.
This trait is implemented automatically if it can be accessed safely as `&T`.

So, why not `Rc`? The reason is `Rc` cannot be passed safely even if it has strong `1` and weak `0` references.
This is due to internal representation that allows optimizations.

Otherwise, entire `Rc` would not be efficient. So what can you do? You can try or fail convert `Rc` to `T::* + Send + Sync`,
and then use partial trait or partial impl for exact that `T::* + Send + Sync`.

Note: Since **usagetimes** guarantee, that is no overlapping usage of `T` fields (`T::*`) there is no guarantee,
that fields will not point to the same data, so fields must be `Send + Sync` (for passing across threads).

Almost same for `RefCell` and other types. As for `RefCell`, it can be accessed as `&T` with **usagetimes**,
so it's unsafe to use this type across threads. Same for other `T::* + Send + !Sync`.

## Example #5: Sending partial mutable borrowing into other thread

Assume, we want to mutate start and end from different threads (see previous example for correct definitions).
Usagetimes allowing this!

```rust
// Here, we say that `start and `end usagetimes can be taken only for T: Sync + Send fields
// As for `len, we says, that there can't be mut access to `start or `end when taking `len usagetime
// But we says nothing about bounds to `len itself, so here we promise to not use len across threads
trait Line<`start: Sync + Send, `end: Sync + Send, `len: `start + `end> {
    // Point trait can't be accessed across threads, just can be used in some of them
    // If you need, you can add extra bounds for this GAT
    type Point: Point;

    // Can be used from other thread, fields under `start are Sync + Send
    fn start_mut(&`start mut self) -> &mut Self::Point;

    // Can be used from other thread, fields under `start are Sync + Send
    fn end_mut(&`end mut self) -> &mut Self::Point;

    // Can't be used from other thread, since we don't bound `len by Sync + Send
    fn len(&`len self) -> &f64;
}

// Here we bound usagetimes to both Line and StraitLine, so we can't have just `start for Line, since it's `start is Sync + Send
impl Line for StraitLine {
    type Point = Point2D;

    // All is fine, `start from Line is Sync explicit, `start from StraitLine is Sync implicit
    fn start_mut(&`start mut self) -> &mut Self::Point {
        &mut self.start
    }

    fn end_mut(&`end mut self) -> &mut Self::Point {
        &mut self.end
    }

    // Can't be used across threads
    fn len(&`len self) -> &f64 {
        self.start.distance(self.other)
    }
}

fn main() {
    let mut strait = StraitLine::default();
    let mut line = &mut strait as &mut dyn Line;

    std::thread::scope(|scope| {
        scope.spawn(|| {
            let mut start = line.start_mut();
            // mut safely here
        });

        // Same as previous, now we have only &mut dyn Line<`*, `!start, `!end>
        scope.spawn(|| {
            let mut end = line.end_mut();
            // mut safely here
        });
    });

    // All borrowings dropped here
    // All just fine and works safely
    println!("Len is {}", line.len());
}
```

What if trait doesn't required to usagetime to be Sync?

```rust
fn main() {
    let mut strait = StraitLine::default();
    let mut line = &mut strait as &mut dyn Line;

    std::thread::scope(|scope| {
        scope.spawn(|| {
            // error: cannot call start_mut since usagetime `start is not Sync
            let mut start = line.start_mut();
        });
    });
}

fn main() {
    let mut strait = StraitLine::default();

    std::thread::scope(|scope| {
        scope.spawn(|| {
            // Assume, start is public
            // Safe, since Point2D is sync implicitly with auto trait Sync and compiler can check this
            let start = &mut strait.start;
        });
    });

    // All borrowings dropped here
    println!("Len is {}", (strait as Line).len());
}
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Definition

Each entity (`struct`, `enum` or `trait`), may provide unique **usagetimes**.
At this point, no boundings allowed (except boundings for used types).

Also, each **usagetime** may contain other usagetimes (but not self).
This allows to use related fields in some context, while included **usagetimes**
cannot be used in the same time (this ensures safety).
These **usagetimes** are called complex. They also can be used for access to some specific members.

Same for simple borrowing, **usagetimes** can have any `ref`s and only one `mut`.

### Structs

When fields declared as public (for `struct`), these fields provides usagetimes with their names.

```rust
struct Struct<
    `first,
    `second,
    `third: `first,
    `fourth: `first + `second,
    `fifth: `third + `second,
    T
> {
    first: `first T,
    second: `second T,
    third: `third T,
    fourth: `fourth T,
    fifth: `fifth T
}

// With public fields
struct Struct<
    `third: `first,
    `fourth: `first + `second,
    `fifth: `third + `second,
    T
> {
    pub first: T,
    pub second: T,
    pub third: T,
    pub fourth: T,
    pub fifth: T
}
```

### Unions

Unions cannot have usagetimes, at all. Allowing that will need to take **usagetime** of used type eternally, at least.
For now I sure, that this is unsound hole, that I don't want to touch.
If you have some thoughts, you can text me (e.g. leave commentary under issue on github).

### Enums

Enumerations itself cannot have any usagetimes, but they may be defined for its variants'es types.

Do you still remember _Example #2_?

```rust
enum PointVariants<`x, `y> {
    D1(&mut Point1D<`x>),
    D2(&mut Point2D<`x, `y>),
    D3(&mut Point3D<`x, `y>)
}
```

Let's replace it by a new one:

```rust
enum PointVariants<`x, `y, `z> {
    D1(&mut Point3D<`x>),
    D2(&mut Point3D<`y>),
    D3(&mut Point3D<`z>)
}
```

This is fine, but compiler must be sure, that we will never access fields when haven't taken appropriate usagetimes:

```rust
impl PointVariants {
    // Here compiler know, that no usagetime taken for PointVariants
    // But it still track that usagetimes doesn't overlapping
    // For example, we can't use mut `test: `x + `y and `x at the same time
    pub fn swap_x_to_y(&mut self) {
        self = match &self {
            PointVariants::D1(point) => {
                // error: cannot take Point3D under `y usagetime, since point is Point3D<`x>
                PointVariants::D2(point)
            }
            // No swap when not x
            _ => _
        }
    }
}
```

Here, in definition, we said, that first variant takes only `` `x `` usagetime,
so obviously, it can be already taken under some other usagetime.

Note: if we replace `&mut T` by `T` with **usagetimes** in `PointVariants`, we can preform this operation,
since nobody can owns one type under different usagetimes.

But we still can define this function, but with passed argument:

```rust
impl PointVariants {
    // Here we need to add runtime check, if we need this argument to be same as borrowed
    pub fn swap_x_to_y(&mut self, maybe_self: &mut Point3D<`y>) {
        self = match &self {
            PointVariants::D1(_) => {
                PointVariants::D2(maybe_self)
            }
            // No swap when not x
            _ => _
        }
    }
}
```

And, most important, that compiler can be sure, that we have two different objects with same **usagetimes**:

```rust
impl PointVariants {
    pub fn replace_x(&mut self, other: &mut Point3D<`x>) {
        self = PointVariants::D1(other)
    }

    pub fn replace_y(&mut self, other: &mut Point3D<`y>) {
        self = PointVariants::D2(other)
    }

    pub fn replace_x_when_y(&`y mut self, other: &mut Point3D<`x>) {
        // error: cannot set PointVariants::D1(&mut Point3D<`x>) to self
        //        since self under `y usagetime
        self = PointVariants::D1(other)
    }
}
```

In examples above, we define **usagetimes** as vertical kebab, what if we try to make it horizontal?

```rust
enum PointVariants<`d1, `d2, `d3> {
    D1(&mut Point1D<`d1 as `x>),
    D2(&mut Point2D<`d2 as `x, `d2 as `y>),
    D3(&mut Point3D<`d3 as `x, `d3 as `y, `d3 as `z>)
}
```

More examples with this kind of enum bounding can be found below.

### Traits

Traits can define **usagetimes** in the same way as structs:

```rust
trait Trait<`first, `second, `third: `first> {
    fn do_all_ref(&self);
    fn do_all_mut(&mut self);

    fn do_first_ref(&`first self);
    fn do_first_mut(&`first mut self);

    fn do_second_ref(&`second self);
    fn do_second_mut(&`second mut self);

    fn do_third_ref(&`third self);
    fn do_third_mut(&`third mut self);
}
```

Here, trait defines which usagetimes can be taken at the same time.
That allows to call several non overlapping methods at once.

With this trait, we can take `` `first `` or `` `third `` and `` `second `` at the same time.

```rust
// first and second can be different, we can use runtime check for pointers
fn do_some_mut(first: &mut Trait<`first>, second: &mut Trait<`second>) {}

// Same, can be different objects
fn do_some_ref(first: &Trait<`first>, third: &Trait<`third>) {}

fn main() {
    let mut tt = Box::new(Struct) as Box<dyn Trait>;

    // Compiler knows, that signature permit only `first and `second usagetimes
    // Since one of them doesn't contain other we can allow this
    do_some_mut(&mut *tt, &mut *tt);

    // Compiler knows, that tt takes with ref `first and ref `third
    // Since, we can't mutates anything at the same time, or read while mutates
    // All is also fine
    do_some_ref(&*tt, &*tt);
}
```

Since here, I want to describe how usagetimes interact with traits that "requires" whole type.

Assume, we have some type that need to default it field when cloning.

```rust
// The most common example is Clone
pub trait Clone: Sized {
    fn clone(&self) -> Self;

    fn clone_from(&mut self, source: &Self) {
        *self = source.clone()
    }
}
```

Type with **usagetimes** can tell to compiler, that actually, it doesn't use some field:

```rust
struct Struct {
    pub no_cloned: f64,
    pub value: f64
}

// Here Clone `_ bounds to Struct<`value>
impl<`value> Clone for Struct<`value> {
    fn clone(&self) -> Self {
        // Note: cannot access to no_cloned, since `_ not bound to `no_cloned
        Struct { value: self.value, no_cloned: 0.0 }
    }

    fn clone_from(&mut self, source: &Self) {
        // Same for source, sine Self is Struct<`value>
        self.value = source.value;
    }
}
```

Same for other types and traits.

When value is dropped, we already know, that type is `` `* ``, so **usagetimes** here are useless.

## Borrowing

Partial borrowing is possible via using `as` keyword (cast) with **usagetime** specified in type:

```rust
fn main() {
    let mut original = Struct { /* ... */ };
    let partial = &mut original as &mut Struct<`_ as `first>;
}
```

In this example `` `_ `` means `Self` and **usagetime** `` `first `` is mut since we cast it as mut type.
For shortcut, choosing **usagetimes** from `` `_ `` can be omitted:

```rust
fn main() {
    let mut original = Struct { /* ... */ };
    let partial = &mut original as &mut Struct<`first>;
}
```

Any **usagetime** can be `ref` or `mut` - taken, or not taken.

```rust
fn main() {
    let mut original = Struct { /* ... */ };
    let partial = &mut original as &mut Struct<ref `first, `second>;
}
```

I this example, `` `second `` inherits same borrowing as one that used in cast (right side).
In case when we try to borrow mut **usagetime**, when struct is borrowed, we will get an error:

```rust
fn main() {
    let mut original = Struct { /* ... */ };
    // Note: without mut keyword, `second becomes ref
    // error: cannot borrow usagetime `second as mut, since it casts as reference
    let partial = &mut original as &Struct<ref `first, mut `second>;
}
```

Usagetimes allows us to have one mutable field and one immutable:

```rust
fn main() {
    let mut original = Struct { /* ... */ };
    let partial = &mut original as &mut Struct<ref `first, `second>;
    // All is fine, first is immutable referenced
    let first = &original.first;
}
```

**Usagetime** can be negated. This is used when some partial borrow occurs:

```rust
fn main() {
    let mut original = Struct { /* ... */ };
    let partial = &mut original as &mut Struct<ref `first, mut `second>;
    // original now is &mut Struct<`*, ref `first, `!second>
    // original cannot be taken under usagetime that contains `second (including self)
    // original can be taken only under ref usagetime that contains `first (also including self)

    // that's why it's allowed
    let first = &original.first;

    partial;
    // At this moment compiler knows, that partial is &mut Struct<ref `first, mut `second>
    // So, after `first usagetime ended, origin becomes &mut Struct<`*, `!second>
    // And after `second usagetime is ended, origin becomes &mut Struct<`*>
    // Note: here we counts any ref and mut same as for simple borrowing
}
```

While **usagetime** begins, we can't have overlapping with other usagetimes:

```rust
fn main() {
    let mut original = Struct { /* ... */ };
    let partial = &mut original as &mut Struct<`second>;
    // error: original can't be taken under `second usagetime
    //        since it already was taken under this usagetime early at line ...
    let _ = &mut original as &mut Struct<`second>;
}
```

**Usagetime borrowing** implements partial borrowing and allows us to get all benefits of true safe partial borrowing.

So, we can put mutable struct into different vec under different **usagetimes**:

```rust
fn main() {
    let mut original = Struct { /* ... */ };

    // Variant I
    {
        // Here only_first: Vec<&mut Struct<`first>
        // Since here original is &mut Struct<`*, `!first>;
        let mut only_first = vec![&mut original as &mut Struct<`first>];

        // Here only_second: Vec<&mut Struct<`second>
        // Since here original is &mut Struct<`*, `!first, `!second>;
        let mut only_second = vec![&mut original as &mut Struct<`second>];

        // Here head can be used as pointer
        let unusable = &original as &Struct<`!>;
        println!("Address is {}", unusable as *const Struct as usize);

        // Here we can use only_first and only_second
        //
        // Note: Variable unusable under `! usagetime
        //       never can access to any of its fields

        // Note: Dereferencing any pointer consider as UB and same is
        //       reading or writing into pointer that can be taken
        //       under some usagetime is also UB (obviously)

        let danger = unusable as *const Struct as *mut Struct;
        unsafe {
            // Note: Here we forget usagetime which is UB and can be done with pointers
            (*danger).first = /* ... */;
        }
    }

    // Variant II
    {
        // Here never_third: Vec<&mut Struct<`*, `!second>
        // Since here original is &mut Struct<`second>;
        let mut never_second = vec![&mut original as &mut Struct<`*, `!second>];

        // Here only_second: Vec<&mut Struct<`second>
        // Since here original is &mut Struct<`!>;
        let mut only_second = vec![&mut original as &mut Struct<`second>];

        // Here head can be used as pointer (actually for this variant cast is redundant)
        let unusable = &original as &Struct<`!>;
        println!("Address is {}", unusable as *const Struct as usize);

        // The difference is, never_second values can be taken under `first or `third
    }

    // And, when vec is dropped, all usagetimes ended and partial borrowings returned
    // Totally safely
    original.first = /* ... */;
}
```

So, what can we say here?

1. When some **usagetime** taken, it's excluded from source. So any related **usagetimes** cannot be taken.
2. We can read `` Struct<`*, `!first> `` as "take all usagetimes except first".
3. Negative usagetimes cannot be specified with `mut` and `ref`. Otherwise, we just left `` ref `first `` to indicate, that this was taken and can be taken further only by reference.
4. We can read `` Struct<`*, ref `first> `` as "Take all usagetimes but keep first ref only"
5. We can have only `` &Struct<`!> `` but not `` &mut Struct<`!> `` since empty (never) **usagetime** cannot access anything.
6. When we take `` `! `` usagetime by reference, struct still can be mutable, and other **usagetimes** also can be mutable (again, `` `! `` can't access anything).

**Usagetimes** are working in the same way for traits and trait objects - there is nothing special.

Assume, we have next definitions:

```rust
struct Point1D {
    pub x: f64
}

struct Point2D {
    pub x: f64,
    pub y: f64
}

struct Point3D {
    pub x: f64,
    pub y: f64,
    pub z: f64
}

trait PointXY<`x, `y> {
    fn x_ref(&`x self) -> &f64;
    fn x_mut(&`x mut self) -> &mut f64;

    fn y_ref(&`x self) -> &f64;
    fn y_mut(&`x mut self) -> &mut f64;
}
```

Here, `PointXY` can be implemented only for `Point2D` and `Point3D`.
We also can implement it for struct like:

```rust
struct Point1DWrapped<`x, `y>(`x Point1D, `y f64);
```

Forgetting implementation (will be discussed later), we can also have **usagetimes** for traits and its objects:

```rust
fn main() {
    let mut point_2d = Point2D { x: 0.0, y: 0.0 };
    // Here point_2d_trait: &mut dyn PointXY<`x>
    // Since here point_2d is Point2D<`*, `!x>
    let point_x_mut = &mut point_2d as &mut dyn PointXY<`x>;
    // Here point_y_ref: &dyn PointXY<ref `y>
    // Since here point_2d is Point2D<`*, `!x, ref `y>
    let point_y_ref = &point_2d as &dyn PointXY<`y>;

    *point_x_mut.x_mut() = 10.0;

    println!("Point2D::y == {}", &point_2d.y);
}
```

## Owning

Same as borrowing, if you need, you can own type under specified **usagetimes**.

```rust
fn main() {
    let point_3d = Point3D { x: 0.0, y: 0.0, z: 0.0 };
    // here point_2d: Point3D<`x, `y>
    // Since here point_3d is moved
    let point_2d = point_3d as Point3D<`x, `y>;

    // Here we can do same things as for point_2d in previous example, but we never have access to z field
}
```

No matter which **usagetime** was taken for owning, compiler still can safely drop this.

```rust
impl Drop for Point3D {
    fn drop(&mut self) {}
}
```

Here, as we can see, drop require type to be under full **usagetime** `` `* ``.
So, with this implementation, we will get next error:

```rust
fn main() {
    let point_3d = Point3D { x: 0.0, y: 0.0, z: 0.0 };

    // Compiler knows, that point_2d owns Point3D, so no one can access to this
    // So, we can call drop even when point_2d is not own point_3d fully
    let point_2d = point_3d as Point3D<`x, `y>;

    // error: point_3d was already moved
    //
    // Note: otherwise, we will call drop twice
    let point_2d_other = point_3d as Point3D<`z>;
}
```

## Bounding

UNRESOLVED QUESTION: We actually can have several implementations for **usagetimes** that are bounded to different **usagetimes** of target,
                     but do we actually need this? For now, for simplifying purpose, it's forbidden.

Bounding allows to define **usagetimes** from source to target.
When bounding, they still can't overlaps, except when we allow to do this explicit.

```rust
trait PointXY<`x, `y> {
    fn x_ref(&`x self) -> &f64;
    fn x_mut(&`x mut self) -> &mut f64;

    fn y_ref(&`x self) -> &f64;
    fn y_mut(&`x mut self) -> &mut f64;
}

// error: PointXY provides usagetime `y that can be bounded automatically for Point1D
impl PointXY for Point1D {
    // no matter
}

impl PointXY for Point1DWrapped {
    fn x_ref(&`x self) -> &f64 {
        &self.0.x
    }

    fn x_mut(&`x mut self) -> &mut f64 {
        &mut self.0.x
    }

    fn y_ref(&`x self) -> &f64 {
        &self.1
    }

    fn y_mut(&`x mut self) -> &mut f64 {
        &mut self.1
    }
}
```

Similar for `Point2D` and `Point3D` since they have same **usagetimes** names that can be bound automatically (by name).
For any type and trait with same generics and lifetimes we can have only one implementation regardless **usagetimes**.

But you can use types wrappers for this propose:

```rust
struct PointYZ(T);

impl PointXY for PointYZ<Point3D<`x as `y, `y as `z>> {
    fn x_ref(&`x self) -> &f64 {
        &self.0.y
    }

    fn x_mut(&`x mut self) -> &mut f64 {
        &mut self.0.y
    }

    fn y_ref(&`y self) -> &f64 {
        &self.0.z
    }

    fn y_mut(&`y mut self) -> &mut f64 {
        &mut self.0.z
    }
}
```

By default, auto bounding happens when same **usagetimes** provided in both trait and type.
Same when we use **usagetimes** that provided by scope:

```rust
enum PointVariants<`x, `y> {
    // Self `x bounded to `x of Point1D
    D1(&mut Point1D<`x>),
    // So on ...
    D2(&mut Point2D<`x, `y>),
    D3(&mut Point3D<`x, `y>)
}
```

So we can bound them manually as follows:

```rust
enum PointVariants<`first, `second> {
    // Self `x bounded to `x of Point1D
    D1(&mut Point1D<`first as `x>),
    // So on ...
    D2(&mut Point2D<`first as `x, `second as `y>),
    D3(&mut Point3D<`first as `x, `second as `y>)
}
```

Also, we can provide different **usagetimes** for different used types:

```rust
enum PointVariants<`d1, `d2, `d3> {
    D1(&mut Point1D<`d1 as `x>),
    D2(&mut Point2D<`d2 as `x, `d2 as `y>),
    D3(&mut Point3D<`d3 as `x, `d3 as `y, `d3 as `z>)
}
```

It's totally safe to check which variant we have while some may read or write to data under that variant:

```rust
// PointVariants<`d1, `d2, `d3>
impl PointVariants {
    // first `x searched in PointVariants, then in d1_mut, then in Point1D
    // `x must be bounded to usagetime of function, here is `d1
    pub fn d1_mut_d1(&`d1 mut self) -> Option<&mut Point1D<`x>> {
        match &mut self {
            PointVariants::D1(point) => Some(&mut point),
            PointVariants::D2(_) => None,
            PointVariants::D3(p) => {
                // error: p.x cannot be used, since it under usagetime `d3
                //        but function requires self to be under usagetime `d1
                p.x;
                None
            }
        }
    }

    // `* includes `d1 which bounded to `x of Point1D
    pub fn all_mut_d1(&mut self) -> Option<&mut Point1D<`x>> {}

    // `d3 is not bound to same variant's type, so we can't guarantee,
    // that there is no usage of Point1D<`x>
    pub fn d3_mut_d1(&`d3 mut self) -> Option<&mut Point1D<`x>> {
        // error: PointVariants bound `x of Point1D to `d1 of PointVariants
        //        but self is under `d3 usage
        if let PointVariants::D1(point) = &mut self {
            return Some(&mut point);
        }
        // Only None can be returned, otherwise we are returning temporary value
        None
    }
}
```

The usagetimes were created with traits kept in mind, so here is all simple.

Let refer to previous examples:

```rust
#[derive(Default)]
struct Point2D<`x, `y> {
    x: f64,
    y: f64
}

trait Point<`x, `y, `both: `x + `y> {
    fn x_ref(&`x self) -> &f64;
    fn x_mut(&`x mut self) -> &mut f64;

    fn y_ref(&`y self) -> &f64;
    fn y_mut(&`y mut self) -> &mut f64;

    // Since, Point<`*> contains Point<`both>, we don't need to specify anything actually
    fn distance(&`both self, other: &Self) -> &f64;

    // But we can, with same usagetime (since `both defined for Self)
    fn distance(&`both self, other: &`both Self) -> &f64;
}

// Automatically bounds `x Point to `x Point2D, `y Point to `y Point2D, both doesn't overlapping with `x and `y in Point2D
impl Point for Point2D {
    fn x_ref(&`x self) -> &f64 {
        self.x
    }

    fn x_mut(&`x mut self) -> &mut f64 {
        self.x
    }

    // Same for y
    // Similar for distance
}

// usagetimes doesn't really split traits and other impls

#[derive(Default)]
struct StraitLine<`start, `end> {
    start: `start Point2D,
    end: `end Point2D,
    color: u32
}

trait Line<`start, `end, `both: `start + `end> {
    fn start_ref(&`start self) -> &Point2D;
    fn start_mut(&`start mut self) -> &mut Point2D;

    fn end_ref(&`end self) -> &Point2D;
    fn end_mut(&`end mut self) -> &mut Point2D;

    fn len(&self) -> &f64
}

impl<`start, `end, `both: `start + `end> Line<`start, `end, `both> for StraitLine<`start, `end> {
    // Similar to point
}
```

In `impl` we define some **usagetimes** names. In trait and type we bound names.
So here compiler must be sure, that we don't have overlapping between names.

```rust
// error: usagetime `end defined in impl block bounds Line's `end to StraitLine's `start
//        but actually usagetime `start in impl block bounds Line's `start to StraitLine's `start already
//        that means that `end overlaps with `start in Line through StraitLine when `end bounds to `start as alias
//
//        (Dots showed when other usagetimes exists, I put it here for example)
//        StraitLine<`start, `end as `start, ...> => StartLine<`start as `start, `end as `start, ...>
//                                                             ~~~~~~~~~~~~~~~~  ^^^^^^^^^^^^^^
//        Line<`start, `end, ...> expects that `start and `end never overlapping
//
// consider: replace `end as `start and try other bound
// consider: `end from impl block is defined for Line and StraitLine. Maybe they can be bound?
impl<`start, `end, `both: `start + `end> Line<`start, `end, `both> for StraitLine<`start, `end as `start> {
    // Similar to point
}
```

Wow, error message is definitely difficult to implement, but still possible.
The question is about properly detection.

Let's ask Bob Piller (compiler), what he thinks:

```txt
First I build complex table to ensure, that complex usagetime `both uses `start and `end of Line.
This was defined for "Line" trait

Complex table (all is fine)

| impl (unique) | impl (complex) |
|---------------|----------------|
| start         | both           |
| end           | both           |

Here we pass usagetimes parameters with same names, so there is nothing difficult:

| impl                                     | required                                 |
|------------------------------------------|------------------------------------------|
| Line<`start, `end, `both>                | Line<`start, `end, `both>                |
| Line<`start, `end, `both: `start + `end> | Line<`start, `end, `both: `start + `end> |

As we can see, impl is bounded to Line with complex lifetime with same names

Then while I was resolving bounding Line<`start, `end, `both> to StraitLine<`start, `end>, I found next error

Main table (overlapping)

| Line       | StraitLine |
|------------|------------|
| start      | start      |
| end        | start      |
| both       | (no bound) |

Here, as we can see, `start from Line and `end from Line was both bounded to StraitLine `start.
And since they overlaps for Line.
```

I hope Bob successfully explain, why we have error with this implementation.
When Bob bounds impl to Line, he starts using original names for consistency
(and in case when names are aliased, he put both):

```rust
impl<`alpha, `beta, `theta: `alpha + `beta> Line<`alpha as `start, `beta as `end, `theta as `both> for StraitLine<`alpha as `start, `beta as `start> {
    // Similar to point
}
```

In this case we will get next tables:

```txt
| impl (unique) | impl (complex) |
|---------------|----------------|
| alpha         | theta          |
| beta          | theta          |

| impl                                                                          | required                                 |
|-------------------------------------------------------------------------------|------------------------------------------|
| Line<`start (alpha), `end (beta), `both (theta)>                              | Line<`start, `end, `both>                |
| Line<`start (alpha), `end (beta), `both (theta) `start (alpha) + `end (beta)> | Line<`start, `end, `both: `start + `end> |

Sine, there is no difference, further actions is redundant

| Line       | StraitLine |
|------------|------------|
| start      | start      |
| end        | start      |
| both       | (no bound) |
```

In case when `Line` trait defined as `` Line<`one, `two, `three: `one + `two ``:

```txt
| impl (unique) | impl (complex) |
|---------------|----------------|
| alpha         | theta          |
| beta          | theta          |

| impl                                                                          | required                                 |
|-------------------------------------------------------------------------------|------------------------------------------|
| Line<`one (alpha), `two (beta), `three (theta)>                               | Line<`one, `two, `three>                 |
| Line<`one (alpha), `two (beta), `three (theta): `one (alpha) + `two (beta)>   | Line<`one, `two, `three: `one + `two>    |

| Line       | StraitLine |
|------------|------------|
| one        | start      |
| two        | start      |
| three      | (no bound) |
```

Here we can see, that `` `one `` and `` `two `` overlaps in Line through the StraitLine `` `start ``.

All is fine, when we pass complex usagetime to trait. Trait dictates which usagetimes can be overlapped explicitly,
but some complex usagetimes can be used as simple for trait:

```rust
struct Struct<`one, `two, `three> {}

trait Trait<`first, `second> {}

impl<`one, `two, `first: `one + `two, `second> Trait<`first, `second> for Struct<`one, `two, `second as `three> {}
// Same, bound with as keyword
impl<`first, `second> Trait<`first, `second> for Struct<`first as `one, `first as `two, `second as `three> {}
```

This can be read as `` `first `` requires both `` `one `` and `` `two `` of `Struct` and `` `second `` is the same with `` `three ``.

## Trait object

Usagetimes are designed to dictate which states can be used concurrently safely,
so it's not possible to implement overlapping usagetimes when it's not allowed.

This concept allows to ensure safety on signature level (same as compiler can check borrowings).
Signature correctness doesn't depends on implementation - that's important moment.

For any scope, compiler can ensure that usagetimes are not overlapped. Since this, we can speak about trait safety.

But the also good one thing is that there's no really special requirements or exclusions.

```rust
struct Struct<`first, `second> {}

trait Trait<`first, `second> {}

impl<`first, `second> Trait<`first, `second> for Struct<`first, `second> {}

// Assume, that Struct have some fields and Trait somehow used all of it
// For this example, assume, you are rust compiler, so you already know that implementations are correct

// With usagetimes we can be sure that this signature working with different objects
fn different_objects<`first>(first: &mut dyn Trait<`first>, other: &mut dyn Trait<`first>) {}

// Here we can be sure, that function can accept same trait object
fn maybe_same_objects<`first, `second>(first: &mut Trait<`first>, second: &mut Trait<`second>) {}

// This function just shows which errors will be got when trying to pass same object in "different_objects" function
// As for me, all is obvious, hope for you same too
fn main() {
    // First can be mut boxed, so there is no real difference
    let mut first = Struct {};
    let first = &mut first as &mut dyn Trait;

    let mut second = Struct {};
    let second = &mut second as &mut dyn Trait;

    // Here compiler knows, that trait object can be safely used under `first and `second usagetimes
    maybe_same_objects(first, first);
    // Different objects, obviously fine
    maybe_same_objects(first, other);

    different_objects(first, other);
    // error: first cannot be under `first usagetime twice (for ref all is fine, of course)
    different_objects(first, first);
}
```

## Thread safety for types and trait objects: `Sync + Send` for **usagetimes**

Very advanced extension over **usagetime** system. When I was designed this feature (whole **usagetimes**),
I asked myself: "Can we pass value under different usagetimes some threads?".

Fortunately, the answer is yes. For this we need that any field under usagetime is `T: Send + Sync`.
This makes us sure that there is no `Rc`, `RefCell` or anything else unsafe stuff.

```rust
#[derive(Default)]
struct Struct {
    // Auto usagetimes for first and danger (since they already public)
    pub first: u64,
    pub danger: Rc<RefCell<u64>>
}

// In contrast to ThreadSafeTrait: Sync + Send, we will never pass the trait object itself to other trait
trait ThreadSafeTrait<`_: Sync + Send> {
    fn borrow_ref(&mut self) -> &u64;
    fn borrow_mut(&mut self) -> &mut u64;
}

// Impl takes only `first usagetimes of Struct
// `_ is default usagetime. Here `_ related to ThreadSafeTrait
//
// Since `_ is Sync + Send, `first also must be Sync + Send, and, as we know, u64 is Sync + Send
// Note: pub field used only because is redundant to provide impl for Struct.
//       Since compiler can check `first usagetime fields during this impl and give error if `first is not Send + Sync
impl ThreadSafeTrait for Struct<`_ as `first> {
    fn borrow_ref(&mut self) -> &u64 {
        &self.first
    }

    fn borrow_mut(&mut self) -> &mut u64 {
        &mut self.first
    }
}

fn action(safe: &mut dyn ThreadSafeTrait) {}

// As we can see, we can actually have safe Sync + Send trait object over non safe type as long, as we don't touch unsafe fields
fn main() {
    let mut st = Struct::default();
    // st is Struct<`first> now
    let shared = &mut st.danger;

    // st is Struct<`!> now
    let object = &mut st as &mut dyn ThreadSafeTrait;

    std::thread::scope(|scope| {
        scope.spawn(|| {
            // Moved, safely accessed
            action(object);
        });
    });

    // Totally safe, no one touch this field
    *shared.borrow_mut() += 1;
}
```

# Drawbacks
[drawbacks]: #drawbacks

No matter which partial borrow concept we implement, they all (including this one)
are very complicated and require feature user to be advanced user of The Rust Programming Language.
But as for this feature, **usagetimes** designed to be so simple as possible.

Forgetting about complexity, partial borrowing allows us to achieve C++ performance with safety and without overhead.
That's making this feature an another example of Zero Cost Abstraction.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

This heading was made first, and only after the concept in the first heading was created.
This topic also was inspired by [digama0 comment](https://github.com/rust-lang/rfcs/pull/3426#issuecomment-1527021158)
on [the other issue](https://github.com/rust-lang/rfcs/pull/3426).

Fortunately, his desire coincides with the requirement of this heading to compare existing alternatives,
so let's get started.

## Implemented one: crate [`partial_borrow`](https://docs.rs/partial-borrow/latest/partial_borrow/)

First of all, we can't be sure which features that stands behind this crate.
So with some fix, this solution may not build anymore (or may, who knows).
The example in `docs.rs` is successfully builds now.

But here we face disadvantages:

- Usage with traits are not represented, so we can't use it for advanced cases,
  or it may require huge changes (I can't really check this).
- Quote "The library currently relies on integer/pointer conversions, 'exposing' its pointers ..."
  means that we may have safety troubles in some rare cases.
- Quote "... feature corresponding to CHERI C/C++’s `__attribute__((cheri_no_subobject_bounds))`"
  means that we may need to use feature that may be not suitable for project (i.e. for security reasons),
  but example builds for me (I leave default settings, and didn't checked out flags passed to upstream compiler(s)).
- Quote "However, there are some Rust libraries which use unsafe to extend the Rust language to provide this facility.
  For example, the combination of partial-borrow with replace_with is unsound" means that we may have UB just
  for using several crates, that supposed to be safe.
- Quote "The provided API is supposed to be safe and sound". I like it, I really want to trust this promise (sarcasm)!
- User crate also need install this crate and use all items from it via `use partial_borrow::prelude::*`.

The author went good journey and you already can try this right now,
but I think this is too dangerous and can be used only in target applications (binary),
when we really need this. I want just say thank you, dear rustacean! So I really hope,
that you, dear reader, don't think that I'm against or being rude with author and their implementation,
I'm just pointing on weak places.

## Proposed one: pre rfc [view types](https://github.com/rust-lang/rfcs/issues/3269)

To begin with I highly dislike this interpretation of partial borrowing idea. Even in example,
we somehow shadowing public function `len` with direct access when field is not public.
Do I need to even continue? Author said that view type can alter visibility, but wait.
Does it not violates [rfc 0001](https://github.com/rust-lang/rfcs/blob/master/text/0001-private-fields.md) that,
ironically, comes **before** process of RFC itself!

So, keep going. It just pre RFC and we can change some aspects,
that's not critical as much as I describe here (sorry for this).
Forgetting the syntax, the feature is not so bad, but its only locally defined
(because traits are unresolved question for now), so we can't have really
expressive things and theoretically may have trait objects,
but we can't have partial borrowing over dyn traits itself.

Also, I want notice, that we need to introduce all variations of type to use view types in user code.
Otherwise how compiler may know which fields need to be mutable for implementation and usage (rhetorical question)?
That logically lead us to only two options:

1. Have advanced traits implementations over original and only original type.
2. Have huge amount of implementations over view types and convert them to each other for providing real partial borrowing.

Provided example is too synthetic, and doesn't explain the author's feature concept as good, as it really can be.
I'm rust enjoyer, and I want further enjoy of coding, but this feature turns Rust to Java.

And even if I dislike this RFC (I put dislike there), I think that this can be implemented relatively easy.
But do we want this feature? This question I would like to leave open.

## Proposed one: RFC [permissions](https://github.com/rust-lang/rfcs/pull/3380)

To be honest, for some reasons, I like this interpretation of partial borrowing issue.
But this, as previous concentrated on local usage.

Firstly, this feature doesn't designed to be worked with trait system and trait objects.
Secondly, this feature introduces keyword `permits` and permissions inside `impl` block.
And that just make no sense for me.

Each implementation for specific type may have different permissions, which is good,
but what if we want to cast type with higher permissions to lower?
The RFC doesn't have answer for this question, so I suppose that you just can't do this.

After some, researching and feedbacks, author decided to bound permissions to lifetime system.
That will lead to lifetime hell - first. And make compiler slower exponential times.
I think that is more better and clever to just not mix warm with soft (idiom).

I can't really add some more then I already wrote, so we continue.

## Work in progress: project [disjoint (maybe dead)](https://github.com/nikomatsakis/fields-in-traits-rfc)

**Note**: Link considered to be marked as `maybe dead` since it's not a part of RFC and still WIP,
which is good, but it can't be guaranteed that repo will live long enough (reference to rust lifetimes 😄).

I already have many "to begin with", "firstly" and other, so I just say thank you.
Thanks to [`@nikomatsakis`](https://github.com/nikomatsakis) and other contributors for developing this aside from rust RFC.
You know, that your concept have many unresolved issues and questions, so you just keep thinking
and spending your time for you vision of partial borrowing solution. Thanks for that.

But, in any barrel of honey exist a spoon of tar (idiom). So, for me is was very difficult to find this project.
It has mentions in issue comments but comments amount is too huge and too easy to skip or forget mention.
I hope authors will open an issue with project links. That will help to give attention from the people,
who can really like and help with your concept, but may just not know about it.

The good side of this concept is that the authors aims to resolve partial borrowing for traits.
But the way that they doing this is imho is awful. Let's take a look on examples:

```rust
// Fields are not part of trait signature, so we need to expose them and remember
trait A {
    f: u32,
    g: u32
}

// Same, just field that can't overlaps with other traits
trait B {
    h: u32
}

// Compiler painful life begins here
trait C: A + B {
    i: u32
}
```

At every implementation compiler need to analyze ALL involved structs with their field and their GATs.
Moreover, disjoint doesn't allow to mix partial mutability when it really possible.

Assume, we have `g` and `h` which overlapping, but as ref (obviously safe) or as mut,
but we can provide safe implementation based on compiler rules (or wrap values with `Rc<RefCell<u32>>` to ensure it safety at runtime).

The game ends here. But I want to continue and say as user: if it is difficult for compiler to analyze disjoints,
then for me this is almost impossible. With this feature we will have spam questions about "Why my disjoint breaks here?"
and answers like "You need to wrap your type with shared runtime borrowed types based of single and multi thread usage".

Potentially, I see disjoint trait as bunch of fields with `T: DerefMut<U>` types. So we can allow user to pass some `RefCell`s and other types into.

Also, I didn't find any mentions about partial mutability for structs itself, so I think is kinda useless for this category of features.
But some people mention this in partial borrow context, so I tried to analyze this too.

Of course, maybe I just don't understand something, but I think it only shows how difficult to use disjoint in practice.

With this we coming to the last one "not an RFC" concept, which takes more popularity then all other ones together.

## Discussion: issue [Partial borrowing (for fun and profit)](https://github.com/rust-lang/rfcs/issues/1215)

To begin with, discussion is going around structs and that makes whole future feature useless for partial borrowing.

Let's take a look on the examples:

```rust
// Original
impl Point {
    pub fn x_mut<'a>(&mut self)
        -> &'a mut f64
        where 'a: &mut self.0.0
    {
        &mut self.0.0
    }
}

// Alternative I
impl Point {
    pub fn x_mut(&mut Point { ref mut x, .. }: &mut Self) -> &mut f64 {
        x
    }
}

// receiver
pub fn point_get_x(&Point { ref x, .. }: &Point) -> &f64 {
    x
}

// Alternative II
impl Point {
    pub fn x_mut<'a>(self: &'a mut Self::x) -> &'a mut f64 {
        self.x
    }
}

// Alternative III (from Reddit)
pub fn point_get_x(p: &Point { x }) -> &f64 { &p.x }

// Alternative IV: Regions (overridden for consistent, originally was used Vec type)
struct Point {
    #[region(x)] x: f64
}

impl Point {
    pub fn x_mut(&region(mut x)) -> &mut f64 { self.x }
}

// Alternative V: Lifetime bounds (overridden for consistent, originally was used Vec type)
struct Structure {
   x: f64
}

impl Structure {
   type 'x;

   // Example with returned value doesn't provided, probably, 'x
   fn x_mut<'s:  'x>(&'s mut self) -> &'x_or_s mut f64 {
      self.x
   }
}

// Alternative VI: Composite lifetimes (omitted)
```

To begin with, I don't like the idea of pattern usage for partial borrowing, since this makes code much less readable.
Also, I can't accept concepts that uses lifetimes for partial borrowing.
This is because lifetimes will have several actually independent functionalities.
So, this will makes lifetimes even more complicated for users and makes a lot of situations,
when compiler must spend its time to understand, what passed lifetime means and how it depends on.

I found one commentary with one I totally agree.
The user [`@glaebhoerl`](https://github.com/glaebhoerl)
says [next](https://github.com/rust-lang/rfcs/issues/1215#issuecomment-124541853):

> I don't like the idea of a type signature requiring knowledge of private fields to understand...

I already mention, that partial borrowing must not to expose private fields.
So the only acceptable solution for me, and I hope us, is regions.

But the problem is the same, it's only for local usage, so we can't partial borrow in traits
(since one implementation can be partial borrow, other can be not - so we can't say is trait is safe for partial borrow).
And I think, that partial borrowing is not worth to implement such huge changes for local only usage.

## Conclusion

I've tried to describe all good and bad sides of the alternatives.
I hope, I make it clear why I don't like them and why Rust need to not implement current conceptions.

In contrast to them, **usagetimes** is advanced functionality, that introduce extended abilities to borrowing with clear syntax.
I tried to check how solution behaves itself in different situations. So this RFC requires attention from experienced users and rust devs,
who may also find disadvantages of this concept.

# Prior art
[prior-art]: #prior-art

Partial borrowing for language with borrowing system is unusual,
so any references (excluding mentioned above) is very difficult to found.
And even then they may be not applicable for Rust.

As a prior art I want to write some requirements for future possible implementations of partial borrowing:

1. Feature must not relay on rust lifetime system, since it become much difficult to understand and compile.
   Also, lifetime elision may break lifetime-based partial borrowing.
2. Feature should have implementation for trait or explicitly explain why not.
   In the second case, it also must so simple, as possible; while the first one may worth it to be difficult.
3. Feature must not conflict with current rust syntax, be clear or, at least stand alone.
   The requirement links with next one.
4. Feature must be optional for libraries and required for users when they use such libraries.
   In other way, feature must not affect devs who don't use this feature.

Probably, that's all lights in the fog that authors can use for future partial borrowing conceptions.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- (Community) Default behavior: do **usagetimes** need to be `Sync + Send` by default?
  In that case we need to use `` `name: ?Sync `` (and `Send` when needed) to ensure that we will never pass it to other thread.
- (Community) If we can use traits as **usagetimes** bound, can we have other trait bounds that ensure, that trait access only `T::field + Trait`?
- (Community) Does **usagetimes** need to have visibility (`pub`, `pub(crate)`, ...)? As for now, their visibility is the same as entity.
- (Community) We actually can have several implementations for **usagetimes** that are bounded to different **usagetimes** of target,
  but do we actually need this? For now, for simplifying purpose, it's forbidden.
- (Community) Does partial borrowing really needed in Rust?

Requires usage cases from community, questions and general clarification.
No unresolved questions related to the core of feature (may be found later).

# Future possibilities
[future-possibilities]: #future-possibilities

I think and hope, that this feature may be used for very high amount of optimizations.

## Roadmap

Probably, we may have next roadmap:

1. Give as much examples as we can to cover any unsound (**usagetimes** intended to be sound, but I need more point of view)
2. Stabilize default requirements. I need feedback for this.
3. When I wrote this RFC, I though that this can be relatively simple to implement
   (binding may cause problems, since it may be not clarified enough for reader).
   But at this point, we can defer implementation for best days.

Most relevant teams (list may be not full):

- `T-compiler` - feature requires changes in compiler core
- `T-style` - feature may be not desired since it may require (may not) to be verbose (I think it worth it, IMHO)
- `T-lang` - obviously, feature affect language itself
- `T-types` - affect type system

Most relevant labels (list may not be full):

- `A-borrow-checker` - feature enhance borrow checker

Since, feature affect type system, all closure, generators and many other also affected.
But, I'm not sure should they all be included.

## Dump ideas

Previously, I tried to think about borrowing path, that can be resolved by compiler locally.
To be short, in case when we trying to use some type mutually twice, compiler may take a look inside.

But this is possible only for exact types (not traits) and locally (I think, we can't safely split type with borrowed paths).
So, if you like this idea, you can freely to develop it in your own RFC (why not?).

Also, I was thinking about new struct type, called `manager`.
This type provides lifetimes for their fields, and store them as `&mut T` only.
So, with this feature, we can try to implement partial borrowing based on lifetimes (they don't bound to each other or Self).

But the problem is how we can be sure that we can drop value. And also, it's too locally.

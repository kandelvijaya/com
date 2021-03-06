+++
author = "kandelvijaya"
date = "2017-05-28T18:43:48+02:00"
description = "Occurances of Functor types in Swift and discussion"
tags = ["haskell", "swift3", "functional programming", "fp"]
title = "Functor: Mapping over things!"

+++

# Prelude

Specialization is the key to mastery; functional programming specializes only on function. Thats all there is. The mastery with FP is to reduce complexity, which is what software engineering is all about. Don't get me wrong, Software engineering is also about delivering product not only over engineering. Delivering real world products used to be the stressful part about functional programming. Not anymore. 

Specialization on functions creates a whole new paradigm. In FP, there is no overhead to think of object, assignment, mutation, shared data. You are left to think only how you glue functions and keep gluing them until your product is a function of given input that produces output. Imagine a web server written in FP; it would take path and query as input and return the resource located there. 

In FP, Function is the unit of reasoning. In OOP, we constructed/invented many cleaver hacks and syntax to get around to solve a particular problem. For instance, we have locks, semaphore, mutex to allow one to do concurrent task while managing a shared resource without deadlocking and getting into race condition. This in itself is a whole new area of study, all because we have mutability. Immutability is de facto in Functional Programming. In such, FP programs can make use of multi cores without any special construct as the order of function execution doesn't really matter. 

What is surprising is the amount of new ideas and papers written on how to deal with certain problems in FP is huge compared to what is being done on OOP or imperative programming. For instance, what are all those funky names like Functor, Applicative, Monad, Monoid, etc. My take on this is: what do you do when all you have is a function? Sure enough you examine it extensively, study the properties and try to exploit certain patterns and prove things like Monad can help FP be pure and still do IO. 

> The more constraints one imposes, the more one frees one's self. And the arbitrariness of the constraint serves only to obtain precision of execution. - Igor Stravinsky

FP is of great interest to Academics, Mathematician, Logicians and pragmatic engineer. It however, requires a different attitude and mindset. 

I grew up learning QBasic, ActionScript3, C, Java, PHP. I now work on Obj-C/Swift. These all use the same principle to tackle a problem. The imperative style. For comparison, I picked up Python in 1 day. I had to mull over 1 month to get Scheme into my mind and more recently I spent 2 weeks besides work and finally did `print("hello world")` equivalent in Haskell. All the things we know in this Imperative world is useless on the other side, FP world. 

In such, FP is not just function composition but equally a different mindset to building software. Similar to the quote; 

> If all you have is a hammer, everything looks like nail. 

FP is not a hammer. Its different. We need to tackle problems differently. Read on this awesome paper (The intro section especially) [Paper: Why Functional Programming?](https://www.cs.kent.ac.uk/people/staff/dat/miranda/whyfp90.pdf) to get more insight.

I have been learning Haskell intensely for about a month or more now and surprising things do make sense, lately. A lot. I sometimes even feel enlightened temporarily and bemused a bit later thinking; did I got it completely or my mind just ran out of working memory. Its a fascinating paradigm. I will now attempt to show some cool stuffs that I grasped and implement them with Swift. Before I do that, I highly encourage you to go through this [Blub Paradox Article](http://wiki.c2.com/?BlubParadox). I hope you have read it long time ago, every programmer does. 

>As long as our hypothetical Blub programmer is looking down the power continuum, he knows he's looking down. Languages less powerful than Blub are obviously less powerful, because they're missing some feature he's used to. But when our hypothetical Blub programmer looks in the other direction, up the power continuum, he doesn't realize he's looking up. What he sees are merely weird languages. He probably considers them about equivalent in power to Blub, but with all this other hairy stuff thrown in as well. Blub is good enough for him, because he thinks in Blub.

So lets not be the Blub programmer, shall we. Today, we will jump directly into **Functor**. Trust me, its not that hard. We will even see Functors in Swift and create some more. 

# Intro
**Functor** is a container data type that provides an interface through which client can pass a function which will be applied to the items inside that container to produce a new container data type.

Lets see list: `Array<Int>`, 
- Its a container data type. It can hold bunch of `Int`s.

## Problem
```swift
let naturalNumbers = [1,2,3,4,5]
```

1. Is there a way I can multiply the list of natural numbers by 2 to produce doubled list?
2. Is there a way I can take a list of natural numbers, apply a function and get back a list of string representing natural numbers?

## Solution
Lets do the Imperative way:
```swift
//1.
var output:[Int] = []
for n in naturalNumbers {
    let applied = n * 2
    output.append(applied)
}

//2.
var output: [String] = []
for n in naturalNumbers {
    let applied = String(n)
    output.append(applied)
}
```

What if we wanted to get squares of items in the list? Do we write another imperative code block:

```swift
var output: [Int] = []
for n in naturalNumbers {
    let applied = n * n
    output.append(applied)
}
```

### Criticizing:
1. We have a lot of repeating code, this is minor to the second point.
2. We are mutating output in every pass. What if this variable is declared 100 lines above the looping construct. What if other parts of code mutate this variable while we are trying to apply a function in loop. 
3. We have a repeating code for a repeating pattern. Can we encapsulate what varies?

### Improvising 1
Lets try to encapsulate what varies. The right encapsulation here is a function. Naturally right!

```swift
func something(input: [Int]) -> [Int] {
    var output: [Int] = []
    for n in input {
        let applied = // what do we do here
        output.append(applied)
    }
    return output
}
```

The above code has 3 problems:

1. What do we name the function?
2. If this function only works on `[Int]` then we need to write similar functions for other types we might want to compute.
3. do we know what function will be n applied to before hand?

### Improvising 2
Lets think for a moment on these 3 lines:
```swift
let applied = n * 2
let applied = String(n)
let applied = n * n
```

We are transforming `n` into some other type by applying a function. `*` is a operator/function. `String()` is a constructor function. 

Let's think more  

- The function takes 1 parameter only which is `n` 
- The function returns 1 value of any other type. The other type can be the same one as `n`. 

Can we generalize this three lines as a transform function:
```swift
let applied = transform(n)
```

where transform would be (don't worry about the T and U. They are placeholder types):

```
func transform<T,U>(input: T) -> U

```

Now we know what applied is going to be: its a transform on the current value. Can we get this transform function as input to the function. 

> Sure, swift has closures and higher order function. 

Lets get back to naming this; for starters we will call `map` (Its takes one type and maps to other)

```swift
func map<T, U>(input: [T], transform: ((T) -> U)) -> [U] {
    var output: [U] = []
    for n in input {
        let applied = transform(n)
        output.append(applied)
    }
    return output
}
```

The above function is a map that takes input as List of `T` and a transform parameter of function type where the function takes `T` value and does something to return `U` kind value. We gather all the `U` values and return that. 

Aha now we have a function that encapsulates the mutation. This function is capable of handling any types of list and any types of transform. 1 function to rule the transform. Nice, right. 

Chances are you know this all by heart, this is the standard `map` function on Collection type of Swift. 

Now lets go beyond the normal comfort zone. Do we need `map` or mapping only on List types. Turns out, especially in FP, maps are essential utility to convert one type to another. A lots of types can be mapped. There's another such type in Swift which has `map`; **Optional**

What about Dictionary? They seem to be in need of `map`. Swift has this baked in. 

What is this all for? In fact, we have just created a **functor**. In the example code above, List is a **functor** type.

```swift
Array<T> => container of zero or more `T` typed items
Optional<T> = container of either .none or .some(T) 
Dictionary<K,V> = container of zero or more (K,V) pairs
```

These 3 types do provide a interface to supply transform function. Its called `map`. 

Hence, We already have **functors** in swift but haven't realized we did. To make things easy, a type that can be mapped is a functor. However, for a type to be a functor it has to have these properties. 

# Properties
```
fmap :: (a -> b) -> f a -> f b  //Haskell 
func map(inputFunctor: Functor<T>, transform: (T -> U)) -> Functor<U> //Pseudo Swift
```

Seems like we need a type `Functor` which is polymorphic in `T`. Remember until this point, we have 3 `map` functions on each `Collection` and `Optional` type. There is no rule that specifies that a new user type will be a functor or that type's `map` will abide by the law and do the right job. Its all informal conformance. 

# Laws
Lets talk about the laws. There are only 2 laws for a type to be functor.

1. If a Functor is applied with identity function it should be the same output.
```swift

// Identity function in Haskell
id :: a -> a
id x = x

// The identity function in swift
func id<T>(input: T) -> T {
    return input
}
```

Now the identity law:
```swift
map(inputFunctor: [1,2,3,4], transform: id) == inputFunctor
```

which will result as these:
```swift
var output: [Int] = []
for n in [1,2,3,4] {
    let applied = id(n)  // this `id` just returns what it was passed
    output.append(applied)
}
```

2. Composing two functions and then mapping the resulting function over a Functor should be the same as first mapping one function over the functor and then mapping the other one.

Before we can state and prove this law in swift, we need a function composition operator. 

```swift
infix operator <>

public func <> <T,U, V>(value1: @escaping((U) -> V),  value2: @escaping ((T) -> U)) -> ((T)-> V) {
    return { x in
        return value1(value2(x))
    }
}

```

The above function `<>` takes 2 function and combines into 1; the types should align. 

Now lets see what it means:
```swift
let mult2 = { x in x * 2 }
let mult3 = { x in x * 3 }

let x = map(inputFunctor: [1,2,3], transform: mult2 <> mult3)

let y1 = map(inputFunctor: [1,2,3], transform: mult3)
let y2 = map(inputFunctor: y1, transform: mult2)

x == y2
```

In Haskell this would be:
```
map (f.g) = map f . map g
```
where `f` and `g` are functions and to `.` is equivalent to our `<>` operator.

*Note: Haskell can infer types almost at all times leaving you to denote the type maybe on 1% case when ambiguity occurs. However, swift is verbose and can cause angle bracket blindness when using generics to provide type placeholder*   

# Haskell

Functor definition
```
class Functor f where
  fmap :: (a -> b) -> f a -> f b
```

Notice Haskell uses `fmap` as the name of the method. We used `map` thats all okay. NOTE: **class** creates a type class which is not related anyway to class from OOP. 

Lets create analogous data type to `Optional` from Swift in Haskell and conform it to be Functor. 

```
-- declaration
data Optional a = None | Some a

-- conformance to Functor
instance Functor Optional where
  fmap f None = None
  fmap f (Some a) = Some (f a)
```

# Making true Functor type in Swift
Informal conformance is a weak promise that can be broken without consent. List, Optional , Dictionaries are Functor in Swift informally. There is no type guarantee. Lets make this **Functor** type and tie them once and for all. 

## How can we model Functor?
Well, its a perfect candidate for Protocol. However, we will run into some limitation of Protocols in swift that will make us go the rough path. Modeling Functor with Struct is a bit easy but well, we cant conform List to a Struct! A class would work but but we want a lightweight interface. 

Ideally, I wanted to express Functor like such
```
protocol Functor<A> {
    func fmap(_ by: ((A) -> B)) -> Functor<B>
}
```

which is equivalent to Haskell's declaration of typeclass functor 
```
class Functor f where 
    fmap:: (a -> b) -> f a -> f b
```

In Haskell, like other FP, a function can only take 1 parameter as input and return 1 parameter as output. Multi argument is achieved via function currying. So the above `fmap` is a function that take a function `a -> b` and returns another function where `f a -> f b`. Appreciate the fact that there is no clutter in the declaration site as the Haskell compiler knows how to curry functions and infer types. This makes writing Haskell so intuitive (once you get a hang of it). 

Back to the point, the above Swift Functor declaration won't work because protocol cannot be them-self generic similar to 
```swift
struct Functor<A> {
    // struct can be generic in this way
}
```

To get a better grasp of why this is the limitation and how Haskell **typeclass** is superior to Swift's type system, I recommend reading this [research paper on How to Make Ad-Hoc polymorphism less Ad-hoc](http://homepages.inf.ed.ac.uk/wadler/papers/class/class.ps).

## Improvising
Fortunately, protocol has associated types that can be used in our modeling case.

```
public protocol Functor {
    associatedtype A
    func fmap<F: Functor, B>(_ by: ((A) -> B)) -> F where F.A == B
}
```

The above declaration states 

1. Current Functor is container of `A` types
2. fmap is a function that works on current functor
3. fmap takes a transform function `A -> B` where input is `A`: current functor item type
4. fmap returns a Functor `F` whose `A` is equal to `B` that is emitted by transform function

Its bit verbose and angle bracketed but it does the work.


## Conforming List to Functor
Time to conform Array<Element> to be Functor. 
```swift
extension Array: Functor {
    public typealias A = Element  // 1

    public func fmap<F, B>(_ by: ((Element) -> B)) -> F where F : Functor, B == F.A {
        var accumulator = [B]()
        for index in self {
            accumulator.append(by(index))
        }
        return accumulator as! F   // 2
    }

}
```


Couple of things to keep in mind:

1. Array has a Element associated type which will be the real item model type. We will use that to denote Functor's Item Type. This is analogous to `Array<Element> ~~~ Functor<Element>`
2. note that we down-casted forcefully to produce a Functor. It wont crash as we just conformed list to be Functor. This is just getting around invented type system. 


## Taking Array Functor for a spin
Lets see the Array Functor in action; shall we!
```swift
let inputFunctor = [1,2,3]

// 1. See the implicit type annotation
let composedAppliedFunctor: Array<Int> = inputFunctor.fmap(mult2 <> mult3)

// returns [6, 12, 18] 
```
To see more example and documented source code; please check this [Github Repo: SwiftFunctor](https://github.com/kandelvijaya/SwiftFunctor)

## Conforming Optional To Functor
Optional is a bit tricky to conform to Functor. First lets see the code and we shall see why exactly?
```swift
extension Optional: Functor {

    // 1. 
    public typealias A = Wrapped

    public func fmap<F, B>(_ by: ((Wrapped) -> B)) -> F where F : Functor, B == F.A {
        switch self {
        case .none:
            //2. This will Crash
            return Optional<B>.none as! F
        case .some(let v):
            let fv = by(v)
            //3. This wont crash
            return Optional<B>(fv) as! F
        }
    }

}
```

Things to note:

1. `Optional<Wrapped>~~~~Functor<Wrapped>`
2. `.none` cannot be casted to Functor because `!` operator is defined on Optional such that it unwraps the eventual value from optional. In our case, its `.none` so it will crash. Think of doing this: `let a: Int? = nil` then `a! + 2`.
3. This wont crash because we do have something. 

Hence, **Optional<Wrapped>**  cannot be represented as true Functor. It only works on happy case. This is the power of formal conformance. 

### Can we model Optional somehow to be Functor?
We can by using another wrapper type that is **`Maybe`** type.
```swift
public struct Maybe<T> {
    //1. 
    public let value: Optional<T>

    public init() {
        value = .none
    }

    public init(with: T) {
        value = .some(with)
    }

}
```

Notes:

1. We encapsulate the real optional inside a **Maybe** struct.

Conforming to Functor is pretty similar as:
```swift
extension Maybe: Functor {

    public func fmap<F, B>(_ by: ((T) -> B)) -> F where F : Functor, F.A == B {
        switch self.value {
        case .none:
            //1. Wont crash
            return Maybe<B>() as! F
        case .some(let v):
            //2. 
            let newv = by(v)
            return Maybe<B>(with: newv) as! F
        }
    }
    
}
```

Points to consider:

1. Immediately we see that when we have `.none` value we return a empty `Maybe` type. This now doesn't crash. 

# Conclusion

To summarize:

1. **Functor** is a type that provides `mapping` capability. A type that can be mapped. 
2. Swift has **Collection** and **Optional** types which are Functor. (Optional  is not pure Functor and we saw why.)
3. Functor are useful abstraction that can turn one kind of data into another. 
4. Check out this [Github Repo & Playground: SwiftFunctor](https://github.com/kandelvijaya/SwiftFunctor) for updates and implementation details. 
5. Functional Programming is scrutinizing pure function to produce mathematically provable, robust and compose-able software. FP is on the rise. 
6. Don t get stuck on **Blub Paradox**


# Whats next:
1. Next issue will go deep into what a total pure function is. 
2. What is referential transparency.
3. OOh!!!! This should be pretty awesome.
Lets say a functor has a partially applied function inside it like `Some (*3)`and we have `Some(4)`, can we map this partially applied function over a data type. Yes, we will see **Applicative Functor** next time around. 


# References:
1. SwiftFunctor [Github Repo](https://github.com/kandelvijaya/SwiftFunctor)
2. Why Functional Programming Matters? [Paper](https://www.cs.kent.ac.uk/people/staff/dat/miranda/whyfp90.pdf)
3. Blub Paradox [Article](http://wiki.c2.com/?BlubParadox)
4. Learn you a Haskell for Great Good [Book](http://learnyouahaskell.com)
5. How to make Ad-Hoc polymorphism less ad-hoc [Paper](https://people.csail.mit.edu/dnj/teaching/6898/papers/wadler88.pdf)
6. Why FP matters? [e book](http://book.realworldhaskell.org/read/why-functional-programming-why-haskell.html)

*Happy Coding.*
*Cheers!*

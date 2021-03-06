+++
author = "kandelvijaya"
date = "2017-06-25T18:26:32+02:00"
description = "Implementing a basic monadic type for network result and discussion"
tags = ["monadic computation", "monad", "functional programming", "fp", "swift"]
title = "Monadic Computation in Swift. Why?"

+++

I won't go into the details of what a Monad is and what exactly it solves. What we will do here is try to come up with a monadic type to encapsulate network result. We will do a side by side comparison with the traditional/classical/imperative and the functional (monadic) approach. In the end you should be able to appreciate and hopefully use the monadic style of wrapping network result without being bothered about the nitty gritty details of what a monad is?

*I will definitely do a long exhaustive article on monad in the weeks to come.*

## A Simple Problem
Almost a month ago, I wrote a lengthy article on [implementing Functor in swift](https://kandelvijaya.com/2017/05/28/fp-functor/). I want to write a Xcode Playground code to fetch the content from that link, stringify it, and count the occurrence of word "functor" in the whole article. Pretty basic stuff.

NOTE: to make async calls and get the eventual result, you will need ask playground to keep the execution indefinite.
```swift
import PlaygroundSupport
...
...
PlaygroundPage.current.needsIndefiniteExecution = true  //I prefer to have this at the end of all the code
```

## 1 way [Most basic]
Heres my Network Service Class
```swift
public class NetworkService {

    private let session = URLSession(configuration: .default)
    private var task: URLSessionDataTask?

    private var eventualResult: Data? {
        didSet {
            if let eventualResult = eventualResult {
                onComplete?(eventualResult)
            }
        }
    }

    public var onComplete: ((Data) -> Void)?

    public init(url: URL) {
        task = session.dataTask(with: url) { [weak self] (data, response, error) in
            if let _ = error { return }
            if response == nil || data == nil { return }
            self?.eventualResult = data
        }
        task?.resume()
    }

}
```

And here's the code to solve the problem:
```swift
private let url = URL(string: "https://kandelvijaya.com/2017/05/28/fp-functor/")!
let service1 = NetworkService(url: url)
service1.onComplete = onReceiving

func onReceiving(data: Data) {
    let string = String(data: data, encoding: .utf8) ?? ""
    do {
        let regexp = try NSRegularExpression(pattern: "functor", options: .caseInsensitive)
        let matches = regexp.matches(in: string, options: [], range: NSMakeRange(0, string.count))
        print(matches.count)
    } catch {
        print("Errored! Throw me")
    }

}

```

**Discussion**:

- Let's be honest, the code above does what it needs to.
- A slight problem, what happens if the url returns different resource i.e. image, video or anything else?
- What happens if the network connection dies, the server cannot be reached or server timed out or other 99 problems that we know making a network request has?
- The code has small surface area, but we already know when it can break.
- Errors are part of what we do, shall we ignore them? On the same note;
> "Nothing in the world is more dangerous than sincere ignorance!"

We shall choose not be ignorant.

## 2 Way [Proper Way]
A bit seasoned developer knows error are part of what we do. So he/she treats them fairly.
And he/she knows/heard about a thing called `Result` enum to gather the choice of either `failure(e)` or `success(v)` where `e` and `v` represent either `Error` or `Value`.

#### Let me rephrase this in a bit more depth.

Think: When you make a network or database or async request to get some value `Value`, 2 things could happen eventually.

- You either get that `Value`
- **OR**
- Something goes wrong `Failure`.

Aha, we have 2 choice. Can we represent this choice with a type? Yes, we can.

- What is choice type? ==> Algebraic Sum type
- What is the other type then? ==> Product type
- Sum Type ==> `Enum` (Not all programming language support proper Sum Type; Java 🤓)
- Product Type ==> `Tuple`, `Struct`, `Class`

Fair enough, lets model this property with `enum`.

```swift
enum Result<Value> {
    case failure(Error)
    case success(Value)
}
```

Had we modeled this type with struct like such:
```swift
struct Result<Value> {
    let failure: Error?
    let success: Value?
}
```
The result we will get could potentially have both failure and success values set. What would that mean? Struct is a product type and hence is not suitable for modeling choice.

### Moving on.... 2 Way
With Result type at hand, we can refactor the NetworkService class to come to this code which treats either error or value and stores in `eventualResult` property.

```swift
class NetworkService<T> {

    private enum ResponseError: Error {
        case badOrIncompleteFormat
        case incompatibleEventualValue
    }

    private let session = URLSession(configuration: .default)
    private var task: URLSessionDataTask?

    //1.
    private var eventualResult: Result<T>? {
        didSet {
            if let eventualResult = eventualResult {
                onComplete?(eventualResult)
            }
        }
    }

    var onComplete: ((Result<T>) -> Void)?

    init(url: URL) {
        task = session.dataTask(with: url) { [weak self] (data, response, error) in
            if let error = error {
                self?.eventualResult = .failure(error)
                return
            }
            if response == nil || data == nil {
                self?.eventualResult = .failure(ResponseError.badOrIncompleteFormat)
                return
            }
            //2.
            guard let dataT = data as? T else {
                self?.eventualResult = .failure(ResponseError.incompatibleEventualValue)
                return
            }
            self?.eventualResult = Result<T>.success(dataT)
        }
        task?.resume()
    }

}

```

And then the calling site would look like this:
```swift
private let url = URL(string: "https://kandelvijaya.com/2017/05/28/fp-functor/")!
let service2 = NetworkService<Data>(url: url)
service2.onComplete = onReceiveImperative

func onReceiveImperative(result: Result<Data>) {
    switch result {
    case let .failure(e):
        print(e)
    case let .success(v):
        let dataString = convertDataToString(data: v)
        switch dataString {
        case let .failure(e):
            print(e)
        case let .success(s):
            let functorCount = countFunctorOccurance(on: s)
            print(functorCount)
        }
    }
}

```


**Discussion:**

- `convertDataToString` and `countFunctorOccurance` are utility functions that we have refactored out. This makes the nested switch case obvious.
- With `NetworkService<T>`, we did our best to handle all the errors that could occur and accumulate it on the `eventualResult` which will be sent to the client.
- `onReceiveImperative` is having to deal with unwrapping and passing 2 times as we have to use 2 different functions to complete the problem at hand. If we had 10 functions to solve this problem, we will definitely have 10 nested switch statement. Similar to the Callback hell, Switch hell. This is the only pain point we have at the moment.


The widespread of obvious and unnecessary `switch .. case` is part of why we become ignorant to error handling and propagation when bootstrapping a simple app, which then with passing time, grows to be forgotten in the core of most probably production ready app. This is where the imperative side ends and the functional side thrives.

Say what you will, refactor how you like, until and unless the principles are concrete, this is a recurring problem to solve. This is exactly where we need a new concept to model the obvious stuff and let us write good readable code that we love to. This is where the monadic operation comes into save the day.


## Way 3    [Functional way]

The repeating happens because we have a Computation Value `Result<V>` but the other function requires its input to be just a normal value `V`. We have to unpack the Computation Value, check if we have proper value inside and if so pass it to the function. If we have to compose with 3 similar functions, then we repeat this pattern 3 times nested.  

What if we can encapsulate this unwrap and pass to function in a utility function. This would help eliminate the nested switch levels. Lets try:

```swift
extension Result {

    func unwrapAndPass<B>(to function: ((Value) -> Result<B>)) -> Result<B> {
        switch self {
        case let  .failure(e):
            return .failure(e)
        case let .success(v):
            return function(v)
        }
    }

}

```

So the extension function above does what we want. It unwraps the current result, if there is a result, passes the result to the function. If there is error, it returns the error.

Lets see how this helped the call site:
```swift
func onReceiveFunctional(result: Result<Data>) {
     let count = result.unwrapAndPass(to: convertDataToString).unwrapAndPass(to: countFunctorOccurance)
     print(count)
}
```

Sweet! 👏

Moreover, if you like operatorOverloading then it becomes much more readable and concise. Important: we preserve all the error handling without being blinded by the nested unwrapping and pattern matching.
```swift
func onReceiveFunctional(result: Result<Data>) {
    let count = result >>= convertDataToString >>= countFunctorOccurance
    print(count)
}
```

How cool is that? 👏👏

The operator overloads are defined like this:
```swift
precedencegroup BindPrecedence {
    higherThan: BitwiseShiftPrecedence
    associativity: left
}
infix operator >>=: BindPrecedence

func >>= <T,U>(_ lhs: Result<T>, _ rhsFunc: ((T) -> Result<U>)) -> Result<U> {
    switch lhs {
    case let  .failure(e):
        return .failure(e)
        case let .success(v):
        return rhsFunc(v)
    }
}

```

In fact, what we did is just encapsulated `unwrapAndPass` with the operator `>>=`. Nothing fancy. But clearly more readable!

## what happens if I have 10 different small functions to do the job now.
==> Chain them and be done. No need to unwrap and pass and switch everywhere.
```swift
    let count = result >>= fn1 >>= fn2 >>= fn3 >>= .... >>= fn10  
```


#### Lets step back and see the code for the 2 utility functions now:
```swift
enum TransformingError: Error {
    case conversionError
    case caughtException
}

func convertDataToString(data: Data) -> Result<String> {
    guard let string = String(data: data, encoding: .utf8) else {
        return .failure(TransformingError.conversionError)
    }
    return Result<String>.success(string)
}

func countFunctorOccurance(on stringContent: String) -> Result<Int> {
    do {
        let regexp = try NSRegularExpression(pattern: "functor", options: .caseInsensitive)
        let matches = regexp.matches(in: stringContent, options: [], range: NSMakeRange(0, stringContent.count))
        return .success(matches.count)
    } catch {
        return .failure(TransformingError.caughtException)
    }
}
```

Notice I have intentionally made the both function of type:: `A -> Result<B>`


## Digesting Monadic operation

1. Lets take a look at `Result<V>`. Result represents **Computation that could fail**. 2 choices, either it has proper value or it has error. *Not both.*
![Result Type image](/img/resultType.png)
2. Lets see the functions `convertDataToString` and `countFunctorOccurance`; both have the signature `f:: X -> Result<Y>`. Each function takes a input of `X` and returns final value `Y` with the possible additional effect of failure captured by `Result`.
![Monadic functions a -> M b](/img/monadicFunction.png)
3. Given we have `Result<A>` and a function of type `(A) -> Result<B>`, how do we compose them? Clearly the types don't match up.
![How to compose Monad and monadic function?](/img/howToMonadicCompose.png)
4. We can unwrap the first `Result<A>` and if it has a proper value `v`, we pass `v` to the function of type `(A) -> Result<B>`, if there was a error on the first `Result<A>`, we simply return that error, never calling the other function. i.e. `unwrapAndPass` or fancy `>>=`
![Composing monadically](/img/composingMonadicallyWithUnwrapAndPass.png)

And if you could imagine, the dark area/ path is the failure path, while the dark green is the success area/path. Now one can imagine chaining multiple functions with `>>=` into the 2 tracked area/path/railway. In fact, there is a wonderfully smart talk about the Railway Oriented Programming, check it out if that interests you.
{{< vimeo 113707214 >}}
And this is the [Railway Oriented Programming article](http://fsharpforfunandprofit.com/posts/recipe-part2/).


## One more thing on `Optional`
If you take a look at how optional chaining works, this is exactly railway oriented programming.

For instance:
```swift
a?.callFirstFunction()?.callOnThat()
```
If `a` happens to be `nil` or `.none`, then eventual result will be `nil`. So `?` is very similar to our `unwrapAndPass` or `>>=` operator. Imagine, how many switch case would you have to type if there was no such sequencing or monadic composing on `Optional<Wrapped>`.

## Conclusion:
1. Sum Type or Enums are great at modeling mutually exclusive choice. Sum types are great for pattern matching. `switch .. case` can simulate lots of pattern matching in Swift.
2. Encapsulating async result in `Result<T>` type and providing `unwrapAndPass` or `>>=` enables one to get the goodies of additional effect (error in our case) handling without having to manually clutter the code with nested `switch ... case`.
3. Don't ignore errors, handle them.
4. `unwrapAndPass` or `>>=` is the composition unit in Functional Programming for Monad (I will get deep into what a monad is next time around) and enables one to do railway oriented programming.


## References
1. [Monads for Functional Programming Research Paper](http://homepages.inf.ed.ac.uk/wadler/papers/marktoberdorf/baastad.pdf)
This is a amazing resource, requires Haskell knowledge and is very long and exhaustive. I have digested more than half the content and experimenting, I will write on monads when I'm done with this paper, hopefully.
2. [Railway oriented programming Video](https://vimeo.com/113707214)


### Cheers and enjoy!

# Swift-Combine-Tutorial

## 1. Introductory Basics

### Example with Notification Center's default publisher

```swift
// Getting a publisher from Notification center for test purpose
let notification = Notification.Name("Test")
var notificationPublisher = NotificationCenter.default.publisher(for: notification)

// attaching a subscriber to the publisher
let subscription = notificationPublisher.sink { notification in
    print("Notification found: \(notification.name)")
}

// Posting a notification
NotificationCenter.default.post(name: notification, object: nil)

// Cancel the subscription when you want, it's like disposable in ReactiveSwift, its type is AnyCancellable
subscription.cancel()

// As cancelled you don't get any more notification in the subscriber block/closure
NotificationCenter.default.post(name: notification, object: nil)

//Output:
//
//Notification found: NSNotificationName(_rawValue: Test)
//
```

Note: We don't often need this, it is added as Apple's developer documentation shared this example

### Custom Subscriber Example


```swift
class CustomSubscriber: Subscriber {
    typealias Input = Int
    
    typealias Failure = Never
    
    func receive(_ input: Int) -> Subscribers.Demand {
        print("Received: \(input)")
        // Return how many values you want
        // You can modify (add) Demand here also
        // .none refers that you don't want to add to the initial Demand
        return .none
    }
    
    func receive(completion: Subscribers.Completion<Never>) {
        print("Completed")
    }
    

    func receive(subscription: Subscription) {
        print("Received subscription")
        
        // Must declare initial number of values you want to observe
        // default is none
        subscription.request(.unlimited)
    }
}

let publisher = [1, 2, 3, 4, 5].publisher

publisher.subscribe(CustomSubscriber())

//Output:
//
//Received: 1
//Received: 2
//Received: 3
//Received: 4
//Received: 5
//Completed
 ```

Note: The difference between Custom subscriber and regular .sink is that .sink has by default .unlimited demand. Mostly we need sink

### Passthrough Subject

```swift
class CustomSubscriber: Subscriber {
    typealias Input = Int
    
    typealias Failure = Never
    
    func receive(_ input: Int) -> Subscribers.Demand {
        print("Received: \(input)")
        // Return how many values you want
        // You can modify (add) Demand here also
        // .none refers that you don't want to add to the initial Demand
        return .none
    }
    
    func receive(completion: Subscribers.Completion<Never>) {
        print("Completed")
    }
    

    func receive(subscription: Subscription) {
        print("Received subscription")
        
        // Must declare initial number of values you want to observe
        // default is none
        subscription.request(.unlimited)
    }
}

let passthroughSubject = PassthroughSubject<Int, Never>()

// Works as Publisher
passthroughSubject.subscribe(CustomSubscriber())
 
passthroughSubject.send(1)
passthroughSubject.send(2)

// Also can receive value with default Subscriber i.e. sink

let cancellable = passthroughSubject.sink { value in
    print("Received from Sink: \(value)")
}

passthroughSubject.send(3)

cancellable.cancel()

passthroughSubject.send(4)

//Output:
//
//Received subscription
//Received: 1
//Received: 2
//Received from Sink: 3
//Received: 3
//Received: 4
```

### CurrentValueSubject

```swift
// CurrentValueSubject can just hold a value, its other functionalities are same as PassthroughSubject

let currentValueSubject = CurrentValueSubject<Int, Never>(5)

currentValueSubject.sink { print($0) }

currentValueSubject.send(6)

currentValueSubject.value = 7

//Output:
//5
//6
//7
```

## 2. Basic operators

### Collect() operator examples

```swift
// Without collect() operator

["A", "B", "C", "D", "E"].publisher
    .sink { print($0) }

//Output:
//A
//B
//C
//D
//E


// With collect() operator
 
["A", "B", "C", "D", "E"].publisher
    .collect()
    .sink { collectedValues in
        print("Printing after collecting: \(collectedValues)")
    }

//Output:
//Printing after collecting: ["A", "B", "C", "D", "E"]

// Collect(n) example

["A", "B", "C", "D", "E"].publisher
    .collect(3)
    .sink {
        print("Printing after collecting: \($0)")
    }

//Output:
//Printing after collecting: ["A", "B", "C"]
//Printing after collecting: ["D", "E"]
```

### Map()

```swift
// Map() on publishers
[1, 2, 3, 4, 5].publisher
    .map { $0*$0 } // Making square of the numbers
    .sink { print($0) }

//Output:
//1
//4
//9
//16
//25
```

### ReplaceNil() operator

```swift
[1, 2, nil, 4].publisher
    .replaceNil(with: 0) // Replaces nil with some given default value
    .sink { print($0) }

//Output:
//Optional(1)
//Optional(2)
//Optional(0)
//Optional(4)
```

### Scan() operator

```swift
let publisher = [1, 2, 3, 4, 5].publisher

publisher
    .scan([]) { result, value -> [Int] in
        return result + [value]
    }.sink { print($0) }

//Output:
//[1]
//[1, 2]
//[1, 2, 3]
//[1, 2, 3, 4]
//[1, 2, 3, 4, 5]
 
 // Scan another example
 let subject = PassthroughSubject<[Int], Never>()

 let cancellable = subject
     .scan([]) { result, value in
         return result + [value]
     }
     .sink { print($0) }

 subject.send([1, 2])

 DispatchQueue.main.asyncAfter(deadline: .now() + 1.0) {
     subject.send([5, 6])
 }


 //Output:
 //[[1, 2]]
 //[[1, 2], [5, 6]] // Comes after 1 seconds

 // Note: The speciality of scan() operator is you can get all the previous output and the new output together
```

### removeDuplicates() 

```swift
// removeDuplicates() basically works as skipRepeats
 [1, 1, 2, 3, 1, 4, 5].publisher
     .removeDuplicates()
     .sink { print($0) }
 
 
 Output:
 //1
 //2
 //3
 //1
 //4
 //5
```

### ignoreOutput()

```swift
// ignoreOutput()- ignores outputs, just shows completion (why on earth somebody will need it :p)
[1, 1, 2, 3, 1, 4, 5].publisher
    .ignoreOutput()
    .sink(
        receiveCompletion: { print("Completed: \($0)")},
        receiveValue: { print($0) }
    )
//Output:
//Completed: finished
```

### dropFirst()
```swift
// dropFirst
[1, 1, 2, 3, 1, 4, 5].publisher
    .dropFirst(2)
    .sink { print($0) }

//Output:
//2
//3
//1
//4
//5
 
 // Note: dropWhile is a similar operator drops values when condition meets
```

### drop(untilOutputFrom: Publisher)

```swift
// drop(untilOutputFrom: Publisher)

let taps = PassthroughSubject<Void, Never>()

let passthroughSubject = PassthroughSubject<Int, Never>()

passthroughSubject
    .drop(untilOutputFrom: taps)
    .sink { print($0) }

passthroughSubject.send(1) // will be dropped

taps.send() // now the passthroughSubject values will be sinked

passthroughSubject.send(2)

//Output:
//2
```

### prefix()

```swift
 // prefix()
 
 [1, 1, 2, 3, 1, 4, 5].publisher
     .prefix(3)
     .sink { print($0) }
 
 //Output:
 //1
 //1
 //2
```
### prefix(while: () -> Bool)
```swift
// prefix(while: )
[1, 1, 2, 3, 1, 4, 5].publisher
    .prefix(while: { $0 < 4 }) // it will pass all values until it meets the condition, once condition met, it will not pass any more values
    .sink { print($0) }
//Output:
//1
//1
//2
//3
//1
```

### prepend()
```swift
// prepend(): attaches before the sequence

[1, 1, 2, 3, 1, 4, 5]
    .publisher
    .prepend([10, 11])
    .collect() // just to avoid new lines in the output :p
    .sink { print($0) }

//Output:
//[10, 11, 1, 1, 2, 3, 1, 4, 5]
 
 // Note: Similarly append() operator attaches elements at the end
```

### switchToLatest()

```swift
// switchToLatest()
 
 let publisher1 = PassthroughSubject<Int, Never>()
 let publisher2 = PassthroughSubject<Int, Never>()
 
 let publishers = PassthroughSubject<PassthroughSubject<Int, Never>, Never>()
 
 publishers
     .switchToLatest() // Will always sink the vlaues of the latest PassthroughSubject hooked to it
     .sink { print($0) }
 
 publishers.send(publisher1) // Hooked with publisher1
 
 publisher1.send(1) // will be sinked
 
 publishers.send(publisher2) // Now hooked with publisher2
 
 publisher2.send(2) // Will be sinked
 
 publisher1.send(3) // Will be ignored as the `publishers` is now hooked with publisher2
 
 //Output:
 //1
 //2
```
### switchToLatest example with Future<>

```swift
var index = 0
func getImageID() -> AnyPublisher<Int, Never> {
    Future<Int, Never> { result in // Future is used for async result production
        // Let's say our task takes 3 second to complete
        DispatchQueue.main.asyncAfter(deadline: .now() + 3.0) {
            result(.success(index))
        }
    }
    .map { $0 } // Receives the result from Future and makes Publisher
    .eraseToAnyPublisher() // Converts to AnyPublisher
}

let taps = PassthroughSubject<Void, Never>()

let cancellable = taps
    .map { _ in getImageID() } // In each tap we request imageID
    .switchToLatest() // We only care about latest request result
    .sink { print("Recieved imageID: \($0)") }

taps.send() // Immediately gets 1 after 3 sec
// requests again in 4 sec, i.e. 1 sec after the first result
DispatchQueue.main.asyncAfter(deadline: .now() + 4.0) {
    index += 1
    taps.send()
}

// requests again in 4.5 sec. i.e. 0.5 sec after the second request
// So the second request is ignored in switchToLatest
DispatchQueue.main.asyncAfter(deadline: .now() + 4.5) {
    index += 1
    taps.send()
}

//Output:
//Recieved imageID: 0
//Recieved imageID: 2
```

### merge()
```swift
let publisher1 = PassthroughSubject<Int, Never>()
let publisher2 = PassthroughSubject<Int, Never>()

publisher1.merge(with: publisher2).sink { print($0) }

publisher1.send(1)
publisher2.send(2)

//Output:
//1
//2
```

### allSatisfy
```swift
[1, 2, 3, 4, 5].publisher
    .allSatisfy { $0 % 2 == 0 }
    .sink { isAllEven in
        print(isAllEven)
    }

// Output:
// false

```

 ### handleEvents and print operators for debugging

 ```swift
 let cancellable = [1, 2, 3, 4, 5].publisher
    .print()
    .sink { _ in } // Not printing anything from sink()
 
 //Output:
 //receive value: (1)
 //receive value: (2)
 //receive value: (3)
 //receive value: (4)
 //receive value: (5)
 //receive finished
 
 ```

```swift
 let cancellable = [1, 2, 3, 4, 5].publisher
     .handleEvents(
         receiveSubscription: { _ in print("Received subscription") },
         receiveOutput: { _ in print("Recieved output") }
     ).sink { _ in }
 
 // There are many other event handlers also in a publisher
 
 cancellable.cancel()
 
 // Output:
 // Received subscription
 // Recieved output
 // Recieved output
 // Recieved output
 // Recieved output
 // Recieved output
 
 ```

### breakpoint operator for debugging
```swift
[1, 2, 3, 4, 5].publisher
    .breakpoint(receiveOutput: { $0 > 4 })
    .sink { print($0) }

// A breakpoint will be started when any condition given is satisfied
```

### debounce
waits specified time AFTER getting an event and publishes the latest value when the waiting period completes. Check detail example: https://developer.apple.com/documentation/combine/fail/debounce(for:scheduler:options:)

### throttle
takes the first event. Then waits specified period and takes only the latest value. Detail: https://developer.apple.com/documentation/combine/fail/throttle(for:scheduler:latest:)

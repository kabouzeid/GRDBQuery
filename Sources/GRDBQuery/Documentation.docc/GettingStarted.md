# Getting Started

A step-by-step guide for using `@Query` in your SwiftUI application. 

## The SwiftUI Environment

**To use `@Query`, first define a new environment key that grants access to the database.**

In the example below, we define a new `dbQueue` environment key whose value is a GRDB [DatabaseQueue]. Some other apps, like the [GRDB demo apps], can choose another name and another type, such as a "database manager" that encapsulates database accesses.

*You are free to choose any type you want*, as long as it is possible to create Combine publishers out of it, the publishers of database values that will feed the SwiftUI views.

The [EnvironmentKey documentation](https://developer.apple.com/documentation/swiftui/environmentkey) describes the procedure:

```swift
import GRDB
import SwiftUI

private struct DatabaseQueueKey: EnvironmentKey {
    /// The default dbQueue is an empty in-memory database
    static var defaultValue: DatabaseQueue { DatabaseQueue() }
}

extension EnvironmentValues {
    var dbQueue: DatabaseQueue {
        get { self[DatabaseQueueKey.self] }
        set { self[DatabaseQueueKey.self] = newValue }
    }
}
```

You will substitute the default empty database with an actual database on disk for your main application:

```swift
import SwiftUI

@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            MyView().environment(\.dbQueue, /* some DatabaseQueue on disk */)
        }
    }
}
```

You will feed SwiftUI previews with databases that you want to preview:

```swift
struct PlayerList_Previews: PreviewProvider {
    static var previews: some View {
        // Empty list
        PlayerList().environment(\.dbQueue, /* empty table of players */)
        
        // Non-empty list
        PlayerList().environment(\.dbQueue, /* non-empty table of players */)
    }
}
```

See the [GRDB demo apps] for examples of such setups.

## Define a Queryable type

**Next, define a ``Queryable`` type for each database request you want to observe.**

For example:

```swift
import Combine
import GRDB
import GRDBQuery

/// Tracks the full list of players
struct PlayerRequest: Queryable {
    static var defaultValue: [Player] { [] }
    
    func publisher(in dbQueue: DatabaseQueue) -> AnyPublisher<[Player], Error> {
        ValueObservation
            .tracking { db in try Player.fetchAll(db) }
            // The `.immediate` scheduling feeds the view right on subscription,
            // and avoids an initial rendering with an empty list:
            .publisher(in: dbQueue, scheduling: .immediate)
            .eraseToAnyPublisher()
    }
}
```

The ``Queryable`` protocol has two requirements: a default value, and a Combine publisher. The publisher is built from the `DatabaseQueue` stored in the environment (you'll adapt this sample code if you prefer another type). The publisher tracks database changes with GRDB [ValueObservation]. The default value is used until the publisher publishes its initial value.

> Note: In the above sample code, we make sure the views are *immediately* fed with database content with the `scheduling: .immediate` option. This prevents any blank state, "flash of missing content", or unwanted initial animation.
>
> The `scheduling: .immediate` option should be removed for database accesses that are too slow, because the user interface would be blocked until the database values are fetched.
>
> If you remove `scheduling: .immediate`, views are initially fed with the default value, and the database content is notified later, when it becomes available. You can for example use a `nil` default value that your view can detect in order to display some waiting indicator, or a [redacted](https://developer.apple.com/documentation/swiftui/view/redacted(reason:)) placeholder.

## Feed a SwiftUI View

**Finally**, you can define a SwiftUI view that automatically updates its content when the database changes:

```swift
import GRDBQuery
import SwiftUI

struct PlayerList: View {
    @Query(PlayerRequest(), in: \.dbQueue)
    var players: [Player]
    
    var body: some View {
        List(players) { player in
            HStack {
                Text(player.name)
                Spacer()
                Text("\(player.score) points")
            }
        }
    }
}
```

> Tip: Some applications want to use `@Query` without specifying the key path to the database in each and every view.
>
> To do so, add somewhere in your application those convenience `Query` initializers:
>
> ```swift
> // Convenience Query initializers for requests
> // that feed from `DatabaseQueue`.
> extension Query where Request.DatabaseContext == DatabaseQueue {
>     init(_ request: Request) {
>         self.init(request, in: \.dbQueue)
>     }
>     init(_ request: Binding<Request>) {
>         self.init(request, in: \.dbQueue)
>     }
>     init(constant request: Request) {
>         self.init(constant:request, in: \.dbQueue)
>     }
> }
> ```
>
> These initializers will streamline your SwiftUI views:
>
> ```swift
> struct PlayerList: View {
>     @Query(PlayerRequest()) // Implicit key path to the database
>     var players: [Player]
>
>     ...
> }
> ```

## Interrupting Automatic Updates 

By default, `@Query` automatically updates the database values for the whole duration of the presence of the view in the SwiftUI engine.

You can spare resources by stopping auto-updates when the view disappears, and restarting them when the view appears. To do so, use the `$players.isAutoupdating` SwiftUI binding, as in the example below:

```swift
import GRDBQuery
import SwiftUI

struct PlayerList: View {
    @Query(PlayerRequest(), in: \.dbQueue)
    var players: [Player]
    
    var body: some View {
        List(players) { player in
            HStack {
                Text(player.name)
                Spacer()
                Text("\(player.score) points")
            }
        }
        // Stop observing the database when the view disappears,
        // and start again when the view appears.
        .mirrorAppearanceState(to: $players.isAutoupdating)
    }
}
```

## How to Handle Database Errors?

**By default, `@Query` ignores errors** published by `Queryable` types. The SwiftUI views are just not updated whenever an error occurs. If the database is unavailable when the view appears, `@Query` will just output the default value.

You can restore error handling by publishing a standard `Result`, as in the example below: 

```swift
import Combine
import GRDB
import GRDBQuery

struct PlayerRequest: Queryable {
    typealias Value = Result<[Player], Error>
    static var defaultValue: Value { .success([]) }
    
    func publisher(in dbQueue: DatabaseQueue) -> AnyPublisher<Value, Never> {
        ValueObservation
            .tracking { db in try Player.fetchAll(db) }
            .publisher(in: dbQueue, scheduling: .immediate)
            .map { players in .success(players) }
            .catch { error in Just(.failure(error)) }
            .eraseToAnyPublisher()
    }
}
```

## Topics

### The @Query property wrapper

- ``Query``

### Feeding @Query with database content

- ``Queryable``

[DatabaseQueue]: https://github.com/groue/GRDB.swift/blob/master/README.md#database-queues
[ValueObservation]: https://github.com/groue/GRDB.swift/blob/master/README.md#valueobservation
[GRDB demo apps]: https://github.com/groue/GRDB.swift/tree/master/Documentation/DemoApps

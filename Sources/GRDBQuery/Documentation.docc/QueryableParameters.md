# Adding Parameters to Queryable Types

Learn how SwiftUI views can configure the database content displayed on screen.

## Overview

When a SwiftUI view needs to configure the database values displayed on screen, it will modify the ``Queryable`` request that feeds the `@Query` property wrapper.

Such configuration can be performed by the view that declares a `@Query` property. It can also be performed by the enclosing view. This article explores all your available options.

## A Configurable Queryable Type

As an example, let's extend the `PlayerRequest` request type we have seen in <doc:GettingStarted>. It can now sort players by score, or by name, depending on its `ordering` property.

```swift
struct PlayerRequest: Queryable {
    enum Ordering {
        case byScore
        case byName
    }

    /// How players are sorted.
    var ordering: Ordering

    static var defaultValue: [Player] { [] }

    func publisher(in dbQueue: DatabaseQueue) -> AnyPublisher<[Player], Error> {
        ValueObservation
            .tracking { db in try fetchValue(db) }
            .publisher(in: dbQueue, scheduling: .immediate)
            .eraseToAnyPublisher()
    }

    private func fetchValue(_ db: Database) throws -> [Player] {
        switch ordering {
        case .byScore:
            return try Player
                .order(Column("score").desc)
                .fetchAll(db)
        case .byName:
            return try Player
                .order(Column("name"))
                .fetchAll(db)
        }
    }
}
```

The `@Query` property wrapper will detect changes in the `ordering` property, and update SwiftUI views accordingly.

> Experiment: You can adapt this example for your own needs. As you can see, you can modify the order to database values, but you can also change how they are filtered. All [GRDB] features are available. 

## Modifying the Request from the SwiftUI View

SwiftUI views can change the properties of the Queryable request with the SwiftUI bindings provided by the `@Query` property wrapper:

```swift
import GRDBQuery
import SwiftUI

struct PlayerList: View {
    // Ordering can change through the $players.ordering binding.
    @Query(PlayerRequest(ordering: .byScore))
    var players: [Player]

    var body: some View {
        List(players) { player in
            HStack {
                Text(player.name)
                Spacer()
                Text("\(player.score) points")
            }
        }
        .toolbar {
            ToolbarItem(placement: .navigationBarTrailing) {
                ToggleOrderingButton(ordering: $players.ordering)
            }
        }
    }
}

struct ToggleOrderingButton: View {
    @Binding var ordering: PlayerRequest.Ordering

    var body: some View {
        switch ordering {
        case .byName:
            Button("By Score") { ordering = .byScore }
        case .byScore:
            Button("By Name") { ordering = .byName }
        }
    }
}
```

In the above example, `$players.ordering` is a SwiftUI binding to the `ordering` property of the `PlayerRequest` request.

This binding feeds `ToggleOrderingButton`, which lets the user change the ordering of the request. `@Query` then redraws the view with an updated database content. 

When appropriate, you can also use `$players.request`, a SwiftUI binding to the `PlayerRequest` request itself.


## Configuring the Initial Request

The above example has the `PlayerList` view always start with the `.byScore` ordering.

When you want to provide the initial request as a parameter to your view, provide a dedicated initializer:

```swift
struct PlayerList: View {
    /// No default request
    @Query<PlayerRequest>
    var players: [Player]

    /// Explicit initial request
    init(initialOrdering: PlayerRequest.Ordering) {
        _players = Query(PlayerRequest(ordering: initialOrdering))
    }

    var body: some View { ... }
}
```

Defining a default ordering is still possible:

```swift
struct PlayerList: View {
    /// Defines the default initial request (ordered by score)
    @Query(PlayerRequest(ordering: .byScore))
    var players: [Player]

    /// Default initial request (by score)
    init() { }
    
    /// Explicit initial request
    init(initialOrdering ordering: PlayerRequest.Ordering) {
        _players = Query(PlayerRequest(ordering: ordering))
    }

    var body: some View { ... }
}
```

> IMPORTANT: The initial request is only used when `PlayerList` appears on screen. After that, and until `PlayerList` disappears, the request is only controlled by the `$players` bindings described above.
>
> This means that calling the `PlayerList(initialOrdering:)` with a different ordering will have no effect:
> 
> ```swift
> struct Container {
>     @State var ordering = PlayerRequest.Ordering.byScore
> 
>     var body: some View {
>         // No effect when the ordering State changes after the `PlayerList`
>         // has appeared on screen:
>         PlayerList(initialOrdering: ordering)
>     }
> }
> ```
>
> To let the enclosing view control the request after `PlayerList` has appeared on screen, you'll need one of the techniques described below.  

## Initializing @Query from a Request Binding

The `@Query` property wrapper can be controlled with a SwiftUI binding, as in the example below:

```swift
struct Container {
    @State var ordering = PlayerRequest.Ordering.byScore

    var body: some View {
        PlayerList(ordering: $ordering) // Note the `$ordering` binding here
    }
}

struct PlayerList: View {
    @Query<PlayerRequest>
    var players: [Player]

    init(ordering: Binding<PlayerRequest.Ordering>) {
        _players = Query(Binding(
            get: { PlayerRequest(ordering: ordering.wrappedValue) },
            set: { request in ordering.wrappedValue = request.ordering }))
    }

    var body: some View { ... }
}
```

With such a setup, `@Query` updates the database content whenever a change is performed by the `$ordering` Container binding, or the `$players` PlayerList bindings.

This is the classic two-way connection enabled by SwiftUI `Binding`.

## Initializing @Query from a Constant Request

Finally, the ``Query/init(constant:in:)`` initializer allows the enclosing Container view to control the request without restriction, and without any SwiftUI Binding. However, `$players` binding have no effect: 

```swift
struct Container {
    var ordering: PlayerRequest.Ordering

    var body: some View {
        PlayerList(constantOrdering: ordering)
    }
}

struct PlayerList: View {
    @Query<PlayerRequest>
    var players: [Player]

    init(constantOrdering ordering: PlayerRequest.Ordering) {
        _players = Query(constant: PlayerRequest(ordering: ordering))
    }

    var body: some View { ... }
}
```

## Summary

**All the ``Query`` initializers we have seen above can be used in any given SwiftUI view.**

```swift
struct PlayerList: View {
    /// Defines the default initial request (ordered by score)
    @Query(PlayerRequest(ordering: .byScore))
    var players: [Player]

    /// Default initial request (by score)
    init() { }
    
    /// Initial request
    init(initialOrdering ordering: PlayerRequest.Ordering) {
        _players = Query(PlayerRequest(ordering: ordering))
    }

    /// Request binding
    init(ordering: Binding<PlayerRequest.Ordering>) {
        _players = Query(Binding(
            get: { PlayerRequest(ordering: ordering.wrappedValue) },
            set: { request in ordering.wrappedValue = request.ordering }))
    }

    /// Constant request
    init(constantOrdering ordering: PlayerRequest.Ordering) {
        _players = Query(constant: PlayerRequest(ordering: ordering))
    }

    var body: some View { ... }
}
```

[GRDB]: https://github.com/groue/GRDB.swift

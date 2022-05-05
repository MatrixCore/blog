---
title: "Dev Blog 01"
date: 2022-05-04T13:03:19+02:00
tags: ["dev", "database"]
author: Finn Behrens
showToc: true
ShowPostNavLinks: true
ShowRssButtonInSectionTermList: true

---

After writing some code for [MatrixCore](https://github.com/MatrixCore/MatrixCore),
[MatrixSQLiteStore](https://github.com/MatrixCore/MatrixSQLiteStore) and
[nio/matrixcore-sqlite](https://github.com/niochat/nio/tree/matrixcore-sqlite),
I decided I should start a blog.

## Deciding on a Database
### Rewriting the database code 1, 2, or 3 times

I started the MatrixCore database code based on Apples [Core Data framework](https://developer.apple.com/documentation/coredata). This did
integrate nicely into `SwiftUI` as it provides a special `viewContext`, which is designed to be used
for UI requests. But the class hierarchies CoreData uses is not that good suited to store chat messages.
Also, many people did say that CoreData does not scale that well with many entries.

This got me to start using [GRDB.swift](https://github.com/groue/GRDB.swift) which worked quite well,
but had no `viewContext`. Therefore I created a `NioAccountStore` which holds a reference to all accounts,
which also makes the creation of the `MatrixCore` client much easier, and so can hold a `TaskHandle` for
the sync background task.

After I decided to use `GRDB.swift` I started implementing the `/sync` task, starting with state events,
so that the client has a general idea of known rooms to store messages under so it can show them.

This worked quite well, but after a short time I noticed that I did not sort my messages and that the
SQLite `rowid` is not a stable identifier, and also I can't insert older messages into the store, which is
required for example for the bridge import feature.
This got me to rethink my database layout again, and search for a way to store the messages in order, but still
leave the option to insert messages at earlier times into the store.

### Swift's `Decoder` protocol and `GRDB`
The matrix protocol has quite an amount of different message and state events, and also is extensible, which
makes designing the database to exactly hold that quite hard. My idea to go around this was to save the raw content json
into the database for example as JSON or plist to save columns.

Thankfully GRDB has a feature built into the library that if an attribute of the struct is mapped to the table
is `Codable` it will automatically be encoded into JSON and stored as text in the database.

But for the `MatrixStateEventType` protocol, which is used to store the State Events, `Decodable` can not be
implemented, as for this the event type is required, which is not part of the content object, but part of the enclosing
JSON. This type is also required to index the database, so I can search for an `m.room.create` Event to get the room
metadata, without scanning all events in the room.

The way to implement this is to haveto have an init function with the arguments decoder and type, but the Decodable protocol
only takes a `Decoder`.
So a way to get the Decoder for the specific column is required. This is possible with the
[`superDecoder(forKey:)`](https://developer.apple.com/documentation/swift/keyeddecodingcontainer/2893648-superdecoder)
function on the `KeyedDecodingContainer`, but GRDB does not update the `CodingPath` so that the resulting Decoder
only has the elements already decoded by the decoder decoding the struct holding the content, and not the subkey/column. [^1]

[^1]: [GRDB.swift#1210](https://github.com/groue/GRDB.swift/issues/1210)

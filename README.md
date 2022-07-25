# Persistent History Tracking Kit

Helps you easily handle Core Data's Persistent History Tracking

![](https://img.shields.io/badge/Platform%20Compatibility-iOS%20|%20macOS%20|%20tvOS%20|%20watchOs-red) ![](https://img.shields.io/badge/Swift%20Compatibility-5.5-red)

[中文版说明](https://github.com/fatbobman/PersistentHistoryTrackingKit/blob/main/READMECN.md)

## What's This？

> Use persistent history tracking to determine what changes have occurred in the store since the enabling of persistent history tracking.  —— Apple Documentation

When Persistent History Tracking is enabled, your application will begin creating transactions for any changes that occur in Core Data Storage. Whether they come from application extensions, background contexts, or the main application.

Each target of your application can fetch the transactions that have occurred since a given date and merge them into the local storage. This way, you can keep up to date with changes made by other persistent storage coordinators and keep your storage up to date. After merging all transactions, you can update the merge date so that the next time you merge, you will only get the new transactions that have not yet been processed.

The **Persistent History Tracking Kit** will automate the above process for you.

## How does persistent history tracking work?

Upon receiving a remote notification of Persistent History Tracking from Core Data, Persistent History Tracking Kit will do the following:

* Query the current author's (current author) last merge transaction time
* Get new transactions created by other applications, application extensions, background contexts, etc. (all authors) in addition to this application since the date of the last merged transaction
* Merge the new transaction into the specified context (usually the current application's view context)
* Update the current application's merge transaction time
* Clean up transactions that have been merged by all applications

For more specific details on how this works, read [在 CoreData 中使用持久化历史跟踪](https://fatbobman.com/posts/persistentHistoryTracking/) or [Persistent History Tracking in Core Data ](https://www.avanderlee.com/swift/persistent-history-tracking-core-data/).

## Usage

```swift
// in Core Data Stack
import PersistentHistoryTrackingKit

init() {
    container = NSPersistentContainer(name: "PersistentTrackBlog")
    // Prepare your Container
    let desc = container.persistentStoreDescriptions.first!
    // Turn on persistent history tracking in persistentStoreDescriptions
    desc.setOption(true as NSNumber,
                   forKey: NSPersistentHistoryTrackingKey)
    desc.setOption(true as NSNumber,
                   forKey: NSPersistentStoreRemoteChangeNotificationPostOptionKey)
    container.loadPersistentStores(completionHandler: { _, _ in })

    container.viewContext.transactionAuthor = "app1"
    // after loadPersistentStores
    let kit = PersistentHistoryTrackingKit(
        container: container,
        currentAuthor: "app1",
        allAuthors: "app1,app2,app3",
        userDefaults: userDefaults,
        logLevel: 3,
    )
}
```

## Parameters

### currentAuthor

The name of the author of the current application. The name is usually the same as the transaction name of the view context

```swift
container.viewContext.transactionAuthor = "app1"
```

### allAuthors

The author name of all members managed by the Persistent History Tracking Kit.

Persistent History Tracking Kit should only be used to manage transactions generated by developer-created applications, application extensions, and backend contexts; other system-generated transactions (e.g. Core Data with CloudKit) are handled by the system itself.

For example, if your application author name is: "appAuthor" and your application extension author name is: "extensionAuthor", then.

```swift
allAuthors: ["appAuthor", "extensionAuthor"],
```

For transactions generated in the backend context, the backend context should also have a separate author name if it is not set to auto-merge.

```swift
allAuthors: ["appAuthor", "extensionAuthor", "appBatchAuthor"],
```

### includingCloudKitMirroring

Whether or not to merge network data imported by Core Data with CloudKit, is only used in scenarios where the Core Data cloud sync state needs to be switched in real time. See [Toggling Core Data's cloud sync state in real time](https://www.fatbobman.com/posts/real-time-switching-of-cloud-syncs-status/) for details on usage

### batchAuthors 

Some authors (such as background contexts for batch changes) only create transactions and do not merge and clean up transactions generated by other authors. You can speed up the cleanup of such transactions by setting them in batchAuthors.

```swift
batchAuthors: ["appBatchAuthor"],
```


Even if not set, these transactions will be automatically cleared after reaching maximumDuration.

### maximumDuration

Normally, transactions are only cleaned up after they have been merged by all authors. However, in some cases, individual authors may not run for a long time or may not be implemented yet, causing transactions to remain in SQLite. In the long run, this can cause a performance degradation of the database.

By setting maximumDuration, Persistent History Tracking Kit will force the removal of transactions that have reached the set duration. The default setting is 7 days.

```swift
maximumDuration: 60 * 60 * 24 * 7,
```

Performing cleanup on transactions does not harm the application's data.

### contexts

The context used for merging transactions, usually the application's view context. By default, it is automatically set to the container's view context.

```swift
contexts: [viewContext],
```

### userDefaults

If an App Group is used, use the UserDefaults available for the group.

```swift
let appGroupUserDefaults = UserDefaults(suiteName: "group.com.yourGroup")!

userDefaults: appGroupUserDefaults,
```

### cleanStrategy

Persistent History Tracking Kit currently supports three transaction cleanup strategies:

* none

  Merge only, no cleanup

* byDuration

  Set a minimum time interval between cleanups

* byNotification

  Set the minimum number of notifications between cleanups

```swift
// Each notification is cleaned up
cleanStrategy: .byNotification(times: 1),
// At least 60 seconds between cleanings
cleanStrategy: .byDuration(seconds: 60),
```

When the cleanup policy is set to none, cleanup can be performed at the right time by generating separate cleanup instances.

```swift
let kit = PersistentHistoryTrackingKit(
    container: container,
    currentAuthor: "app1",
    allAuthors: "app1,app2,app3",
    userDefaults: userDefaults,
    cleanStrategy: .byNotification(times: 1),
    logLevel: 3,
    autoStart: false
)
let cleaner = kit.cleanerBuilder()

// Execute cleaner at the right time, for example when the application enters the background
clear()
```

### uniqueString

The string prefix for the timestamp in UserDefaults.

### logger

The Persistent History Tracking Kit provides default logging output. To export Persistent History Tracking Kit information through the logging system you are using, simply make your logging code conform to the PersistentHistoryTrackingKitLoggerProtocol.

```swift
public protocol PersistentHistoryTrackingKitLoggerProtocol {
    func log(type: PersistentHistoryTrackingKitLogType, message: String)
}

struct MyLogger: PersistentHistoryTrackingKitLoggerProtocol {
    func log(type: PersistentHistoryTrackingKitLogType, message: String) {
        print("[\(type.rawValue.uppercased())] : message")
    }
}

logger:MyLogger(),
```

### logLevel

The output of log messages can be controlled by setting logLevel:

* 0 Turn off log output
* 1 Important status only
* 2 Detail information

### autoStart

Whether to start the Persistent History Tracking Kit instance as soon as it is created.

During the execution of the application, the running state can be changed by start() or stop().

```swift
kit.start()
kit.stop()
```

## Requirements

    .iOS(.v13),
    
    .macOS(.v10_15),
    
    .macCatalyst(.v13),
    
    .tvOS(.v13),
    
    .watchOS(.v6)

## Install

```swift
dependencies: [
  .package(url: "https://github.com/fatbobman/PersistentHistoryTrackingKit.git", from: "1.0.0")
]
```

## License

This library is released under the MIT license. See [LICENSE](https://github.com/fatbobman/persistentHistoryTrackingKit/blob/main/LICENSE) for details.

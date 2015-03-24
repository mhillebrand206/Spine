Spine
=====
Spine is an Swift framework for consuming a JSON API that adheres to the jsonapi.org spec.

Installation
============
Spine uses [Carthage](https://github.com/Carthage/Carthage). Add `github "wvteijlingen/Spine"` to your Cartfile. See the [Carthage documentation](https://github.com/Carthage/Carthage#adding-frameworks-to-an-application) for instructions on how to integrate with your project using Xcode.

Quickstart
==========
### 1. Instantiate a Spine
```swift
let baseURL = NSURL(string: "http://api.example.com/v1")
let spine = Spine(baseURL: baseURL)
```

### 2. Register your resource classes
Every resource is mapped to a class that implements the `ResourceProtocol` protocol. Spine comes with a `Resource` class that implements this protocol.
A `Resource` subclass should override the variables `resourceType` and `attributes`. The `resourceType` should contain the type of resource in plural form. The `attributes` array should contain an array of attributes that must be persisted. Attributes that are not in this array are ignored.

Each class must be registered using a factory method. This is done using the `registerResource` method.

```swift
// Resource class
class Post: Resource {
	dynamic var title: String?
	dynamic var body: String?
	dynamic var creationDate: NSDate?
	dynamic var author: User?
	dynamic var comments: LinkedResourceCollection?

	override class var resourceType: String {
		return "posts"
	}

	override var attributes: [Attribute] {
		return attributesFromDictionary([
			"title": PropertyAttribute(),
			"body": PropertyAttribute().serializeAs("content"),
			"creationDate": DateAttribute().serializeAs("created-at"),
			"author": ToOneAttribute(User.resourceType),
			"comments": ToManyAttribute(Comment.resourceType)
		])
	}
}


// Register resource class
spine.registerResource(Post.resourceType) { Post() }
```

### 3. Fetching resources
```swift
// Using simple find methods
spine.find(["1", "2"], ofType: Post.self) // Fetch posts with ID 1 and 2
spine.findOne("1", ofType: Post.self)  // Fetch a single posts with ID 1
spine.find(Post.self) // Fetch all posts
spine.findOne(Post.self) // Fetch the first posts

// Using a complex query
var query = Query(resourceType: Post.self)
query.include("author", "comments", "comments.author") // Sideload relationships
query.whereProperty("upvotes", equalTo: 8) // Only with 8 upvotes
query.addAscendingOrder("created-at") // Sort on creation date
spine.find(query)
```

All fetch methods return a Future with `onSuccess` and `onFailure` callbacks.

### 4. Saving resources
```swift
spine.save(post).onSuccess {
    println("Saving success")
.onFailure { error in
    println("Saving failed: \(error)")
}
```
Extra care MUST be taken regarding related resources. Saving does not automatically save any related resources. You must explicitly save these yourself beforehand. If you added a new create resource to a parent resource, you must first save the child resource (to obtain an ID), before saving the parent resource.

### 5. Deleting resources
```swift
spine.delete(post).onSuccess {
    println("Deleting success")
.onFailure { error in
    println("Deleting failed: \(error)")
}
```
Deleting does not cascade on the client.

### 6. Read the wiki
The wiki contains much more information about using Spine.


Memory management
=================
Spine suffers from the same memory management issues as Core Data, namely retain cycles for recursive relationships. These cycles can be broken in two ways:

1. Declare one end of the relationship as `weak` or `unowned`.
2. Use the `unloadResource` function to unload resources and break cycles when you are done with a resource.

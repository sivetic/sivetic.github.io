---
layout: post
title: Asynchronous programming with Scala futures
draft: true
---


There are many way to accomplish asynchonous programming in Scala: using spawn with a thread pool, actors, futures, any number of Java threading libraries, etc.  Various Scala libraries provide different implementations for each of the asynchronous approaches, making the choice more difficult.  As of 2.10, Scala attempts to unify the approaches by deprecating spawn, and merging futures and actors into a common library.

### Using spawn

The project I am on has relied on running code blocks in a separate thread using spawn, and supplying a callback function to deal with results of the asynchronous code block.  While the approach has worked, it is far from ideal, and has resulted in callback hell.  Code is difficult to read and debug, and error handling leaves something to be desired.  Scala 2.10 has deprecated the use of spawn, giving us yet another reason to switch.

The existing code utilizing spawns is relatively easy to use in case we want to fetch one single item

```scala
def fetch[T](key: String, handler: (T) => Unit) {
    spawn {
        val result:T = ... // Fetch object from the database
        handler(result)
    }
}
```

Using the `fetch` method defined above simple as well:

```scala
def doSomething() {
    def handler(p: Profile) {
        println("Profile found")
        println(p)
    }

    val key = "..." // Key by which to fetch the profile
    fetch[Profile](key, handler(_: Profile))
}
```

Let's examine above code a little closer before moving on:

Define a `fetch` function, accepting a String key to query against our data store, and a handler to be called upon completion of our query.

```scala
def fetch[T](key: String, handler: (T) => Unit) {
```


Run the block of code enclosed in {} in a separate thread.  As we have no code `spawn`, there is nothing else for the `fetch` function to do, so it returns immediately.

```scala
spawn { ... }
```

Slight aside: the block of code enclosed by {} is actually an anonymous function defined in Scala, and passed to the `spawn` function as the first parameter.  `spawn` is not a special Scala keywords, it is nothing more than a function.  Here's the definition of `spawn`:

```scala
def spawn(p: => Unit)(implicit runner: TaskRunner = defaultRunner): Unit =
```

#### Callback hell

Using `spawn` and callbacks in the simple example above presented no major difficulties.  We weren't handling errors, however we could have just as easily added an error handler, or defined a type consisting of a Success or Failure, wrapped the queried type, and passed it to the handler.

We begin to notice troubles as soon as we need to chain dependent asynchronous operations.  Using a somewhat contrived example of querying for a user object given a username, using the userId to determine user's address, and finally using the user's address to find the closest restaurant, we can demonstrate the difficulty in managing a set of callbacks three levels deep:

```scala
def fetch[T](key: String, handler: (T) => Unit) { .... }

val userName = "test_user"

// For brewity, assume whatever key we pass will magically return our object
fetch[User](userName, handleUserFound)

def handleUserFound(user: User) {
    fetch[Location](user.id, handleLocationFound)
}

def hanldeLocationFound(location: Location) {
    fetch[Restaurant](location.geolocation, handleRestaurantFound)
}

def handleRestaurantFound(restaurant: Restaurant) {
    // Do something with the result, such as draw it on a map
}
```

This is an extremely simplistic example, however it begins to illustrate the difficulty behind multiple nested callbacks.  Imagine those callbacks nested deeper, perform some real world operations, and sprinkled throughout the code in various traits and classes.  Understanding and debugging this kind of code becomes very difficult.


### Using future


### Retrying



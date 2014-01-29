---
layout: post
title: Exploring Scala Options
---

# About Scala options

Scala `Option` is a very convenient way to deal with objects that may not be defined.  Most languages tend to represent an undefined object as `null`, leading to all kinds of null-checking code, and uncaught `NullPointerExceptions`.  In fact, `null` is such a bad idea that its inventor, Tony Hoare, called it his billion dollar mistake [1][ref1].

This post will not be about whether you should use `null` or not; instead, you will learn how to use the Scala Option feature to simplify coding if you do decide to avoid `null`s in Scala (as is highly recommended).

## What is an `Option`?

Option is a Scala class used to represent values that are either an instance of Some[A] if the value is present, or an instance of None in case the value is not present (similar to `null`).  Options are monads, and as such provide several important methods that allow for operations and composition: `map`, `foreach`, `filter`, and `flatMap`.  In addition, Options provide other useful methods such as `isEmpty`, `get`, `getOrElse`, `orElse`, etc.

When you first start using `Option`, it is really tempting to make use of `Opion.get` - do not fall into temptation.  `Option.get` will throw an exception if the Option value is `None`, which is equivalent to not using Options and accessing a null object.  Instead, using the various other methods defined on the `Option` class in order to make the most of Scala.

## Creating an `Option`

The easiest way to create an `Option` is to use the `apply` method in the `Option` companion object.

```
val option1 = Option("Foobar")
```

You can also define a value that is empty, or `None`:

```
val option2 = None
```

Options come in very handy when interacting with libraries that rely on null values:

```
val input = null
val option3 = Option(null) // option3 will be None
```

*Important Note*: You can also create an instance of the `Some` class by wrapping values with `Some()`.  Unlike `Option.apply` however, `Some.apply` will wrap the null value instead of returning an instance of `None`, so be especially careful when using `Some.apply`:

```
val option4: Option[String] = Some(null) // option4 is an instance of Some[String] with value set to null
val option5: Option[String] = Option(null) // option5 is an instance of None
```


## Working with `Option`s

### Transform values

Use `Option.map` to transform a value where the transformation function returns a value *not* wrapped in an `Option`:

```
def toUpper(s: String) = s.toUpperCase

val result = None.map(toUpper) // Returns None
val result = Option("foo").map(toUpper) // Returns Some("FOO")
val result = Option("12345").map(_.length) // Returns Some(5)
```

The equivalent of above code using `match` instead:

```
def toUpper(s: String) = s.toUpperCase

val result = Option("foo") match {
	case None => None
	case Some(x) => Some(toUpper(x))
}
```

And for fun, an equivalent without using functional programming concents:

```
def toUpper(s: String) = s.toUpperCase
val foo = Some("foo")

val result = 
	if (foo.isDefined)
		Some(toUpper(foo.get))
	else
		None
```


## Apply a method to the `Option`'s value

Sometimes we want to apply a method to the option's value, such as print the option's value, or perform a unit of work.  In such cases, where we do not care about the value returned from our method (i.e. we do not want to transform the option's value), we use `Option.foreach`:

```
def log(msg: String): Unit = println(s"Message Length: ${msg.length}")
Option("I am a log statement").foreach(log) // Prints "Message Length: 20")
None.foreach(log) // Does nothing

Option("Another Example").foreach(println) // Prints "Another Example"
```

Above code can also be rewritten in a more verbose mode, though use of `foreach` is encouraged due to brewity:

```
Option("I am a log statement") match {
	case Some(s) => log(s)
	case None =>
}

val o = Option("I am a log statement")
if (o.isDefined)
	log(o.get) // we know o.get will return a value since we already tested it
```

## Combining multiple `Option`s

Occasionally we need to combine several methods, each of which returns an option, to return a result.  Database and web service access will often need to make combine results from multiple method calls that return options.  Generally speaking, we do not want to return a nested Option; instead we want to flatten the results to a single `Option[A]` value.  Here is an example of how we can combine options using `Option.flatMap`:

```scala
// Database accessor methods that return user's data
def getUserById(id: Long): Option[User] = {...}
def getBusinessByUserId(userId: Long): Option[Business] = {...}
def getBusinessProfileByBusinessId(businessId: Long): Option[BusinessProfile] = {...}

// Combine results of above methods to return employee count at user's business
val id = 100
val businessDetails: Option[(String, Int)] = getUserById(id).flatMap { user =>
	getBusinessByUserId(user.userId).flatMap { business =>
		getBusinessProfileByBusinessId(business.businessId).map { businessProfile =>
			(business.name, businessProfile.employeeCount) // return the business name and employee count
		}
	}
}
```

In the above example, we will return the employee count if and only if we've successfully retrieved our user, business, and business profile.  If any of the three calls return `None`, we will immediately return `None` as our employee count, indicating that we were not able to find all the necessary data.

Deep nesting issues become immediately apparent with above code as soon as we start combining results from more than a couple methods.  Thankfully, Scala has a convenient way of combining options using the `for-yield` construct.  `for-yield` comprehensions are only syntactic sugar - they get compiled to a standard map/flatMap expression.  However they hide a lot of ugly code for us, so its usability is immediatley apparent:

```
val businessDetails: Option[(String, Int)] = for {
	user <- getUserById(id)
	business <- getBusinessByUserId(user.userId)
	businessProfile <- getBusinessProfileByBUsinessId(business.businessId)
} yield {
	(business.name, businessProfile.businessId)
}
```

Once again, if any one of the three calls returns `None`, `businessDetails` will also be set to `None`.


## Flattening nested `Option`s

If you ever end up with nested options in form of `Option(Option[A])`, calling `flatten` will return the inner option or `None` if the inner option is also `None`:

```
val o1 = Option("foo")
val o2 = None
val o3 = Option(o1) // Option(Option("foo"))
val o4 = Option(o2) // Option(None)

val o5 = o3.flatten // Option("foo")
val o6 = o4.flatten // None
```

`Option.flatten` can also be written using `match`:

```
Option(Option("foo")) match {
	case Some(x) => x
	case None => None
}
```


## Testing whether `Option` value is defined

`Option.isEmpty` returns true if option value is not defiend (i.e. if it is `None`), false otherwise
`Option.isDefined` returns true if the option value is defined, false otherwise 

```
Option("foo").isEmpty // false
None.isEmpty // true
Option("foo").isDefined // true
None.isDefined // false
```

Once again, the equivalent written using `match`:

```
// Option.isEmpty
Option("foo") match {
	case Some(_) => false
	case None => true
}

// Option.isDefined
Option("foo") match {
	case Some(_) => true
	case None => false
}
```

## Working with `Option` values

Sometimes we want to retrieve the option value instead of applying a transformation function to the value.  In such cases, we can use one of several methods:

	- `Option.getOrElse` returns option's value if present, the supplied default otherwise. Note that the supplied default can either be a value, or a method returning a value.
	- `Option.orElse` returns the same option if nonempty, or the supplied default option otherwise.  As with getOrElse, the default can be a method.  Unlike getOrElse, orElse will return an `Option[A]`
	- `Option.orNull` returns the option value if nonempty, null otherwise.  `orNull` is useful when interacting with 3rd party libraries that expect `null` values, otherwise should be avoided in idiomatic Scala code.
	- 'Option.get' returns the option value if nonempty, throws NoSuchElementException otherwise.  Use of `Option.get` is highly discouraged in idiomatic Scala.

Examples:

```
val full: Option[String] = Option("Glass is full")
val empty: Option[String] = None

full.getOrElse("Glass is empty") // returns "Glass is full"
empty.getOrElse("Glass is empty") // returns "Glass is empty"

def default = "Glass is empty"
empty.getOrElse(default) // pass method returning a string, returns "Glass is empty".  Only evaluated if option is `None` (i.e. lazy evaluation)

full.orElse(Option("Glass is empty")) // returns Option("Glass is full")
empty.orElse(Option("Glass is empty")) // returns Option("Glass is empty")

full.orNull // returns "Glass is full"
empty.orNull // returns null

full.get // returns "Glass is full"
empty.get // throws NoSuchElementException
```


## Further reading

So far I've explained core methods of the `Option` Scala object.  There are dozens more useful methods that you can explore by reading the [API Docs][ref2].


 [ref1]: http://www.infoq.com/presentations/Null-References-The-Billion-Dollar-Mistake-Tony-Hoare
 [ref2]: http://www.scala-lang.org/api/2.10.2/index.html#scala.Option
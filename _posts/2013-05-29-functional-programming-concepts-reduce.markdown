---
layout: post
title: Functional programming concepts - reduce (fold right)
---


### Introduction

There has been some discussion at Protegra recently about functional programming.  Coincidentally, I ran across a presentation titled [Functional Programming in 5 Minutes](http://slid.es/gsklee/functional-programming-in-5-minutes), or more accurately Learn to Curry in 5 Minutes.

For those interested in learning about currying, please refer to the presentation linked above, to the Wikipedia article on [Currying](http://en.wikipedia.org/wiki/Currying), or any other on-line functional programming resource.

Instead of focusing on currying as an introduction to functional programming, I will use a simple example of list aggregation to demonstrate the effectiveness of functional programming.

### Defining a List

Everyone should hopefully be familiar with the linked list data structure.  In it's most basic implementation, a linked list is composed of nodes containing a datum of some type, and a link to the next node in the list.  A list may be empty, or may contain one or more elements.  Lists containing no elements, also known as empty lists, are defined as `Nil`.  Lists containing elements are defined as `cons * (listof *)`, where [cons](http://en.wikipedia.org/wiki/Cons) is a function defining an object that contains two values (or pointers to values).  In the case of our list, `cons` defines an object containing a datum of some type * and a reference to the rest of the list.

A list containing three elements (1, 2, and 3) can be expressed using the above definition as follows:
`cons 1 (cons 2 (cons 3 Nil))`

In other words, we construct the first object with 1 as the datum, and a pointer to the rest of the list.  The second object is constructed with 2 as the datum and a reference to the rest of the list.  The third object is constructed with 3 as the datum, and a reference to an empty list (`Nil`), indicating that the third element is the final element.

Because `cons` notation is quite verbose, we can also express lists using curly braces: `{1, 2, 3}`.  I am avoiding the use of square bracket notation for lists in order to avoid confusion with arrays, which are generally defined using the square brackets ([]).

### List Aggregation in Imperative Languages

Now that we have a definition of a list, we can explore a very simple list aggregation operation - summation.  Determining the sum of a list is a frequently performed operation.  In an imperative language, we would typically sum all items in a list in the following fashion:

```js
function summation(list) {
	var sum = 0;
	for (var i=0; i<list.length; i++) {
		sum += list[i];
	}
	return sum;
}
```

The code should be familiar and self-explanatory.  Most of us have written the same method dozens of times, and will write it many more times.

Now let's assume that our program calls for computing the product of a list.  Once again, in an imperative language we would write a familiar method:


```js
function product(list) {
	var prod = 1;
	for (var i=0; i<list.length; i++) {
		prod *= list[i];
	}
	return prod;
}
```

The code was easy to write, and is self-explanatory.  I copy-pasted the summation method, renamed the method and some variables, and replaced the + operator with the * operator.  At this point we have two methods that are almost identical, but we need to keep both around.  Keeping both methods increases size of our code base, which has a direct effect on maintainability of the code.

#### Note on Concurrency

The summation method, as written above, may only be executed by a single thread.  While a good number of projects and developers do not care enough about concurrency, the increase in problem and data sizes we are facing today, combined with an increase in availability of CPU and GPU cores, means we will start to find more problems that will need to take advantage of concurrency in order to keep processing times reasonable.  Unfortunately, imperative code does not easily grant us that ability.


### List Aggregation Using Recursion

We can rewrite our summation algorithm using recursion.  For purposes of our sample code, we will assume our list has two extra methods - `head` and `tail`.  The `head` method returns the first element in the list, while the `tail` method returns a pointer to the rest of the list.

We can define the sum of an empty list to be 0, meaning that our function will terminate recursion when it reaches the end of the list.  Calling the `tail` method in the last element of the list will return an empty list, `Nil`, however for purposes of our language, we will assume that a list has a length property, which we will test to check if it is == 0.

```js
function summation(list) {
	if (list.length == 0) {
		return 0;
	} else {
		return list.head + summation(list.tail);
	}
}
```

We have replaced a for loop with a recursive call, which has also made our code slightly more difficult to reason about.  Recursion however leads very nicely into the functional programming solution to the summation problem.


### List Aggregation in Functional Programming

An implementation of summation in functional programming is typically done using recursion, exactly as in the previous example.  The recursion used is done so frequently, functional programming has a name for the pattern - fold right (also called reduce).  Let's begin by rewriting the summation function in Scala:

```scala
// Note: we have defined summation() as a function within the List class, which allows us to use isEmpty(), head(), and tail()
def summation(): Int =
	if (isEmpty) 0
	else head + tail.summation
```

Here is a line by line breakdown of the function:

Function definition, accepting a single paramter `l` of type List[Int], and returning an Int
`def summation(l: List[Int]): Int =`

If our list is empty, return 0
`if (l.isEmpty) 0`

Otherwise, if our list is not empty, return value of the head of the list plus the sum of the rest of the list.
`else l.head + tail.summation`

The first improvement we can make to our code is to make it generic, so we are not limiting ourselves to Ints:

```scala
// Note: A is the type of our generic list, namely List[A]
def summation(z: A): A =
	if (isEmpty) z
	else head + tail.summation(z)
```

We introduce a new paramter, `z`, of the same type as our list, which specifies the value sum of an empty list.  In case of numbers, `z` will be 0.  In case of Strings, we can define `z` to be an empty string, or anything else we may want to.


#### Fold Right

Next improvement is where functional programming becomes interesting.  To understand the code snippet, we need to recognize that in functional programming, functions can be passed to other functions as regular parameters.  We begin by defining a sum function:

```scala
def sum(x: A, y: A): A = x + y
```

Next, we remove the concept of summation from our summation function, and make it a general recursive aggregation function, called foldRight:

```scala
def foldRight(z: A)(f: (A, A) => A): A = 
	if (isEmpty) z
	else f(head, tail.foldRight(z)(f))
```

Let's dissect the code line by line to try and understand the flow.

Define a function `foldRight` which takes two arguments: `z`, the result of `foldRight` function call for an empty list, and `f` the function to apply to our list elements.  The function `f` requires as an input two parameters of the same type, and returns a value of the exact same type.  Looking back a few lines, our `sum` function is the perfect candidate, as it takes two parameters of type A as input, and returns a value of type A as output.
`def foldRight(z: A)(f: (A, A) => A): A =`

If the list is empty, we return `z`.

```scala
if (isEmpty) z
```

If the list is not empty, we return the value of a call to function `f`.  The first parameter to function `f` is the current value of head of list.  The second parameter is the value calculated by calling foldRight for the remainder of the list.  The resulting call stack is a recursive set of calls to `foldRight`, terminating at an empty list.

```scala
else f(head, tail.foldRight(z)(f))
```


Above may initially be somewhat confusing to those not used to recursion and functional programming.  Let's examine the order of operations when we call `foldRight` on a list of three integers, `{1, 2, 3}`:

```scala
// Define our sum function to return the sum of two values
def sum(x: A, y: A): A = x + y

// The initial call with our entire list
{1,2,3}.foldRight(0)(sum)

// foldRight called with a non-empty list encountered, return the sum of the 
// first value (head) and result of foldRight for the rest of the list (tail)
sum(1, {2,3}.foldRight(0)(sum))

// foldRight called with a non-empty list again, apply sum function once again
sum(1, sum(2, {3}.foldRight(0)(sum)))

// Final foldRight non-empty list call, apply sum function again
sum(1, sum(2, sum(3, {}.foldRight(0)(sum))))

// Empty list encountered.  We've specified 0 to be the value of applying 
// foldRight to an empty list, so we return 0 from the final foldRight call
sum(1, sum(2, sum(3, 0)))

// Apply sum functions in order
sum(1, sum(2, 3))
sum(1, 5)
 = 6
```

#### Revisiting List Product

Earlier we wrote a method that calculates the product of a list by copying and pasting our existing summation function.  Since we have just defined our foldRight aggregation function, we can accomplish the same task with much more code re-use:

First we define a function to calculate the product of two values:

```scala
def product(x: A, y: A): A = x * y
```

Then we can invoke our `foldRight` function to determine the product of all values in a list.  We only need to remind ourselves that product of an empty list will be 1, and not 0 as in the summation (when dealing with real numbers at least).

```scala
val p = list.foldRight(1)(product)
```

Using the `foldRight` method, we can easily apply an arbitrary calculation to our list elements, and expect the a valid result every time.  The amount of code written becomes much less, and once the behaviour of `foldRight` is understood, the clarity of code increases.


#### Note on Concurrency

While our imperative version of list aggregation suffered from being single threaded, our functional programming equivalent does not suffer the same fate.  In the case of sums and products, we are not concerned about the order we sum our elements in.  That means we can modify the implementation of `foldRight` to be concurrent.  Since the only job of `foldRight` is to invoke the function supplied on the list elements, and since `foldRight` and the function supplied cannot actually modify the list contents, we have can be assured that the output will be exactly the same in a concurrent setting as it would be in a single threaded setting.  Usefulness of making aggregation and iteration functions concurrent become apparent when dealing with large datasets, however that is out of scope for the current blog post.  See [MapReduce](http://en.wikipedia.org/wiki/Mapreduce) for one of the best examples of concurrency advantages in functional programming.


### Conclusion

Through use of a trivial example, I have attempted to illustrate the differences between the imperative and the functional approach to list aggregation.  The examples shown should illustrate how we are able to leverage functional programming to modularize our programs.  Modularizing provides us with a benefit of keeping code small, clear, easy to maintain, and highly re-usable.  Writing readable and maintainable code should be something every developer aims to achieve.

Disclaimer: The post is in no way meant to be a deep dive into functional programming.  I consider myself to have only scratched the surface of functional programming, and I am far from an expert.
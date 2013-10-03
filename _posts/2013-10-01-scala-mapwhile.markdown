---
layout: post
title: Scala mapWhile - map with a conditional predicate
---


Earlier today I needed to apply a map function to a list for as long as a predicate was satisfied; once the predicate was false, I need the map function to bail.  I was somewhat surprised to find Scala has no built-in method to accomplish such a task.

The easiest method would be to use `takeWhile` and `map` in combination:

```scala
val ids = (1 to 100).toList
def f(x: Int) = DB.fetch[Profile](x)
def p(x: Profile) = x.isArchived
val profiles = longs.takeWhile(p(f(_))).map(f(_))
// Spoiler - we are fetching the profile twice...
```

This does not work so well when the predicate is based around the map function, which is an expensive computation.  In my case, I need to iterate over a list of profile ids, for each id retrieve a profile from the database, then test the profile for a certain condition, stopping the iteration as soon as the predicate fails.  Instead of returning profile ids, as would be the case with takeWhile, I need to return the profile.

`List.takeWhile` allows testing each profile for the predicate, however it returns the original profile id, and not the newly fetched profile data, which I require.  I came up with two solutions, one using a while loop and one using recursion (optimized for tail recursion).

```scala
class MapWhileList[A](self: List[A]) {
  def mapWhile[B](f: A => B, p: B => Boolean): List[B] = {
    val b = new ListBuffer[B]
    var these = self
    var a:Option[B] = Some(f(these.head))
    while (!these.isEmpty && p(a.get)) {
      b += a.get
      these = these.tail
      a = if (!these.isEmpty) Some(f(these.head)) else None
    }
    b.toList
  }

  def mapWhileRec[B](f: A => B, p: B => Boolean): List[B] = {
    @tailrec
    def loop(xs: List[A], acc: List[B]): List[B] = {
      if (xs.isEmpty) acc
      else {
        val b = f(xs.head)
        if (!p(b)) acc
        else loop(xs.tail, b :: acc)
      }
    }

    loop(self, Nil).reverse
  }
}

implicit def mapWhileList[A](self: List[A]) = new MapWhileList(self)

// Usage (slightly contrived)
def f(x: Int): Profile = DB.fetch[Profile](id) // expensive computation
def p(x: Profile): Boolean = !x.isArchived // We stop when we encounter an archived Profile

val ids = (1 to 100).toList // list of profile IDs
val mw1 = ids.mapWhile(f, p)
val mw2 = ids.mapWhileRec(f, p)
```


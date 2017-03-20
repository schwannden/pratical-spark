# Stream

Another application of lazy evaluation is that we can work with large data structure more efficiently. And something only lazy evaluation can do, is that we can work with infinite data structure.

### Stream Trait and Object

```Scala
package object Stream {
  import java.util.NoSuchElementException

  trait Stream[+A] {
    def isEmpty: Boolean
    def filter(p: A => Boolean): Stream[A]
    def take(n: Int): Stream[A]

    val head: A
    val tail: Stream[A]
    val toList: List[A]
  }

  object Stream {
    def cons[T](hd: T, tl: => Stream[T]): Stream[T] = new Stream[T] {
      override def isEmpty: Boolean = false

      override def filter(p: T => Boolean): Stream[T] =
        if (p(head)) Stream.cons(head, tail.filter(p))
        else tail.filter(p)

      def take(n: Int): Stream[T] = {
        if (n <= 0) empty else cons(head, tail.take(n-1))
      }

      override val head: T = hd
      override lazy val tail: Stream[T] = tl
      override lazy val toList: List[T] = head :: tail.toList
    }

    val empty = new Stream[Nothing] {
      override def isEmpty: Boolean = true
      override def take(n: Int) = this
      override def filter(p: Nothing => Boolean) = this
      override lazy val head = throw new NoSuchElementException("empty.head")
      override lazy val tail = throw new NoSuchElementException("empty.tail")
      override lazy val toList: List[Nothing] = Nil
    }
  }

}
```

### Examples

Constructing Stream object and print it

```Scala
import Stream._

val t = Stream.cons(1, Stream.cons(2, Stream.cons(3, Stream.empty)))
t.toList
```

We can now define natural number as follows

```Scala
def from(n: Int): Stream[Int] = Stream.cons(n, from(n+1))
val nat = from(0)
```

And implement the Sieve of Eratosthenes algorithm for prime number

```Scala

def seive(s: Stream[Int]): Stream[Int] =
  Stream.cons(s.head, seive(s.tail filter (_ % s.head != 0)))
```

And define prime number on top of this algorithm is straight forward

```Scala
val prime = seive(from(2))
```

Print the first 200 primes

```Scala
prime.take(200).toList
```




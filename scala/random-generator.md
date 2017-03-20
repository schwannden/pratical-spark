# Random number generation

Intuitively, we can define the following trait for random number

```Scala
trait Generator[+T] {
  def generate: T
}
```

The an integer random number generator would be

```Scala
val integers = new Generator[Int] {
  val rand = new java.util.Random
  override def generate = rand.nextInt()
}
```

So `integers.generate` yields a random integer. And we can further define random boolean generator as

```Scala
val integers = new Generator[Int] {
  override def generate = integers.generate > 0
}
```

And random pair as

```Scala
 val integers = new Generator[Int] {
  override def generate = (integers.generate, integers.generate)
}
```

This quickly becomes cumbersome when we need larger random structures. We hope to be able to compose new random generator by function composition. For example, we hope to be able to define random generators as follows

```Scala
val booleans = integers map (_>0)

val pairs: Generator[(Int, Int)] = for(
  i <- integers;
  j <- integers
) yield (i, j)
```

This is not only less codes, but more elegant and intuitive.

### Expanding Random Trait

To use for loop in composition, we need map and flatMap function in Random trait. The type of flatMap should be a Monad \(more on this later\), and the semantics of map should be

```Scala
map(xs) = xs flatMap (x => unit(x))
```

So let's define our random trait as

```Scala
package object Random {
  trait Generator[+T] {
    self =>

    def generate: T

    def map[S](f: T => S): Generator[S] = new Generator[S] {
      override def generate = f(self.generate)
    }

    def flatMap[S](f: T => Generator[S]): Generator[S] = new Generator[S] {
      override def generate = f(self.generate).generate
    }
  }
}
```

Together with some helper method

```Scala
val integers = new Generator[Int] {
val rand = new java.util.Random
override def generate = rand.nextInt()
}


def single[T](x: T): Generator[T] = new Generator[T] {
override def generate = x
}

def choose(low: Int, high: Int): Generator[Int] = new Generator[Int] {
override def generate = integers.generate % (high - low) + low
}

def oneOf[T](xs: T*): Generator[T] =
for(i <- choose(0, xs.length)) yield xs(i)
```

### Usage example

Random boolean and pairs can be defined as

```Scala
import Random._

val booleans = integers map (_>0)

val pairs: Generator[(Int, Int)] = for(
  i <- integers;
  j <- integers
) yield (i, j)
```

Random list

```Scala

val lists: Generator[List[Int]] = for(
  empty <- booleans;
  list <- if (empty) emptyList else nonEmptyList
) yield list

def emptyList = single(Nil)

def nonEmptyList = for (
  head <- integers;
  list <- lists
) yield head :: list
```

Even random tree

```Scala
trait Tree
case class Leaf(x: Int) extends Tree {
  override def toString: String = "(" + x + ")"
}
case class Node(l: Tree, r: Tree) extends Tree {
  override def toString: String = "(" + l + ")" + "(" + r + ")"
}

def leaf = integers map (Leaf(_))

def inners = for (
  l <- trees;
  r <- trees
) yield Node(l, r)

def trees: Generator[Tree] = for (
  isLeaf <- booleans;
  tree <- if (isLeaf) leaf else inners
) yield tree
```




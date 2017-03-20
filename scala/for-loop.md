Scala for loop is implemented in terms of map and flatMap, while the conditions is implemented as filter method.

For example

```Scala
for (x <- xs) yield expr(x)
```

is expanded to

```Scala
xs.map(x => expr(x))
```



And

```Scala
for (
  x <- xs;
  y <- ys
) yield expr(x, y)
```

is expanded to

```Scala
xs.flatMap(x => ys.map(y => expr(x, y)))
```




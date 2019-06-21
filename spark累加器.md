spark2.x之后，之前的accumulator被废除，用AccumulatorV2代替

```
@deprecated("use AccumulatorV2", "2.0.0")
class Accumulator[T] private[spark] (
```

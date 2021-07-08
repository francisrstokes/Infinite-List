# lazy-infinite
A Fantasy Land Compliant, Lazily Evaluated Infinite List Data Structure.

`lazy-infinite` uses generators to define a potentially infinite data structure, and allows you to describe transforming the elements in that structure without evaluating it.

## Installation

```bash
npm i lazy-infinite
```

## Usage

```javascript
const Infinite = require('lazy-infinite');

// Create a representation of an infinite structure
const naturalNumbers = Infinite.generator(function*() {
  let x = 0;
  while (true) yield x++;
});

// Transform to yield new infinite structures
const primes = naturalNumbers
  .filterDependent((x, list) => {
    if (!(x > 1 && (x % 2 === 1 || x === 2))) return false;
    for (let i = 0; i < list.length; i++) {
      const y = list[i];
      if (y > x / 2) break;
      if (x % y === 0) return false;
    }
    return true;
  });

// Concretely evaluate only what is required
const first1000Primes = primes.take(1000);
// -> [ 2, 3, 5, 7, 11, 13, 17, 19, 23, 29, 31, 37, 41, 43, 47, 53, 59, 61, 67, 71, ...
```

## API

### take

`take :: Infinite a ~> Integer -> [a]`

`take` receives an argument *n*, and returns a concrete list with *n* elements.

**Example**
```javascript
naturalNumbers.take(5);
// -> [ 0, 1, 2, 3, 4 ]
```

### takeContinuous

`takeContinuous :: Infinite a ~> Integer -> [[a], Infinite a]`

`takeContinuous` is just like take, but it returns an array of two items: the *concrete list* and a new *Infinite* that produces elements where the previous one left off.

**Example**
```javascript
let [concrete, nextPrimes] = primes.takeContinuous(5);
console.log(concrete);
// -> [ 2, 3, 5, 7, 11 ]

[concrete, nextPrimes] = nextPrimes.takeContinuous(5);
console.log(concrete);
// -> [ 13, 17, 19, 23, 29 ]
```

### nth

`nth :: Infinite a ~> Integer -> a`

`nth` takes an argument *n* and returns the *nth* element of the concretely evaluated list.

**Example**
```javascript
primes.nth(5);
// -> 11
```

### drop

`drop :: Infinite a ~> Integer -> Infinite a`

`drop` takes a number *n* and returns a new `Infinite` with the first *n* elements removed.

**Example**
```javascript
naturalNumbers
  .drop(5)
  .take(5);
// -> [ 5, 6, 7, 8, 9 ]
```

### map

`map :: Infinite a ~> (a -> b) -> Infinite b`

`map` takes a function and applies it to every element in the list.

**Example**
```javascript
naturalNumbers
  .map(x => -x)
  .take(5);
// -> [ -0, -1, -2, -3, -4 ]
```

### flatMap

`flatMap :: Infinite a ~> (a -> [b]) -> Infinite b`

`flatMap` runs a function on every element of the list and concatenates the results.

**Example**
```javascript
naturalNumbers
  .flatMap(x => naturalNumbers.take(x))
  .take(10);
// -> [ 0, 0, 1, 0, 1, 2, 0, 1, 2, 3 ]
```

### mapIndexed

`mapIndexed :: Infinite a ~> (a -> Int -> b) -> Infinite b`

`mapIndexed` is just like [map](#map), except the function it is passed receives an index as it's second argument.

**Example**
```javascript
primes
  .mapIndexed((x, i) => `Prime #${i}: ${x}`)
  .take(5);
// -> [
//      'Prime #0: 2',
//      'Prime #1: 3',
//      'Prime #2: 5',
//      'Prime #3: 7',
//      'Prime #4: 9'
//    ]
```

### filter

`filter :: Infinite a ~> (a -> Bool) -> Infinite a`

`filter` takes a function, and drops any elements from the infinite list that return `false` when this function is applied.

**Example**
```javascript
naturalNumbers
  .filter(x => x % 2 === 0)
  .take(5);
// -> [ 0, 2, 4, 6, 8 ]
```

### filterIndexed

`filterIndexed :: Infinite a ~> (a -> Int -> b) -> Infinite b`

`filterIndexed` is just like [filter](#filter), except the function it is passed receives an index as it's second argument.

**Example**
```javascript
primes
  .filterIndexed((x, i) => i % 2 === 0)
  .take(5)
// -> [ 2, 5, 9, 13, 19 ]
```

### filterDependent

`filterDependent :: Infinite a ~> (a -> [a] -> Bool) -> Infinite a`

`filterDependent` is just like [filter](#filter), except that the function it is passed receives the *list of items before it* as the second argument.

**Example**
```javascript
naturalNumbers
  .filterDependent((x, list) => x !== list.length)
  .take(5);
// -> [ 1, 2, 3, 4, 5 ]
```

### zip

`zip :: Infinite a ~> Infinite b -> Infinite [a, b]`

`zip` takes another `Infinite`, and returns a new Infinite whose elements are an array with the original value and a corresponding value from the other infinte.

This can be used to create custom *indexing*.

**Example**
```javascript
const fibonacci = Infinite.generator(function* () {
  let a = 0;
  let b = 1;
  while (true) {
    yield b;
    const tmp = b;
    b += a;
    a = tmp;
  }
});

primes
  .zip(fibonacci)
  .take(5);
// -> [ [2, 1], [3, 1], [5, 2], [7, 3], [9, 5] ]
```

### intersperse

`intersperse :: Infinite a ~> Infinite b -> Infinite a|b`

`intersperse` takes another `Infinite`, and returns a new Infinite whose elements alternate between the first and second Infnites.

**Example**
```javascript
const fibonacci = Infinite.generator(function* () {
  let a = 0;
  let b = 1;
  while (true) {
    yield b;
    const tmp = b;
    b += a;
    a = tmp;
  }
});

primes
  .intersperse(fibonacci)
  .take(5);
// -> [ 2, 1, 3, 1, 5 ]
```

### toGenerator

`toGenerator :: Infinite a ~> () -> Generator a`

`toGenerator` returns a generator function that represents this `Infinite`s computation.

**Example**
```javascript
const primeGeneratorFn = primes.toGenerator();

const primeGen = primeGeneratorFn();
primeGen.next();
// -> { value: 2, done: false }
primeGen.next();
// -> { value: 3, done: false }
primeGen.next();
// -> { value: 5, done: false }
```

### Infinite.generator

`Infinite.generator :: Generator a -> Infinite a`

`Infinite.generator` takes a potentially infinite generator function and returns an `Infinite` list.

**Example**
```javascript
const odds = Infinite.generator(function*() {
  let x = 1;
  while (true) {
    yield x;
    x += 2;
  }
});
```

### Infinite.toGenerator

`Infinite.toGenerator :: Infinite a -> Generator a`

`Infinite.toGenerator` takes an `Infinite` list and returns a generator function that represents it's computation.

**Example**
```javascript
const primeGeneratorFn = Infinite.toGenerator(primes);

const primeGen = primeGeneratorFn();
primeGen.next();
// -> { value: 2, done: false }
primeGen.next();
// -> { value: 3, done: false }
primeGen.next();
// -> { value: 5, done: false }
```

### Infinite.from

`Infinite.from :: (a -> a) -> a -> Infinite a`

`Infinite.from` takes *next value* function and a *start* value, and returns an `Infinite` with an automatically constructed iterator.

**Example**
```javascript
const odds = Infinite.from(x => x + 2, 1);
```

### Infinite.fromIterable

`Infinite.fromIterable :: Iterable a -> Infinite a`

`Infinite.fromIterable` takes anything conforming to the `Iterable` interface and returns an `Infinite`.

**Example**
```javascript
Infinite.fromIterable([1,2,3,4,5,6,7]).take(5)
// -> [ 1, 2, 3, 4, 5 ]
```

## Fantasy Land

Supports\*: `Functor` and `Filterable`.

*\*Since a truly valid `of` static method cannot be written for `Infinite` it does not properly conform to either interface*

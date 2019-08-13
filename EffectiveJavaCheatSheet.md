[TOC]



## Creating and Destroying Objects

### 1. Static factory methods

Benefits:

* ***have names***

  Relevant when multiple constructors needed: e.g. `BigInteger.probablePrime(..)`

* ***are not required to create a new object each time.***

  Use preconstructed instances (immutable classes), cache instances, instance-controlled classes, Flyweight (reuse instances, cache internally)

* ***they can return an object of any subtype of their return type***

  hide implementation classes, decide in runtime about suitable implementation, 

  return an interface

* ***the class of the returned object need not exist when the class containing the method is written***

  dependency injection

Limitations:

* ***classes without public or protected constructors cannot be subclassed***

  use composition instead of inheritance


> **Conventions:**	
>
> - `from(.)`: type-conversion - `Date.from(instant)`
> - `of(..)`: aggregation (multiple parameters) - `EnumSet.of(JACK, QUEEN, KING)`
> - `instance(..)`: return an instance
> - `newInstance(..)`: return **new** instance



### 2. Builder instead of telescoping constructors

Bad alternative to telescoping constructors - JavaBeans pattern (using `setters` after creation) leaves object in inconsistent state partway through its construction and precludes making class immutable.

**The Builder pattern is a good choice when designing classes whose constructors or static factories would have more than a handful of parameters**

Hierarchical builders:

```java
NyPizza pizza = new NyPizza.Builder(SMALL).addTopping(HAM).build();
```

```java
public abstract class Pizza {
    public enum Topping { HAM, MUSHROOM }

    final Set<Topping> toppings;

    abstract static class Builder<T extends Builder<T>> {
        EnumSet<Topping> toppings = EnumSet.noneOf(Topping.class);

        public T addTopping(Topping topping) {
            toppings.add(Objects.requireNonNull(topping));
            return self();
        }

        abstract Pizza build();
        protected abstract T self();
    }

    Pizza(Builder<<?> builder) {
        toppings = builder.toppings.clone(); // See Item 50
    }
}

public class NyPizza extends Pizza {
    public enum Size { SMALL, MEDIUM, LARGE }

    private final Size size;

    public static class Builder extends Pizza.Builder<Builder> {
        private final Size size;

        public Builder(Size size) {
            this.size = Objects.requireNonNull(size);
        }

        @Override
        public NyPizza build() {
            return new NyPizza(this);
        }

        @Override
        protected Builder self() {
            return this;
        }
    }

    private NyPizza(Builder builder) {
        super(builder);
        size = builder.size;
    }
}
```



### 3. Singleton: private constructor or `Enum`

* **Making a class a singleton can make it difficult to test its clients** (cannot substitute mocks)
* **Prefer static factory method vs public static final INSTANCE** - can change to non-singleton in the future
* **Deserialization breaks singleton property** - implement `readResolve` method to fix
* **Enum is a singleton** - the best approach, deserialization is safe



### 4. Noninstantiability using private default constructor

* Attempting to enforce noninstantiability by making a class abstract does not work
* A class can be made noninstantiable by including a private constructor (which throws Exception)
* Private constructor prevents subclassing
* Use only for utility classes



### 5. Prefer dependency injection to hardwiring resource

* Static utility classes and singletons are inappropriate for classes whose behavior is parameterized by an underlying resource.
* Dependency injection provides flexibility and testability
* Pass the resource into the constructor (or factory method or builder) when creating a new instance
* Dependency injection is equally applicable to constructors, static factories and builders



### 6. Avoid creating unnecessary objects (performance)

* You can avoid creating unnecessary objects by using **static factory methods**
* Example with pattern matching - use following approach to avoid re-compilation of same pattern into finite state machine

```java
public class RomanNumerals {
    private static final Pattern ROMAN = Pattern.compile(.);

    static boolean isRomanNumeral(String s) {
        return ROMAN.matcher(s).matches();
    }
}
```

* **Prefer primitives to boxed primitives**, and watch out for unintentional autoboxing
* For example, the `keySet()` method of the `Map` interface returns a `Set` view of the `Map` object, consisting of all the keys in the map, **but every call to `keySet()` on a given `Map` object may return the same `Set` instance**



### 7. Eliminate obsolete object references (memory leaks)

* Whenever a **class manages its own memory**, the programmer should be alert for memory leaks
* Take care when implementing **caches and data structures**

```java
    public Object pop() {
        if (size == 0) throw new EmptyStackException();
        Object result = elements[--size];
        elements[size] = null; // Eliminate obsolete reference
        return result;
    }
```

* Another common source of memory leaks is **listeners and other callbacks**
  * Store only weak references to callbacks (`WeakHashMap`)



### 8. Avoid finalizers and cleaners[9]

* Finalizers and cleaners are dangerous, unpredictable and generally unnecessary
* **Have your class implement `AutoCloseable[7]`**, and require its clients to invoke the close method on each instance when it is no longer needed (**preferably using try-with-resources**)



### 9. Prefer `try-with-resources`[7] to `try-finally`

* `try-finally` is ugly when used with more than one resource (multiple nesting)
* When 2 exceptions thrown - first in `try {..}` and second in `finally {..}`, the second exception completely obliterates the first one. There is no record of the first exception in the exception stack trace
* Usage of `try-with-resources` with optional `catch(.)` clause:

```java
    static String firstLineOfFile(String path, String defaultVal) {
        try (BufferedReader br = new BufferedReader(new FileReader(path))) {
            return br.readLine();
        } catch (IOException e) {
            return defaultVal;
        }
    }
```

* If exceptions are thrown by both `try (.)` and the (invisible) `close()`, the latter exception
  is suppressed in favor of the former. **These suppressed exceptions are printed in the stack trace with a notation saying that they were suppressed**.



## Common `Object` methods

### 10. Obey the `equals(.)` contract

**You may not** override `equals(.)` when:

* Instance is inherently unique (like `Thread`)
* No need for the "logical equality"
* Private or package-private classes if `equals(.)` is never invoked

**You better override** `equals(.)` when:

* Class is a **value class** (needing "logical equality")

**`equals(.)` should:**

* represent **equivalence relation** (reflexive, symmetric, transitive)
* be **consistent** - multiple invocations yield same results
* return `false` for `equals(null)`

**Problem: **

**There is no way to extend a class and add a value component (field) while preserving the `equals(.)` contract.** 

Usually - you will violate symmetry, transitivity or Liskov substitution principle. Example for symmetry violation (when mixing Point and ColorPoint equality check):

```java
public class Point {
    ...
    @Override
    public boolean equals(Object o) {
        if (!(o instanceof Point)) return false;
        Point other = (Point) o;
        return other.x == x && other.y == y;
    }
}

public class ColorPoint extends Point {
    ...
    @Override
    public boolean equals(Object o) {
        // If we use o.getClass() here - we'll violate Liskov substitution principle
        if (!(o instanceof ColorPoint)) return false; 
        ColorPoint other = ((ColorPoint) o);
        // Broken - violates symmetry!
        return super.equals(o) && other.color == color;
    }
}
```

```java
    Point p = new Point(1, 2);
    ColorPoint cp = new ColorPoint(1, 2, Color.RED);
    // p.equals(cp) is true
    // cp.equals(p) is false
```

**Solution: **

**Use composition!**

```java
public class ColorPoint {
    private final Point point;
    private final Color color;
    
    ...
    
    @Override
    public boolean equals(Object o) {
        if (!(o instanceof ColorPoint)) return false;
        ColorPoint other = (ColorPoint) o;
        // Good job!
        return other.point.equals(point) && other.color.equals(color);
    }
}
```

**Recipe for `equals(.)`**:

1. Use the == operator to check if the argument is a reference to this object (if yes, return true)
2. Use the `instanceof` operator (`null instanceof [AnyType]`  is false) to check if the argument has
   the correct type. If not, return false
3. Cast the argument to the correct type
4. For each “significant” field in the class, check if that field of the argument matches the corresponding field of this object
5. Always override `hashCode()` when you override `equals(.)`
6. Advice is not to use **derived fields** (those fields, that can be computed from other fields)



### 11. Always override `hashCode()` when you override `equals(.)`

`hashCode()` should:

* be consistent - multiple invocations yield same results
* be the same if objects are equal according to `equals(.)` 
* whenever possible - return different values if objects are not equal according to `equals(.)`

```java
// The worst possible legal hashCode implementation - never use!
// Hash tables will degenerate to Linked Lists
@Override public int hashCode() { return 42; }
```

Advices:

* Cache hashCode value for Immutable classes

* Use `Objects.hash(..)` for convenience (but a bit slower)

* For primitive types use e.g. `Long.hashCode(longNum)`

* For references - invoke `hashCode()` recursively

* For arrays - use `Arrays.hashCode(.)` 

* Don't use **derived fields** (those, that can be computed from other fields)

* Combine hash codes from multiple fields

  ```java
  // Typical hashCode method
  @Override public int hashCode() {
  	int result = Short.hashCode(areaCode);
  	result = 31 * result + Short.hashCode(prefix);
  	result = 31 * result + Short.hashCode(lineNum);
  	return result;
  }
  ```

* The value 31 was chosen because it is an odd number and a prime, also hashcode calculation 

  optimized by JVM as `31 * i == (i << 5) - i`



### 12. Always override `toString()`

Advices:

* Providing a good `toString()` implementation makes your class much more pleasant to use and makes systems using the class easier to debug
* When practical, the `toString()` method should return all of the interesting information contained in the object
* Prefer to not specify exact format in documentation - to avoid client dependency on the output of your `toString()` - so that you can change format easily in the future
* Provide programmatic access to the information contained in the value returned by `toString()`



### 13. Override `clone()` judiciously

* Providing correct `clone()` implementation is hard, it's contract is weak () and object creation is done without constructor call (extralinguistic object creation mechanism)

* Immutable classes should never provide a `clone()` method

* A better approach to object copying is to provide a **copy constructor** or **copy factory**, which offer more flexibility and are easier to implement

  * ```java
    // Copy constructor
    public Yum(Yum yum) { ... };
    ```

  * ```java
    // Copy factory
    public static Yum newInstance(Yum yum) { ... };
    ```

    

### 14. Consider implementing `Comparable`


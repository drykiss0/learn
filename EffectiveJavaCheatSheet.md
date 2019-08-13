### Creating and Destroying Objects

1. **Static factory methods**:

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
   > - `from(..)`: type-conversion 
   > - `of(..)`: aggregation (multiple parameters)
   > - `instance(..)`: return an instance
   > - `newInstance(..)`: return **new** instance
   
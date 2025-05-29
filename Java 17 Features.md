# Java 17 Features

###  Sealed Classes 
Sealed classes let you control which other classes or interfaces may extend or implement them, enabling a more declarative form of hierarchy control.

public abstract sealed class Shape
    permits Circle, Rectangle, Triangle { ... }

public final class Circle extends Shape { ... }
public final class Rectangle extends Shape { ... }
public non-sealed class Triangle extends Shape { ... }


### Benefits:

Exhaustiveness in switch (future patterns)

Better modeling of fixed hierarchies

Stronger encapsulation of class hierarchies


### Pattern Matching for switch
Extends pattern matching to switch expressions and statements, allowing more concise and type-safe dispatch.

static String formatter(Object o) {
    return switch (o) {
        case Integer i -> String.format("int %d", i);
        case Long l    -> String.format("long %d", l);
        case Double d  -> String.format("double %f", d);
        default        -> o.toString();
    };
}

### Benefits:

Exhaustiveness checking

Guards (case String s && s.length() > 5 -> …)

Multiple patterns per label


### Pattern matching for instanceof 
bundles the test and the cast into one, making code safer and more concise.

#### Instead of
if (obj instanceof String) {
    String s = (String) obj;
    System.out.println(s.length());
}

#### you write
if (obj instanceof String s) {
    // inside this block, 's' is already a String
    System.out.println(s.length());
}

obj instanceof String s both checks that obj is a String and—if it is—binds it to the local variable s.

The scope of s is the true branch of the if.



### Enhanced Pseudo-Random Number Generators 

A new java.util.random API providing:

Jumpable and splittable PRNG algorithms

A unified, extensible interface for future algorithms

Better statistical qualities than legacy Random

RandomGenerator rng = RandomGenerator.of("L64X128MixRandom");
int n = rng.nextInt(100);


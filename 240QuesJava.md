##Q1. What is JVM? Why is Java called platform-independent?
JVM (Java Virtual Machine): It is the runtime engine that executes Java bytecode. JVM is responsible for converting bytecode into machine-specific code.

Platform Independence:

Java source code (.java) is compiled into bytecode (.class).

This bytecode can run on any machine that has a compatible JVM.

Hence, Java is "Write Once, Run Anywhere" (WORA).

 
// Sample
public class Hello {
    public static void main(String[] args) {
        System.out.println("Hello, Java!");
    }
}
##Q2. What is the difference between JDK, JRE, and JVM?
Component	Description
JDK (Java Development Kit)	Includes JRE + development tools (javac, javadoc).
JRE (Java Runtime Environment)	Contains JVM + core libraries. It’s needed to run Java apps.
JVM	Executes the bytecode line by line.

##Q3. What are the features of Java?
Object-Oriented

Platform Independent

Robust and Secure

Multithreaded

Distributed

Architecture-neutral

Dynamic and Extensible

##Q4. What is the difference between path and classpath variables?
Path: Tells OS where to find executables (e.g., javac, java).

Classpath: Tells JVM where to find .class files and external libraries.

 
# Example
export PATH=/usr/bin/java
export CLASSPATH=.:/home/user/myapp/classes
##Q5. What are primitive data types in Java?
Java has 8 primitives:

byte, short, int, long (integer types)

float, double (floating-point)

char (character)

boolean (true/false)

 
int a = 10;
double pi = 3.14;
boolean flag = true;
##Q6. What is the default value of a primitive type?
Type	Default Value
int	0
boolean	false
double	0.0
object reference	null

##Q7. What are wrapper classes?
They "wrap" primitive types into objects. For example:

int → Integer

boolean → Boolean

Useful in Collections (like List<Integer>) which work with objects.

 
int x = 10;
Integer y = Integer.valueOf(x);
##Q8. What is autoboxing and unboxing?
Autoboxing: Primitive → Wrapper

Unboxing: Wrapper → Primitive

 
Integer i = 10; // autoboxing
int j = i;      // unboxing
##Q9. What is a constructor?
A constructor is used to initialize objects. It has the same name as the class and no return type.

 
class Person {
    Person() {
        System.out.println("Constructor called");
    }
}
##Q10. What is constructor overloading?
Multiple constructors with different parameter lists.

 
class Person {
    Person() {}
    Person(String name) {}
    Person(String name, int age) {}
}
##Q11. What is the purpose of the static block?
It’s used to initialize static variables. Runs once when the class is loaded.

 
class Example {
    static {
        System.out.println("Static block called");
    }
}
##Q12. Can we execute a program without the main method?
From Java 7 onwards, it’s not possible for standalone applications. JVM looks for the main() method.

##Q13. What is the difference between static and non-static methods?
Static: Belongs to the class; can be called without object.

Non-static: Belongs to object instance.

 
static void show() {}
void print() {}
##Q14. What is method overloading?
Same method name, different parameters. Resolved at compile-time.

 
int sum(int a, int b) {}
double sum(double a, double b) {}
##Q15. What is method overriding?
Subclass provides specific implementation for a method in its superclass.

 
class A {
    void show() {}
}
class B extends A {
    @Override
    void show() {}
}
##Q16. What is the use of final keyword in Java?
Variable: Value can’t change.

Method: Can’t be overridden.

Class: Can’t be extended.

##Q17. What is the difference between this and super?
this: Refers to current object.

super: Refers to parent class object/method.

 
super.show();
this.name = name;
##Q18. Can a constructor be private?
Yes. Used in Singleton patterns or factory classes.

 
class Singleton {
    private Singleton() {}
}
##Q19. What is the use of instanceof keyword?
Checks if an object is an instance of a class or subclass.

 
if (obj instanceof String) {}
##Q20. What is the difference between break and continue?
break: exits the loop.

continue: skips current iteration.

##Q21. What is the difference between == and e##Quals()?
==: compares references.

e##Quals(): compares values (can be overridden).

 
String a = "hello";
String b = new String("hello");
System.out.println(a == b);         // false
System.out.println(a.e##Quals(b));    // true
##Q22. What is a package in Java?
A namespace to group related classes.

 
package com.example.utils;
##Q23. What is the access scope of protected access modifier?
Accessible:

Within same package.

In subclass (even if in different package).

##Q24. What is the final class?
Cannot be extended.

 
final class Constants {}

##Q25. What is the difference between interface and abstract class?
Feature	Interface	Abstract Class
Methods	All abstract (until Java 7)	Can have both
Fields	Public static final	Any access modifier
Multiple Inheritance	Yes	No

##Q26. What are naming conventions in Java?
Class: CamelCase (e.g., CustomerAccount)

Variable/Method: camelCase (e.g., calculateTotal)

Constant: UPPER_SNAKE_CASE (e.g., MAX_USERS)

##Q27. Why should we follow naming conventions?
Improves readability, maintenance, and aligns with community standards.

##Q28. What is a JavaBean?
A reusable component that follows conventions:

Public no-arg constructor

Getters and setters

Serializable

 
public class Student implements Serializable {
    private String name;
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
}
##Q29. What is the importance of the toString() method?
Returns string representation of an object.

 
@Override
public String toString() {
    return "Student[name=" + name + "]";
}
##Q30. What is the purpose of e##Quals() and hashCode()?
Used in Collections like HashMap, HashSet.

e##Quals(): checks logical e##Quality.

hashCode(): provides hash value.

 
@Override
public boolean equals(Object o) { ... }

@Override
public int hashCode() { ... }

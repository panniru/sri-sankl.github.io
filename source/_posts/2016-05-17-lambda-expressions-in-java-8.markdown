---
layout: post
title: "Lambda Expressions in Java 8"
date: 2016-05-17 22:42:12 +0530
comments: true
categories: 
---
> Lambda Expression is a way of writing an implamementation of a method for later execution.

Lambdas are also known as *closures* or *anonymous functions*.


##### Syntax:
  Lambda expression consists of three parts. 
  1. A paranthesized set of parameters.
  2. An arrow
  3. A body
``` java Lambda Expression
    () -> //my single statement;
    (a, b) -> //my single statement;
``` 
<!--more--> 
 If we have multiple statements in body part, we have to write them inside the curly braces and the last statement should return the result.
``` java Lambda Expression
	() -> {
		//my steatement1
		//.....
		return someValue; 
	}
``` 
<b> *Note:* </b>Lambdas can not `break` or `continue` out of the body.
##### Functional Interface:
  It is an interface which requires only one method to be implemented in order to satisfy the requirements of the interface.
``` java MySampleInteface.java
public interface MySampleInterface {
	void mySampleMethod();
}
```
 We can implement this interface using an anonynous class
 
``` java
		MySampleInterface msi = new MySampleInterface() {
			@Override
			public void myMethod() {
				// My implementation logic
			}
		};
 msi.myMethod();

```
 We can simplify this entire story into a single statement using lambdas.
``` java
    		MySampleInterface msi = () -> System.out.println("I am implementing a functional interface");
         msi.myMethod();
```


  This is how the lambdas are implemented. Thre is no ambiguity around which method of the interface lambda is trying to implement.
<br/>
The designers of java8 also provided an annotation `@FunctionalInterface` to serve as a documentations hint that an interface is a functional interface. Its not mandatory to keep this annotation though.
<br/>
Java has some functional interfaces, such as `Runnable`, `Comparator<T>` etc.
``` java
Runnable r = () -> System.out.println("Lambda Implemented run method");
r.run();
```
``` java
	Comparator<String> c = (String lhs, String rhs) -> {
					System.out.println("Comparing "+lhs+ " with "+ rhs );
					return lhs.compareTo(rhs);
				};
	int result = c.compare("Hello", "world");
```
##### Type Inference
 Java 8 is providing enhanced type inferences. In the above example the target type is a`Comparator<String>`, the objects passed to the lambda should be String or its subtype. 
 In this case we can leave the `String` declaration in front of `lhs` and `rhs`.
``` java
	Comparator<String> c = (lhs, rhs) -> {
					return lhs.compareTo(rhs);
				};
	int result = c.compare("Hello", "world");
```

For the first in the java world lambda reference cannot directly be assigned to the `Object` class. Since `Object` is not a functional interface. We have to cast to its target type to assign to a `Object` type reference.
``` java
    Object obj = (Runnable) () -> System.out.println("Hello World");
```
##### Lixical Scoping:
 When we are working with a inner class, we need to write a special syntax to access the outer scope from the inner class. 

``` java
public class Hello {
	Runnable r = new Runnable() {
		@Override
		public void run() {
			System.out.println(Hello.this);
			System.out.println(Hello.this.toString());

		}
	};

	public String toString() {
		return "I am in Hello class";
	}
}
```
We can simplify this using lambds. Lambdas are lexically scoped. This means that they recognizes their immediate environment as the next outer most scope.
 

``` java
public class Hello {
	Runnable r = () -> {
		System.out.println(this);
		System.out.println(toString());
	};

	public String toString() {
		return "I am in Hello class";
	}
}
```
`this` no longer refers to the lambda itself. Both the above examples provides the same output.

Lambdas can refer to it's immediate outer scoped local variables but these are effectively final meaning that its final in all but name. 
 It can refer but can not modify local variable `message`

``` java
public static void main(String args[]) {
		String message = "Hello";
		Runnable r1 = () -> System.out.println(message);

		r1.run();

	}
```
 
``` java
	public static void main(String args[]) {
		StringBuilder message = new StringBuilder("Hello");
		Runnable r1 = () -> {
				System.out.println(message);
				message.append(" ");
				message.append("World");
				System.out.println(message);
		};
		r1.run();
	}
```
 Here `final` modifier applied to only the reference but not on the right side object.
 
##### Method References:
Lambdas not only difined at the point of usage. We can also define lambdas for later use.  
``` java
public class Person {
	public String firstName;
	public String lastName;
	public int age;

	public Person(String firstName, String lastName, int age) {
		this.firstName = firstName;
		this.lastName = lastName;
		this.age = age;
	}

	public static final Comparator<Person> compareByFirstName = (lhs, rhs) -> lhs.firstName.compareTo(rhs.firstName);
	public static final Comparator<Person> compareByLasteName = (lhs, rhs) -> lhs.lastName.compareTo(rhs.lastName);

	public String toString() {
		return "FirstName: " + firstName;
	}

	public static void main(String args[]) {
		List<Person> persons = new ArrayList<Person>();

		persons.add(new Person("Sri", "Sankl", 22));
		persons.add(new Person("Nani", "Hero", 26));
		persons.add(new Person("Rehman", "A.R", 26));
		persons.add(new Person("Bahubali", "Rajamouli", 21));
		System.out.println(persons);

		Collections.sort(persons, Person.compareByFirstName);

		System.out.println(persons);

	}
}
```
Comparator `compareByFirstName` is defined as a static final field and it can be referenced as any other static field could be referenced.

>Thank you for reading the post. Feel free to comment.
>Happy Coding..:)

######References:
 1. http://www.oracle.com/technetwork/articles/java/architect-lambdas-part1-2080972.html

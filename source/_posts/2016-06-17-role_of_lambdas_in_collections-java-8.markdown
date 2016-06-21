---
layout: post
title: "Role of Lambdas in Collections - Java 8"
date: 2016-06-17 21:39:06 +0530
comments: true
categories: Java
---
  Collections would have look different if lambdas were part of java from beginning. Lambdas are popular in functional programming and some other object oriented programming languages like Ruby.
<br>
It might be better to implement a new collection framework `collection II` using lambdas, but it would be a big task. Instead, java came up with a strategy of adding extension methods to existing interfaces(such as `Collection`, `List` or `Iterable`) and adding new interfaces(such as `Sream`) that are retrofitted onto existing classes.
<br>
  Major advantages of using lambdas in collections are
<br>
1. *Parallelism*
2. *Lazy evaluation*
<!--more--> 
For example, An Employee class will looks like.
<br>
![Employee.java](/images/EmpClass.png) 

##### External vs Internal Iteration
If I want to set all the employees `dept` to *PROD*. I need to write a `for` loop to iterate over all the elements in the collection.
``` Java External Iteration
		for (Employee employee : employees) {
			employee.setDept("PROD");
		}
```
 The Collection framework provides a way to its client to iterate over the elements through `iterator()`. The `for` loop uses the iterator and traverse sequentially through all the elements of the collection. 
<br>
The disadvantages using this approch are
<br>
  1. It is inherently serial. It travels in order of which they are inserted into collection.
<br>
  2. There is no opportunity to mange the control flow, parellelism, short circuting or laziness to improve the performance.
 
 The alternative to this approach is an *internal iteration*. 
``` Java Internal Iteration
employees.forEach((emp) -> emp.setDept("PROD"));
```
 
 Here control flow is in the hands of library. Collections provides a way to the client to interact with its internal methods. The client deligates the snippets of code to the library to excecute. So library can decide parellel processing or lazy evalution based on the given criteria. So the internal iteration helps library to process elements in the collection parelelly. The `forEach()` method may not use the parellel processing, but there is a possibility to implement parellelism by using internal iteration.
  Internal iteration also provides a way to *pipeline* operations togather.


For example, If I want to shift all the *PRDO* dept employees to *R&D*, we could say
``` Java 
		employees.stream()
			.filter(emp -> emp.getDept() == "PROD")
			.forEach(emp -> emp.setDept("R&D"));
```
`filter()` returns a stream of employees who are matching with the given condition to `forEach()` function.
<br>
If I want to find, how much the company is spending on sales dept, then we could do
<br>
``` Java
		employees.stream()
			.filter(emp -> emp.getDept() == "SALES")
			.mapToLong(emp -> emp.getSal())
			.sum();
```
##### Eager vs Lazy Evaluation
 In an eager type of evaluation, when `filter()` method gets excecuted it draws all the elements from the collection and applies the condition(Predicate) and provides a new Collection. The new collection will be used by `mapToLong()` method. In the same way `mapToLong()` method also returns a new collection. So every statement when it returns accumulates the result.


 Where as in Lazy , filtering is only done when we start iterating the elements of results of the filter method. The `filter()` method instead of reading all the elements to return the result, it draws the first element from the source and applies the predicate. The succeded element will be deligated to the next method `mapToLong()` which is also a lazy operation. Methods such as `filter()`, `map()` are *naturally lazy*. So the return type of these methods cannot be a Collection, they return again a `stream`. The tail methods such as `sum()`  or `forEach()` are eager operations. They read all elements deligated from lazy methods and returns the result. Normally tail methods are *naturally eager*. 

 We can acheive a significant increase in performance by doing series of lazy operations and an eager operation such as `filter-map-forEach`(*lazy-lazy-eager*) or `filter-filter-mapToLong-sum`(*lazy-lazy-lazy-eager*). 

 For example, If we are doing filtering, maping and accumulating on a collection. The lazy operations such `filter()`, `map()`, `sum()` will do in single pass of data instead of three passes.

##### Streams
  The `Stream` interface is fundamental interface that intended to use in variety of senarios, including the Collections API.
<br>
Streams differs from Collections in several ways:


 1.  *No Storage:* Streams dont store any data. It transforms data from source through various operations. Stream method produces a new stream these are known as *Intermediate Operaions*; those that do not are *terminal operations*
<br>
 2. *Functional in nature:* In functional programming languages functions are the first class citizens and they only returns result instead of modifying the state of params. In the same way stream operations produces results but they will not modify the data.
<br>
 3. *Laziness-seeking :* Many stream operation such as `filter()`, `map()` are lazy in nature. We can only traverse as many elements as we need to find the answer. For example "find the first employee who is having Rs. 10000  salary" not need to look up all the elements in the collection.
<br>
 4. *Bounds optional:* We can draw limited no of elements from the infinite collection of data source. For example, we can only take 20 elements from the infinite array of numbers. 
  

All the [stream](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Stream.html) operations are not fitted in the Collections. Instead `stream()` method is added in Collections which return the `Stream` interface.
<br>
Stream operations can operate in either seqrial or parellel; whether stream is serial or parellel is a property of a stream source. We have `parallelStream()` method to return the parellel stream.
 
##### Laziness and short-circuiting
  If we are looking for the first employee in a dept whose salary is more than Rs.10000. 
``` Java
    Optional<Employee> firstEmp = employees.stream()
                       .filter(emp -> emp.getDept() == "SALES")
				         .filter(emp -> emp.getSal() >= 10000).findFirst();
```
  Instead of iterating all the elements in the collection or source, `filter()` operation finds one for which predicate is true. The `findfirst()` methods returns `Optional`, because there may be cases where no element matching the criteria.

##### Sorting became simpler
   In the previous [post](http://sri-sankl.github.io/blog/2016/06/17/advantages-of-lambdas-in-java-8/) we have seen how to simplify sorting using comparator with lambdas    
``` Java
public static final Comparator<Employee> compareByName = (lhs, rhs) -> lhs.getName().compareTo(rhs.getName());
```
Java8 provides even simpler machanism to do this

 If I have two X type objects to comapare based on a value returned by the functions of each X. On the `Comparator` class a `comparing` function is lambda it takes a function and then extracts a value out of the object and returns a `Comparator` that sorts based on that.


``` Java
 public static Comparator<Employee> BY_NAME = Comparator.comparing(Employee::getName);
```
 If I have to sort the employees by dept and then by salary. Java8 provides the method composition which is known as *functional composition*. 

``` Java
Collections.sort(employees, Employee.BY_DEPT.thenComparing(Employee.BY_SAL));
```
 We can combine the `Comparator` instances in varios ways.
We can also write the above statment without assigning lambdas to `BY_DEPT` or `BY_SAL` variables as.
``` java
Collections.sort(employees, Comparator.comparing(Employee::getDept).thenComparing(Employee::getSal));
```
######References:
 1. http://cr.openjdk.java.net/~briangoetz/lambda/lambda-libraries-final.html
 2. http://www.oracle.com/technetwork/articles/java/architect-lambdas-part2-2081439.html


>Thank you for reading the post. Feel free to comment.
>Happy Coding..:)

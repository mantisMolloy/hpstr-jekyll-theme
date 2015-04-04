---
layout: post
title: "Java 8: Lamda Expressions"
modified:
categories: blog
description: An introduction to Java 8 Lamdas
excerpt: This post covers the basics of lamda expressions through full examples. I cover everything you need to know about lamda expressions for the OCA Java 8 exam. 
tags: [Java 8, Lamda, OCA]
image: 
  feature: code.jpg
share: true
date: 2015-04-03T14:29:12+01:00
---

# Simple Lamdas

In this post we will scratch the surface of Java 8 lamda expressions. This is by no means a guide to everything Java 8 has to offer in the way of lamda expressions, but it will cover the basics and covers everything you need to know for the OCA Java 8 exam.

## What is a Lamda Expression?

Functional programming is nothing new. Languages like Haskell and lisp have been around for 25 & 50 years. Even JavaScript can be considered a functional language, but the definition of a functional language is still somewhat debatable as seen in <a href="http://www.reddit.com/r/programming/comments/8krbo/erlang_is_not_functional_response_to_scala_is_not/">this</a> Reddit article on Erlang. 

Java 8 introduces the concept of **lamda expressions** to the Java OOP world. A lamda expression is a block of code which can be passed around. A lamda expression doesn't have a name like a traditional function would. Lamda expressions resemble anonymous inner classes but they behave differently and have different scope as will be outlined later in this post. You may have heard of lamdas referred to as closures in other programming languages

## Let's Start With an Example

First lets look at an example of a list of car objects where we want to print only cars that match a certain criteria. Let's see how we would do this without lamdas and then see how we can improve the code by using simple Java 8 lamdas. 

Lets say we have the following simple class that models a car

{% highlight java %}
public class Car {
  private String  model;
  private boolean sunRoof;
  private boolean airCon;

  public Car (String model, boolean sunRoof, boolean airCon) {
    this.model   = model;
    this.sunRoof = sunRoof;
    this.airCon  = airCon;
  }

  public boolean  hasSunRoof  { return sunRoof;}
  public boolean  hasAirCon   { return airCon; }
  public String    toString   { return model; }
}
{% endhighlight %}

One way we might try print car objects without lamdas is to create a new class which checks our car objects and returns whether the car objects matches our criteria. 

Lets create an interface for all our classes that will check a car for a particular attribute.

{% highlight java %}
public interface Check {
  boolean test (Car c);
}
{% endhighlight %}

Now we need to create a class which checks for sun roof. Our **CheckSunRoof** class which implements **Check** will look like this

{% highlight java %}
public class CheckSunRoof implements Check {
  public boolean test (Car c){
    return c.hasSunRoof();
  }
}
{% endhighlight %}

Simple stuff so far, right? Ok now lets see how we would use these functions

{% highlight java %}
public class PrintWithoutLamdas {
  public static void main (String[] args) {
    
    List <Car> cars = New ArrayList<Car>();
    cars.Add(new Car ("Audi", true, true));
    cars.Add(new Car ("BMW", false, true));
    cars.Add(new Car ("Lada", false, false));

    printBy(cars, new CheckSunRoof());
  }

  public static void printBy(List<Cars> list, Check check) {
    for (Car car : list) {
      if (check.test(car)) {
        System.out.println(car + " ");
      }
    }
  }
}
{% endhighlight %}

Pretty straight forward. Now let's say we want to print cars that have air con. What would we do? Well we could write a new **CheckAirCon** class and pass that to the **printBy()** method. Or seeing as the second parameter of the **printBy()** method is an interface we could just write an anonymous inner class.

Enter Java 8. Instead of writing a new class we can simply pass in a lamda expression. No need to write a new **CheckAirCon** class or no need to write an anonymous inner class. The only change would be the following code when calling the printBy() method.

{% highlight java %}
printBy(cars, car -> car.hasAirCon());
{% endhighlight %}

So the whole thing with lambdas would look like this. Note that with this implementation we would still need to write the **Check** interface (although more on predicates later) but we would not need the **CheckSunRoof** class or a **CheckAirCon** class.

{% highlight java %}
public class PrintWithLamdas {
  public static void main (String[] args) {
    
    List <Car> cars = New ArrayList<Car>();
    cars.Add(new Car ("Audi", true, true));
    cars.Add(new Car ("BMW", false, true));
    cars.Add(new Car ("Lada", false, false));

    printBy(cars, car -> car.hasAirCon());
    printBy(cars, car -> car.hasSunRoof());
  }

  public static void printBy(List<Cars> list, Check check) {
    for (Car car : list) {
      if (check.test(car)) {
        System.out.println(car + " ");
      }
    }
  }
}
{% endhighlight %}

The code above used a concept called **Deferred Execution**. Basically this means it defined a piece of code that will be run later.

## So what just happened there?

When figuring out what to do with a lamda expression Java relies on context. Java mapped the lamda expression onto the method in the **Check** interface. First let's analyse the format of a lamda expression.

{% highlight java %}
parameter -> body

car -> car.hasAirCon()
{% endhighlight %}

So car is the **parameter** and car.hasAirCon() is the **body**. When a lamda expression is passed as a parameter it tries to map itself onto the interface. The method that is called in the our example is

{% highlight java %}
boolean test (Car c);
{% endhighlight %}

Since the method takes a **Car** the lamda parameter must be a **Car**. Since the method returns a boolean the lamda must also return a boolean. If car.hasAirCon() didn't return a boolean this would not compile.

## Predicates

So the lambda expression is mapping itself onto the **test** method in the **Check** interface. An interface like the **Check** interface is called a **functional interface** for best practice we can annotate it with **@FunctionalInterface**. Does this mean we have to write an interface everytime we want to use a lambda. Well no, Java provides an interface which can be used for our case. Through generics the **Predicate** interface in the **java.util.function** package will cater for our needs.

{% highlight java %}
public interface Predicate<T> 
  boolean test(T);
}
{% endhighlight %}

Now a full example of how we print cars using no extra classes or interfaces and just lamda expressions would look like this 

{% highlight java %}
public class Car {
  private String  model;
  private boolean sunRoof;
  private boolean airCon;

  public Car (String model, boolean sunRoof, boolean airCon) {
    this.model   = model;
    this.sunRoof = sunRoof;
    this.airCon  = airCon;
  }

  public boolean  hasSunRoof  { return sunRoof;}
  public boolean  hasAirCon   { return airCon; }
  public String    toString   { return model; }

  public static void printBy(List<Cars> list, Predicate<Car> check) {
    for (Car car : list) {
      if (check.test(car)) {
        System.out.println(car + " ");
      }
    }
  }

  public static void main (String[] args) {
    
    List <Car> cars = New ArrayList<Car>();
    cars.Add(new Car ("Audi", true, true));
    cars.Add(new Car ("BMW", false, true));
    cars.Add(new Car ("Lada", false, false));

    printBy(cars, car -> car.hasAirCon());
    printBy(cars, car -> car.hasSunRoof());
  }
}
{% endhighlight %}

Simple! 

Java 8 has integrated the Predicate interface into some existing classes. For example in ArrayList the removeIf() method takes a Predicate. Lets say we have a list of Strings and we want to remove all Strings that begin with 'z'. We could write a loop or we could use a lamda expression.

{% highlight java %}
List<String> myWords = new ArrayList<>();
myWords.add("Tom");
myWords.add("zoo");
myWords.add("elephant");
System.out.println(myWords);     // [Tom, zoo, elephant]
myWords.removeIf(s -> s.charAt(0) == 'z');
System.out.println(myWords);     // [Tom, elephant]
{% endhighlight %}

## More Complex Lamda expressions

What if we the Predicate interface doesn't suit our needs? Well Java has packaged interfaces for commonly used lamda expression signatures. They can be found in **java.util.function**. In total there are 45 ready made functional interfaces written with generics that many lamda expressions you want to use will map onto. If you can't find anything in **java.util.function** you can always write your own functional interface!

## Scope

Unlike anonymous inner classes lamda expressions can access variables outside of their own body. So in our example we can use something like this.

{% highlight java %}
boolean test = true;
printBy(cars, a -> a.hasAirCon() == test);
{% endhighlight %}

Lamdas can access instance and static variables. Method parameters and local variables can be accessed but not assigned new values.

{% highlight java %}
(i, j) -> { int i = 0; return 0; } //will not compile

(i, j) -> { int c = i; return 0; } // ok 
{% endhighlight %}

## Syntax

The syntax we have been using up to the last example is the simplest form of a lamda expression. There are different options available. Parentheses around the parameter is optional when there is one parameter and it doesn't have its type declared. When using curly braces we must use a return statement.

{% highlight java %}

// All equally valid

car -> car.hasAirCon()                    // one parameter and no declaration
(Car car) -> return car.hasAirCon()       // declare parameter
(Car car) -> { return car.hasAirCon(); }  // declare parameter type and state the return
{% endhighlight %}

We can declare multiple parameters, but note that we must use parentheses here.

{% highlight java %}
(String a, String b) -> a.equlas(b)
{% endhighlight %}


## Summary

In this post we touched on basic lamda expression functionality. There is much more to learn about lamdas and the new features of Java 8 but this post covers everything you need for the OCA exam. 

Not covered is the full extent of the **java.util.function** package or using Java 8 streams with filters and lamda expressions. Using lamda expressions with streams and filters not only makes your code more succinct but makes it more efficient code becomes easily parallelizable.
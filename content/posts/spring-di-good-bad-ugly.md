---
title: "Spring DI - The Good, the Bad and the Ugly"
date: 2019-08-03T14:33:47+05:30
draft: false
tags:
  - Java
  - Spring
  - DI
categories:
  - "Software Engineering"
images:
  - images/logo-100x100.png
---
[Spring](https://spring.io) is a mature applications development framework for Java. Combined with the power of Spring Boot it is a competitive choice for modern micro-services based application development.

Dependency injection is one of the core services provided by the framework. Spring supports constructor injection and setter injection. Lets take simple example of a hypothetical `OrderService` class depending on an `OrderRepository` implementation and discuss the different options available to us.

## The Good - Constructor Injection
In constructor injection as the name implies we pass dependencies via the constructor. 

{{< highlight java >}}
@Service
public class OrderService {

    private final OrderRepository orderRepository;

    @Autowired
    public OrderService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository; 
    }
}
{{< /highlight >}}

There are multiple advantages of using constructor injection. First and foremost you can 
guarantee that when you create an instance of your class is is valid and usable right away.
The constructor enforces the [invariant](https://en.wikipedia.org/wiki/Class_invariant) that `OrderService` needs an `OrderRepository` at all times to work correctly.

Another benefit of using construction injection is that your class is not tightly coupled to
the Spring framework. You can always create and use the `OrderService` without a Spring context. For 
example your unit tests will not need a custom runner.

As an architect what I love about constructor injection is it drives you towards low coupling.
Having 5+ parameters in your constructor will surely raise a flag and make you rethink your design.

## The Bad - Setter Injection
Setter injection depends on a setter method to inject dependencies.

{{< highlight java >}}
@Service
public class OrderService {

    private OrderRepository orderRepository;

    @Autowired
    public setOrderRepository(OrderRepository orderRepository) {
        this.orderRepository = orderRepository; 
    }
}
{{< /highlight >}}

If you look closely you will notice that we lose our class invariant because we had to drop the final
modifier from our instance field. Now there is no guarantee that we will always have an `OrderRepository` that we can call.

Another disadvantage is that high coupling is not so obvious unlike constructor injection.But still you have the capability of using your class without Spring.

## The Ugly - Autowired Fields
{{< highlight java >}}
@Service
public class OrderService {

    @Autowired
    private OrderRepository orderRepository;

}
{{< /highlight >}}

Coming from a .NET background and having worked with DI frameworks like [StructureMap](http://structuremap.github.io/) and [Autofac](https://autofac.org/) I was shocked to see code snippets like the above when I first saw them.

If you use this method you class is tightly coupled to the Spring framework in addition to the disadvantages discussed in setter injection. I don't think there is any valid reason to use `@Autowired` private fields.

## Conclusion
After evaluating the DI options available in Spring my recommendations are,

- Always prefer constructor injection.
- Use setter injection if you must (but not because it would reduce constructor parameters)
- Avoid `@Autowired` fields like the plague
---
title: "Dependency inversion in C: struct-of-function-pointers and building libraries"
date: 2025-11-16T12:00:00-08:00
draft: false
tags: ["C","architecture","embedded", "dependency", "inversion"]
---

# Dependency Inversion in Embedded C Without Runtime Polymorphism

A common mistake in embedded software architecture is coupling hardware implementation details too closely to the business rules of the application. This creates brittle code that is difficult to port to new systems with identical high-level semantics but different physical interfaces.

The solution is **dependency inversion**: high-level policies should not depend on low-level details. This topic has been covered extensively elsewhere, so here’s the short version: lower-level components must conform to an interface defined at a higher level. Control still flows from high to low abstraction layers, but the *dependencies* flow upward—high-level code is unaware of how the interface is concretely implemented.

Languages with full runtime polymorphism natively support this. C does not. There is no virtual function table or class hierarchy. But a vtable is just a table of function pointers—something C *can* express. So in C, we provide our own layer of indirection using structs of function pointers.

Let’s ground this in an example.

## A Simple Logging Interface

Suppose we’re designing a logging facility. The system needs two APIs:

1. An initialization function that accepts a module name to prepend to each log message.
2. A log function that accepts a message to log.

A naïve embedded implementation might tightly couple these to a specific implementation - such as logging to stdout. But this has a serious drawback - its very difficult to change the logging interface for a given component. What if we wanted a component to sometimes log to stdout, and sometimes log to a file? The indirection through an interface provides this flexibility.

## Defining the Interface
So, let's define an abstract interface—implemented via function pointers and let the high-level component code depend only on that.
{{< highlight c >}}
{{<readfile "dependency-inversion-c/core/include/logger_interface.h" >}}
{{< /highlight >}}

### Implementing Two Concrete Loggers
Now, we create two low-level implementations of the high-level logger interface. Note how they depend on the logger *interface*. Depending on a logger does **not** imply depending on a concrete transport.

#### stdout Logger
{{< highlight c >}}
{{<readfile "dependency-inversion-c/plugins/stdout_logger/include/stdout_logger.h" >}}
{{< /highlight >}}

{{< highlight c >}}
{{<readfile "dependency-inversion-c/plugins/stdout_logger/src/stdout_logger.c" >}}
{{< /highlight >}}

#### File Logger
{{< highlight c >}}
{{<readfile "dependency-inversion-c/plugins/file_logger/include/file_logger.h" >}}
{{< /highlight >}}

{{< highlight c >}}
{{<readfile "dependency-inversion-c/plugins/file_logger/src/file_logger.c" >}}
{{< /highlight >}}

## Using the Logger

Let's create a component that uses a logger. We'll make a generic worker that does some task - in our case, logs a message. This worker depends only on the logging *interface* and not a particular logger.

{{< highlight c >}}
{{<readfile "dependency-inversion-c/core/components/include/worker.h" >}}
{{< /highlight >}}

{{< highlight c >}}
{{<readfile "dependency-inversion-c/core/components/src/worker.c" >}}
{{< /highlight >}}

## Pulling it all together
Its now trivial to create a `main` which instantiates a few different workers, and flexibly select which logger to use. We'll create three loggers: two that utilize the stdout implementation and one that logs to a file.

{{< highlight c >}}
{{<readfile "dependency-inversion-c/app/main.c" >}}
{{< /highlight >}}

If we compile this all together and run, we should see the two stdout loggers emit their messages on the console, and the file logger's output in `log.txt`. The makefile in the source code for this example defines the required target.
```bash
make > /dev/null
./demo
[stdout_logger_1] Component did some work
[stdout_logger_2] Component did some work
cat log.txt
[file_logger] Component did some work
```

## Verifying Dependency Inversion

To confirm our high-level logger has*no dependency on any concrete implementation, let's investigate the `worker.o` file created for the worker component. Run `make worker.o` to create the object file for our worker component. We can then use `nm` to prove there are no undefined symbols. This demonstrates our dependency inversion worked - our high-level component has no dependency on a specific implementation of the logger!
```bash
make worker.o
nm worker
0000000000000030 T _component_do_work
0000000000000000 T _component_init
000000000000006c s l_.str
0000000000000000 t ltmp0
000000000000006c s ltmp1
0000000000000088 s ltmp2
```

## The Cost of Indirection
While this abstraction is very low cost, it is not *zero* cost. We need to dereferenc our function pointer, which incurs a small runtime cost each time we log. For almost all applications, this is completely negligable.

## Other Options
What are the other options for tackling this sort of problem? There's a few options.

1. Define entirely separate worker components for each logger type: worker-file, worker-stdout, etc
2. Conditionally compile our worker or logger libraries with different implementations depending on the desired configuration.

The first bullet is hitting the problem with a hammer - introduce tons of duplication and maintain tightly coupled interfaces. This makes the project much more expensive to maintain and less scalable. The second option is even worse - we've made it impossible to instantiate multiple types of loggers in a single translation unit. Its my opinion that conditional compilation is nearly always a code smell and should be avoided at all costs - a topic for another time.

## Conclusion

We’ve now walked through implementing dependency inversion in C without runtime polymorphism. Using function-pointer interfaces allows you to decouple high-level policy from hardware-specific implementations with minimal overhead. This pattern produces more testable, more portable, and more maintainable embedded systems.
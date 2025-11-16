---
title: "Dependency inversion in C"
date: 2025-11-16T12:00:00-08:00
draft: false
tags: ["C","architecture","embedded", "dependency", "inversion"]
---
Suppose we’re developing a system in need of a logging facility. The logger needs two APIs:

1. An initialization function that accepts a module name to prepend to each log message.
2. A log function that accepts a message to log.

A naïve implementation might couple the logger directly to a specific output, like stdout. But this has a serious drawback - it's very difficult to change the logging interface for a given component. What if we wanted a component to sometimes log to stdout, and sometimes log to a file? What if we are working on an embedded platform and need to emit log messages via UART or SPI, or some other serial interface?

A common mistake in embedded software architecture is coupling hardware implementation details too closely to the business rules of the application. This creates brittle code that is difficult to port to new systems with identical high-level logic but differences in the presentation layer.

The solution is **dependency inversion**: high-level policies should not depend on low-level details. This topic has been covered extensively elsewhere, so here’s the short version: lower-level components must conform to an interface defined at a higher level. Control still flows from high to low abstraction layers, but the *dependencies* flow upward—high-level code is unaware of how the interface is concretely implemented.

Languages with full runtime polymorphism natively support this. C does not. There is no virtual function table or class hierarchy. But a vtable is just a table of function pointers—something C *can* express. So in C, we provide our own layer of indirection using structs of function pointers.

Let’s dig into our logger example. The naive implementation might be structure like this, with the a component utilizing a logger containing a direct or transitive dependency on a specific logger implementation:
{{< mermaid >}}
graph TD
    Main --> Worker
    Worker --> StdoutLogger
{{< /mermaid >}}

Dependency inversion frees components from needing to know the details of how logging is implemented. The resulting architecture looks as follows:
{{< mermaid >}}
graph TD
    Main --> Worker
    Main --> StdoutLogger
    Main --> FileLogger
    Worker --> LoggerInterface
    StdoutLogger --> LoggerInterface
    FileLogger --> LoggerInterface
{{< /mermaid >}}

It's worth a note that it is totally fine - and in fact expected - that `main` depends on the low-level details. Ideally, `main` - or whatever the entry point for your application is - should be the centralized location where all concrete implementations are defined and injected into the more abstract components.

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

## Using the Logger in a Component

Let's create a component that uses a logger. We'll make a generic worker that does some task - in our case, logs a message. This worker depends only on the logging *interface* and not a particular logger.

{{< highlight c >}}
{{<readfile "dependency-inversion-c/core/components/include/worker.h" >}}
{{< /highlight >}}

{{< highlight c >}}
{{<readfile "dependency-inversion-c/core/components/src/worker.c" >}}
{{< /highlight >}}

## Wiring together into an Application
It's now trivial to create a `main` which instantiates a few different workers, and flexibly select which logger to use. We'll create three loggers: two that utilize the stdout implementation and one that logs to a file. Each logger instance maintains its own module name buffer, allowing multiple workers to share the same logger implementation while retaining independent module names.

{{< highlight c >}}
{{<readfile "dependency-inversion-c/app/main.c" >}}
{{< /highlight >}}

If we compile this all together and run, we should see the two stdout loggers emit their messages on the console, and the file logger's output in `log.txt`. The makefile in the source code for this example defines the required target.
```bash
make > /dev/null
./demo
[stdout_logger_1] Worker did some work
[stdout_logger_2] Worker did some work
cat log.txt
[file_logger] Worker did some work
```

## Verifying Dependency Inversion

To confirm the components implementing business logic - like `worker` - have no dependency on any concrete implementation, let's investigate the `worker` object file. If we run `make worker.o` to create the object file for our worker component, we can then use `nm` to prove there are no undefined symbols. This demonstrates our dependency inversion worked - our high-level component has no dependency on a specific implementation of the logger!
```bash
make worker.o
nm worker
0000000000000030 T _worker_do_work
0000000000000000 T _worker_init
000000000000006c s l_.str
0000000000000000 t ltmp0
000000000000006c s ltmp1
0000000000000088 s ltmp2
```

## The Cost of Indirection
While this abstraction is very low cost, it is not *zero* cost. We need to dereference our function pointer, which incurs a small runtime cost each time we log. For almost all applications, this is completely negligible.

## Other Options
What are the other options for tackling this sort of problem? There's a few options.

1. Define entirely separate worker components for each logger type: worker-file, worker-stdout, etc
2. Conditionally compile our worker or logger libraries with different implementations depending on the desired configuration.

The first bullet is hitting the problem with a hammer - introduce tons of duplication and maintain tightly coupled interfaces. This makes the project much more expensive to maintain and less scalable. The second option is even worse - we've made it impossible to instantiate multiple types of loggers in a single translation unit. It's my opinion that conditional compilation is nearly always a code smell and should be avoided at all costs - a topic for another time. In short, both alternatives reduce flexibility and maintainability compared to dependency inversion.

## Conclusion

We’ve now walked through implementing dependency inversion in C without runtime polymorphism. Using function-pointer interfaces allows you to decouple high-level policy from hardware-specific implementations with minimal overhead. This pattern produces more testable, more portable, and more maintainable embedded systems.

The full source code for this example is available on [GitHub](https://github.com/sam-w-yellin/dependency-inversion-c).

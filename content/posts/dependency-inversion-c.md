---
title: "Dependency Inversion in C: A Technique for Extensible Embedded Software"
date: 2025-11-16T12:00:00-08:00
draft: false
tags: ["C","architecture","embedded", "dependency", "inversion"]
---
The core value proposition of software is flexibility—but in embedded systems, we often lose that advantage. Codebases become tightly coupled to a specific hardware revision, making even small platform changes expensive and risky. As hardware complexity grows linearly, software complexity grows exponentially. Costs rise. Schedules slip. Eventually organizations become resistant to improving their own products because the software architecture can’t absorb change.

This risk is avoidable. By intentionally separating the high-level business rules from low-level hardware details, we regain the flexibility software is supposed to provide. One of the most effective techniques for achieving this separation is [dependency inversion](https://en.wikipedia.org/wiki/Dependency_inversion_principle). In short, lower-level components implement an interface defined at a higher level. Control still flows from high to low abstraction layers, but the *dependencies* flow upward. High-level code is unaware of how the interface is concretely implemented. In an embedded context, this paradigm allows the software architecture to adapt quickly and cheaply to new hardware iterations without rewriting core logic.

Languages with full runtime polymorphism natively support this paradigm. They can define abstract base classes from which concrete implementations are derived, and the components depending on these interfaces can depend only on the base class type. C does not have such an elegant built-in solution for inverting dependencies. There is no virtual function table or class hierarchy. But a vtable is just a struct of function pointers - something C expresses naturally. So in C, we provide our own layer of indirection using structs of function pointers.

In this post, we explore dependency inversion in C through a concrete, practical example: designing a flexible logging interface.

## A Simple Logger

Suppose we’re developing a system in need of a logging facility. The logger needs two APIs:

1. An initialization function to set up any resources required by the logger.
2. A log function that accepts a message to log.

A naïve implementation might couple the logger directly to a specific output, like stdout. But this has a serious drawback: it becomes difficult to change *how* a component logs. What if we wanted a component to sometimes log to stdout, and sometimes log to a file? What if we are working on an embedded platform and need to emit log messages via UART or SPI, or some other serial interface?

The solution is **dependency inversion**: 

Let’s dig into our logger example. The naïve implementation might look like this: the component depends directly or transitively on a specific logger implementation.
{{< mermaid >}}
graph TD
    Main --> Worker
    Worker --> StdoutLogger
{{< /mermaid >}}

Dependency inversion frees components from needing to know the details of how logging is implemented. It also leads to sparser build dependency graphs - components that depend only on the interface don’t need to be recompiled when implementations change. The resulting architecture looks as follows:
{{< mermaid >}}
graph TD
    Main --> Worker
    Main --> StdoutLogger
    Main --> FileLogger
    Worker --> LoggerInterface
    StdoutLogger --> LoggerInterface
    FileLogger --> LoggerInterface
{{< /mermaid >}}

It’s worth noting that it is not only acceptable but expected that main depends on low-level details. Ideally, `main` - or whatever the entry point for your application is - should be the centralized location where all concrete implementations are defined and injected into the more abstract components.

## Defining the Logger Interface
So, let's define an abstract interface—implemented via function pointers and let the high-level component code depend only on that. A real implementation would probably be a variadic function, but we'll just log a simple string for simplicity.

**File**: `core/components/logger/include/logger_interface.h`
{{< highlight c >}}
{{<readfile "dependency-inversion-c/core/components/logger/include/logger_interface.h" >}}
{{< /highlight >}}

### Implementing Two Concrete Loggers
Now, we create two low-level implementations of the high-level logger interface. Notice how they depend on the logger interface. Components which do logging do not directly utilize either of these concrete implementations.

#### stdout Logger
**File**: `plugins/stdout_logger/include/stdout_logger.h`
{{< highlight c >}}
{{<readfile "dependency-inversion-c/plugins/stdout_logger/include/stdout_logger.h" >}}
{{< /highlight >}}
**File**: `plugins/stdout_logger/src/stdout_logger.c`
{{< highlight c >}}
{{<readfile "dependency-inversion-c/plugins/stdout_logger/src/stdout_logger.c" >}}
{{< /highlight >}}

#### File Logger
**File**: `plugins/file_logger/include/file_logger.h`
{{< highlight c >}}
{{<readfile "dependency-inversion-c/plugins/file_logger/include/file_logger.h" >}}
{{< /highlight >}}
**File**: `plugins/file_logger/src/file_logger.c`
{{< highlight c >}}
{{<readfile "dependency-inversion-c/plugins/file_logger/src/file_logger.c" >}}
{{< /highlight >}}

## Using the Logger in a High-Level Component

Let's create a component that uses a logger. We'll make a generic worker that does some task - in our case, logs a message. This worker depends only on the logging *interface* and not a particular logger.

**File**: `core/components/worker/include/worker.h`
{{< highlight c >}}
{{<readfile "dependency-inversion-c/core/components/worker/include/worker.h" >}}
{{< /highlight >}}

**File**: `core/components/worker/src/worker.c`
{{< highlight c >}}
{{<readfile "dependency-inversion-c/core/components/worker/src/worker.c" >}}
{{< /highlight >}}

## Wiring Together into an Executable
It's now trivial to create a `main` which instantiates a few different workers, and flexibly select which logger to use. We'll create three loggers: two that utilize the stdout implementation and one that logs to a file. Each logger instance maintains its own module name buffer, allowing multiple workers to share the same logger implementation while retaining independent module names.

**File**: `app/main.c`
{{< highlight c >}}
{{<readfile "dependency-inversion-c/app/main.c" >}}
{{< /highlight >}}

If we compile this all together and run, we should see the two stdout loggers emit their messages on the console, and the file logger's output in `log.txt`. The makefile in the source code for this example defines the required target.
```bash
make > /dev/null
./demo
[stdout1] Worker did some work
[stdout2] Worker did some work
cat log.txt
[file1] Worker did some work
```

## Verifying Dependency Inversion

To confirm the components implementing business logic - like `worker` - have no dependency on any concrete implementation, let's investigate the `worker` object file. If we run `make worker.o` to create the object file for our worker component, we can then use `nm` to prove there are no undefined symbols. This demonstrates the dependency inversion worked: the worker component has zero dependency on any concrete logger implementation.

```bash
make worker.o
nm worker.o
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
What are the other options for tackling this sort of problem?

1. Define entirely separate worker components for each logger type: worker-file, worker-stdout, etc
2. Conditionally compile our worker or logger libraries with different implementations depending on the desired configuration.

The first bullet is hitting the problem with a hammer - introduce tons of duplication and maintain tightly coupled interfaces. This makes the project much more expensive to maintain and less scalable. The second option is even worse - we've made it impossible to instantiate multiple types of loggers in a single translation unit. It's my opinion that conditional compilation is nearly always a code smell and should be avoided at all costs - a topic for another time. In short, both alternatives reduce flexibility and maintainability compared to dependency inversion.

## Conclusion

We’ve now walked through implementing dependency inversion in C - a language without native support for dynamic polymorphism. Using function-pointer interfaces allows you to decouple high-level policy from hardware-specific implementations with minimal overhead. This pattern produces more testable, more portable, and more maintainable embedded systems.

The full source code for this example is available on [GitHub](https://github.com/sam-w-yellin/dependency-inversion-c). If you’d like to discuss this pattern or how it might apply to your system, feel free to reach out: sam.w.yellin@gmail.com.

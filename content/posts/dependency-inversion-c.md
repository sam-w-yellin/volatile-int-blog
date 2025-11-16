---
title: "Dependency inversion in C: struct-of-function-pointers and building libraries"
date: 2025-11-15T12:00:00-08:00
draft: false
tags: ["C","architecture","embedded"]
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

A naïve embedded implementation might tightly couple these to a serial port:

```c
#include <stdio.h>
#include <string.h>

// Hypothetical serial driver
#include "serial.h"

static char g_module_name[32];

void init_logger(const char *module_name, size_t name_len) {
    serial_init(115200);
    size_t copy_len = name_len < (sizeof(g_module_name) - 1)
                        ? name_len
                        : (sizeof(g_module_name) - 1);

    memcpy(g_module_name, module_name, copy_len);
    g_module_name[copy_len] = '\0';
}

void log(const char *msg) {
    serial_write("[", 1);
    serial_write(g_module_name, strlen(g_module_name));
    serial_write("] ", 2);
    serial_write(msg, strlen(msg));
    serial_write("\n", 1);
}
```

Any code depending on this logger would now be untestable on platforms without a serial port.

## Defining the Interface
Instead, we define an abstract interface—implemented via function pointers—and let high-level code depend only on that.
```c
#include <stddef.h>

typedef void (*logger_init_fn)(const char *module_name, size_t name_len);

typedef void (*logger_log_fn)(const char *msg);

// Struct representing a logger interface (vtable-like)
typedef struct {
    logger_init_fn init;
    logger_log_fn log;
} logger_interface_t;
```

### Implementing Two Concrete Loggers
Now, we create two low-level implementations of the high-level logger interface. Note how they depend on the logger *interface*. Depending on a logger does **not** imply depending on a concrete transport.

#### stdout Logger
```c
#include <stdio.h>
#include <string.h>
#include "logger_interface.h"

static char stdout_module_name[32];

void stdout_logger_init(const char *module_name, size_t name_len) {
    size_t copy_len = name_len < (sizeof(stdout_module_name) - 1)
                        ? name_len
                        : (sizeof(stdout_module_name) - 1);

    memcpy(stdout_module_name, module_name, copy_len);
    stdout_module_name[copy_len] = '\0';
}

void stdout_logger_log(const char *msg) {
    printf("[%s] %s\n", stdout_module_name, msg);
}

logger_interface_t stdout_logger = {
    .init = stdout_logger_init,
    .log  = stdout_logger_log
};
```

#### Serial Port Logger
```c
#include <string.h>
#include "logger_interface.h"
#include "serial.h"

static char serial_module_name[32];

void serial_logger_init(const char *module_name, size_t name_len) {
    serial_init(115200);

    size_t copy_len = name_len < (sizeof(serial_module_name) - 1)
                        ? name_len
                        : (sizeof(serial_module_name) - 1);

    memcpy(serial_module_name, module_name, copy_len);
    serial_module_name[copy_len] = '\0';
}

void serial_logger_log(const char *msg) {
    serial_write("[", 1);
    serial_write(serial_module_name, strlen(serial_module_name));
    serial_write("] ", 2);
    serial_write(msg, strlen(msg));
    serial_write("\n", 1);
}

logger_interface_t serial_logger = {
    .init = serial_logger_init,
    .log  = serial_logger_log
};
```

## High-Level Logger (Policy Layer)

Consumers of the logger should not depend on any specific implementation. The question is: how does the high-level logger know which implementation to use?

Two strategies:

1. Pass the interface struct pointer into every logging function.
2. Pass the interface only into the initialization function, which stores it in a file-scope static variable.

The first approach pollutes function signatures.  
The second is cleaner but requires careful linking discipline and is not thread-safe.

### Strategy 1: Passing the Pointer to Each API
```c
#include "logger_interface.h"

void log(const logger_interface_t *logger, const char *msg) {
    logger->log(msg);
}

void init_logger(const logger_interface_t *logger, const char *module_name, size_t name_len) {
    logger->init(module_name, name_len);
}
```

### Strategy 2: Storing the Pointer Only During Initialization
```c
#include "logger_interface.h"

static logger_interface_t *active_logger = NULL;

void log(const char *msg) {
    if (active_logger != NULL) {
        active_logger->log(msg);
    }
}

void init_logger(const logger_interface_t *logger, const char *module_name, size_t name_len) {
    active_logger = logger;
    active_logger->init(module_name, name_len);
}
```

## Using the Logger

Let's look at how we can use these 
```c
#include "logger_interface.h"
#include "logger.h"     // High-level logger
#include "logger_stdout.h"  // Logger implementation

int main(void) {
    init_logger(&stdout_logger, "MAIN", 4);

    log("Hello, World!");

    return 0;
}
```
Compile and run:
```bash
gcc -Wall -Wextra \
    main.c \
    logger.c \
    logger_stdout.c \
    -o logger_example

./logger_example
```

## Verifying Dependency Inversion

To confirm our high-level logger has*no dependency on any concrete implementation, compile it into a static library:
```bash
gcc -c logger.c -o logger.o
ar rcs liblogger.a logger.o
```
Inspect the symbol table with `nm`:
```bash
nm liblogger_top.a
00000000 T init_logger
00000010 T log
                 U logger_init_fn
                 U logger_log_fn
```
We see dependencies only on the abstract interface symbols.

## Conclusion

We’ve now walked through implementing dependency inversion in C without runtime polymorphism. Using function-pointer interfaces allows you to decouple high-level policy from hardware-specific implementations with minimal overhead. This pattern produces more testable, more portable, and more maintainable embedded systems.
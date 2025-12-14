---
title: "Crunch Dev Blog #1: Designing a Message Definition Protocol for the Embedded World"
date: 2025-12-01T13:00:00-08:00
draft: false
tags: ["crunch", "C++", "serdes", "serialization", "message"]
---

There are [a](https://protobuf.dev/) [lot](https://mavlink.io/en/) [of](https://github.com/6over3/bebop) [different](https://msgpack.org/index.html) [tools](https://flatbuffers.dev/) for defining structured messages and working with their serialized representations for communication over-the-wire. I have worked with open source, closed source, and hand-rolled toolchains tackling this problem. Every tool I have seen has a serious design flaw that makes them very difficult to use correctly in mission-critical contexts. Validation - both of individual fields and of whole messages - is completely decoupled from the message definitions. This is a fundamental gap in how messages are specified. A comprehensive description of what makes a message valid is equally important in system design as the types and names of the fields.

About a year ago I started thinking about a better type of message definition format. One designed with embedded targets and validation at the forefront. Enter **Crunch**: a message definition protocol with a few design principles that distinguish it from other options.

1. Semantic field verification is *opt-out* rather than  *opt-in*. Fields are defined not only with a name and a type, but a set of validity criteria.
2. The serialization protocol is swappable. You can choose a flexible serialization based on tag-length-value, or a speed optimized static serialization, or define your own.
3. No external dependencies (for C++ targets. Some other languages may require libraries for API binding).
4. Message integrity checks are built-in, and opt-out.

I am going to document the development of *Crunch* in a series of blog posts. This first one is going to focus on The Dream - requirements, high level interface design, the development environment. I'll also lay out the development roadmap.

# Background on Message Protocols
We just don't have good support for message protocols that treat validity and serialization flexibility as first-class concerns.

`protobuf` is probably the most widely used message definition and serialization protocol there is. It's the backbone of `gRPC` and a huge number of software projects. The folks at [buf](https://buf.build/) realized that semantic validation of fields and messages is a critical feature gap and developed the [protovalidate extension](https://github.com/bufbuild/protovalidate) for `protobuf` - the backbone of gRPC and as far as I am aware, the most widely used message definition and serialization protocol around.  However, the implementations of this tool have a number of drawbacks.

1. `protovalidate` relies on runtime parsing of Google CEL. As a result, it is incompatible with environments that cannot support dynamic memory allocation. 
2. `protovalidate` has a long list of dependencies and is not trivial to set up - especially for systems not using `buf` to manage `protobuf`-related dependencies.
3. `protobuf` itself does not provide strong support for embedded systems - relying instead on third-party extensions such as [nanopb](https://github.com/nanopb/nanopb) and [capnproto](https://capnproto.org/). Any attempts to extend `protovalidate` support to embedded systems involves bridging multiple plugins maintained by separate folks.

Protocols like MAVLink do include some built-in structural validation (checksums, CRC extra, strict field layouts), but they don’t provide semantic validation of field values or cross-field relationships.

There are other problems with many of these protocols which make them unfit for some embedded and resource constrained applications. `protobuf`, and many other protocols, provide only a single serialization/deseralization protocol which optimizes for on-the-wire size, flexibility, and extensibility of message definitions. This is an important consideration. But its not the *only* consideration. The tag-length-value schema sacrifices serialization and deserialization speed for protocol flexibility. The custom message frameworks I've seen in-house in my career utilized more rigid protocols that enabled much faster message processing.  

**Comparison of Popular Message Frameworks Against Crunch's Design Goals**

| System         | Static Memory Allocation | Built-In Integrity Checking (CRC/Parity) | Flexible Serialization Protocol (Choose TLV vs Static Layout) | Semantic Field Validation |
|----------------|--------------------------|------------------------------------------|----------------------------------------------------------------|---------------------------|
| **Crunch**     | ✔ | ✔ | ✔ | ✔ |
| **Protobuf**   | ✘ | ✘ | ✘ | ✘ |
| **nanopb**     | ✔ | ✘ | ✘ | ✘ |
| **Bebop**      | ✘ | ✘ | ✘ | ✘ |
| **MAVLink**    | ✔ | ✔ | ✘ | ✘ |
| **FlatBuffers**| ✔ | ✘ | ✘ | ✘ |
| **MessagePack**| ✘ | ✘ | ✘ | ✘ |


# Requirements and Design Goals
Here's what I want *Crunch* to do:
1. Validation of fields and messages must be *opt-out* instead of *opt-in* and a first-class aspect of message definition.
2. Function without any dynamic memory allocation.
3. Serialization formats should be plugins, all working from a single message definition. *Crunch* will include both a TLV and a read/write speed optimized serialization format.
4. Create highly performant bindings for working with messages in any languages.

Some of the things I'm not setting out to do (note some of these may be possible, but I won't constrain the design to enable them):
1. Make the messages compatible with `protobuf` wire format or any other language specification (Although it would be technically possible via the extensible serialization format, there will be no attempt to unify with the `.proto` format used to declare `protobuf` messages).
2. Be optimally performant for every single computer architecture and memory hierarchy.
3. Have *native* implementations in languages outside C++.
4. Be compatible with C++ versions older than C++23.

# The Dream
Once I decided to actually build this thing, the very first step I took was to try and figure out what I wanted the ergonomics to be. At least in the first iteration, message definition is going to be done directly in C++ - abstracting out a full DSL is a future problem and I want to make sure I have a really solid handle on how I want this to work under the hood with the language I use most often. 

This is the very first interface design I've come up with and is what I'm going to implement to start out. In this example, we have two messages. One of the messages uses the other as a field. This is not a comprehensive description of the protocol, but it is a north star through development. Implementation of a system that meets this interface design is going to be the first major milestone.

## Message Definition
First, the message definition format:
```c++
#include <crunch>

enum class MyEnum : Crunch::EnumType {
    E1,
    E2,
    E3
};

struct BarMessage {
    using Crunch::types;
    using Crunch::validators;
    types::VInt8<
        0 /* field ID */,
        validators::positive
    > barint;
    types::VArray<
        1,
        3 /* 3 elements */,
        VInt8<
            /* no id for array elements */
            validators::positive
        >;
    > bararray;
}

struct FooMessage {
    using Crunch::types;
    using Crunch::validators;

    types::VInt32<
        0,
        validators::positive,
        validators::required
    > field1;
    types::VEnum<
        1,
        MyEnum,
        validators::enum_in<E1, E2>
    > field2;
    types::VMsg<
        2,
        BarMessage,
        validators::required
    > field3;
    types::VBool<
        3,
        BarMessage,
        validators::none
    > field4;
    static constexpr auto const (FooMessage& m) validate -> std::expected<void, validators::ValidationError> {
        if (m.field1.get() > m.field3.get().barint.get())
            return {};
        return std::unexpected({"field1 must be > barint"})
    };
}
```
Highlighting a few of the important aspects of the design here:
1. Everything that can be done at compile time is done at compile time. Template parameters, `constexpr` field and message validators.
2. We don't use inheritance. We'll see this in the serdes section, but instead of inheriting from a base `Message` class, we utilize template concepts to enforce messages are serializable per the protocol(s).
3. We use a monadic type to bubble up errors. *Crunch* will not use exceptions so that it is totally applicable to environments with real-time requirements.
4. You have to *opt-out* of validation. There is no option to omit a validator. If you don't want a validator, you have to specify that with the `none` validator.
5. Messages have a field ID which will be used by serialization protocols requiring a field tag.

This is omitting some complexity that will be present in the final implementation. For example, until we have reflection in C++26 we'll probably need to maintain a macro that defines a list of field names. But the spirit is captured above.

## Message Serialization/Deserialization

This example shows how messages can be instantiated, validated, and sent/received over the wire.

```c++
#include <crunch>
#include "messages/foo.h"

int main() {
    FooMessage m;
    static auto foomsg_buffer = Crunch::GetBuffer<FooMessage, Crunch::CRC16, Crunch::serdes::TLV>();

    const auto r = Crunch::Validate(m);
    if (r.has_error()) {
        std::print("{}", r.error());
    }

    Crunch::Serialize(foomsg_buffer, m);

    if (Crunch::Deserialize(foomsg_buffer)) {
        std::printf("{}", r.error());
    }
}
```
Some of the important things to note:
1. Message classes do not provide utilities like `GetBuffer`, `Serialize`, or `Deserialize`. We don't want to involve inheritance in our implementation. Instead, *Crunch* provides these functions as templatized utilities that require the `CrunchMessage` on the message template parameters.
2. To meet the core requirement of flexible serialization, we provide a parameter to the interface for buffer creation that specifies which protocol to use for serialization.
3. I think its critically important we include CRC by default in all message serialization/deserialization. *Crunch* will ship with an efficient CRC16 - with the option to use a parity bit, totally omit, or define your own CRC algorithm.

As previously stated, the first iteration of the design will use the C++ class definitions as the source of truth for message definitions. But its pretty evident that given sufficient restrictions on the types via concepts, we could derive a domain specific language in the future.

## Fields
Let's go through what field types I plan to support and what the programatic interface is to each one.

### Field Types
We need to support a comprehensive set of field types. The field types will dictate what validators can be utilized, ergonomics for accessing the field, and how they are serialized for any given serialization protocol. The most important principle in creating these types is that all field sizes must be known at compile time.

To start out, I plan to have *Crunch* support these field types:

1. Signed Integers (Int8, Int16, Int32, Int64)
2. Unsigned Integers (UInt8, UInt16, UInt32, UInt64)
3. Floating Point (F32, D64)
4. String<MaxSize>
5. Bool
6. Array<MaxCount, Type>
7. Map<MaxCount, Key, Value>
8. User-defined messages

### Field Accessors
Building in validation means we need to establish a reasonable interface and be clear when the validators run. We want the following behavior:

1. When we write a new value into a field, it should be validated.
2. Reads from a message should assume the message has been validated.
3. We should make it possible to sidestep validation at any point, but it should be an *explicit* request.
4. We should have a notion of if a field is "set" or not. This will enable TLV serialization schemes to omit them in serialization, and for static schemes to specify if there is garbage values in a field or not during deserialization.
5. Arrays, Maps, and User-defined messages all need to validate both any other types they use, and themselves. However, setting only should run the used-type validators. The top-level map/array/user message validator should run only when `Crunch::Validate`, `Crunch::Serialize`, or `Crunch::Deserialize` is called. The reason for this is that its super common to "build up" messages and aggregate types such that their intermediate forms are not valid, and we don't want to make that difficult. 

Let's say that we have a message `Foo` with the following definition:

```c++
struct FooMessage {
    using Crunch::types;
    using Crunch::validators;

    types::VInt32<
        0,
        validators::positive
    > field1;
    types::VArray<
        1,
        20,
        VInt16<
            validators::less_than<100>
        >
        validators::not_empty
    > field2;
    static constexpr auto const (FooMessage& m) -> std::expected<void, validators::ValidationError> {
        return {};
    };
}
```

We should implement the following ergonomics for field access from outside the class:
```c++
    FooMessage m;
    {
        const auto success = m.field1.set(10); // passes validation
        if (success.has_error()) {
            // ...
        }
    }

    {
        const auto success = m.field2.add(101); // should fail
        if (success.has_error()) {
            // ...
        }
    }

    // fails on field2 length
    if (Crunch::Validate(m).has_error()) {
        // ...
    };
```

### Field Validators
*Crunch* should provide a reasonable set of validators to apply to fields. We'll do a whole post on designing the validators. We can establish a few ground rules now though. Field validators:

1. Are `constexpr` functions.
2. Are pure functions.
3. Do not depend on the values in other fields in the message.
4. Are not recursive.
5. Can be defined by a user

## Message Definition
The message definitions are defined as classes. The classes need to maintain the following properties:

1. All member variables must be a *Crunch* field.
2. Define a `static constexpr` message-level validator for cross-field validation, which takes only a `const` reference to the message type itself as input.
3. Ensure all field IDs are unique.

# The Roadmap
There's quite a lot of work ahead. The following is my plan at the outset and is very much subject to change as I learn more about the problem space. The development roadmap is as folows:

## Basic End-to-End functionality
I want to get something up and running with full message definition with field and message level validation and serialization/deseralization running ASAP rather than building out any one layer in its entirety to start out. This should validate a lot of the basic ideas and ergonomic design. I'll omit any unit level testing at this phase because I'm really going to be focused on proof-of-concept.

1. Get a dev environment set up. Build system selection, linters, static analysis, CI. Cross-compilation support is critical from day 1 because I need to set up a way to test functionality and performance on an embedded device.
2. Implement the message and field template concepts
3. Implement the signed integer types and a small number of validators.
4. Support nested messages.
5. Implement the core *Crunch* APIs: `GetBuffer`, `Validate`, `Serialize`, `Deserialize`
6. Implement a single serdes protocol.
7. Write an end-to-end integration test on a single computer (like my laptop).
8. Demonstrate communication via *Crunch* between an embedded device and another compute node.

## Supporting Infrastructure
Once the end-to-end functionality is in place, I want to create a more robust development environment.

1. Implement unit tests around functionality I'm confident will be sticking around. 
2. Implement a fuzz testing framework for the serialization protocols and validators. This not only means building integrations with fuzzers like AFL, but also tooling to generate arbitrary message definitions for use in the fuzzers.
3. Track metrics such as code size, on-target performance, and test coverage in CI.
4. Set up some sort of cross-compute-node integration test infrastructure for quickly iterating on cross-platform tests.
5. Implement tests utilizing multiple compilers - at least both gcc and clang.

## Full Field Type Support
After we have all the supporting infrastructure for testing in place, build out the rest of the types.

1. Support all of the scalar types enumerated above.
2. Support the aggregate types.

For each of these types, make sure we have everything well covered via our test infrastructure.

## The Second Serdes Protocol 
I'm not yet sure which protocol I'll start with - but I need to implement the other one at this point.

1. Define the second serdes protocol, complete with testing.
2. Expand test coverage to verify behavior when `Serialize`/`Deserialize` encounter a message encoded in the wrong protocol.
3. Implement a "universal" `Deserialize` API that takes in a message and inspects the header to select the right deserialization implementation.

## Language Interoperability
At this point, *Crunch* will be fully featured for C++. The next challenge is language interoperability. My plan is to generate thin wrappers around the C++ APIs. I'm going to target getting it working with two languages. 

1. Demonstrate interoperability with C.
2. Demonstrate interoperability with Rust.

Long term, I think that this is an excellent opportunity to take advantage of C++26's reflection to autogenerate out native bindings in other languages. Which leads to the last part of the plan.

## Domain Specific Language
Once *Crunch* is fully functional, and we have demonstrated the ability to go from C++ -> other languages, we'll finally abstract out the final DSL so we can write `.crunch` message definitions. Of everything in this project, this step is what I know the least about. Maybe that's why I'm delaying it so long in the process. I have not implemented or utilized a lexer before, so this feels a bit like the wild west. I don't want to start on it until I'm really confident in the C++ API and am confident that I can bind that API to other languages.

# Conclusion
It's hard not to be reminded of the [classic XKCD on standards](https://xkcd.com/927/) when thinking about introducing a new message format. Even still, I believe that *Crunch*'s two primary value propositions of opt-out validation and static memory allocation are critical to a significant number of technical domains, and are underserved by the existing set of options.

I wouldn't recommend *Crunch* for everyone. It is not designed to be as widely supported as `protobuf`, or to serve a niche domain such as `MAVLINK`. But for embedded projects that need to prioritize speed and program correctness above all else, I hope it becomes a solid choice.

If you're interested in following along with the *Crunch* development journey consider subscribing to the [newsletter](https://volatileint.dev/newsletter)!
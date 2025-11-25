---
title: "Implementing a Generic Closed-Loop Controller in Modern C++"
date: 2025-11-16T12:00:00-08:00
draft: false
build:
  list: false
  publishResources: true
tags: ["C++","setpoint", "concept", "embedded", "template", "lambda"]
---

# A Case Study in Software Evolution
In this post, we're going to implement a generic C++ component capable of implementing a wide variety of closed-loop control systems. We'll end with a controller capable of implementing whatever algorithm you'd want, with configurable setpoints, consistent error handling, and support for logging using modern C++ concepts. 

Before we dive in, note that the final code is not simple. In the course of reading, you may balk at the complexity introduced for the sake of consistent abstraction. Hopefully, the culmination of what we have built presented in the last section of the post sufficiently motivates the complexity tradeoff. That being said, this approach is **not** a one-size-fits-all solution.  hWhile this approach
First, let's kick things off with a scenario.

## A Simple Bang-Bang Controller
Picture yourself as an embedded software engineer working at a scrappy startup designing a revolutionary new technology with a healthy mix of hardware and software co-development. Your co-worker - a thermal engineer responsible for maintaining the health of a temperature-sensitive component of the system - approaches you, asking for software to regulate the temperature of their system:

- A thermoelectric cooler (henceforth referred to as a TEC) connected to the component that needs to be cooled. The engineer connected this to a simple switch which will apply either 0 or 100% power.
- A thermocouple connected to the device being cooled to sense the temperature.
- A microcontroller wired to control the TEC using the thermocouple as feedback.

< add chart >

The thermal engineer explains that the requirements here are really simple. All their analysis shows that they just need to keep the component between 20 and 80 degrees Celsius. The component generates heat during operation, so the only element needed is the cooler. The temperature is expected to change slowly, so they suspect that it will be sufficient to just occasionally monitor the temperature, and if it gets to 80 degrees, turn on the cooler and keep it on until the thermocouple reports 20 degrees.

You write a very minimal program:
```c++
#include <thread>
#include <chrono>
#include "gpio.hpp"
#include "adc.hpp"

int32_t raw_adc_to_deg_uc(uint16_t raw) {
        // Simple conversion example
        return static_cast<float>(raw) * 0.1f;
}

void control_tec(Gpio& tec, Adc& thermocouple) {
    const auto raw_temp_adc = thermocouple.read();
    const auto temp_uc = raw_adc_to_deg_uc(raw_temp_adc);

    if (temp_uc >= 80) {
        tec.Set(true);
    } else if (temp_uc <= 20) {
        tec.Set(false);
    }
}

int main() {
    Gpio tec{ 1 }; // TEC is on pin 1
    Adc thermocouple{ 2 }; // thermocouple on pin 2

    tec.Configure(Gpio::Output);
    thermocouple.Configure();

    while(true) {
        control_tec(tec, thermocouple);
        std::this_thread::sleep_for(std::chrono::minutes(1));
    }
}
```

This implementation is functional, but it has a number of problems. 

## Implementing Controller Scaffolding
What happens when the control law needs to change? What about when the GPIO is changed for a PWM output, or when the on-chip ADC is replaced with a more accurate off-chip device?

And what about testability? The folks who write these control laws are often not the embedded software teams. The control law is inextricably tied to the hardware used for gathering feedback and effecting output. Is the controls team expected to maintain simulation infrastructure for hardware to iterate on control laws?

How do we make sure this code actually serves all our purposes? It doesn't appear to handle errors. It doesn't provide insight into the status of the controller. It doesn't have any sense of configuration - the bounds are hard-coded. We have no facilities to log state, or understand if our feedback or actuation mechanisms have failed to accomplish their tasks.

Finally, what about consistency across the codebase? If we are working in a system with lot of controllers - as many embedded systems and embedded software platforms must support - how do we make sure they're all implemented in a consistent way? consistency is critically important for maintaining developer velocity in complex code bases, and so it is a matter of utmost importance that we do not arbitrarily do the same thing in wildly different ways.

These problems can be addressed by providing *scaffolding* for the domain of control law implementations. 

### Requirements for a Generic Controller
Requirements for the scaffolding can be derived from the above use cases:

1. Control laws shall be independently testable from any specific hardware configuration.
2. Controller feedback and actuation mechanisms shall explicitly handle all IO and configuration errors.
3. Controllers shall emit representations of their current state suitable for logging.
4. The data flow between the feedback, control law, and actuation mechanism shall be explicitly defined in order to enforce consistency.

We also have these design goals: 
1. It should be *easy* to write control laws that abide by the requirements.
2. To the extent possible the constraints imposed by the requirements should be checked at compile-time.

### Components of a Controller
The controller consists of three logically distinct components: 
1. A purely mathematical control law that receives inputs and computes outputs in order to achieve some goal.
2. A feedback mechanism which interacts with some external interface to collect measurements.
3. An output mechanism which effects the values commanded by the control law into the world.

In a given step of the controller data flows first into the feedback component. Then, the data is converted to a format useable by the control law which uses the feedback to calculate the next output value. The output value from the controller is converted to the required data type to emit to a physical component. 

{{< mermaid >}}
flowchart LR
  S[Feedback Sensor]
  FB[Feedback Component<br/>e.g., AdcFeedback]
  LAW[Control Law<br/>e.g., BangBangLaw]
  ACT[Actuator Component<br/>e.g., GpioActuator]
  AHW[Actuator Hardware]

  S -->|Raw measurement| FB
  FB -->|Converted measurement| LAW
  LAW -->|Command / control output| ACT
  ACT -->|Actuation signal| AHW
{{< /mermaid >}}

Let's ground this in our TEC example. The ADC is the feedback. ADCs read a voltage. The control law for a TEC is going to deal with temperatures in degrees C, so the voltage must be converted to degrees. The control law emits a boolean - the TEC is on or off. For the bang-bang controller discussed earlier, the output GPIO uses the same data type as the controller output.

It is now clear that the data types for input and output from the control law *constrain* the feedback and actuator components.. The control law cannot provide its own adapters - or we ahve violated the independent testability requirement. The components close to the hardware must implement their own conversion layers.

Template concepts - introduced in C++20 - are naturally able to express these constraints. Each of these components - feedback, actuator, law - can be defined as a `concept`.
 
#### ControlLaw Interface
Let's start out writing a concept for our most abstract component - the `ControlLaw`. The concept requires that each law defines its input and output types - `Measurement` and `Command` respectively. Laws must also define a `State` type.

There are two required APIs for a `ControlLaw`:

1. `Initialize` configures the internal `ControlLaw` state. It must return the initialized state, or a string in the case of an error.
2. `Compute` takes in a measurement, and returns a tuple of the next `Command` and its current `State`. 

These interfaces enforce two of our requirements - that the controller implements error handling, and that we have visibility into the current state of the controller. 

```c++
template <typename C>
concept ControlLaw = requires(C law, const typename C::Measurement& m) {
    typename C::Measurement;
    typename C::Command;
    typename C::State;

    { law.Initialize() } -> std::same_as<std::expected<typename C::State, std::string>>;

    {
        law.Compute(m)
    }
    -> std::same_as<std::expected<std::pair<typename C::Command, typename C::State>, std::string>>;
};
```

Notice that the controller is completely independent of the actuator and feedback mechanisms. This decoupling enables our testability requirement - a class abiding by the `ControlLaw` concept can be instantiated independent of the larger controller infrastructure.

#### External Interfaces
Now let's implement our Feedback and Actuator concepts. As discussed earlier, these interfaces are dependent on the `ControlLaw`'s defined data types. This is represented in our concepts as a template parameter for the `ControlLaw`, from which we extract the `Measurement` and `Command` data types:

```c++
template <typename FB, typename Law>
concept Feedback = ControlLaw<Law> && requires(FB fb) {
    typename FB::State;

    { fb.Configure() } -> std::same_as<std::optional<std::string>>;

    {
        fb.Read()
    } -> std::same_as<
        std::expected<std::pair<typename Law::Measurement, typename FB::State>, std::string>>;

    { fb.Convert(std::declval<typename FB::Raw>()) } -> std::same_as<typename Law::Measurement>;
};

template <typename AC, typename Law>
concept Actuator = ControlLaw<Law> && requires(AC ac, const typename Law::Command& cmd) {
    typename AC::State;

    { ac.Configure() } -> std::same_as<std::optional<std::string>>;

    { ac.Write(cmd) } -> std::same_as<std::expected<typename AC::State, std::string>>;

    { ac.Convert(std::declval<typename Law::Command>()) } -> std::same_as<typename AC::State>;
};
```

These concepts closely mirror each other. They are designed to encapsulate external-facing dependencies and abstract away conversion to the `ControlLaw`types. The interface entails:
1. A `Configure` function that ensures the output interfaces are ready for control and returns a string on error. 
2. A function to interface with the external world - `Read` and `Write` for the `Feedback` and `Actuator` interfaces respectively. 
3. A `Convert` function that goes from the respective internal type to the data type expected by the template `ControlLaw`.

Just as in the `ControlLaw` concept, `std::expected` and `std::optional` return types are used to enforce error handling and state reporting.

#### Integrated Controller
Finally, we implement an integrated `Controller` class. This class is templated on a specific set of `Actuator`, `Feedback`, and `ControlLaw` concept implementations. It orchestrates the dataflow between components and exposes the external interface to the represented controller.

```c++
template <ControlLaw Law, Feedback<Law> FB, Actuator<Law> ACT>
class Controller
{
   public:
    using Measurement = typename Law::Measurement;
    using Command = typename Law::Command;

    Controller(FB& feedback, ACT& actuator, Law& law) : fb_(feedback), act_(actuator), law_(law) {}

    std::expected<typename Law::State, std::string> Initialize()
    {
        if (auto err = fb_.Configure()) return std::unexpected(*err);
        if (auto err = act_.Configure()) return std::unexpected(*err);
        auto law_state = law_.Initialize();
        if (!law_state.has_value()) return std::unexpected(law_state.error());
        return law_state.value();
    }

    std::expected<ControllerState<FB, Law, ACT>, std::string> Step()
    {
        auto fb_result = fb_.Read();
        if (!fb_result.has_value()) return std::unexpected(fb_result.error());
        auto [measurement, fb_state] = *fb_result;

        auto law_result = law_.Compute(measurement);
        if (!law_result.has_value()) return std::unexpected(law_result.error());
        auto [command, law_state] = *law_result;

        auto act_result = act_.Write(command);
        if (!act_result.has_value()) return std::unexpected(act_result.error());
        auto act_state = *act_result;

        ControllerState<FB, Law, ACT> state{
            .feedback_state = fb_state, .control_state = law_state, .actuator_state = act_state};
        return state;
    }

   private:
    FB& fb_;
    ACT& act_;
    Law& law_;
};
```

## Revisiting the Bang-Bang Controller
All the pieces are now in place to implement controllers using our generic template scaffolding. The following example implements the simple bang-bang controller for a TEC discussed earlier.

First - we need to create concrete implementations of our ADC feedback, GPIO actuation, and bang-bang control law concepts. The implementations are included below without much exposition - they are fairly straightforward components.

**File**: `example/components/feedback_controller/plugins/feedback/include/adc_feedback.hpp`
{{< highlight c >}}
{{<readfile "feedback-controller/example/components/feedback_controller/plugins/feedback/include/adc_feedback.hpp" >}}
{{< /highlight >}}

**File**: `example/components/feedback_controller/plugins/actuators/include/gpio_actuator.hpp`
{{< highlight c >}}
{{<readfile "feedback-controller/example/components/feedback_controller/plugins/actuators/include/gpio_actuator.hpp" >}}
{{< /highlight >}}

**File**: `example/components/feedback_controller/plugins/laws/include/bangbang.hpp`
{{< highlight c >}}
{{<readfile "feedback-controller/example/components/feedback_controller/plugins/actuators/include/gpio_actuator.hpp" >}}
{{< /highlight >}}

It is worth noting the framework is flexible for whatever configuration any given aspect of the controller requires. In this case, we provide min and max thresholds for the bang-bang controller.

With these components - and the simulated concrete ADC and GPIO interfaces defined by the (linked GitHub repo)[https://github.com/sam-w-yellin/feedback-controller]  - we can actually instantiate and execute a controller. Instantiation is simple:

```c++
// Simulated hardware
SimulatedAdc adc;
SimulatedGpio gpio;

// Bang-bang control law
BangBangLaw law(20 /* min */, 80 /* max */);

// ADC Feedback
auto convert_adc = [](uint16_t raw) -> BangBangLaw::Measurement
{ return static_cast<BangBangLaw::Measurement>(raw) / 10; };
AdcFeedback<BangBangLaw, decltype(convert_adc)> feedback(adc, convert_adc);

// GPIO Actuator
auto convert_gpio = [](bool cmd) -> BangBangLaw::Command { return cmd; };
GpioActuator<BangBangLaw, decltype(convert_gpio)> actuator(gpio, convert_gpio);

// Integrated controller
Controller<BangBangLaw, decltype(feedback), decltype(actuator)> controller(feedback, actuator,
                                                                            law);
```

We can now use the `controller` object to enact our control - first using the `Initialize` API to setup the hardware and control laws, and then the `Step` function to read inputs into and write outputs from the control law.

Let's briefly explore what kinds of errors we get if we violate our concepts. One of the constraints is that our `Convert` functions map to the types expected by the templated `ControlLaw`. So, instead of converting the ADC read to an `int32_t` (implicit through the `BangBangLaw::Measurement`), let's try converting to a `float`:

```c++
auto convert_adc = [](uint16_t raw) -> float
{ return static_cast<float>(raw) / 10; };
AdcFeedback<BangBangLaw, decltype(convert_adc)> feedback(adc, convert_adc);
```

We're going to get a compile-time error - and if you're used to C++ templates you'll agree this error is *much* nicer than errors you get without concepts:

```bash
/Users/syellin/blog/examples/feedback-controller/example/bangbang_sim/main.cpp:85:5: error: constraints not satisfied for class template 'Controller' [with Law = BangBangLaw, FB = AdcFeedback<BangBangLaw, (lambda at /Users/syellin/blog/examples/feedback-controller/example/bangbang_sim/main.cpp:76:24)>, ACT = GpioActuator<BangBangLaw, (lambda at /Users/syellin/blog/examples/feedback-controller/example/bangbang_sim/main.cpp:81:25)>]
   85 |     Controller<BangBangLaw, decltype(feedback), decltype(actuator)> controller(feedback, actuator,
      |     ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
/Users/syellin/blog/examples/feedback-controller/include/feedback_controller_interface.hpp:56:27: note: because 'Feedback<AdcFeedback<BangBangLaw, (lambda at /Users/syellin/blog/examples/feedback-controller/example/bangbang_sim/main.cpp:76:24)>, BangBangLaw>' evaluated to false
   56 | template <ControlLaw Law, Feedback<Law> FB, Actuator<Law> ACT>
      |                           ^
/Users/syellin/blog/examples/feedback-controller/include/feedback_controller_interface.hpp:42:57: note: because type constraint 'std::same_as<float, typename BangBangLaw::Measurement>' was not satisfied:
   42 |     { fb.Convert(std::declval<typename FB::Raw>()) } -> std::same_as<typename Law::Measurement>;
```

We are told *exactly* what is wrong - our `Convert` function return type of `float` is not equal to `BangBangLaw::Measurement`. Similar errors will occur if any other aspect of the contract is violated. So we have achieved the design goals - we both have compile-time checks that we are well formed, and the code is easy to debug when done incorrectly.

## A Control Simulation
The following code utilizes the bang-bang controller - instantiated through our scaffolding - to run 1000 steps of the control simulation. This example demonstrates all our requirements - error handling, loggability, strict enforcement at compile-time.

**File**: `example/bangbang_sim/main.cpp`
{{< highlight c >}}
{{<readfile "feedback-controller/example/components/feedback_controller/plugins/actuators/include/gpio_actuator.hpp" >}}
{{< /highlight >}}

After building and executing this simulation, and running the included `graph_log.py` script - you'll see an output like this:

![bang-bang controller output](bangbang-graph.png)

## Exercises for the Reader
I encourage you to try extending this functionality yourself! Consider these two challenges:

1. Implement the `Convert` function at compile-time rather than as a lambda passed into the constructor.
2. Enable runtime configuration of the controller via dependency inversion by creating a base class for each of the concepts.

These exercises both may be very useful to some applications, and are great ways to apply the patterns learned in this post.

## Conclusion
The design we've implemented demonstrates all the requirements and design goals we set out to achieve. If you are operating in an ecosystem of control laws, this pattern is almost certainly useful. I have implemented similar patterns at multiple organizations and reaped the benefits of consistent control interfaces and implementations at scale. The benefits of cleanly defined component boundaries and separate responsibilities, enforced error checking and state exposure, and swappable feedback/actuator/law implementations come at the cost of complexity. Not only is this implementation more verbose than the simple controller we saw at the beginning - it is undoubtedly more complex. Understanding this implementation fully demands quite a lot from the code maintainers. To reaffirm - this solution makes sense in an *ecosystem* of controllers.

I hope the patterns reviewed here are useable in your own code. The feedback controller interface is available on [GitHub](https://github.com/sam-w-yellin/feedback-controller) as a single-header to anyone who wants to use it or try out the exercises outlined above. The simulation framework included in the repo also could prove useful to anyone iterating on controllers.

If you found this article valuable, consider subscribing to the [newsletter](https://volatileint.dev/newsletter) to hear about new posts! If you have any feedback or questions, reach out to me at sam@volatileint.dev!
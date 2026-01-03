---
title: "A Month with Antigravity: The Good, The Bad, and The Ugly"
description: "A deep dive into the realities of using Antigravity, Opus 4.5, and Gemini 3 Pro for greenfield development and architectural maintenance."
date: 2026-01-02T12:00:00-08:00
draft: false
tags: ["ai", "antigravity", "dead air workflow", "software engineering"]
---

I've been on paternity leave over the past few months, and between sessions playing with my son, rocking him to sleep, and changing diapers, I made a point to try and stay sharp and learn some new skills. There's nothing more topical in the software engineering world than agentic AI for development, and I had an idea for a few new greenfield projects, so I signed up for the "pro" tier of Google's AI infrastructure, downloaded the [Antigravity](https://antigravity.google/) IDE, and got to work on building. I entered with very little experience - I had used Claude with an outdated model a few times, but was essentially entirely new to the ecosystem. I tried out all of the models natively available in Antigravity as I could. Most of my time was spent with Opus 4.5 and Gemini 3 Pro. I gave the less powerful models a try too (such as Sonnet 4.5 and GPT-OSS 120B), but their performance was so far below the two more powerful models that I abandoned their use soon after starting.

In this article I'm going to share my experiences working daily with Google's AI integrated development environment over the past month.

# The Good

## Bootstrapping a Project is *Fast* 
The ability to quickly bootstrap a project from nothing is, bar none, the most exciting part of the agentic AI workflow. This is the place where most of my ideas used to die, and Antigravity has dramatically reduced that failure mode.

I have found my projects have the most success when I focus on an end-to-end feature at the outset rather than fully fleshing out an entire layer of functionality (a practice I picked up after reading [The Pragmatic Programmer](https://pragprog.com/titles/tpp20/the-pragmatic-programmer-20th-anniversary-edition/)). Historically, just getting to that first vertical slice was painful. Setting up a development environment, wiring together build systems, choosing libraries, and getting everything into a runnable state carried a significant upfront cost. The project needed to really be worth it to justify that effort. Many of my ideas have fizzled out before they ever got off the ground.

With Antigravity, I am genuinely elated at how quickly I can reach the first end-to-end features. My approach to bootstrapping is to write a very detailed description of the application. I treat the initial prompt like a lightweight design document. I describe:

- What we are building
- What software architecture I want to utilize
- What underlying technologies I want
- What build system I want

I also explicitly ask the agent to create a kernel of end-to-end functionality to build off of. I describe a vertical slice of the final system that can build and run in as much detail as possible, down to the classes and interfaces I want designed. I ask it to focus on the development ergonomics of that end-to-end feature while adhering to the described architecture. These bootstrapping prompts have been multiple pages long.

The generated infrastructure is generally just OK. The interfaces between software modules are often unwieldy, the directory and file structure is far from what I would choose long term, and most aspects need significant massaging. Nearly all of the code in that initial infrastructure survives only through the first iteration. Even still, this approach has been an order of magnitude improvement over the traditional experience of starting a new project. There are two reasons this approach works so well:

1. The act of fixing these problems is fun. From the very first time you sit down to write any code yourself, you're improving on a real part of your idea. It is hard to overstate the value of iterating on something tangible. Expanding and refactoring code is hard - especially so in large projects. But starting from nothing is even more difficult, and the AI agents can essentially skip that step for you.
2. I found that the models, especially Opus 4.5, are pretty dang good at build system configuration. I use Bazel for most of my personal projects, and I can definitively say I have not written a single line of Starlark since downloading Antigravity.

> Tip #1: When bootstrapping a new project with an AI agent, write extremely detailed prompts. Output quality always seems to go up as my specificity increases.

## AI is a Good Chameleon
I found that both Gemini 3 Pro and Opus 4.5 were very effective at implementing new features which introduced no new architectural patterns. As a concrete example, one project I am building implements an RPC (remote procedure call) software interface. It took quite a while to coerce the AI to implement the first RPCs correctly, but after that initial struggle, new RPCs following similar design patterns to existing ones have been very rapidly developed. Within a few hours working with the tools, I was able to implement new concepts utilizing existing patterns much faster than I would have been able to do it myself, and found that I have been making fewer and fewer corrections to the implementations as my prompts improved.

On that note, I offer an important lesson I learned. Be explicit about what pattern you wanted the AI to follow to implement a given feature - even if it feels obvious. An anecdote from that project with the RPCs will demonstrate this well: I prompted the Opus 4.5 agent to expand a service to implement the ability to update a piece of state contained within one of the application services. Similar functionality existed already elsewhere in the code via the RPC mechanism. I was very surprised when I reviewed the output to see the RPC mechanism was completely sidestepped and instead a new backdoor state update mechanism was created that opened a file to look for overrides, even when we already had dozens of other RPCs in the system implementing functionality very closely related to this new feature. I rolled back those changes, and issued the same prompt, but added another line to the prompt - "Utilize the RPC mechanism for the feature". The resulting output was perfect and accepted without modification.

> Tip #2: Be explicit about what architectural patterns to follow for a given feature request.

## Antigravity's Plan Workflows
When you prompt the AI to pursue some task, if it is of sufficient complexity, the agent will inform you that the task requires planning and approval, and generate a markdown document describing its understanding of your intent and the specific steps it will take. It will also highlight questions about the process which you are prompted to clarify.

This experience felt similar to talking through a new feature with a coworker. On countless occasions, the design iteration through the implementation plan unveiled some new understanding of what I wanted to happen or revealed a corner case I had not accounted for yet.

Occasionally, the plan would ask some rather insightful questions about the architectural impacts of a change, but as will be discussed later, these insights were few and far between. The impact of a change on the larger ecosystem was rarely considered and is not one of the benefits of AI at this stage. The implementation plan was fantastic for technical implementation details of a specific, isolated code block and for system-level user-facing design impacts, not long-term maintainability.

> Tip #3: Find an AI workflow that involves planning stages. But don't rely on those plans to create something maintainable long term.

# The Bad

## Antigravity UI Bugs
Antigravity is a new project and no doubt has a lot of moving pieces in the background, so I'm sure these will improve with time. But there are a lot of bugs.

Anecdotally, I'd say that you should expect the agent to crash every hour or so. These crashes manifest in one of two ways: an error that the agent could not respond and that you need to start a new conversation, or a broken file link which crashes the agent. It's entirely opaque what causes both of these error types - perhaps dev logs exist somewhere that could provide insight, but as a user I never took the time to dig in.

The context of these conversations is very difficult to recover. It's especially frustrating when the AI crashes in the middle of a feature, leaving code in a broken state which is hard to reason about. When you start a new conversation and the context is gone, the agents often struggle to recover.

> Tip #4: Commit between every single buildable checkpoint in the development of a new feature. These tools are not stable, context is hard to recover, and starting over is almost always easier than recovering.

## No Respect for the Rules
While iterating on features with the agent, I found myself repeatedly specifying the same pieces of feedback. Stop creating monolithic functions, add tests around some new behavior, run the tests before saying you're done with a feature, etc. Antigravity provides an interface to specify "rules" which the agents are supposed to take into account, defined inside markdown files. Some of the rules I attempted to utilize include:

> Prefer small functions. If you have lots of if/else logic, put the contents inside each into utility functions when possible. Generally try and keep functions less than 10 lines.

> All components of this project are released and deployed together. You should never worry about backwards interface compatibility. If you refactor code, just delete what you don't need instead of marking a deprecation step.

I never had any success getting the agents to reliably follow rules like these. The output consistently violated explicit instructions, even when the rules were simple, local, and directly relevant to the code being written. Fixing these violations required constant follow-up prompts, rework, and manual cleanup. Furthermore, every corrective prompt consumes tokens while adding little to no project value. You are paying to restate constraints the tool already claims to understand. I cannot imagine using these tools in a production codebase with stricter standards, deeper abstractions, or a larger set of architectural rules. The ability to constrain agents to a set of rules they actually follow is not a nice-to-have. It is a prerequisite for using these tools in any serious engineering environment. Until that exists, the burden of enforcing standards remains entirely on the human.

## Useless Comments
This complaint is specifically for Gemini 3 Pro - it writes absolutely terrible comments. 

The comment-to-code ratio is often greater than one, and the comments rarely document anything useful. Instead of explaining intent, invariants, or non-obvious behavior, the comments restate the code, narrate obvious control flow, or serve as a kind of internal monologue. Given the earlier issue with rules being ignored, I found myself routinely following up feature prompts with a second instruction: “go back and remove all the useless comments”.

Including that instruction in the initial prompt almost never works. Gemini seems to have a very strong bias toward producing copious expository comments, and the only reliable way to suppress them is with a singular, direct follow-up prompt. Amusingly, the model appears to have a decent internal understanding of what constitutes a useless comment. When explicitly asked to remove them, it does a surprisingly good job without needing further guidance.

The most problematic form of these comments is what I would describe as "stream-of-consciousness" commentary. These are long, rambling comment blocks where the model appears to be asking itself a question, then answering those questions, sometimes expressing shock and disbelief at what it has found. Take this hilarious example from when I asked Gemini to implement a new sort of coordinate transform in one of my projects:

> // GS position is lat/lon in msg, but we need ECI/ECEF. 
> // Currently GroundStation message has lat/lon. LinkGeometry needs ECI.
> // Wait, GroundStation message HAS NO POSITION in XYZ? 
> // We need to convert Lat/Lon to ECEF/ECI.
> // Assuming Earth rotation is handled elsewhere or ignored (ECEF=ECI for GS at t=0?).
> // Let's implement simple LatLonToECEF.

This was inserted in-line with the "complete" code. When using Gemini 3 Pro, I would estimate no less than 15 percent of the generated output is expository, "stream of consciousness" comments.

> Tip #5: Follow up prompts with explicit direction to remove useless comments, especially when using Gemini.

## Dead-Air Workflow
The primary development workflow I used followed a prompt-review-iterate loop. That time between "prompt" and "review", especially for tasks of significant scope, can be hard to fill productively. I had hoped that I could just go off and work on some other part of the codebase, but I found that difficult for two reasons:

1. The agents produce text output as they are reasoning through a task that describes their current stage. Frequently, while watching this output, I found the AI going down a bad path. It might have been discarding some feedback I gave (there is something distinctly infuriating about reading an AI agent describe its intent to ignore your specific direction), or making a design decision which will have sweeping impacts when something much simpler is better. But whatever the reason, it's hard to feel comfortable leaving things unsupervised. When you catch it early, you can enter a corrective comment which redirects things before they get too off the rails. So review of the thought-process output starts to become an important subtask, meaning the workflow really becomes prompt-supervise-review-iterate.
2. If you end up touching the same files the AI is working on, you'll create conflicts. As developers we are used to this conceptually - we use version control to resolve merge conflicts all the time - but the mechanics here are worse. Working with an AI is more like working with someone live on a Google Doc than collaborating on a git repo.
3. Even working in completely different files is risky. If the AI builds the project and encounters a failure, for example because you are halfway through writing a function and the code is not yet buildable, the agent will see the error and attempt to fix it. In doing so, it will often delete or comment out your in-progress changes. This happened to me repeatedly.

I refer to this entire phase as the dead-air portion of the workflow. You are not meaningfully productive, but you also cannot fully disengage. I have not found a better solution than either watching the agent’s reasoning output and intervening when necessary, or walking away entirely and accepting the risk that something goes seriously wrong.

# The Ugly

## A Complete Disregard for Maintainability

The book [Software Engineering at Google](https://abseil.io/resources/swe-book) defines software engineering as "programming integrated over time". My time with Opus 4.5 and Gemini 3 Pro has solidified the suspicions I already had - these tools are abysmal at software engineering. Without oversight from a human, especially in greenfield projects with few intentionally crafted architectural patterns to immitate, the models are simply incapable of producing maintainable software which is resiliant in the face of evolving dependencies and requirements.

One of the projects I have been developing has a UI component. I wanted to focus my development on the backend, so I designed the interface contract between the frontend and backend, and let the AI agent fully handle developing an ncurses-based terminal UI. Long term, I planned on replacing the UI with something more polished, and so I saw this as a good opportunity to observe how the AIs handled maintenance of a component with essentially no oversight from me.

I gave basically no direction on this UI component. I didn't review the code that was produced. I didn't give any feedback on how to architect the changes. I didn't advise for or against any specific refactors. I simply communicated when I needed a new UI feature, and I would try it out after being made aware the agent was complete.

Eventually, about two weeks into the project, the UI simply became unusable. I asked to add a new feature which seemed, on its surface, completely mundane. The UI froze, and the agent spent about 45 minutes iterating, attempting to find the bug. Frustrated, I called the AI-driven-component-architecture experiment over, and took my first look at what was produced.

1. There was a single `main.cpp` file for the UI.
2. The file was over 3000 lines long.
3. It consisted of about 4 functions, with one of them being over 1500 lines long.
4. Most of the logic was contained within a *very* deeply nested series of `if` statements inside `while` loops.
5. State was shared via globals modified by lambdas which captured the globals as references.

I have no idea how it ever got to this state. I would have expected these models would ship with some sort of heuristics about usability. And if not usability, that there would be some recognition from the agents that the rest of the codebase, which I had been intimately involved in architecting, was so radically different from this small corner I did not touch.

While this was the most egregious misstep (well, long series of missteps) I saw an AI agent make thus far, it is far from the only one. I already mentioned the incident where an entire new state management interface was introduced for a new feature instead of using a well-established mechanism. And there are countless more examples. I cannot think of a single instance where I set out to make a new feature that introduced some novel architectural element and I did not need to intervene on some aspect of the design for the sake of sanity.

> Tip #6: You have to own the architecture. At this point, AI agents are simply incapable of self-driven maintainable code.

> Tip #7: Just because an AI agent is writing code in your project does not mean the architecture is unimportant. After a sufficient amount of architectural neglect, neither human nor AI will be able to make changes to the code.


## Serving a Cake Half-Cooked
While the architectural incompetence is the most insidious and dangerous aspect of the AI development experience, the most frustrating is when the agent claims a feature is done or an issue is addressed when it has not been. Usually this has manifested in one of two ways:

1. In the course of task execution, the AI has decided that some part of the work is "too hard". You'll know this happened because when you look back in the reasoning stream, you'll find some indication that some step of the feature would drive a lot of change to some area or another, and so the functionality was stubbed out, marked TODO, or simply totally ignored. This omitted functionality is very rarely presented by the agent in its task summaries, and is only discovered via code review or actually attempting to use the features which were skipped.
2. Less often, but frequently enough to be notable, the agent will confidently report incorrect results. I have seen agents run tests, have the tests fail, and then report they passed. I have seen agents claim a bug is resolved when it has not written a test or even attempted to build the code after it runs.

Sometimes, the agent will take a much more reasonable approach to the source of the first issue. When it sees a task that becomes complicated, it will pause and prepare an explanation of the consequences of the task, and solicit feedback on its plan. I love when it does this and wish that this behavior was more consistent. I hope future iterations of the models refuse to change the scope of completion criteria without communicating the plan.

I don't want a half-cooked cake. I don't want the chef to tell me that black forest cake is too hard, so it gave me an Oreo instead.

> Tip #8: You have to review every single line of code. AI tools are a far cry from perfect, and they *will* make mistakes.

# Conclusion
Overall, I really have been enjoying my time with Antigravity. It took me quite a lot of time and self-reflection to extract significant value from them. In some ways, you have to pull on your interpersonal toolbelt to derive the most value. Working with the most recent models has felt a bit like working with an prodigious toddler - someone who has the ability to execute on anything you give them in the immediate future but lacks the foresight to understand the long-term consequences of their actions.

There's been a lot of discourse online where software engineers make claims like "AI writes 100% of my PRs". I can see how the agentic tools like Antigravity could consume many, maybe even most, of the programming tasks for codebases with well-established patterns and diligent engineers reviewing outputs. But, at least for now, these current models are incapable of driving maintainability. Woe unto thee who does not provide rigorous, detailed review of the outputs of an AI agent.

Which leads me to my last thought. I have a deep concern that these tools will produce poorly understood systems. It is simply too easy to trust the outputs of AI agents, especially when they appear to work. Especially when using the tools to write code you may not have been able to write yourself. I have been reflecting on the ncurses-based UI software component. How many new, AI-driven projects already have been installed as the backbone of critical infrastructure without sufficient architectural oversight? It seems likely to me that many such projects are running on borrowed time.

These tools are powerful. But it is critical to fully appreciate what they are good at, and what they are not good at. Be like [Uncle Ben](https://www.youtube.com/watch?v=guuYU74wU70). Because if you rely on these tools to replace your own brain in the development process, nobody will hold the AI agent responsible.

Feel free to reach out to me if you have any thoughts, or know how to address any of the issues I came across: sam@volatileint.dev

If you found this article interesting, consider subscribing to the [newsletter](https://volatileint.dev/newsletter) to hear about new posts!
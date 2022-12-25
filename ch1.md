# 1 Go: Simple to learn but hard to master 1
What matters in the end isn’t the number of mistakes we make, but our capacity to learn from them. This assertion also applies to programming. The seniority we acquire in a language isn’t a magical process; it involves making many mistakes and learning from them. The purpose of this book is centered around this idea. It will help you, the reader, become a more proficient Go developer by looking at and learning from 100 common mistakes people make in many areas of the language

## 1.1 Go outline 2
Why does Go not have feature X? Your favorite feature may be missing because it doesn’t fit, because it affects compilation speed or clarity of design, or because it would make the fundamental system model too difficult.

* ***Stability***: Even though Go receives frequent updates (including improvements and security patches), it remains a stable language. Some may even consider this one of the best features of the language.
* ***Expressivity***: We can define expressivity in a programming language by how naturally and intuitively we can write and read code. A reduced number of keywords and limited ways to solve common problems make Go an expressive language for large codebases.
* ***Compilation***: As developers, what can be more exasperating than having to wait for a build to test our application? Targeting fast compilation times has always been a conscious goal for the language designers. This, in turn, enables productivity.
* ***Safety***: Go is a strong, statically typed language. Hence, it has strict compile-time rules, which ensure the code is type-safe in most cases

## 1.2 Simple doesn’t mean easy 3
Go is simple to learn but not necessarily easy to master.

Most of the blocking bugs are caused by inaccurate use of the message-passing paradigm via channels, despite the belief that message passing is easier to handle and less error-prone than sharing memory.

## 1.3 100 Go mistakes 4
Why should we read a book about common Go mistakes?
The best time for brain growth is when we’re facing mistakes.

Mistakes have a facilitative effect. The main idea is that we can remember not only the error but also the context surrounding the mistake. This is one of the reasons why learning from mistakes is so efficient

> Tell me and I forget. Teach me and I remember. Involve me and I learn.
 —Unknown

### 1.3.1 Bugs
Although accurate tests should be a way to discover such bugs as early as possible, we may sometimes miss cases because of different factors such as time constraints or complexity. Therefore, as a Go developer, it’s essential to make sure we avoid common bugs.
### 1.3.2 Needless complexity
A significant part of software complexity comes from the fact that, as developers, we strive to think about imaginary futures. Instead of solving concrete problems right now, it can be tempting to build evolutionary software that could tackle whatever future use case arises. However, this leads to more drawbacks than benefits in most cases because it can make a codebase more complex to understand and reason about.

Getting back to Go, we can think of plenty of use cases where developers might be tempted to design abstractions for future needs, such as interfaces or generics. This book discusses topics where we should remain careful not to harm a codebase with needless complexity.

### 1.3.3 Weaker readability
When programming in Go, we can make many mistakes that can harm readability. These mistakes may include nested code, data type representations, or not using named result parameters in some cases. Throughout this book, we will learn how to write readable code and care for future readers (including our future selves).

### 1.3.4 Suboptimal or unidiomatic organization
Another type of mistake is organizing our code and a project suboptimally and unidiomatically.

### 1.3.5 Lack of API convenience
We can think about many situations such as overusing `any` types, using the wrong creational pattern to deal with options, or blindly applying standard practices from object-oriented programming that affect the usability of our APIs.

### 1.3.6 Under-optimized code
Under-optimized code is another type of mistake made by developers. It can happen for various reasons, such as not understanding language features or even a lack of fundamental knowledge. Performance is one of the most obvious impacts of this mistake, but not the only one.

### 1.3.7 Lack of productivity
Being comfortable with how a language works and exploiting it to get the best out of it is crucial to reach proficiency.

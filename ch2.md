# 2 Code and project organization
## 2.1 #1: Unintended variable shadowing
The scope of a variable refers to the places a variable can be referenced

In Go, a variable name declared in a block can be redeclared in an inner block. This principle, called *variable shadowing*, is prone to common mistakes.

```go
var client *http.Client
if tracing {
    client, err := createClientWithTracing()
    if err != nil {
        return err
    }
    log.Println(client)
} else {
    client, err := createDefaultClient()
    if err != nil {
        return err
    }
    log.Println(client)
}
// Use clien
```
we use the short variable declaration operator `:=` in both inner blocks to assign the result of the function call to the inner client variables—not the outer one. As a result, the outer variable is always `nil`.

> NOTE This code compiles because the inner client variables are used in the logging calls. If not, we would have compilation errors such as client declared and not used.

* The first option uses temporary variables in the inner blocks
* The second option uses the assignment operator `=` in the inner blocks to directly assign the function results to the client variable. However, this requires creating an error variable because the assignment operator works only if a variable name has already been declared.

```go
var client *http.Client
var err error
if tracing {
    client, err = createClientWithTracing()
    if err != nil {
        return err
    }
} else {
    // Same logic
}
```

Imposing a rule to forbid shadowed variables depends on personal taste. For example, sometimes it can be convenient to reuse an existing variable name like err for errors. Yet, in general, we should remain cautious because we now know that we can face a scenario where the code compiles, but the variable that receives the value is not the one expected

## 2.2 #2: Unnecessary nested code
```go
“func join(s1, s2 string, max int) (string, error) {
    if s1 == "" {
        return "", errors.New("s1 is empty")
    } else {
        if s2 == "" {
            return "", errors.New("s2 is empty")
        } else {
            concat, err := concatenate(s1, s2)     ❶
            if err != nil {
                return "", err
            } else {
                if len(concat) > max {
                    return concat[:max], nil
                } else {
                    return concat, nil
                }
            }
        }
    }
}
 
func concatenate(s1 string, s2 string) (string, error) {
    //
}
```
Let’s try this exercise again with the same function but implemented differently.

```go
func join(s1, s2 string, max int) (string, error) {
    if s1 == "" {
        return "", errors.New("s1 is empty")
    }
    if s2 == "" {
        return "", errors.New("s2 is empty")
    }
    concat, err := concatenate(s1, s2)
    if err != nil {
        return "", err
    }
    if len(concat) > max {
        return concat[:max], nil
    }
    return concat, nil
}
 
func concatenate(s1 string, s2 string) (string, error) {
    // ...
}
```

> Align the happy path to the left; you should quickly be able to scan down one column to see the expected execution flow.

the second version requires scanning down one column to see the expected execution flow and down the second column to see how the edge cases are handled

In general, the more nested levels a function requires, the more complex it is to read and understand

* When an if block returns, we should omit the else block in all cases.

```go
if foo() {
    // ...
    return true
} else {
    // ...
}

// Instead, we omit the else block like this:

if foo() {
    // ...
    return true
}
// ...”
```

With this new version, the code living previously in the else block is moved to the top level, making it easier to read.

* We can also follow this logic with a non-happy path

```go
if s != "" {
    // ...
} else {
    return errors.New("empty string")
}

// An empty s represents the non-happy path. Hence, we should flip the condition.
if s == "" {
    return errors.New("empty string")
}
```

This new version is easier to read because it keeps the happy path on the left edge and reduces the number of blocks

Writing readable code is an important challenge for every developer. Striving to reduce the number of nested blocks, aligning the happy path on the left, and returning as early as possible are concrete means to improve our code’s readability.

## 2.3 #3: Misusing init functions
Sometimes we misuse `init` functions in Go applications. The potential consequences are poor error management or a code flow that is harder to understand

### 2.3.1 Concepts
An `init` function is a function used to initialize the state of an application. It takes no arguments and returns no result (a `func()` function). When a package is initialized, all the constant and variable declarations in the package are evaluated. Then, the `init` functions are executed.

```go
package main 

import "fmt"

var a = func() int {
    fmt.Println("var")        ❶
    return 0
}()
 
func init() {
    fmt.Println("init")       ❷
}
 
func main() {
    fmt.Println("main")       ❸
}
```
❶ Executed first

❷ Executed second

❸ Executed last

```go
package main
 
import (
    "fmt"
    "redis"
)
 
func init() {
    // ...
}
 
func main() {
    err := redis.Store("foo", "bar")
    // ...
}
```
```go
package redis
 
// imports
 
func init() {
    // ...
}

func Store(key, value string) error {
    // ...
{
```

Because `main` depends on redis, the redis package’s `init` function is executed first, followed by the `init` of the `main` package, and then the `main `function itself

We can define multiple `init` functions per package. When we do, the execution order of the `init` function inside the package is based on the source files’ alphabetical order

We shouldn’t rely on the ordering of `init` functions within a package. Indeed, it can be dangerous as source files can be renamed, potentially impacting the execution order

We can also define multiple `init` functions within the same source file

We can also use `init` functions for side effects, requires the foo package to be initialized. We can do that by using the `_` operator this way.

```go
package main
 
import (
    "fmt"
    _ "foo"
)
func main() {
    // ...
}
```

In this case, the `foo` package is initialized before `main`. Hence, the `init` functions of foo are executed.

Another aspect of an `init` function is that it can’t be invoked directly

### 2.3.2 When to use init functions
where using an `init` function can be considered inappropriate: holding a database connection pool

```go
var db *sql.DB
 
func init() {
    dataSourceName :=
        os.Getenv("MYSQL_DATA_SOURCE_NAME")
    d, err := sql.Open("mysql", dataSourceName)
    if err != nil {
        log.Panic(err)
    }
    err = d.Ping()
    if err != nil {
        log.Panic(err)
    }
    db = d
}
```

three main downsides:

* First, error management in an `init` function is limited. Indeed, as an `init` function doesn’t return an error, one of the only ways to signal an error is to `panic`, leading the application to be stopped. In our example, it might be OK to stop the application anyway if opening the database fails. However, it shouldn’t necessarily be up to the package itself to decide whether to stop the application. Perhaps a caller might have preferred implementing a retry or using a fallback mechanism. In this case, opening the database within an `init` function prevents client packages from implementing their error-handling logic.
* Another important downside is related to testing. If we add tests to this file, the `init` function will be executed before running the test cases, which isn’t necessarily what we want (for example, if we add unit tests on a utility function that doesn’t require this connection to be created). Therefore, the `init` function in this example complicates writing unit tests.
* The last downside is that the example requires assigning the database connection pool to a global variable. Global variables have some severe drawbacks, for example:

	* Any functions can alter global variables within the package.
	* Unit tests can be more complicated because a function that depends on a global variable won’t be isolated anymore.

In most cases, we should favor encapsulating a variable rather than keeping it global

```go
func createClient(dsn string) (*sql.DB, error) {    ❶
    db, err := sql.Open("mysql", dsn)
    if err != nil {
        return nil, err                             ❷
    }
    if err = db.Ping(); err != nil {
        return nil, err
    }
    return db, nil
}
```

Using this function, we tackled the main downsides discussed previously. Here’s how:

- The responsibility of error handling is left up to the caller.
- It’s possible to create an integration test to check that this function works.
- The connection pool is encapsulated within the function

```go
func init() {
    redirect := func(w http.ResponseWriter, r *http.Request) {
        http.Redirect(w, r, "/", http.StatusFound)
    }
    http.HandleFunc("/blog", redirect)
    http.HandleFunc("/blog/", redirect)
 
    static := http.FileServer(http.Dir("static"))
    http.Handle("/favicon.ico", static)
    http.Handle("/fonts.css", static)
    http.Handle("/fonts/", static)
 
    http.Handle("/lib/godoc/", http.StripPrefix("/lib/godoc/",
        http.HandlerFunc(staticHandler)))
}
```

In this example, the `init` function cannot fail (`http.HandleFunc` can panic, but only if the handler is `nil`, which isn’t the case here). Meanwhile, there’s no need to create any global variables, and the function will not impact possible unit tests. Therefore, this code snippet provides a good example of where `init` functions can be helpful

- They can limit error management.
- They can complicate how to implement tests (for example, an external dependency must be set up, which may not be necessary for the scope of unit tests).
- If the initialization requires us to set a state, that has to be done through global variables.

## 2.4 #4: Overusing getters and setters
Data encapsulation refers to hiding the values or state of an object.

It is also considered neither mandatory nor idiomatic to use getters and setters to access struct fields

using getters and setters presents some advantages, including these:

* They encapsulate a behavior associated with getting or setting a field, allowing new functionality to be added later (for example, validating a field, returning a computed value, or wrapping the access to a field around a mutex).
* They hide the internal representation, giving us more flexibility in what we expose.
* They provide a debugging interception point for when the property changes at run time, making debugging easier.

If we fall into these cases or foresee a possible use case while guaranteeing forward compatibility, using getters and setters can bring some value. For example, if we use them with a field called balance, we should follow these naming conventions:

* The getter method should be named `Balance` (not GetBalance).
* The setter method should be named `SetBalance`.

```go
currentBalance := customer.Balance()
if currentBalance < 0 {
    customer.SetBalance(0)
}
```

In summary, we shouldn’t overwhelm our code with getters and setters on structs if they don’t bring any value. We should be pragmatic and strive to find the right balance between efficiency and following idioms that are sometimes considered indisputable in other programming paradigms

## 2.5 #5: Interface pollution
Interface pollution is about overwhelming our code with unnecessary abstractions, making it harder to understand.
### 2.5.1 Concepts
We use interfaces to create common abstractions that multiple objects can implement. What makes Go interfaces so different is that they are satisfied implicitly.

Custom implementations of the `io.Reader` interface should accept a slice of bytes, filling it with its data and returning either the number of bytes read or an error.

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}
```

```go
type Writer interface {
    Write(p []byte) (n int, err error)
}
```
Let’s assume we need to implement a function that should copy the content of one file to another. We could create a specific function that would take as input two `*os.Files`. Or, we can choose to create a more generic function using `io.Reader` and `io.Writer` abstractions:

```go
func copySourceToDest(source io.Reader, dest io.Writer) error {
    // ...
}
```

This function would work with `*os.File` parameters (as `*os.File` implements both `io.Reader` and `io.Writer`) and any other type that would implement these interfaces. For example, we could create our own `io.Writer` that writes to a database, and the code would remain the same. It increases the genericity of the function; hence, its reusability.

Furthermore, writing a unit test for this function is easier because, instead of having to handle files, we can use the `strings` and `bytes` packages that provide helpful implementations

```go
func TestCopySourceToDest(t *testing.T) {
    const input = "foo"
    source := strings.NewReader(input)
    dest := bytes.NewBuffer(make([]byte, 0))
    err := copySourceToDest(source, dest)
	if err != nil {
		t.FailNow()
	}

	got := dest.String()
	if got != input {
		t.Errorf("expected: %s, got: %s", input, got)
	}
}
```

While designing interfaces, the granularity (how many methods the interface contains) is also something to keep in mind. A known proverb in Go ([https://www.youtube.com/watch?v=PAAkCSZUG1c&t=318s](https://www.youtube.com/watch?v=PAAkCSZUG1c&t=318s)) relates to how big an interface should be:

> The bigger the interface, the weaker the abstraction. —Rob Pike

Indeed, adding methods to an interface can decrease its level of reusability. `io.Reader` and `io.Writer` are powerful abstractions because they cannot get any simpler. Furthermore, we can also combine fine-grained interfaces to create higher-level abstractions. This is the case with `io.ReadWriter`, which combines the reader and writer behaviors:

```go
type ReadWriter interface {
    Reader
    Writer
}
```

**NOTE** As Einstein said, Everything should be made as simple as possible, but no simpler. Applied to interfaces, this denotes that finding the perfect granularity for an interface isn’t necessarily a straightforward process.

### 2.5.2 When to use interfaces
When should we create interfaces in Go?

- Common behavior
- Decoupling
- Restricting behavior

#### Common behavior
The first option we will discuss is to use interfaces when multiple types implement a common behavior. In such a case, we can factor out the behavior inside an interface

```go
type Interface interface {
    Len() int               ❶
    Less(i, j int) bool     ❷
    Swap(i, j int)          ❸
}
```
❶ Number of elements
❷ Checks two elements
❸ Swaps two elements

are we necessarily interested in the implementation type?

In many cases, we don’t care. Hence, the sorting behavior can be abstracted, and we can depend on the `sort.Interface`.

Finding the right abstraction to factor out a behavior can also bring many benefits.

#### Decoupling
Another important use case is about decoupling our code from an implementation. If we rely on an abstraction instead of a concrete implementation, the implementation itself can be replaced with another without even having to change our code.

One benefit of decoupling can be related to unit testing.

#### Restricting Behavior
restricting a type to a specific behavior.
```go
type IntConfig struct {
    // ...
}
 
func (c *IntConfig) Get() int {
    // Retrieve configuration
}
 
func (c *IntConfig) Set(value int) {
    // Update configuration
}
```
we are only interested in retrieving the configuration value, and we want to prevent updating it. How can we enforce that, semantically, this configuration is read-only, if we don’t want to change our configuration package? By creating an abstraction that restricts the behavior to retrieving only a config value:

```go
type intConfigGetter interface {
    Get() int
}
```

```go
“type Foo struct {
    threshold intConfigGetter
}
 
func NewFoo(threshold intConfigGetter) Foo {    ❶
    return Foo{threshold: threshold}
}
 
func (f Foo) Bar()  {
    threshold := f.threshold.Get()              ❷
    // ...
}
```
❶ Injects the configuration getter
❷ Reads the configuration

we can only read the configuration in the `Bar` method, not modify it. Therefore, we can also use interfaces to restrict a type to a specific behavior for various reasons, such as semantics enforcement.

### 2.5.3 Interface pollution
developers coming from C# or Java may found it natural to create interfaces before concrete types. However, this isn’t how things should work in Go.

abstractions should be discovered, not created. What does this mean? It means we shouldn’t start creating abstractions in our code if there is no immediate reason to do so.

We shouldn’t design with interfaces but wait for a concrete need. Said differently, we should create an interface when we need it, not when we foresee that we could need it.

What’s the main problem if we overuse interfaces? The answer is that they make the code flow more complex. Adding a useless level of indirection doesn’t bring any value; it creates a worthless abstraction making the code more difficult to read, understand, and reason about. If we don’t have a strong reason for adding an interface and it’s unclear how an interface makes a code better, we should challenge this interface’s purpose. Why not call the implementation directly?

**NOTE** We may also experience performance overhead when calling a method through an interface. It requires a lookup in a hash table’s data structure to find the concrete type an interface points to. But this isn’t an issue in many contexts as the overhead is minimal.

if it’s unclear how an interface makes the code better, we should probably consider removing it to make our code simpler

## 2.6 #6: Interface on the producer side
where should an interface live?
- Producer side—An interface defined in the same package as the concrete implementation
- Consumer side—An interface defined in an external package where it’s used

It’s common to see developers creating interfaces on the producer side, But in Go, in most cases this is not what we should do.

_abstractions should be discovered, not created_. This means that it’s not up to the producer to force a given abstraction for all the clients. Instead, it’s up to the client to decide whether it needs some form of abstraction and then determine the best abstraction level for its needs.

perhaps one client won’t be interested in decoupling its code. Maybe another client wants to decouple its code but is only interested in the `GetAllCustomers` method. In this case, this client can create an interface with a single method, referencing the Customer struct from the external package:

```go
package client
 
type customersGetter interface {
    GetAllCustomers() ([]store.Customer, error)
}
```

The main point is that the `client` package can now define the most accurate abstraction for its need (here, only one method). It relates to the concept of the Interface-Segregation Principle (the **_I_** in **_SOLID_**), which states that no client should be forced to depend on methods it doesn’t use. Therefore, in this case, the best approach is to expose the concrete implementation on the producer side and let the client decide how to use it and whether an abstraction is needed.

the abstractions defined in the `encoding` package are used across the standard library, and the language designers knew that creating these abstractions up front was valuable

An interface should live on the consumer side in most cases. However, in particular contexts (for example, when we know—not foresee—that an abstraction will be helpful for consumers), we may want to have it on the producer side. If we do, we should strive to keep it as minimal as possible, increasing its reusability potential and making it more easily composable

## 2.7 #7: Returning interfaces
why returning an interface is, in many cases, considered a bad practice in Go.

Hence, in general, returning an interface restricts flexibility because we force all the clients to use one particular type of abstraction. In most cases, we can get inspiration from Postel’s law ([https://datatracker.ietf.org/doc/html/rfc761](https://datatracker.ietf.org/doc/html/rfc761)):

> Be conservative in what you do, be liberal in what you accept from others. —Transmission Control Protocol

If we apply this idiom to Go, it means:

- Returning structs instead of interfaces
- Accepting interfaces if possible

Of course, there are some exceptions. As software engineers, we are familiar with the fact that rules are never true 100% of the time. The most relevant one concerns the error type, an interface returned by many functions. We can also examine another exception in the standard library with the io package:

```go
func LimitReader(r Reader, n int64) Reader {
    return &LimitedReader{r, n}
}
```

The function returns an exported struct, `io.LimitedReader`. However, the function signature is an interface, `io.Reader`. What’s the rationale for breaking the rule we’ve discussed so far? The `io.Reader` is an up-front abstraction. It’s not one defined by clients, but it’s one that is forced because the language designers knew in advance that this level of abstraction would be helpful (for example, in terms of reusability and composability)

if we know (not foresee) that an abstraction will be helpful for clients, we can consider returning an interface. Otherwise, we shouldn’t force abstractions; they should be discovered by clients. If a client needs to abstract an implementation for whatever reason, it can still do that on the client’s side.

### 2.8 #8: any says nothing
In Go, an interface type that specifies zero methods is known as the empty interface, `interface{}`. With Go 1.18, the predeclared type `any` became an alias for an empty interface; hence, all the `interface{}` occurrences can be replaced by `any`. In many cases, `any` can be considered an overgeneralization; and as mentioned by Rob Pike, it doesn’t convey anything [https://www.youtube.com/watch?v=PAAkCSZUG1c&t=7m36s](https://www.youtube.com/watch?v=PAAkCSZUG1c&t=7m36s)

In assigning a value to an `any` type, we lose all type information, which requires a type assertion to get anything useful out of the `i` variable

```go
package store
 
type Customer struct{
    // Some fields
}
type Contract struct{
    // Some fields
}
 
type Store struct{}
 
func (s *Store) Get(id string) (any, error) {
    // ...
}
 
func (s *Store) Set(id string, v any) error {
    // ...
}
```

Because we accept and return `any` arguments, the methods lack expressiveness. If future developers need to use the `Store` struct, they will probably have to dig into the documentation or read the code to understand how to use these methods. Hence, accepting or returning an `any` type doesn’t convey meaningful information. Also, because there is no safeguard at compile time, nothing prevents a caller from calling these methods with whatever data type, such as an `int`.

```go
s := store.Store{}
s.Set("foo", 42)
```

By using `any`, we lose some of the benefits of Go as a statically typed language. Instead, we should avoid `any` types and make our signatures explicit as much as possible.

Having more methods isn’t necessarily a problem because clients can also create their own abstraction using an interface. For example, if a client is interested only in the Contract methods.

What are the cases when any is helpful?

```go
type ContractStorer interface {
    GetContract(id string) (store.Contract, error)
    SetContract(id string, contract store.Contract) error
}
```

Because we can marshal `any` type, the Marshal function accepts an `any` argument:

```go
func Marshal(v any) ([]byte, error) {
    // ...
}
```

If the query is parameterized (for example, `SELECT * FROM FOO WHERE id = ?`), the parameters could be `any` kind. Hence, it also uses `any` arguments:

```go
func (c *Conn) QueryContext(ctx context.Context, query string,
    args ...any) (*Rows, error) {
    // ...
}
```

In summary, `any` can be helpful if there is a genuine need for accepting or returning any possible type (for instance, when it comes to marshaling or formatting). In general, we should avoid overgeneralizing the code we write at all costs. Perhaps a little bit of duplicated code might occasionally be better if it improves other aspects such as code expressiveness

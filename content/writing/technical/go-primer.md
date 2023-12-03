---
title: concurrent drill
date: 2023-11-30T13:40:03-05:00
draft: false
---

Concurrency-- managing multiple threads of execution-- is a difficult thing to
get right.

The wild west of concurrent execution is a stark departure from the
well-trodden path of sequential programming, and can quickly introduce errors.
I think this added complexity prevents some programmers from messing around
with it.

...And I think that's unfortunate, because suckin' at something is the first
step to being sorta good at something [^1] .

With that said, I'm not familiar with a better programming language to broach
this subject than Go. Objectively, it's a simple language [^2] and most
programmers will find it easy to read. More importantly, some of the
spectacular language features will explain concurrent execution clearly.

Using a practical example, we're going to try and model a contrived military
activity using normal (synchronous) coding practices, exploring some Go syntax
as we progress, and we'll see how far it takes us. Then, when we run into
issues, we'll introduce concurrency, and evaluate the outcome.

Let's get underway!

## life and times of an officer cadet

I can't think of a better candidate for the following exercise than an
Officer Cadet. They're programmed to do exactly what you tell them, and they
possess a little less agency when compared to their non-commissioned
peers. 

Here's how you might model a `Cadet`.

```golang
// main.go

import "fmt"


type Cadet struct {
    name string
}

func (c Cadet) announce() {
    fmt.Printf("Cadet %s, reporting for duty!\n", c.name)
}

func main() {
    cadet := Cadet{ "charlie" }

    cadet.announce()
}

// $ go run main.go
// Output:
// Cadet charlie, reporting for duty!
```

Looks good, but that function declaration is a little... *Weird*.

That's the syntax for a method definition-- and the preamble before the
function name is equivalent to `self`  or  `this` in other programming
languages.

You can read about methods {{< link "https://gobyexample.com/methods"
>}}here{{</ link >}}.

## marching up and down the square 

One of the first things recruits learn in the military is drill. Well-executed
drill movements project discipline to onlookers, and can look pretty cool.

{{< youtube id="0xvXMEnwyYM?start=206" >}}

However, there's a cadence to drill movements, you see. Frankly, if we're going
to build our own drill platoon, we'll need to nail the timings.

Let's consider the time it takes for a cadet to turn three seconds, and define
a method named `Turn()`.

```golang
// main.go

import "time"


func (c Cadet) Turn() {
    fmt.Printf("Cadet %s, starting turn...\n" c.name)
    time.Sleep(3 * time.Second)
    fmt.Printf("Cadet %s, finished turning!\n", c.name)
}
```

Aside from the fact that you don't actually announce turns doing drill, this is
completely fine. We should scale this together, and put a whole platoon of these
folks and start booking performances. We'd be in the Houston Texan's stadium in
no time. 

Here's a little team of them:

```golang
func main() {
    // instantiate a slice of cadets
    cadets := []Cadet{
        {"boggis"},
        {"bunce"},
        {"bean"},
    }
}
```
Alright. Three cadets, ready to turn on my command. How do I issue a command to
all of them? Well, unlike the real military, yelling won't move things along,
so let's try iterating over the cadets and issuing each one a `Turn()`.

While we're at it, for undisclosed reasons, let's measure the time it takes for
this whole affair to transpire.

```golang
// main.go
// <SNIP>

func main() {
    cadets := []Cadet{
        {"boggis"},
        {"bunce"},
        {"bean"},
    }

    start := time.Now()
    for _, cadet := range cadets {
        cadet.Turn()        
    }
    fmt.Println(time.Since(start))
}

// $ go run main.go
// Cadet boggis starting turn...
// Cadet boggis finished turn!
// Cadet bunce starting turn...
// Cadet bunce finished turn!
// Cadet bean starting turn...
// Cadet bean finished turn!
// 9.00379525s
```

The output looks great! Really great! Wait...

No, seriously, wait. Chances are, you're still waiting. You're waiting nine
seconds, at the minimum. This is, as we say in the military, UNSAT [^2]

This sort of movement is *supposed* to be done in unison, so it *should* be
taking three seconds, max. We've tried nothing, and we're all out of ideas.

## ~~everything everywhere~~ all at once 

We need to think outside of the box, because I can't think of a typical 
programming construct that will let us model this **"unison"** behaviour.

Perhaps, given the subject of this article, you've recognized a case for
concurrency. We want to do multiple things, well, concurrently. Here's the textbook
definition of concurrency:

> *Concurrency is when tasks can start, complete, and run in overlapping time periods.*

We want these cadets to turn at the same time. Now, the real punch line is how easy it is
to express this behaviour in Go.

The (titular, and comically simple) answer to this problem is two letters: `go`

```golang
start := time.Now()
for _, cadet := range cadets {
    go cadet.Turn()        
}
elapsed := time.Since(start)
fmt.Println(elapsed)
```

Within the loop, this simple addition tells the program to execute each
`cadet.Turn()` in a separate, lightweight thread of execution called a `goroutine`.

Emphasis on lightweight. As of writing, each `goroutine` invocation incurs
about 2 kB of memory on the stack. Consequently, it's possible to spawn over a
million goroutines concurrently. This is in stark comparison to OS-level
threads, which delve into the kernel and incur much more overhead.  

Don't take my word for it though, let's run this program.

```sh
$ go run main.go
58.542µs
Cadet bunce starting turn...
```

See footnote. [^2]

Looks like our program executed in a few microseconds, which doesn't leave a lot of room for
drill. We're left more confused than when we started.

I encourage you to run this program a few more times. You'll probably get
different output each time. Try to reason about is happening. 

---

Ready? Here's what happened above:

1. The Go runtime scheduled each `cadet.Turn()` to be executed concurrently, alongside the `main` thread.
2. Once the loop finished, the `main` thread of execution kept running.
3. Before hitting the end of `main`, the `Cadet{"bunce"}.Turn()` goroutine began execution.
4. Then the program ended, because `main` finished.

The program doesn't wait until all goroutines are done executing. The head
honcho, `main`, calls the shots. As you may have observed, point 3 is
consequently non-deterministic.

Evidently, we need a way to tell the `main` thread to wait. Here's a simple solution:

```golang
start := time.Now()

for _, cadet := range cadets {
    go cadet.Turn()        
}

time.Sleep(3 * time.Second)

elapsed := time.since(start)
fmt.Println(elapsed)

// $ go run main.go
// Output:
// Cadet boggis starting turn...
// Cadet bunce starting turn...
// Cadet bean starting turn...
// Cadet bean finished turn!
// Cadet bunce finished turn!
// Cadet boggis finished turn!
// 3.000826042s
```

Mission accomplished, and for real this time. Maybe you're wondering why the order of
execution isn't the same as scheduled, and for that, you're going to need to
take it up with the scheduler. 

A more pressing issue is how depressingly hacky this solution feels-- Inserting
`time.Sleep` to orchestrate threads is a valid strategy, but it's not a very
good one. For example, we won't always know how long to `Sleep` for.

Let's explore a better synchronization strategy. 

## hurry up and wait

The Go standard library provides the `sync` package, which includes the `WaitGroup` object.

This object will help us express the following behaviour:

1. Launch all goroutines
2. Wait (in the main thread) for all goroutines to run to completion 
3. Finally, continue executing in the main thread

Under the hood, a `WaitGroup` is a simple counter. Increment the count with
`Add` and decrement it with `Done`. Finally, `Wait` will block the current
thread if the counter is non-zero, and continue execution otherwise. Easy!

We need to modify our `Turn` method signature to accept a pointer to the
`WaitGroup`. 

```golang
import "sync"


func (c Cadet) Turn(wg *sync.WaitGroup) { 
    fmt.Printf("Cadet %s, starting turn...\n" c.name)
    time.Sleep(3 * time.Second)
    fmt.Printf("Cadet %s, finished turning!\n", c.name)
    wg.Done()
}

func main() {
    // SNIP
    var wg sync.WaitGroup
    for _, cadet := range cadets {
        wg.Add(1)
        go cadet.Turn(&wg)
    }
    wg.Wait()
}

// $ go run main.go
// Cadet bean starting turn...
// Cadet bunce starting turn...
// Cadet boggis starting turn...
// Cadet boggis finished turn!
// Cadet bunce finished turn!
// Cadet bean finished turn!
// 3.001287583s
```

This is much better. We can scale up the number of cadets, or tinker with the
amount of work done in the `Turn` method without worrying about syncing up
the main thread. Sweet!

Let's take stock, though. It's unfortunate that we needed to modify the
signature of an otherwise perfectly fine method to do our concurrent bidding. 

We've lost the ability to tell a `Cadet` to turn without a `sync.WaitGroup`
lying around, which is a significant change to our API. Thankfully,
there's a more idiomatic solution.

```golang
import "sync"


func (c Cadet) Turn() { 
    fmt.Printf("Cadet %s, starting turn..." c.name)
    time.Sleep(3 * time.Second)
    fmt.Printf("Cadet %s, finished turning!", c.name)
}

func main() {
    var wg sync.WaitGroup
    for _, cadet := range cadets {
        wg.Add(1)

        go func() {
            cadet.Turn()
            wg.Done()
        }()
    }
    wg.Wait()
}
```

This is an example of an *anonymous function* in Go [^3] . Anonymous
insofar that the function doesn't actually have a name-- we declare it, and
then immediately execute it. Better still, our locally-scoped function can close
over [^4]  the `wg` and `cadet` variables at each iteration of the loop. 

While it's a little unsightly at first exposure, it's a super-powered feature.
We've gift-wrapped a package of synchronous code for concurrent execution. As a
result, a `Cadet` retains its straightforward API, and isn't tightly coupled to
any synchronization primitives. Wow!

Our final code is available here.

## kaizen (further reading)

The seminal book on concurrency in Go is shockingly {{< link
"https://www.oreilly.com/library/view/concurrency-in-go/9781491941294/"
>}}Concurrency in Go{{< /link >}}.


[^1]: Sage advice from {{< link "https://www.youtube.com/watch?v=Gu8YiTeU9XU" >}}Jake the Dog{{< /link >}}.
[^2]: The Go programming language has 25 keywords, and the entire language spec is around 50 pages.
  By contrast, the Java SE 12 specification is almost 770 pages. Yowza.
[^3]: Unsatisfactory.
[^4]: In JavaScript, this is commonly referred to as an IIFE (pronounced
    "iffy"), or "Immediately Invoked Function Expression". 
[^5]: These are better known as *closures* (because they "close over",
    or capture, locally scoped variables). The compiler is smart enough to pass
    references to the variables, which are made available on the heap. 

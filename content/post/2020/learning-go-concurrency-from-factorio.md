---
title: "Learning Go Concurrency From Factorio"
author: "ropnop"
draft: false
date: 2020-06-28T10:36:39-05:00
summary: "Go's concurrency model confused me at first, but it finally clicked when I thought of it like building an assembly line in Factorio"
toc: true
share_img: "/images/2020/06/factorio-1.jpg"
tags: ["go"]
---

# Introduction
If I haven't said it before: I love writing Go. Something about the language just clicks with me, and the more I write the more I enjoy it. Lately, I have been re-writing some of my old tools in Go, and I take every opportunity I can to learn or try something new with the language. For my last project, I needed to take full advantage of concurrency in Go and I struggled for a while to fully wrap my head around it. I kept hitting race conditions and never ending loops, and it stemmed from the fact that I wasn't thinking in the correct way.

Finally, I started to think of Go's concurrency as an assembly line of sorts, and my brain instantly made a connection to the game [Factorio](https://factorio.com/), which I used to play a lot. If you haven't heard of or played Factorio before, it's essentially a factory simulator where you design and build automated production lines and scale up. The game rewards you for being as efficient as possible, and identifying bottlenecks and designing logistical solutions are how you can expand.

Even though I had read plenty of tutorials and examples online about Go concurrency, this analogy stuck with me and I decided to share it. If you're new to Go or struggle to fully grok concurrency like me, hopefully this will help you!

# Basics of Concurrency
Go has all the building blocks for writing concurrent code baked in to the language already. It's a first class citizen, and IMO one of the strongest features of Go. What confused me at first (but now I love), is that as opposed to Python's `asyncio` or JavaScript, there are no special keywords like `async` and `await`, or the concept of "promises" (eww). All you need to run concurrent code is the single keyword `go`.

Of course, because that's all there is, the language forces you to think of how to manage and control the concurrency. This was the hard part for me.

To write concurrent code, these are the basic components include with Go:

 * **Go routines** - these are the "workers" that you can spin off with the `go` keyword. They are functions that will just run in the background until they are done, you force them to stop, or your program exits.
 * **Channels** - these are how you pass data between the "workers"
 * **Waitgroups** and **Context** - these are language construcst to help you control and stop workers and understand when things are "done"

# Factorio and Go
Even after writing concurrent Go code utilizing channels and waitgroups (mostly by following tutorials or copying existing tools), I still struggled to design good concurrent code and constantly wound up writing code that didn't work and either exited too quickly or never exited at all. Until I started thinking of Factorio and my Go program like a giant factory :)

In Factorio, two of the main components of an assembly line are "inserters" and "belts". Belts move items around, and inserters are used to either put an item on a belt, or take an item off a belt and do something with it. Belts and inserters are simple - but when used correctly they can create amazingly complex factories. Here's an example of a belt with two inserters. The one of the left is taking an item from storage and putting it on the belt; the one on the right is taking an item off of the belt and putting it in storage.

![factorio inserters](/images/2020/06/inserters_gif.gif)

This is bascially how concurrency works in Go. The inserters are Go routines, and the belts are channels. The inserter on the left is writing to a channel, and the inserter on the right is reading from the channel. In just the same we you can design amazingly efficient factories with just belts and inserters in Factorio, you can write amazingly efficient Go with just Go routines and channels.

## Simple Example
If we take that simple example above of one belt and two inserters, we can basically express it in Go code with two Go routines (inserters) and a channel (belt)

```go
func Inserter1(belt chan int) {
    i := 0
    for {
        time.Sleep(1 * time.Second)
        belt <- i
        i++
    }
}

func Inserter2(belt chan int) {
    for {
        select {
        case i := <-belt:
            fmt.Printf("[Inserter2] I got %d\n", i)
        }
    }
}

func main() {
    belt := make(chan int)
    go Inserter1(belt)
    go Inserter2(belt)
    for {
        // this loops forever so our "factory" never stops
    }
}
```
Here we define two functions to represent the "inserters". `Inserter1` just continually puts numbers on a "belt", and `Inserter2` just loops forever reading and printing any numbers it finds on the "belt". Our "belt" is a channel. When creating a channel, you have to specify what data type it will use (as opposed to Factorio, where you can put anything on a belt). In this case I just use `int`, but in practice it can be any arbitrary type.

Our main function creates the channel and spins off the two Go routines (inserters) and tells them to use the same channel (belt). One important thing to note is that Go programs always want to exit, and will exit as soon as the main function returns. If we didn't have an inifinte loop, the program would exit immediately after starting the inserters and nothing would actually happen.

When we run this program, the channel and go routines will happily run forever (just like the belt and the inserters in the GIF above).

## Closing Channels - Deleting Belts
Unlike Factorio, where we're happy to just run our factory 24/7 and never quit, in a Go program we probably want to control things a bit more. In the tools I've written, I also have a finite set of "work" that needs to be done and want to exit the program when it's complete. This is where Go forces you to think about how to control concurrent work, and there's a few patterns you can follow.

First, unlike Factorio, in Go you can actually destroy a belt when you no longer need it (aka close the channel). Imagine if in Factorio when the inserter placed the last piece on the belt it destroyed it, and the receiving inserter would stop once the belt was gone. This is one method for signalling when work is done. To implement this, let's change `Inserter1` to stop after 5 numbers and then close the channel:

```go
func Inserter1(belt chan int) {
    for i := 0; i < 5; i++ {
        time.Sleep(1 * time.Second)
        belt <- i
    }
    fmt.Println("[Inserter1] My work is done. Closing the channel")
    close(belt)
    return
}
```

We then need to change `Inserter2` to stop working when the channel is closed. In Go, we can do this by adding a check when we pull a number off the belt. This is a pattern called the "comma ok idiom" which is really useful to know in Go. Basically if the channel is closed, it will "not be ok", so we can close that function as well.

```go
func Inserter2(belt chan int) {
    for {
        select {
        case i, ok := <-belt:
            if ok {
                fmt.Printf("[Inserter2] I got %d\n", i)
            } else {
                fmt.Println("[Inserter2] The belt is gone. I'm done")
                return
            }
        }
    }
}
```

If we run our program again, we can see after 5 numbers the inserters stop:
```
$ go run factory.go
[Inserter2] I got 0
[Inserter2] I got 1
[Inserter2] I got 2
[Inserter2] I got 3
[Inserter1] My work is done. Closing the channel
[Inserter2] I got 4
[Inserter2] The belt is gone. I'm done
```
There's a problem though: the program continues to run forever because we infinite looped it. We need a way to "catch" when the inserters are all done so we can succesfully exit. 

## Notify Channel - Adding a Belt
One way we can make sure the work is all done before we exit is to use yet another channel. Go routines can read/write to as many channels as needed, and we can always sit and wait from something (block) on any channel. In Factorio terms, imagine if when the second inserter finished all it's work it placed a "done" product on another belt which looped back to the beginning. When we see the "done" product on the belt, we know we're finished and can exit.

To do that, we can add another channel, called "done". This channel will be for booleans, e.g. true we are done. And we will "block" until that channel recieves something. This blocking is crucial for working with concurrent Go code. It's akin to "await" - Go will not proceed to the next line until it recieves something:

```go
func main() {
    belt := make(chan int)
    done := make(chan bool)
    go Inserter1(belt)
    go Inserter2(belt, done)
    <-done //wait for our finished product before exiting
    fmt.Println("[Main Factory] All done! Closing up shop")
}
```

Now all that's left to do is put something on the "done" channel when everything is ready. This should go on the "last step" - i.e. when `Inserter2` returns. We can use a `defer` for that:

```go
func Inserter2(belt chan int, done chan bool) {
    defer func() {
        done <- true
    }()
    for {
        select {
        case i, ok := <-belt:
            if ok {
                fmt.Printf("[Inserter2] I got %d\n", i)
            } else {
                fmt.Println("[Inserter2] The belt is gone. I'm done")
                return
            }
        }
    }
}
```

Now when we run our program we actually exit when all the work is done:

```
$ go run factory.go
[Inserter2] I got 0
[Inserter2] I got 1
[Inserter2] I got 2
[Inserter2] I got 3
[Inserter2] I got 4
[Inserter1] My work is done. Closing the channel
[Inserter2] The belt is gone. I'm done
[Main Factory] All done! Closing up shop
```

Everything you'd ever want to do with concurrent code can be done with just Go routines and Channels. Just like designing complext factories in Factorio with nothing but inserters and belts. However, there's a few things you can leverage to make things even more efficient/easier to manage.

## Multiple Inserters - Single Belt
In Factorio, you rarely just have a single inserter doing something on a belt. That's not very efficient, since that one inserter becomes the bottleneck. Instead, you place multiple identical inserters doing the same thing on the same belt:

![multiple inserters](/images/2020/06/multiple_inserters.gif)

This way if the first inserter is busy, the second or third will pick it up, etc. In Go, this is really useful when our Go routines need to do something that takes time (e.g. make a network request). While waiting on the first one, the next Go routine will pick up the next item on the channel. When the first one is done, it returns to the channel and grabs the next.

To demonstrate this, we can spin up 3 instances of `Inserter2` in our code:

```go
for i := 0; i < 3; i++ {
    go Inserter2(belt, done)
}
```

The problem now becomes - how do we know when *all* of the instances of `Inserter2` are done? Our code waits for just the first item to come across the "done" channel. We need to wait for all three. So we could solve it by just blocking three times:

```go
<-done
<-done
<-done
fmt.Println("[Main Factory] All done! Closing up shop")
```

But this becomes hard to manage manually - and this is where [WaitGroups](https://gobyexample.com/waitgroups) come in. WaitGroups are essentially syntactic sugar to make that blocking strategy easier. In this case, we'll create a WaitGroup for our Inserter2 cluster, and then wait for all of them to finish. Our WaitGroup will completely replace our "done" channel:

```go
func Inserter2(belt chan int, wg *sync.WaitGroup) {
    defer wg.Done()
    for {
        select {
        case i, ok := <-belt:
            if ok {
                fmt.Printf("[Inserter2] I got %d\n", i)
            } else {
                fmt.Println("[Inserter2] The belt is gone. I'm done")
                return
            }
        }
    }
}

func main() {
    belt := make(chan int)
    go Inserter1(belt)

    var i2WaitGroup sync.WaitGroup
    for i := 0; i < 3; i++ {
        i2WaitGroup.Add(1)
        go Inserter2(belt, &i2WaitGroup)
    }
    i2WaitGroup.Wait()
    fmt.Println("[Main Factory] All done! Closing up shop")
}
```

We're basically doing the exact same thing as the "done" channel, but using `sync.WaitGroup`. For every worker we spin up, we call `Add(1)` on the WaitGroup. Every time a worker finishes (i.e. returns), we call `Done()` on the WaitGroup. Finally, we block on the WaitGroup with `Wait()`, which will block until the number of "Dones" matches the number of "Adds". The benefit of this approach is it's a bit easier to read, and we can dynamically loop and add as many workers as we want.

In my code, I like to use "done" channels when I only have one instance of a worker, but WaitGroups when I will spin up multiple.

## Cutting the Power - Cancelling Everything with Context
The last thing I wanted to show was how to handle graceful shutdowns. There's not a perfect Factorio analogy here - but I kind of think of it as the powerlines that power the inserters. In Factorio, each inserter requires power. If the powerline is cut, the all stop. And you can define multiple circuits throughout your factory to switch on and off. How can we control turning off parts of our Factory in Go?

Our current program will run until either all the work is done, or until we forcibly kill it with Ctrl-C. As programs get more complex, that's not a very good approach. We'll eventually want to handle things like timeouts, or if we're doing something that is rather sensitive and we can't afford to just kill the program randomly, we want to gracefully handle interrupts. 

This is finally where [Context](https://gobyexample.com/context) comes in. Context took me a long time to grok, and TBH, there's features I still don't fully understand. But the way I came to think of it is that it's just a global channel we can use across all of our Go workers. Context is like the powerlines that run throughout our factory and power the inserters. It's definitely not a perfect analogy since Context can carry more than just "on/off" information, but that's usually how I use Context so I'm sticking with it.

To use contexts effectively, every "worker" should accept a context as a parameter, and always check if the context is "Done" during its work. For example, let's rewrite `Inserter2` to support a context:

```go
func Inserter2(ctx context.Context, belt chan int, wg *sync.WaitGroup) {
    defer wg.Done()
    for {
        select {
        case i, ok := <-belt:
            if ok {
                fmt.Printf("[Inserter2] I got %d\n", i)
            } else {
                fmt.Println("[Inserter2] The belt is gone. I'm done")
                return
            }
        case <-ctx.Done():
            fmt.Printf("[Inserter2] Cancelling my work because : %q\n", ctx.Err())
            return
        }
    }
}
```

When using a context, we always want to catch `ctx.Done()`. In this case, we're basically telling `Inserter2` to just cancel its remaining work and exit (return). It will also say why by calling `ctx.Err()`. Now if anything anywhere else in our code closes the context, it will notify all the workers to stop what they're doing and exit. How can we close the context? 

If we wanted a timout, in main, we can define a background context and then apply a timeout to it (say, 3 seconds):

```go
backgroundCtx := context.Background()
ctx, cancel := context.WithTimeout(backgroundCtx, 3*time.Second)
for i := 0; i < 3; i++ {
    i2WaitGroup.Add(1)
    go Inserter2(ctx, belt, &i2WaitGroup)
}
```

In this case, `cancel` is a function we can also manually call to immediately close the context. If anything goes wrong for example, calling `cancel` is the same as forcing a timeout. If we run the code now, we can see that all of our `Inserter2` instances bail out after 3 seconds because the context timed out:

```
$ go run factory.go
[Inserter2] I got 0
[Inserter2] I got 1
[Inserter2] I got 2
[Inserter2] Cancelling my work because: "context deadline exceeded"
[Inserter2] Cancelling my work because: "context deadline exceeded"
[Inserter2] Cancelling my work because: "context deadline exceeded"
[Main Factory] All done! Closing up shop
```

Lastly, let's say we want to manually call `cancel` if our program is interrupted, e.g. with a Ctrl-C. This is easy to do as well in Go, and we'll leverage another Go routine worker and a channel. We can create a channel for OS Signals, and a worker who just listens for the correct signal and calls `cancel()`. In Factorio, imagine a dedicated belt you use as a kill switch. When you put something on it, an inserter grabs it and cuts the power to the factory. Let's finally define our "KillSwitch" worker:

```go
func KillSwitch(cancel context.CancelFunc) {
    sigs := make(chan os.Signal)
    signal.Notify(sigs, syscall.SIGINT)
    <-sigs
    fmt.Println("[Kill Switch] Ctrl-C pressed. Cancelling everything")
    cancel()
}
```

And we spin it off from our main function:

```go
func main() {
    belt := make(chan int)
    go Inserter1(belt)

    var i2WaitGroup sync.WaitGroup

    backgroundCtx := context.Background()
    ctx, cancel := context.WithTimeout(backgroundCtx, 3*time.Second)
    go KillSwitch(cancel)

    for i := 0; i < 3; i++ {
        i2WaitGroup.Add(1)
        go Inserter2(ctx, belt, &i2WaitGroup)
    }
    i2WaitGroup.Wait()
    fmt.Println("[Main Factory] All done! Closing up shop")
}
```

Now when we run our "factory", it can exit for 3 reasons:
 * All the work is done. `Inserter1` finished and closed the belt
 * It took longer than 3 seconds. Our context deadline was hit
 * Someone hit "Ctrl-C" and cancelled everything

If you start it up and hit Ctrl-C, you can see it works:

```
$ go run factory.go
[Inserter2] I got 0
[Inserter2] I got 1
^C[Kill Switch] Ctrl-C pressed. Cancelling everything
[Inserter2] Cancelling my work because: "context canceled"
[Inserter2] Cancelling my work because: "context canceled"
[Inserter2] Cancelling my work because: "context canceled"
[Main Factory] All done! Closing up shop
```

# Conclusion
I know it's not a perfect analogy, but it helped me grok Go routines, channels, waitgroups and contexts. Hopefully this helps you as well. Concurrency in Go is simple to implement, but complex because it forces you to think of and handle everything. There are a lot of "foot guns" with concurrency, so thinking out what I'm trying to accomplish and thinking of if like a Factorio assembly line has really helped me. In my next post, I'll dive into the code and my thought process of how I impelemented concurrency in [go-windapsearch](https://github.com/ropnop/go-windapsearch) for processing LDAP entries, and it's all rooted in my Factorio mindset :)









---
title: "Rewriting a bloated Python script in Go"
date: 2020-05-24T15:56:21-05:00
author: "ropnop"
summary: "How I re-wrote windapsearch (an old Python script of mine to perform LDAP recon) in Go using concurrency, channels, interfaces, and custom JSON marshalers"
toc: true
share_img: ""
tags: ["go", "python", "ldap", "active directory"]
draft: true
---

# Introduction
It's no secret that I've been loving learning and writing Go lately. After starting with [kerbrute](https://github.com/ropnop/kerbrute), my first Go project, I've started using it at work almost daily, and recently dove deep down into Windows syscalls with Go to create [go-clr](https://blog.ropnop.com/hosting-clr-in-golang/). I wanted another fun project/challenge, so I decided to re-write [windapsearch](https://github.com/ropnop/windapsearch) from scratch in Go.

Windapsearch was almost entirely coded in one night in a hotel room in Vegas after getting the idea attending Defcon in 2016. Since then I've admittedly negelected it, but earlier this year merged in a PR to finally support Python 3. In the back of my mind, however, I was never super happy with it as it was a quick hack-job, and I always wanted to refactor it.

In this post, I wanted to showcase some of the things I learned in the process of the re-write, and some cool new features. This whole project was basically a fun excuse to learn more things about Go, so I thought I'd share. I'm still by no means an expert in Go, so if anyone reading this has other tips/efficiencies please share!

If you want to try out the new Go version of `windapsearch`, check out the latest release on Github: https://github.com/ropnop/go-windapsearch

## Motivations
When deciding to re-write `windapsearch` in Go, there were a few improvements and features I definitely wanted to add:

 * The Python script had a ton of code duplication. Adding new LDAP queries was a matter of rewriting some common functions, and chaining another if statement to the arguments. I wanted to clean it up
 * The Python script choked on really large queries. The way the LDAP searching worked, the script was waiting for every single response to come back, storing them in a dict, then processing them and writing to a file/stdout. For extremely large queries it took forever and sometimes crashed. I wanted to process and write results as they came in over the wire
 * The CSV export feature was super hacky and never really worked how I wanted. I wanted machine-friendly output, but CSV was not the right choice. I wanted to see if I could convert LDAP responses to JSON
 * Windows AD LDAP attributes come back in weird formats sometimes. For example, SIDs are base64 encoded bytes, timestamps are longs that represent an interval, etc. I wanted to covert these to more friendly output that we're used to seeing in other AD tools

Taking what I knew about Go, and researching other Go topics I wanted to explore, I learned and implemented the following language features to tackle the motivations above:

 * Breaking up windapsearch into related packages, and using an Interface to handle each LDAP "module" that I wanted
 * Utilizing channels and go routines to process and write results as they were available, instead of waiting for the full LDAP search to finish
 * Writing a custom JSON Marshaler for LDAP entries
 * Generating code for common AD Attributes and their syntaxes to convert

I'll talk through each of these. Before diving in, I want to mention this wouldn't have been possible without the awesome [go-ldap](https://github.com/go-ldap/ldap) package that does all the heavy lifiting behind the scenes.

# Interfaces and Modules
Breaking down the original `windapsearch` code, it's really pretty simple. It looks for a DC, opens an LDAP session, sends an LDAP request, and prints the LDAP response. Each "module" is really only crafting a filter with some default attributes and using the already established LDAP connection to send the request and print the response.

I decided to split the code up into three disctint parts:
 * `windapsearch` - the core code that handles command line options, and the parsing/writing of results
 * `ldapsession` - the underlying LDAP connection for sending requests and passing back results
 * `modules` - the high level definitions for each custom LDAP filter

*Note: "modules" is an overloaded term. I'm not referring to go modules or libraries*

The core `windapsearch` code shouldn't care what the contents of the module are, it just needs to be told what to run and it should run it. This is where I decided to use a Go [interface](https://gobyexample.com/interfaces). Every "module" must fulfill the interface defined below:

```go
type Module interface {
	Name() string
	Description() string
	FlagSet() *pflag.FlagSet
	DefaultAttrs() []string
	Run(session *ldapsession.LDAPSession, attrs []string) error
}
```
The benefit of using an interface, is my core `windapsearch` code can be high level. It doesn't care or need to know what the modules are, it just knows that they all will have these functions. Name and Description are used for the module list on the help page, `FlagSet` is used for modules to define their own custom command line args, and `DefaultAttrs` allows modules to specify default attributes that make sense given the context. Finally the `Run` function just needs to take in an already connected LDAP session, run whatever queries need to be run, and return an error.

For example, here is a complete, simple module:
```go
type AdminObjects struct{}

func init() {
	AllModules = append(AllModules, new(AdminObjects))
}

func (AdminObjects) Name() string {
	return "admin-objects"
}

func (AdminObjects) Description() string {
	return "Enumerate all objects with protected ACLs (i.e admins)"
}

func (AdminObjects) FlagSet() *pflag.FlagSet {
    // this module doesn't have any other command line flags, but we still have to return a *pflag.FlagSet
	return pflag.NewFlagSet("adminobjects", pflag.ExitOnError)
}

func (AdminObjects) DefaultAttrs() []string {
	return []string{"cn"}
}

func (AdminObjects) Run(session *ldapsession.LDAPSession, attrs []string) error {
	filter := "(adminCount=1)"
	sr := session.MakeSimpleSearchRequest(filter, attrs)
	return session.ExecuteSearchRequest(sr)
}
```
If `-m admin-objects` is specified at the command line, `windapsearch` will choose this module from `AllModules`, save it as `w.Module` and simply execute it by passing in the already configured ldap session:

```go
err := w.Module.Run(w.LDAPSession, attrs)
if err != nil {
    return err
}
```

Breaking the code up this way let me write the modules really rapidly, without having to worry about the underlying LDAP connections or results parsing. For more examples of modules, including some more advanced ones, see the complete [list here](https://github.com/ropnop/go-windapsearch/tree/master/pkg/modules)

This was my first time using interfaces in Go, and before this I hadn't fully understood them or seen the value. Now I get it!

# Concurrency and Channels
To tackle the second problem I had with 

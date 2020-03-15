---
title: "Hosting the CLR and executing .NET assemblies from Go"
author: "ropnop"
draft: false
slug: "hosting-clr-in-golang"
date: 2020-03-15
summary: "Write up of my journey figuring out how to host the CLR and execute .NET assemblies from memory in pure Go."
toc: true
share_img: "/images/2020/03/go-clr-dll.PNG"
tags: ["golang", "windows", ".net", "clr"]
---

# Intro
A while back, I was [nerd sniped](https://xkcd.com/356/) on Twitter when someone asked if Go could be used to run .NET assemblies (DLLs or EXEs). Although I've been writing a lot of Go recently (and love it), I had never done much with Go on Windows, and certainly nothing advanced like interacting with syscalls or working with CGo before. This sounded like a really fun challenge and I decided to spend some time digging into it. After reading the amazing [Black Hat Go](https://nostarch.com/blackhatgo) book, I thought maybe I knew enough to make it work and that it couldn't be that complicated. I was wrong. It was really hard. But in the end I got a PoC working, learned a ton, and decided to share my journey.

Before I jump in, I want to state that I was (and still am) a complete n00b when it comes to .NET. I've never written anything more advanced than a Hello World. I also don't really know C/C++, and even less when it comes to C/C++ on Windows. With that being said, there are probably better ways to do this, and I probably have things wrong. But I wanted to share for others. Nobody is immediately an expert on anything, and code doesn't come together magically the first time. I went from knowing nothing to having something working in a few weekends just by Googling and playing around. So this is the story of how I stumbled, bumbled, and Stack Overflow'd my way into running .NET assemblies in Golang. 

## tl;dr
I'm releasing my PoC code on Github: [go-clr](https://github.com/ropnop/go-clr). This package let's you host the CLR and execute DLLs from disk or managed assemblies from memory. In short, you can run something like:

```go
import (
    clr "github.com/ropnop/go-clr"
    log
    fmt
)

func main() {
    var exeBytes = []byte{0xa1 ....etc}
    retCode, err := clr.ExecuteByteArray(exeBytes)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf(["[+] Exit code: %d\n", retCode)
}
```

Check out the examples for more code and the [GoDocs](https://godoc.org/github.com/ropnop/go-clr) for exposed structs and functions.

# Background - Syscalls vs CGo
There are two main ways within Go you can interact with Windows APIs: syscalls or CGo. Syscalls require identifying and loading DLLs from the system and finding the exported function you want to call, where CGo lets you write C to do the "heavy lifting" and then call it from Go. For example, here's two ways to pop a Windows message box in Golang. First, using CGo to write the function in C and then calling it from Go:

```go
package main

/*
#include <windows.h>

void SayHello() {
	MessageBox(0, "Hello World", "Helo", MB_OK);
}
*/
import "C"

func main() {
	C.SayHello()
}
```
One of the problems with this approach is CGo requires a GNU Compiler, which means simply running `go build` out of the box won't work. You need to set up a build system with something like [msys2](https://www.msys2.org/). Go is also very opinionated about where you can include header files from, and I encountered some errors importing more advanced things I needed. The other problem is you need to write C. 

The other way to call the Windows API is using exported functions from DLLs. For example:

```go
// +build windows
package main

import (
	"fmt"
	"syscall"
	"unsafe"
)

func main() {
	user32 := syscall.MustLoadDLL("user32.dll")
	messageBox := user32.MustFindProc("MessageBoxW")
	text, _ := syscall.UTF16PtrFromString("Hello World!")
	caption, _ := syscall.UTF16PtrFromString("Hello")
	MB_OK := 0
	ret, _, _ := messageBox.Call(
		uintptr(0),
		uintptr(unsafe.Pointer(text)),
		uintptr(unsafe.Pointer(caption)),
		uintptr(MB_OK))
	fmt.Printf("Returned: %d\n", ret)
}
```
This example requires quite a lot more Go code, but is much easier to build (a simple `go build` on Windows will work). It finds `user32.dll` and the exported `MessageBoxW` function, then makes a syscall to the function with necessary pointers. It requires converting strings to UTF16 pointers and makes use of `unsafe.Pointer`. I know, it looks really messy and confusing - but IMO this is the better way to do it. It let's us write "pure" Go with no C dependencies. This is how I decided to write `go-clr`.

For a great introduction to calling the Windows API from Go and using `unsafe`, I found this article to be an amazing resource: https://medium.com/jettech/breaking-all-the-rules-using-go-to-call-windows-api-2cbfd8c79724

# Background - Calling Managed Code from Unmanaged Code
Like I mentioned, when I started this journey I had barely any knowledge of .NET and how it worked. I started by reading a few articles to understand what was necessary and the concept of "managed" vs "unmanaged" code came up. "Unmanaged" code usually refers to low level C/C++ code that is compiled and linked and "just works" if you execute the instructions. "Managed" code refers to code that is written to target .NET and will not "just work" without the CLR. I came to think of it conceptually as requiring an interpreter - e.g. you can't just run Python code without Python being installed and calling the interpreter. You can't just run .NET code without calling the CLR.

The Common Language Runtime, or CLR, is what is used by .NET to "interpret" the code. I still don't fully understand it, but I conceptually came to think of it as Microsoft's version of the Java Virtual Machine (JVM). When you write Java, it compiles to an intermediate bytecode and a JVM is required to execute it. Similarly then, when you write .NET, it compiles to an intermediate language and requires a CLR to execute it. I'm probably wrong and missing a lot, but but in the end all I needed to know was you need to create or attach to a CLR before you can execute .NET assemblies.

Knowing these basic terms and concepts helped greatly when Googling. I was able to Google things like "hosting CLR" and "running managed code from unmanaged" and get very good results.

Two of those results had excellent examples of running managed code from C/C++:
 * [clr_via_native.c](https://gist.github.com/xpn/e95a62c6afcf06ede52568fcd8187cc2) - gist by [xpn](https://twitter.com/_xpn_)
 * [Hosting the CLR the Right Way](https://www.mode19.net/posts/clrhostingright/)

These two articles had examples of launching .NET assemblies from C. Since I knew how to open a message box in Go by calling the Windows API, I figured I had all the skills I needed to recreate their code in Go. How hard could it be? Yeah...I just had to draw the rest of the owl:

![the rest of the owl](/images/2020/03/rest-of-owl.jpg)

# Part 1 - Loading a Managed DLL from Disk
I set about trying to basically re-write the two examples I found into pure Go using syscalls. xpn's code was only 70 lines, and after reading it over several times, I got the basic steps down that were necessary:

 1) Create a "MetaHost" instance
 2) Enumerate the runtimes
 3) Get an "Interface" to that runtime
 4) "Start" the Interface
 5) Call "ExecuteInDefaultAppDomain" and pass in the DLL and arguments

What was extremely helpful to me was to have xpn's code open in Visual Studio. This way I could right click on functions/constants and "View Definition" to see where they were coming from.

## Calling CLRCreateInstance
The first step in the whole process is to create a MetaHost instance by calling a native function. It looked like this in the sample code:
```C
CLRCreateInstance(CLSID_CLRMetaHost, IID_ICLRMetaHost, (LPVOID*)&metaHost)
```
The [MSDN docs](https://docs.microsoft.com/en-us/dotnet/framework/unmanaged-api/hosting/clrcreateinstance-function) on that function let me know that it is part of `MSCorEE.dll`. So to load in Go:

```go
var (
	modMSCoree            = syscall.NewLazyDLL("mscoree.dll")
	procCLRCreateInstance = modMSCoree.NewProc("CLRCreateInstance")
)
```

It takes 3 arguments: `clsid`, `riid` and `ppInterface`. The first two arguments are pointers to GUIDs, and the third is a pointer to a a pointer that will point to the new MetaInstance created. How to create the GUIDs in Go? First I right clicked in Visual Studio to see the definition for the constant `CLSID_CLRMetaHost`. It is defined in `metahost.h` and looks like this:

```C
EXTERN_GUID(CLSID_CLRMetaHost, 0x9280188d, 0xe8e, 0x4867, 0xb3, 0xc, 0x7f, 0xa8, 0x38, 0x84, 0xe8, 0xde);
```
This took me several hours of Googling. I went down the wrong path at first of trying to convert the GUID to a string. But then I discovered the `syscall` package has a [GUID type already](https://golang.org/pkg/syscall/?GOOS=windows#GUID). Fortunately, the `metahost.h` definitions matched up exactly with the parameters the struct expected, and I could re-create the GUID in Go by copying the values in like this:

```go
import "golang.org/x/sys/windows"
CLSID_CLRMetaHost   = windows.GUID{0x9280188d, 0xe8e, 0x4867, [8]byte{0xb3, 0xc, 0x7f, 0xa8, 0x38, 0x84, 0xe8, 0xde}}
```
To get the pointer argument, I used an empty `uintptr` variable. The final call to the `CLRCreateInstance` function looked like this:

```go
var pMetaHost uintptr
hr, _, _ := procCLRCreateInstance.Call(
  uintptr(unsafe.Pointer(&CLSID_CLRMetaHost)),
  uintptr(unsafe.Pointer(&IID_ICLRMetaHost)),
  uintptr(unsafe.Pointer(&pMetaHost))
)
checkOK(hr, "procCLRCreateInstance")
```
I passed in unsafe pointers to the GUID structs and an unsafe pointer to a null pointer (essentially). If the function returns successfully, the value of `pMetaHost` should be populated with the actual memory address of the new CLR MetaHost instance.

The function returns an `HRESULT`. If the value is equal to 0, it was successful. So I wrote a helper function to compare an `hr` to zero and panic if it failed:
```go
func checkOK(hr uintptr, caller string) {
    if hr != 0x0 {
        log.Fatalf("%s returned 0x%08x", caller, hr)
    }
}
```

And it worked! I could see the value of `pMetaHost` was populated with an address. Now what to do with it?

## Recreating Interfaces in Go
This is where everything got really hard, really fast. Calling exported functions from DLLs was fairly straightforward, but now I was dealing with pointers which pointed to interfaces which pointed to other functions. I knew the next step in the chain was to call `metaHost->EnumerateInstalledRuntimes`, but all I had was a pointer to a metaHost object in memory. 

Again, I am terrible at C/C++. But I knew enough to know that if memory layout all matched up, I could "cast" a pointer to an object. Fortunately, the same holds true for Go using the unsafe package. If I recreated the `ICLRMetaHost` interface as a struct in Go, it would be possible to convert my pointer to it. Once again, right clicking and "Viewing Definition" on the ICLRMetaHost interface let me see it defined in `metahost.c`. It appeared to be defined twice: once for C++ and once for C. I focused on the C interface:
![clrmetahost c interface](/images/2020/03/clrmetahost_c_interface.PNG)

Looked like it actually defined two interfaces: `ICLRMetaHostVtbl` and then `ICLRMetaHost`, which only has a pointer the Vtbl. I assumed that `STDMETHODCALLTYPE` would just be a pointer to a function, so a `uintptr` in Go would be fine. I followed along with the "C Structs & Go Structs" section on the [post](https://medium.com/jettech/breaking-all-the-rules-using-go-to-call-windows-api-2cbfd8c79724) I referenced earlier and ended up with this defined in Go:

```go
//ICLRMetaHost Interface from metahost.h
type ICLRMetaHost struct {
	vtbl *ICLRMetaHostVtbl
}

type ICLRMetaHostVtbl struct {
	QueryInterface                   uintptr
	AddRef                           uintptr
	Release                          uintptr
	GetRuntime                       uintptr
	GetVersionFromFile               uintptr
	EnumerateInstalledRuntimes       uintptr
	EnumerateLoadedRuntimes          uintptr
	RequestRuntimeLoadedNotification uintptr
	QueryLegacyV2RuntimeBinding      uintptr
	ExitProcess                      uintptr
}

func NewICLRMetaHostFromPtr(ppv uintptr) *ICLRMetaHost {
	return (*ICLRMetaHost)(unsafe.Pointer(ppv))
}
```

The `NewICLRMetaHostFromPtr` function takes the pointer I get from `CLRCreateInstance` and returns an ICLRMetaHost object. This all appeared to work fine. I even did a `fmt.Printf("+%v", metaHost)` on the object and could see that every struct field was populated with a pointer, so it looked like it was working.

Now to call a function like `EnumerateInstalledRuntimes`, I could use `syscall.Syscall` with the address of the function stored in `ICLRMetaHost.vtbl.EnumerateInstalledRuntimes`. Right?

## Calling Interface Methods
The [EnumerateInstalledRuntimes](https://docs.microsoft.com/en-us/dotnet/framework/unmanaged-api/hosting/iclrmetahost-enumerateinstalledruntimes-method) method only takes in one parameter: a pointer to a pointer for the return value (an enumerator). So I implemented the call like this:

```go
var pInstalledRuntimes uintptr
hr, _, _ := syscall.Syscall(
  metaHost.vtbl.EnumerateInstalledRuntimes,
  1,
  uintptr(unsafe.Pointer(&pInstalledRuntimes)),
  0,
  0,
  0
)
checkOK(hr, "metaHost.EnumerateInstalledRuntimes")
```

*Note: The `0`s are necessary since `syscall.Syscall` requires 6 arguments, but only uses whatever is necessary.*

Except...it didn't work. No matter what I tried or tinkered with, I never got a good return value (i.e. `0`). At this point I was well beyond my comfort level and had no idea how to troubleshoot, so I considered giving up and scrapping the whole idea. However one night I kept Googling certain terms I found in the C interface, like "vtbl" and "IUnknown" along with "golang", and stumbled on a goldmine. 

Turns out I can't just treat these functions as native functions. They are methods of a COM object. I'll be honest - I had heard of Component Object Model (COM), but knew nothing about it and didn't have the time or patience to learn it. I still don't understand it. But I found this amazing Stack Overflow answer about implementing COM methods in Golang: https://stackoverflow.com/questions/37781676/how-to-use-com-component-object-model-in-golang

Apparently, when calling a COM method, I have to make sure `AddRef` and `Release` are implemented, and pass a pointer to the object itself as the first argument. I essentially copied the code from that SO answer and boom - I got a good return code for the `EnumerateInstalledRuntimes` function:

```go
func (obj *ICLRMetaHost) EnumerateInstalledRuntimes(pInstalledRuntimes *uintptr) uintptr {
	ret, _, _ := syscall.Syscall(
	  obj.vtbl.EnumerateInstalledRuntimes,
	  2,
	  uintptr(unsafe.Pointer(obj)),
	  uintptr(unsafe.Pointer(pInstalledRuntimes)),
	  0)
	return ret
}
```

## Implementing Additional Interfaces
That one SO answer cracked this all open for me. Now I knew how to implement the C style interfaces I was finding in header files in Visual Studio and call the functions I needed. Next up was implementing the `IEnumUnknown` interface, since that's what `EnumerateInstalledRuntimes` points to. I got really good at quickly copying the interfaces from header files to Go with Vim macros, and I just copied/pasted the standard function implementations. One thing I did was not bother implementing functions I was never going to call - it didn't matter as long as they were included in the struct definition for padding.

After the `IEnumUnknown` interface, I also needed the [ICLRRuntimeInfo](https://docs.microsoft.com/en-us/dotnet/framework/unmanaged-api/hosting/iclrruntimeinfo-interface) interface from `metahost.h` as well.

To enumerate installed runtime versions as strings, I had to do a little memory hackery. [GetVersionString](https://docs.microsoft.com/en-us/dotnet/framework/unmanaged-api/hosting/iclrruntimeinfo-getversionstring-method) doesn't actually return a string, but writes a UTF16 string to a spot in memory. So in Go, I allocated a 20 byte buffer array and passed the pointer to that as the spot to write the string to. Then I convert the buffer array to a UTF16 string that's Go friendly. One thing I learned was to ensure I pass the pointer to the first element of the array - not the array itself. Ultimately the loop for enumerating runtimes looked like this:

```go
var rutimes []string
var pRuntimeInfo uintptr
var fetched = uint32(0)
var versionString string
versionStringBytes := make([]uint16, 20)
versionStringSize := uint32(len(versionStringBytes))
var runtimeInfo *ICLRRuntimeInfo
for {
    hr = installedRuntimes.Next(1, &pRuntimeInfo, &fetched)
    if hr != 0x0 {
        break
    }
    runtimeInfo = NewICLRRuntimeInfoFromPtr(pRuntimeInfo)
    if ret := runtimeInfo.GetVersionString(&versionStringBytes[0], &versionStringSize); ret != 0x0 {
        log.Fatalf("GetVersionString returned 0x%08x", ret)
    }
    versionString = syscall.UTF16ToString(versionStringBytes)
    runtimes = append(runtimes, versionString)
}
fmt.Printf("[+] Installed runtimes: %s\n", runtimes)
```
Amazingly, this actually worked the first time I ran it and I couldn't believe it. Maybe I was finally starting to get the hang of writing C in Go ;)

## Executing a DLL From Disk
I won't go in to all the other interfaces I created, but I essentially just went one by one converting the C from [xpn](https://gist.github.com/xpn/e95a62c6afcf06ede52568fcd8187cc2) and this [blog](https://www.mode19.net/posts/clrhostingright/). I ended up implementing:
 * [ICLRRuntimeInfo](https://docs.microsoft.com/en-us/dotnet/framework/unmanaged-api/hosting/iclrruntimeinfo-interface)
 * [ICLRRuntimeHost](https://docs.microsoft.com/en-us/dotnet/framework/unmanaged-api/hosting/iclrruntimehost-interface)
 * [IUnknown](https://docs.microsoft.com/en-us/windows/win32/api/unknwn/nn-unknwn-iunknown)

Finally I got to the point where I was ready to try out my implementation of ICLRRuntimeHost's [ExecuteInDefaultAppDomain](https://docs.microsoft.com/en-us/dotnet/framework/unmanaged-api/hosting/iclrruntimehost-executeindefaultappdomain-method) method. I wrote a dead simple C# program to test (which I still had to look up online how to do - told you..I don't know C#...):

```csharp
using System;
using System.Windows.Forms;

namespace TestDLL
{
    public class HelloWorld
    {
        public static int SayHello(string foobar)
        {
            MessageBox.Show("Hello from a C# DLL!");
            return 0;
        }
    }
}
```
And I compiled it to a DLL with: `csc -target:library -out:TestDLL.dll TestDLL.cs`. Then in Go I converted the filename, type name, method name and argument to UTF16 string pointers and called the method:

```golang
pDLLPath, _ := syscall.UTF16PtrFromString("TestDLL.dll")
pTypeName, _ := syscall.UTF16PtrFromString("TestDLL.HelloWorld")
pMethodName, _ := syscall.UTF16PtrFromString("SayHello")
pArgument, _ := syscall.UTF16PtrFromString("foobar")
var pReturnVal *uint16
hr = runtimeHost.ExecuteInDefaultAppDomain(
    pDLLPath,
    pTypeName,
    pMethodName,
    pArgument,
    pReturnVal
)
checkOK(hr, "runtimeHost.ExecuteInDefaultAppDomain")
fmt.Printf("[+] Assembly returned: 0x%x\n", pReturnVal)
```
And it worked!! 
![DLL from disk](/images/2020/03/go-clr-dll.PNG)

After getting it working, I cleaned up the code a little bit and added some helper functions. You can see the full example from start to finish here: [DLLFromDisk.go](https://github.com/ropnop/go-clr/blob/master/examples/DLLfromDisk/DLLfromDisk.go)

I also added a wrapper function that basically automates the entire process, which you can call with just:

```go
ret, err := clr.ExecuteDLLFromDisk(
  "TestDLL.dll",
  "TestDLL.HelloWorld",
  "SayHello",
  "foobar")
```

So that was cool and all...but what I really wanted to be able to do was load the assembly from memory. This required the DLL existing on disk, and I had dreams and visions of downloading a DLL or embedding it inside a Go binary and just having it execute. I thought the hard part was done and this would be easy....I was wrong.

# Part 2 - Executing Assemblies from Memory
My initial thought was to leverage virtual filesystem in Go, like [vfs](https://github.com/blang/vfs) or [packr2](https://github.com/gobuffalo/packr/tree/master/v2) and keep the DLL in memory. Of course I soon realized that was impossible, since the path to the DLL was being passed to a native function I had no control over and that function would always look on disk.

I also looked through the MSDN documents for ICLRRuntimeHost and couldn't find any reference to loading or executing things from memory. But I remembered hearing and seeing others do this in some offensive tools, so I turned back to Google and found two tools that were executing .NET assemblies from memory using native code:

 * [Donut](https://github.com/TheWover/donut) - especially [rundotnet.cpp](https://github.com/TheWover/donut/blob/master/DonutTest/rundotnet.cpp)
   * Also this [blogpost](https://thewover.github.io/Introducing-Donut/) about Donut and CLR
 * [GrayFrost](https://github.com/GrayKernel/GrayFrost) - especially [Runtimer.cpp](https://github.com/GrayKernel/GrayFrost/blob/master/GrayFrost/Runtimer.cpp)

 Looking at that example code, I realized they had to use the [deprecated CLR methods](https://docs.microsoft.com/en-us/dotnet/framework/unmanaged-api/hosting/deprecated-clr-hosting-functions) to achieve in memory execution. So I needed to rewrite and re-implement a lot more in Go, specifically the [ICORRuntimeHost](https://docs.microsoft.com/en-us/dotnet/framework/unmanaged-api/hosting/icorruntimehost-interface). 

 The new high level flow looked like this:
   1) Create a MetaHost instance
   2) Enumerate the installed runtimes
   3) Get RuntimeInfo to latest installed version
   4) Run BindAsLegacyV2Runtime()
   5) Get ICORRuntimeHost interface
   6) Get default app domain from interface
   7) Load assembly into app domain
   8) Find entrypoint to loaded assembly
   9) Call entrypoint

Yeah...quite a bit more complicated than just running a DLL from disk.

## Loading an Assembly into Memory
After implementing ICORRuntimeHost and AppDomain in Go, I realized looking at Donut and GrayFrost's code that in order to call the `Load_3` method inside an AppDomain, the bytecode had to be in a specific format: namely a [SafeArray](https://docs.microsoft.com/en-us/archive/msdn-magazine/2017/march/introducing-the-safearray-data-structure). 

This was a fun rabbit hole to go down (sarcasm). I needed to figure out how to convert a byte array in Go to a Safe Array in memory. First, I created a Go struct based on the definition in `OAld.h`:

```go
type SafeArray struct {
	cDims      uint16
	fFeatures  uint16
	cbElements uint32
	cLocks     uint32
	pvData     uintptr
	rgsabound  [1]SafeArrayBound
}

type SafeArrayBound struct {
	cElements uint32
	lLbound   int32
}
```

After a few more nights of Googling and reading public C/C++ code on Github that implemented safe arrays, I realized I could create a SafeArray through a native function ([SafeArrayCreate](https://docs.microsoft.com/en-us/windows/win32/api/oleauto/nf-oleauto-safearraycreate)), and then use a raw memory copy to put bytes in the correct place. First, to create the SafeArray:

```go
var rawBytes = []byte{0xaa....} // my executable loaded in a byte array
modOleAuto, err := syscall.LoadDLL("OleAut32.dll")
must(err)
procSafeArrayCreate, err := modOleAuto.FindProc("SafeArrayCreate")
must(err)

size := len(rawBytes)
sab := SafeArrayBound{
    cElements: uint32(size),
    lLbound:   0,
}
runtime.KeepAlive(sab)
vt := uint16(0x11) // VT_UI1
ret, _, _ := procSafeArrayCreate.Call(
    uintptr(vt),
    uintptr(1),
    uintptr(unsafe.Pointer(&sab)))
sa := (*SafeArray)(unsafe.Pointer(ret))
```
I still couldn't figure out what the hell `vt` is and should be. I honestly just copied the value from Donut (`0x11`) which corresponds to a `VT_UI1`, and it worked so I stuck with it.

The procedure returns a pointer to a created SafeArray. Now our actual data (the bytes) need to be copied into memory where `safeArray.pvData` points to. I couldn't figure out a way to do this in native Go, so I imported and used `RtlCopyMemory` from `ntdll.dll` to perform a raw memory copy:

```go
modNtDll, err := syscall.LoadDLL("ntdll.dll")
must(err)
procRtlCopyMemory, err := modNtDll.FindProc("RtlCopyMemory")
must(err)

ret, _, err = procRtlCopyMemory.Call(
    sa.pvData,
    uintptr(unsafe.Pointer(&rawBytes[0])),
    uintptr(size))
```
Since SafeArrayCreate allocates the memory based on the `cElements` value (which is equal to the size of our byte array), we can just copy directly to that point in memory. Surprisingly, this worked. 

I ultimately ended up wrapping this in a helper function, [CreateSafeArray](https://github.com/ropnop/go-clr/blob/master/safearray.go#L37), that I would take in a byte array and return a pointer to a SafeArray in memory.

## Finding and Calling the Entry Point
Once a SafeArray was created, it could be loaded into an AppDomain with the `Load_3` method:

```go
func (obj *AppDomain) Load_3(pRawAssembly uintptr, asmbly *uintptr) uintptr {
	ret, _, _ := syscall.Syscall(
		obj.vtbl.Load_3,
		3,
		uintptr(unsafe.Pointer(obj)),
		uintptr(unsafe.Pointer(pRawAssembly)),
		uintptr(unsafe.Pointer(asmbly)))
	return ret
}
```
This gave me a pointer to an [Assembly object](https://docs.microsoft.com/en-us/dotnet/api/system.reflection.assembly?view=netframework-4.8). I needed to next implement the Assembly interface in Go. Looking at in Visual Studio (from `mscorlib.tlh`) it looked like any other interface I had implemented:

![assembly interface](/images/2020/03/assembly_interface.PNG)

But at this point, not matter what I tried, I kept getting memory out of bound exceptions when calling the `get_EntryPoint` and `Invoke_3` methods. Something was really wrong and I couldn't figure it out. Since I had passed my comfort zone wayyy back - I was again almost ready to just give up. For probably 6 straight nights I kept re-reading and re-writing my code over and over but couldn't figure it out.

I eventually started using debuggers to read memory and compare hex dumps between a working C++ program and my Go program, but still couldn't spot the difference. 

I started searching Github for other Go projects that were doing Windows API calls and found [w32](https://github.com/JamesHovious/w32) from James Hovious. I actually knew of this project before, but decided to really start reading his source code. I came across his implementation of [IDispatch](https://github.com/JamesHovious/w32/blob/master/idispatch.go). I suddenly remembered that the interface definition for Assembly mentioned `IDispatch`. I saw in his code that the vtbl struct included some additional methods I didn't have:

```golang
type pIDispatchVtbl struct {
	pQueryInterface   uintptr
	pAddRef           uintptr
	pRelease          uintptr
	pGetTypeInfoCount uintptr
	pGetTypeInfo      uintptr
	pGetIDsOfNames    uintptr
	pInvoke           uintptr
}
```

I don't know what an IDispatch is and WTF I have no idea what these other functions are or what they do, but it dawned on me that if they are missing from my struct definition, then the memory won't line up correctly. I added them in to the start of the [Assembly struct](https://github.com/ropnop/go-clr/blob/master/assembly.go#L22-L25) and everything started working! I wasted nearly 2 weeks chasing this bug. Sometimes I hate computers.

### Calling the Entry Point
After finding the method entry point and creating a MethodInfo object, the last step is to just invoke the function with the `Invoke_3` method. This function is defined in `mscorlib.tlh` as:

![invoke_3 definition](/images/2020/03/invoke_3_interface.PNG)

My initial thought when I saw that was "great...what's a VARIANT". Looking at the definition of VARIANT in `OAldl.h`, my heart sank even further. It was some crazy struct with UNION definitions accounting for all different types of values. To simplify things, I decided to not bother trying to implement passing arguments to the method so I could just use a null variant and null pointer as the parameters to `Invoke_3`. This is what Dount and GrayFrost do as well.I searched for "variant" and "golang" and found some implementations in the [go-ole project](https://github.com/go-ole/go-ole/blob/master/variant_amd64.go) that I could use.

Putting it together:
```go
safeArray, err := CreateSafeArray(exebytes)
must(err)
var pAssembly uintptr
hr = appDomain.Load_3(uintptr(unsafe.Pointer(&safeArray)), &pAssembly)
checkOK(hr)
assembly := NewAssemblyFromPtr(pAssembly)
var pEntryPointInfo uintptr
hr = assembly.GetEntryPoint(&pEntryPointInfo)
checkOK(hr)
methodInfo := NewMethodInfoFromPtr(pEntryPointInfo)
var pRetCode uintptr
nullVariant := Variant{
    VT:  1,
    Val: uintptr(0),
}
hr = methodInfo.Invoke_3(
    nullVariant,
    uintptr(0),
    &pRetCode)
checkOK(hr)
fmt.Printf("[+] Executable returned code %d\n", pRetCode)
```
To test it, I created a Hello World C# EXE by Googling "Hello World C# EXE":

```csharp
using System;

namespace TestExe
{
    class HelloWorld
    {
        static void Main()
        {
            Console.WriteLine("hello fom a c# exe!");
        }
    }
}
```
I built it with `csc TestExe.cs`, then loaded it in to a byte array in Go:

```go
exebytes, err := ioutil.ReadFile("TestExe.exe")
must(err)
```

And then created a SafeArray from it and went through the magic incantations necessary to get everything into place, and....

![exe from memory](/images/2020/03/go-clr-exe-from-memory.PNG)

IT WORKED!! After several weeks of reading, Googling, copying/pasting code, trying things, building, crashing, re-building, re-crashing, debugging, giving up, coming back, feeling stupid, feeling smart, feeling stupid again and then feeling accomplished - I finally got it working. What a journey.

You can see the full example for loading an EXE from memory [here](https://github.com/ropnop/go-clr/blob/master/examples/EXEfromMemory/EXEfromMemory.go).

# Conclusion
This was a really challenging but very rewarding experiment. I went from knowing next to nothing about the CLR, COM, and Go syscalls to building something pretty cool. That being said - I don't think this is "production" ready and probably never will be. To be honest, I don't understand enough of what it's doing to really be able to troubleshoot, and I've noticed it can be pretty unstable.

But my hope in releasing the code is to show how others how it's possible and give building blocks to expand upon for writing custom tooling. I also wanted to write up this post about my journey to hopefully inspire others that you can be a complete noob on a topic and still figure things out - there's so much good information out there in the depths of Google, Github, and Stack Overflow.

Looking forward to seeing how other's use/build upon this research! Let me know if you have any questions (and I'll try my best to answer). Also, if you understand the stuff better than me - please let me know where I'm wrong and where I could improve the code! For example I seem to be hitting garbage collection issues in Go where randomly things fail if I add to may "fmt.Printf" calls. I don't get it. Sometimes I hate computers :)

-ropnop

## References and Acknowledgements
I found the following pages very helpful. In no particular order:
 * https://gist.github.com/Arno0x/386ebfebd78ee4f0cbbbb2a7c4405f74
 * https://www.mode19.net/posts/clrhostingright/
 * https://github.com/RickStrahl/wwDotnetBridge/pull/18/commits/307e70ac70c17be5744ce6cc0cd1b0a0ee947758
 * https://yizhang82.dev/calling-com-from-go
 * https://stackoverflow.com/questions/37781676/how-to-use-com-component-object-model-in-golang
 * https://github.com/AllenDang/w32/blob/c92a5d7c8fed59d96a94905c1a4070fdb79478c9/typedef.go
 * https://www.codeproject.com/Articles/607352/Injecting-NET-Assemblies-Into-Unmanaged-Processes
 * https://www.unknowncheats.me/forum/general-programming-and-reversing/332825-inject-net-dll-using-clr-hosting.html
 * https://blog.xpnsec.com/rundll32-your-dotnet/
 * http://sbytestream.pythonanywhere.com/blog/clr-hosting
 * https://www.codeproject.com/Articles/416471/CLR-Hosting-Customizing-the-CLR
 * https://thewover.github.io/Introducing-Donut/
 * https://github.com/TheWover/donut/blob/9c07b2fde9ac489fffca409fbd7c2228c90c3373/loader/clr.h
 * https://github.com/TheWover/donut/blob/9c07b2fde9ac489fffca409fbd7c2228c90c3373/loader/inmem_dotnet.c
 * https://github.com/go-ole/go-ole/blob/master/idispatch.go#L5
 * https://medium.com/jettech/breaking-all-the-rules-using-go-to-call-windows-api-2cbfd8c79724
 * https://gist.github.com/xpn/e95a62c6afcf06ede52568fcd8187cc2


































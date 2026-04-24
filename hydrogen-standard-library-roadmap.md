# Hydrogen Standard Library Roadmap

## Goal

Build a fully usable Hydrogen standard library in small, practical phases.

Hydrogen should be standalone. It should not depend on the C standard library, .NET, the CLR, or another language runtime.

The standard library should grow from the smallest possible freestanding core into a complete Australis OS development platform.

---

# Library Layers

```text
Hydrogen.Core        // always available, no OS dependency
Hydrogen.Memory      // allocators, slices, buffers
Hydrogen.Runtime     // optional runtime services
Hydrogen.Text        // strings, UTF-8, formatting
Hydrogen.IO          // platform-backed IO
Australis.Kernel     // kernel-only APIs
Australis.Userspace  // app/service APIs
Australis.UI         // windowing/UI APIs
```

## Design Rule

Every library layer should clearly state:

- Whether it works in freestanding mode
- Whether it requires heap allocation
- Whether it requires Australis OS
- Whether it requires the Hydrogen runtime
- Whether it is safe, unsafe, or mixed

---

# Phase 0 — Standard Library Foundations

## Purpose

Create the rules and structure before adding lots of APIs.

This phase prevents the standard library from becoming messy later.

## Deliverables

```text
/std
  /Hydrogen.Core
  /Hydrogen.Memory
  /Hydrogen.Runtime
  /Hydrogen.Text
  /Hydrogen.IO
  /Australis.Kernel
  /Australis.Userspace
  /Australis.UI
```

## Decisions to Make

- Naming convention: `PascalCase`
- File/module structure
- Public/private/internal rules
- How imports work
- How docs are generated
- How tests are written
- How freestanding builds exclude unavailable APIs

## Required Compiler Support

- module imports
- public/private visibility
- basic package/project references
- build targets
- standard library lookup path

## Completion Criteria

This phase is complete when a Hydrogen file can import a standard library module:

```hy
use Hydrogen.Core;

public function Main(): int {
    return 0;
}
```

---

# Phase 1 — Hydrogen.Core

## Purpose

Create the always-available base library.

`Hydrogen.Core` should work everywhere:

- kernel mode
- userspace mode
- freestanding mode
- runtime-enabled mode
- runtime-disabled mode

It should have no OS dependency.

## Modules

```text
Hydrogen.Core.Types
Hydrogen.Core.Result
Hydrogen.Core.Option
Hydrogen.Core.Math
Hydrogen.Core.Diagnostics
Hydrogen.Core.Convert
Hydrogen.Core.Bitwise
Hydrogen.Core.Compare
Hydrogen.Core.Panic
```

## Features

### Primitive Type Helpers

Support common type aliases and helpers:

```hy
u8
u16
u32
u64
i8
i16
i32
i64
usize
isize
bool
char
byte
void
```

### Result

```hy
Result<T, E>
Ok(value)
Error(error)
```

Example:

```hy
function Divide(a: int, b: int): Result<int, string> {
    if (b == 0) {
        return Error("Cannot divide by zero");
    }

    return Ok(a / b);
}
```

### Option

```hy
Option<T>
Some(value)
None
```

Example:

```hy
function FindUser(id: int): Option<User> {
    ...
}
```

### Panic

```hy
Panic("Something went wrong");
```

Behaviour depends on target:

- freestanding: halt or trap
- kernel: kernel panic
- userspace: terminate process
- debug: include file/line if available

### Math

Start small:

```hy
Math.Min(a, b)
Math.Max(a, b)
Math.Abs(x)
Math.Clamp(value, min, max)
```

Avoid floating-point-heavy APIs until later.

### Bitwise Helpers

Useful for OS work:

```hy
Bits.Set(value, bit)
Bits.Clear(value, bit)
Bits.Toggle(value, bit)
Bits.Has(value, bit)
Bits.AlignUp(value, alignment)
Bits.AlignDown(value, alignment)
```

### Conversion

Basic explicit conversions:

```hy
Convert.ToInt(...)
Convert.ToUInt(...)
Convert.ToBool(...)
```

Do not make conversions too magical.

## Required Compiler Support

- generics
- basic structs/enums
- static functions
- compile-time constants
- basic error messages

## Completion Criteria

You can write small Hydrogen programs without touching OS APIs:

```hy
use Hydrogen.Core;

function Main(): int {
    var result = Math.Clamp(150, 0, 100);
    return result;
}
```

---

# Phase 2 — Hydrogen.Memory

## Purpose

Create safe and unsafe memory tools.

This is one of the most important libraries for Hydrogen because the language is low-level and OS-focused.

## Modules

```text
Hydrogen.Memory.Pointer
Hydrogen.Memory.Slice
Hydrogen.Memory.Buffer
Hydrogen.Memory.Allocator
Hydrogen.Memory.FixedBuffer
Hydrogen.Memory.Layout
Hydrogen.Memory.Unsafe
```

## Features

### Pointers

```hy
ptr<T>
null
&value
*pointer
```

Helper APIs:

```hy
Pointer.IsNull(pointer)
Pointer.Offset(pointer, count)
Pointer.Cast<TFrom, TTo>(pointer)
```

### Slices

A slice is a pointer plus a length.

```hy
slice<T>
```

Example:

```hy
function Fill(buffer: slice<byte>, value: byte): void {
    for (var i = 0; i < buffer.Length; i++) {
        buffer[i] = value;
    }
}
```

### Buffers

Owned memory regions:

```hy
Buffer<T>
Buffer<byte>
```

Possible behaviour:

- stores pointer
- stores length
- knows allocator
- can be disposed/freed

Example:

```hy
var buffer = Buffer<byte>.Create(1024);
defer buffer.Free();
```

### Allocator Interface

Hydrogen should not assume one allocator.

```hy
interface IAllocator {
    function Allocate(size: usize, alignment: usize): ptr<byte>;
    function Free(pointer: ptr<byte>): void;
}
```

Default allocators can come later.

### Layout Helpers

```hy
sizeof<T>()
alignof<T>()
offsetof<T>("Field")
```

Example:

```hy
comptime assert(sizeof<GdtEntry>() == 8);
```

### Memory Operations

```hy
Memory.Copy(dest, src, count)
Memory.Move(dest, src, count)
Memory.Set(dest, value, count)
Memory.Zero(dest, count)
Memory.Compare(a, b, count)
```

These must be Hydrogen-native, not libc wrappers.

## Required Compiler Support

- pointer types
- `unsafe`
- `defer`
- fixed-size arrays
- basic compile-time functions
- optional bounds checks

## Completion Criteria

You can allocate, use, and free memory in Hydrogen:

```hy
use Hydrogen.Memory;

function Main(): int {
    var buffer = Buffer<byte>.Create(1024);
    defer buffer.Free();

    Memory.Zero(buffer.Data, buffer.Length);

    return 0;
}
```

---

# Phase 3 — Hydrogen.Text

## Purpose

Create a clear, honest text model.

Hydrogen strings should be UTF-8 by default, but the library must not pretend that bytes and characters are the same thing.

## Modules

```text
Hydrogen.Text.String
Hydrogen.Text.Utf8
Hydrogen.Text.Rune
Hydrogen.Text.Format
Hydrogen.Text.Parse
Hydrogen.Text.Ascii
Hydrogen.Text.StringBuilder
```

## Features

### String Model

Recommended model:

```hy
string    // UTF-8 text
byte      // raw 8-bit value
char/rune // Unicode scalar value
```

Avoid making `string[index]` mean “character at index”.

Prefer:

```hy
text.ByteLength
text.ByteAt(index)
text.RuneCount()
text.RuneAt(index)
text.Runes()
```

### ASCII Helpers

Very useful early on:

```hy
Ascii.IsDigit(c)
Ascii.IsLetter(c)
Ascii.IsWhitespace(c)
Ascii.ToUpper(c)
Ascii.ToLower(c)
```

### UTF-8 Helpers

```hy
Utf8.Validate(bytes)
Utf8.DecodeNext(bytes, index)
Utf8.EncodeRune(rune, buffer)
```

### Formatting

Start simple:

```hy
Format.Int(value)
Format.UInt(value)
Format.Hex(value)
Format.Bool(value)
```

Later:

```hy
String.Format("Value: {0}", value)
```

### Parsing

```hy
Parse.Int("123")
Parse.UInt("123")
Parse.Bool("true")
```

Return `Result`, not exceptions:

```hy
var value = try Parse.Int("123");
```

### StringBuilder

For runtime/userspace builds:

```hy
var builder = StringBuilder.Create();
defer builder.Free();

builder.Append("Hello ");
builder.Append("Hydrogen");

var text = builder.ToString();
```

## Required Compiler Support

- string literals
- byte arrays
- slices
- Result
- optional heap allocation
- iteration

## Completion Criteria

You can format and parse basic text without libc:

```hy
use Hydrogen.Text;

function Main(): int {
    var number = try Parse.Int("42");
    var text = Format.Int(number);
    return 0;
}
```

---

# Phase 4 — Hydrogen.Runtime

## Purpose

Create optional runtime services.

This layer should not be required for freestanding or kernel code unless explicitly enabled.

## Modules

```text
Hydrogen.Runtime.Startup
Hydrogen.Runtime.Heap
Hydrogen.Runtime.GC
Hydrogen.Runtime.Panic
Hydrogen.Runtime.TypeInfo
Hydrogen.Runtime.Debug
Hydrogen.Runtime.Console
Hydrogen.Runtime.Threading
```

## Features

### Startup

Responsible for:

- setting up runtime state
- calling `Main`
- handling return codes
- calling global initializers if supported

### Heap

A runtime heap for normal applications.

```hy
Runtime.Heap.Allocate(size)
Runtime.Heap.Free(pointer)
```

### Optional Garbage Collection

GC should be optional.

Project config example:

```json
{
  "runtime": "native",
  "gc": false
}
```

or:

```json
{
  "runtime": "managed",
  "gc": true
}
```

### Panic Handling

Runtime-enabled panic can include:

- message
- file
- line
- stack trace
- process termination

### Type Info

Optional future feature:

```hy
typeof<T>()
value.GetType()
```

Should not be required for low-level builds.

### Console

Runtime-level console for debugging:

```hy
Console.Write("Hello");
Console.WriteLine("Hydrogen");
```

In freestanding builds, this should either be unavailable or target-specific.

## Required Compiler Support

- runtime profile selection
- entrypoint generation
- optional metadata
- optional GC hooks
- build modes

## Completion Criteria

A normal Hydrogen userspace app can start, allocate memory, print text, and exit.

```hy
use Hydrogen.Runtime;

public function Main(): int {
    Console.WriteLine("Hello from Hydrogen Runtime");
    return 0;
}
```

---

# Phase 5 — Australis.Kernel

## Purpose

Create the APIs required to write Australis OS kernel components in Hydrogen.

This layer should be available only in kernel/freestanding targets.

## Modules

```text
Australis.Kernel.Boot
Australis.Kernel.CPU
Australis.Kernel.Interrupts
Australis.Kernel.Memory
Australis.Kernel.Paging
Australis.Kernel.Ports
Australis.Kernel.Devices
Australis.Kernel.Debug
Australis.Kernel.Scheduler
Australis.Kernel.Syscalls
Australis.Kernel.Panic
```

## Features

### Boot

```hy
@entry
function KernelStart(info: ptr<BootInfo>): void {
    Kernel.Main(info);
}
```

### CPU

```hy
CPU.Halt()
CPU.DisableInterrupts()
CPU.EnableInterrupts()
CPU.ReadCr3()
CPU.WriteCr3(value)
```

Some of these require `unsafe`.

### Ports

For x86/x64 port IO:

```hy
Ports.In8(port)
Ports.Out8(port, value)
Ports.In16(port)
Ports.Out16(port, value)
```

### Interrupts

```hy
@interrupt(14)
function PageFault(frame: ptr<InterruptFrame>): void {
    Panic.Fatal("Page fault");
}
```

APIs:

```hy
Interrupts.Register(vector, handler)
Interrupts.Enable()
Interrupts.Disable()
```

### Kernel Memory

```hy
PhysicalMemory.AllocatePage()
PhysicalMemory.FreePage(page)
VirtualMemory.Map(...)
VirtualMemory.Unmap(...)
```

### Paging

```hy
Paging.MapPage(virtualAddress, physicalAddress, flags)
Paging.UnmapPage(virtualAddress)
Paging.Flush(address)
```

### Debug Output

```hy
KernelDebug.WriteLine("Booting Australis...");
```

This might write to serial, framebuffer, or debug console.

### Panic

```hy
Kernel.Panic("Fatal kernel error");
```

Should halt safely.

## Required Compiler Support

- freestanding mode
- no libc
- no mandatory heap
- inline assembly or compiler intrinsics
- interrupt attributes
- section attributes
- packed structs
- fixed memory layout

## Completion Criteria

You can write a minimal kernel entry in Hydrogen:

```hy
use Australis.Kernel;

@entry
function KernelStart(info: ptr<BootInfo>): void {
    KernelDebug.WriteLine("Australis booted from Hydrogen");
    CPU.HaltLoop();
}
```

---

# Phase 6 — Hydrogen.IO

## Purpose

Create platform-backed input/output.

This layer is not always available. It depends on the current target.

For userspace apps, it talks to Australis OS services.
For hosted builds, it may talk to the host platform.
For kernel builds, it should generally be unavailable or replaced by kernel debug output.

## Modules

```text
Hydrogen.IO.File
Hydrogen.IO.Directory
Hydrogen.IO.Path
Hydrogen.IO.Stream
Hydrogen.IO.Console
Hydrogen.IO.Device
```

## Features

### File

```hy
File.Open(path)
File.ReadAllText(path)
File.ReadAllBytes(path)
File.WriteAllText(path, text)
File.Exists(path)
File.Delete(path)
```

Use `Result`:

```hy
var text = try File.ReadAllText("config.txt");
```

### Directory

```hy
Directory.Exists(path)
Directory.Create(path)
Directory.List(path)
```

### Path

```hy
Path.Join(a, b)
Path.Extension(path)
Path.FileName(path)
Path.DirectoryName(path)
```

### Stream

```hy
interface Stream {
    function Read(buffer: slice<byte>): Result<usize, IOError>;
    function Write(buffer: slice<byte>): Result<usize, IOError>;
    function Close(): void;
}
```

### Console

```hy
Console.WriteLine("Hello");
Console.ReadLine()
```

## Required Compiler Support

- Result
- strings
- slices
- runtime or platform syscall support
- interfaces or traits

## Completion Criteria

A userspace Hydrogen app can read/write files and print output.

```hy
use Hydrogen.IO;

function Main(): int {
    var text = try File.ReadAllText("hello.txt");
    Console.WriteLine(text);
    return 0;
}
```

---

# Phase 7 — Australis.Userspace

## Purpose

Create the standard APIs for Australis OS applications and services.

This layer should wrap Australis system calls in safe Hydrogen APIs.

## Modules

```text
Australis.Userspace.Process
Australis.Userspace.Thread
Australis.Userspace.Syscall
Australis.Userspace.Permissions
Australis.Userspace.Environment
Australis.Userspace.Time
Australis.Userspace.Services
Australis.Userspace.App
```

## Features

### Process

```hy
Process.CurrentId()
Process.Exit(code)
Process.Spawn(path, args)
```

### Thread

```hy
Thread.Sleep(ms)
Thread.Start(function)
Thread.CurrentId()
```

### Syscall

Low-level syscall wrapper:

```hy
Syscall.Invoke(number, arg1, arg2, arg3)
```

Most users should not need this directly.

### Permissions

```hy
Permissions.Has("filesystem.read")
Permissions.Request("network")
```

### Environment

```hy
Environment.Get("HOME")
Environment.Args()
Environment.CurrentDirectory()
```

### Time

```hy
Time.Now()
Time.UtcNow()
Time.Sleep(ms)
```

### App Metadata

```hy
App.Name()
App.PackageId()
App.Version()
```

## Required Compiler Support

- target-aware builds
- syscall attributes or wrappers
- project/app metadata
- Result
- optional async later

## Completion Criteria

A Hydrogen Australis app can interact with the OS without manually calling syscalls.

```hy
use Australis.Userspace;

function Main(): int {
    Console.WriteLine(App.Name());
    Thread.Sleep(1000);
    return 0;
}
```

---

# Phase 8 — Australis.UI

## Purpose

Create the standard UI layer for Hydrogen applications on Australis OS.

This should come after userspace APIs are stable.

## Modules

```text
Australis.UI.App
Australis.UI.Window
Australis.UI.View
Australis.UI.Controls
Australis.UI.Layout
Australis.UI.Input
Australis.UI.Graphics
Australis.UI.Theme
Australis.UI.Events
```

## Features

### Application

```hy
var app = UIApp.Create();
app.Run();
```

### Window

```hy
var window = Window.Create("My App", 800, 600);
window.Show();
```

### Controls

Start small:

```text
Button
Label
TextBox
Panel
Image
ListView
```

Example:

```hy
var button = Button.Create("Click me");
button.OnClick += function() {
    Console.WriteLine("Clicked");
};
```

### Layout

```text
StackPanel
Grid
DockPanel
Canvas
```

### Input

```hy
Mouse.Position()
Keyboard.IsKeyDown(Key.Enter)
```

### Graphics

```hy
Canvas.DrawRect(...)
Canvas.DrawText(...)
Canvas.DrawImage(...)
```

### Theme

```hy
Theme.Current()
Theme.SetAccentColor(...)
```

## Required Compiler Support

- delegates/events or function references
- objects or structs with methods
- userspace runtime
- strings
- memory management
- app manifest support

## Completion Criteria

You can build a simple graphical Australis app in Hydrogen:

```hy
use Australis.UI;

function Main(): int {
    var app = UIApp.Create();

    var window = Window.Create("Hydrogen App", 800, 600);
    var button = Button.Create("Click me");

    button.OnClick += function() {
        Console.WriteLine("Hello from UI");
    };

    window.Content = button;
    window.Show();

    return app.Run();
}
```

---

# Phase 9 — Full Standard Library Hardening

## Purpose

Make the standard library reliable, documented, tested, and pleasant to use.

## Work Items

### Documentation

Every public type/function should include:

- description
- example
- target availability
- safety notes
- allocation behaviour
- possible errors

Example:

```text
File.ReadAllText(path)

Available:
- Australis userspace
- hosted native mode

Unavailable:
- freestanding
- kernel mode

Allocates:
- yes

Errors:
- FileNotFound
- PermissionDenied
- InvalidUtf8
```

### Tests

Create tests for:

- Core
- Memory
- Text
- Runtime
- IO
- Kernel APIs where testable
- Userspace APIs
- UI basics

### Build Matrix

Test standard library under:

```text
freestanding-debug
freestanding-release
kernel-debug
kernel-release
userspace-debug
userspace-release
hosted-debug
hosted-release
```

### Examples

Create example projects:

```text
examples/hello-world
examples/file-reader
examples/memory-buffer
examples/kernel-hello
examples/userspace-app
examples/ui-counter
```

### Diagnostics

Improve error messages when unavailable APIs are used:

```hy
use Hydrogen.IO;
```

In freestanding mode:

```text
error HY2004: Hydrogen.IO is not available in freestanding builds

help: use Australis.Kernel.Debug for kernel debug output
```

## Completion Criteria

The standard library feels usable by someone other than you.

---

# Phase 10 — Complete v1 Standard Library

## Purpose

Declare a stable first version.

This does not mean the library is finished forever. It means users can trust the core APIs.

## v1 Requirements

### Hydrogen.Core

- Result
- Option
- Math
- Panic
- conversions
- bit helpers
- primitive helpers

### Hydrogen.Memory

- pointers
- slices
- buffers
- allocator interface
- memory copy/move/set/zero
- layout helpers

### Hydrogen.Text

- UTF-8 validation
- byte/rune APIs
- formatting
- parsing
- basic StringBuilder

### Hydrogen.Runtime

- startup
- heap
- panic handling
- console
- debug support
- optional GC hooks, even if GC is not complete

### Hydrogen.IO

- files
- directories
- paths
- streams
- console

### Australis.Kernel

- boot
- CPU
- interrupts
- ports
- paging
- physical memory
- panic/debug

### Australis.Userspace

- process
- thread
- syscalls
- environment
- time
- permissions
- app metadata

### Australis.UI

- app
- window
- controls
- events
- basic layout
- input
- drawing basics

## v1 Completion Criteria

Hydrogen is v1-standard-library-ready when you can build:

1. A freestanding kernel component
2. A userspace command-line tool
3. A file-reading utility
4. A graphical Australis app
5. A small library/package used by another Hydrogen project

All without relying on libc, .NET, CLR, or another language runtime.

---

# Suggested Build Order

## First Working Target

Build this first:

```text
Hydrogen.Core
Hydrogen.Memory
Hydrogen.Text minimal
Australis.Kernel.Debug
```

This lets you write kernel-level proof-of-concept code.

## Second Working Target

Then build:

```text
Hydrogen.Runtime minimal
Hydrogen.IO.Console
Australis.Userspace.Process
Australis.Userspace.Syscall
```

This lets you write userspace command-line apps.

## Third Working Target

Then build:

```text
Hydrogen.IO.File
Hydrogen.Text formatting/parsing
Hydrogen.Memory.Buffer
```

This lets you write real tools.

## Fourth Working Target

Then build:

```text
Australis.UI
Australis.Userspace.App
Australis.Userspace.Permissions
```

This lets you write graphical Australis applications.

---

# Final Direction

The Hydrogen standard library should prove that Hydrogen is not a C wrapper, not a C# clone, and not dependent on another runtime.

It should show that Hydrogen is:

> A standalone native language with its own compiler, own standard library, own runtime model, and first-class Australis OS integration.

The standard library should be built from the bottom up:

```text
Core -> Memory -> Text -> Runtime -> Kernel -> IO -> Userspace -> UI
```

Each phase should produce something usable, even before the whole roadmap is finished.

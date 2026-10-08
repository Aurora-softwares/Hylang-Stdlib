# Hydrogen standard library

<img src="assets/hydrogen.icon.svg" style="display: block;margin-left: auto; margin-right: auto; width: 30%;" />

This repository contains the standard-library design plan. It does not yet contain installable `.hyproj` library implementations. Existing `System.*` APIs are built into the [Hydrogen compiler runtimes](https://github.com/Aurora-Softwares/Hylang-Compiler).

For available APIs and exact compiler-route support, use the [builtin reference](https://aurora-softwares.github.io/Hylang-Docs/runtime/builtins) and [current release guide](https://aurora-softwares.github.io/Hylang-Docs/getting-started/current-release/).

## Use the existing builtins

No separate package is required for console output, file text/byte I/O, and integer parsing:

```hylang
public class Program {
    public static int Main() {
        System.IO.File.WriteAllText("message.txt", "Hello, Hydrogen!\n");
        System.Console.Write(System.IO.File.ReadAllText("message.txt"));
        System.Console.WriteLine(System.Convert.ToInt32("42"));
        return 0;
    }
}
```

This complete program creates/replaces `message.txt` in the process working directory. Save it as `example.hy` and compile it with the released Linux x86-64 compiler:

```bash
hy compile example.hy -o example
chmod +x example
./example
```

Native names should be fully qualified. Strings are byte-based, file reads require seekable files in the native runtime, and failed native I/O terminates with an error.

## API families

| Family | Available implementation |
| --- | --- |
| `System.Console.Write`/`WriteLine` | Native and SDK |
| `System.IO.File` text/bytes/existence | Native and SDK |
| `System.Convert.ToInt32` | Native and SDK; returns `int` (32-bit in the current native release) |
| `System.Runtime.GC.Collect`/`GetAllocatedBytes` | Native source APIs |
| `System.Collections.List<T>` | SDK builtin only |
| `System.Runtime.Buffer`, `Memory`, `BinaryPrimitives` | SDK builtins only |
| `System.Testing.Assert` | SDK builtin only |

The SDK supplies broader generic/unsafe/runtime features than the self-hosted native compiler. Builtin recognition during native checking does not guarantee emission support.

## Library design

The [standard-library design plan](hydrogen-standard-library-roadmap.md) proposes reusable layers for core types, memory, runtime, text, I/O, and Australis-specific services. Proposed `Hydrogen.*`/`Australis.*` package names and example APIs in that document are future designs, not imports currently supplied by this repository.

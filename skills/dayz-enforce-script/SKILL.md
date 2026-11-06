---
name: dayz-enforce-script
description: Enforce Script (DayZ .c) rules for DayZ 1.30. Use before writing or editing any DayZ .c script, when one fails to compile, or when a script error in the log needs explaining.
---

# Enforce Script for DayZ

Enforce looks like C# or C++, but its parser is much smaller and code that looks valid often isn't. Write only what this page
or the vanilla scripts already do.

`patterns.md`, next to this file: open it before writing RPC, net sync, persistence, player connect hooks, timers or a JSON
config. Vanilla's shape for each, with the vanilla file it comes from.

Vanilla code is the best reference for signatures and call sites: grep the game's scripts (`P:\scripts` after extracting the
game data with DayZ Tools, or `scripts.pbo` unpacked). An `override` must copy the vanilla signature exactly.

## The loop

Keep it tight: write, pack, boot, read the log, fix. Done when the script log has no `SCRIPT (E)` lines.

- **Pack** the mod folder into a PBO (Addon Builder or any packer) under `@MyMod/addons/`.
- **Server code:** start `DayZServer_x64.exe` with `-mod=@MyMod` (and `-profiles=<folder>`), then read the newest
  `script_*.log` in the profile folder. A compile error stops the server before the mission starts; the log names the file
  and line.
- **Client-only code** (UI, input, the camera): start the client offline on a mission (`DayZ_x64.exe -mission=<mission folder>
  -mod=@MyMod`) and read its script log (`%LocalAppData%\DayZ\script_*.log`).
- Fix the first error first: one broken line often causes several messages after it.

## The rules

Each rule has a name. Most broken code breaks one of these.

**Line rule: a newline ends an expression.** Continue a line only right after `(`, `[`, or a `,` in a call, `new` or `{}` list.
```c
bool ok = Math.Max(a,
    b) > 1;                      // OK
bool b = x > 1 &&                // FAIL: the line ends in an operator
    x < 3;
string t = s
    .Substring(0, 1);            // FAIL: the line starts with '.', an operator or ','
return
    1;                           // FAIL: return takes its value from its own line
```

**One-name rule: one name per local per function.** A second `for (int i …)`, the same name in sibling `if`/`else` blocks,
shadowing in a nested block, a reused `foreach` variable, or a local named like a parameter or a global all give `Multiple
declaration of variable`. Declare `int i;` once and reuse it.

**Ref rule: weak references die.** `new` into a member without `ref` is a compile error. `array<X>` (no `ref`) drops a `new`
object at once: use `array<ref X>` and `ref X m_X;`. A factory's result (`m_Panel = Panel.Make(...)`) stored in a plain member
compiles and is freed at once too: the next line reads null.

**Swallow rule: a braceless body that is `return`, `break`, `continue` or a nested `if` also takes the next statement on its
line.** `if (n <= 1) return 1; return 2;` puts both returns in the `if`; `if (c) return; int y = 1;` skips the `= 1`. Put the
body on its own line or in braces.

**Cast rule: `(name)` followed by a non-operator is a cast.** `G((x))`, `if ((b))`, `int y = (x);`, `-(x)` and `Sleep(2000);`
fail with `Broken casting` when more code follows on the same line (`int y = (x);` ending its line compiles). Drop the extra
parentheses; write `Sleep(2000 + 0)`.

**Return rule: every non-void function ends with a top-level `return`.** A `switch` whose cases all return, or
`while (true) { return x; }`, still gives `No return statement`.

**Enum rule: values must be plain numbers.** `C = A | B`, `D = 1 << 3`, `E = 2 + 3` and `F = Other.X` silently delete that
member, and later uses fail with `Can't find variable`. Write `C = 3`, `D = 8`.

**Default rule: default arguments must be literals.** `void F(int a = 1 + 1)` or `= Math.Max(1, 2)` compiles and passes **0**.

**Target rule: the target type decides division.** `float a = 7 / 2;` is 3.5, `int a = 7 / 2;` is 3. A float target makes
`%` float modulo, which doesn't exist: `float x = 1 + (i % 10);` is `Unknown operator '%'`; put `i % 10` in an `int` local
first. **`1 / 0` with constants crashes the whole server** while compiling.

**Bool rule: `&&`/`||` used as a non-`bool` value crashes the server** while compiling, even in a function never called:
`"x" + (a && b)`, `Print(a && b)`, `string.Format("%1", a || b)`. Put it in a `bool` local first:
`bool ok = a && b; Print("x" + ok);`. Comparisons alone (`"x" + (a == 1)`) are fine.

**Component rule: `v[i] = …` can't use `v` whole on the right.** There `v` reads as a float: `spot[1] = F(spot)`,
`spot[1] = spot.Length()` and `spot[1] = vector.Dot(spot, up)` fail (`Cannot convert 'float' to 'vector'`), and every later
use of `spot` in the function fails too. Components are fine (`spot[1] = SurfaceY(spot[0], spot[2])`). Go through a local:
`float y = F(spot); spot[1] = y;`.

**Constructor rule: one constructor per class.** A second one fails at the call site. The base constructor gets the same
arguments as the subclass's, so their parameter lists must match.

**Constructor-call rule: a bare `X(args)` calls class X's constructor**, even inside a class with its own method X. When the
constructor takes those arguments, nothing is logged and the method never runs. Name methods unlike any class (`Advance`, not
`Step` or `Timer`), or call `this.X()`.

**Depth rule: recursion stops at 127 frames, and big frames crash the server.** Past the cap it's a `Stack overflow` script
error; with many locals per frame the server dies earlier, and a function with 257 locals crashes it while compiling.

**Null rule: a null object doesn't stop the script.** A method call on null is skipped (it returns whatever that method last
returned); a field read on null ends the function. Check `Cast` results before using them.

## Not in the language

| Wrong | Use |
|---|---|
| `a ? b : c` | `if`/`else` |
| `do { } while (c);`, `for (;;)` | `while (true) { …; if (!c) break; }` |
| `a = b = c`, assignment inside `()`, conditions or arguments | separate statements |
| `a %= b`, float `%`, `vec / 2`, `vec *= 2` | `a = a % b`, `Math.ModFloat(x, y)`, `vec * 0.5`, `vec = vec * 2` |
| `a[0]++`, `a[0] += 1` | `a[0] = a[0] + 1` |
| `new X().M()` | `(new X()).M()` |
| `X::M()`, `o->x` | `X.M()`, `o.x` |
| `'c'`, `"a" "b"`, multi-line strings, `\` continuation | `"c"`, `"a" + "b"` |
| `0X1F`, `2E3`, `.5`, `5.`, `1.5f`, `0b101` | `0x1f`, `2e3`, `0.5`, `5.0`, `1.5`, `5` (`010` is ten) |
| `+x` | `x` |
| `string s = 5;`, `int i = "5";`, `5 + "a"` | `"" + 5`, `"5".ToInt()`, `"" + 5 + "a"` |
| `null` for `string`/`int` | `""`, `0` |
| `var x`, `auto` member or global | a type; `auto` only for initialised locals |
| `#define N 5`, `#if`, `#undef` | `const int N = 5;`, `#ifdef` |
| `public`, `abstract`, `interface`, `virtual`, `final`, nested classes, generic functions, `new T()`, constructor overloads | — |
| `goto` and labels, `foreach` over a string or with an untyped or existing variable | — |
| `foreach (T v : F())` over a call's result | `array<T> a = F(); foreach (T v : a)` |

## Shape of the language

```c
modded class PlayerBase                      // 'modded'/'sealed' before 'class'
{
    protected ref array<ref Foo> m_List = new array<ref Foo>();   // member initialisers may call/new
    static const int MAX = 5;

    void PlayerBase() {}                     // constructor = class name, one per class
    void ~PlayerBase() {}                    // destructor, no parameters
    override void EOnInit(IEntity other, int extra)  // signature must match the base exactly
    {
        super.EOnInit(other, extra);
    }
}

class Box<Class T> : Managed { ref T m_V; }  // ':' or 'extends'
enum EMode : EBase { A = 1, B, C = 0x10 }    // literal values only; newline or ',' between members
typedef array<string> TStringArray;          // file scope only

void G(int a, out int b, inout array<int> c, notnull Foo d, int e = 5) {}   // max 16 params, literal defaults last
foreach (int i, string v : arr) {}           // typed variables declared here
switch (s) { case "a": Foo(); break; default: break; }   // case may be any expression; no fallthrough with code
float f = (float)i;  Foo f2 = Foo.Cast(obj);  Class.CastTo(f3, obj);
```

- Always write `override`: without it the method still overrides but warns. A wrong signature with `override` is
  `no function with this name in the base class`.
- `private` hides from subclasses; `protected` doesn't. `this` and `super` don't exist in static methods.
- `"text" + int/float/bool/vector/object` works; the left operand must be the string.
- Always end statements with `;`. Some lines compile without one, but a statement ending in `new X()` doesn't.
- Attributes are calls: `[NonSerialized()]`, `[Obsolete("msg")]`.
- Every loaded mod's name is a define: `#ifdef SomeMod`.
- Script modules see only earlier ones: `3_Game` can't use a type from `4_World`.

## Errors → cause

The common ones. For any other message, find a vanilla script that does the same thing and match its shape.

| Logged | Cause |
|---|---|
| `Syntax error`, `Broken expression (missing ';'?)` | the line rule, `?:`, an unsupported literal or keyword, `new X().M()` |
| `Unknown type '&&'`, `Error in parameters` | a line that starts with an operator (the line rule); an unknown type |
| `Expected ')', not a '…'` / `Expected ',' or ')', not a '…'` | the line rule: a break inside a condition or before `,` |
| `Broken casting` | the cast rule |
| `Multiple declaration of variable 'x'` | the one-name rule |
| `Cannot convert 'float' to 'vector'` on `v[i] = F(v)`, then errors on every later use of `v` | the component rule |
| `No return statement in function returning non-void` | the swallow rule or the return rule |
| `Can't find variable 'X'` | a typo; a member dropped by the enum rule; a bare enum member (use `E.X`); a type from a later module |
| `Incompatible parameter 'x'` | no implicit conversion (int↔string, null for primitives) |
| `Operator '++' is not supported on temporary values` | `a[0]++`, `Get()++` |
| `Variable 'o' is not strong ref (missing 'ref'?)` | the ref rule |
| `Too many parameters for 'X' method` | the constructor rule, or extra arguments |
| `Function 'M' is marked as override, but there is no function…` | the override's signature differs from the base |
| `Assign operator '=' not allowed here`, `Unknown operator '='` | assignment used as a value, or chained |
| `Opened scope at the end of file, missing '}' ?` | unbalanced braces, often after one of the above |
| The server stops while compiling, with no error | the bool rule, a constant `1 / 0`, or a function too long or with too many locals |

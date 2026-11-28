# Frinet for Windows (fork)

> **Original project:** [synacktiv/frinet](https://github.com/synacktiv/frinet) by Synacktiv
> ([blog post](https://www.synacktiv.com/publications/frinet-reverse-engineering-made-easier),
> [SSTIC 2026 talk, in French](https://www.sstic.org/2026/presentation/spatial_frinet/)).
> The original README describes building and using Frinet on **Linux**.
> This fork adds features and documents how to install and use Frinet on **Windows**.

<p align="center">
<img alt="Frinet" src="screenshots/frinet.png"/>
</p>

## Contents

1. [What is Frinet?](#1-what-is-frinet)
2. [What this fork adds](#2-what-this-fork-adds)
3. [Installation on Windows](#3-installation-on-windows)
4. [Your first trace, step by step](#4-your-first-trace-step-by-step)
5. [How big should a trace be? Exceptions and how to avoid problems](#5-how-big-should-a-trace-be-exceptions-and-how-to-avoid-problems)
6. [How the windows work together in IDA](#6-how-the-windows-work-together-in-ida)
7. [Troubleshooting](#7-troubleshooting)
8. [Reference: tracer options](#8-reference-tracer-options)
9. [Credits and license](#9-credits-and-license)

---

## 1. What is Frinet?

Frinet records **everything a function does while it runs**: every instruction it executes,
every register that changes and every byte of memory it reads or writes. You can then open
that recording in IDA Pro and move through it like a video: forwards, backwards, or straight
to any moment, and see the state of the program at that moment.

Frinet has four parts:

| Part | What it is | What it produces |
|---|---|---|
| **Tracer** (`tracer/trace.py`) | A Python script that uses [Frida](https://frida.re) to hook a function in a running program and record it | a `.tenet` trace file (text), plus a `.calls.txt` call list (this fork) |
| **Indexer** (`frinet.exe`) | A Rust command-line program | a `.db` file: the trace in a fast, searchable form |
| **Python module** (`frinet_db.pyd`) | A Rust library that IDA's Python can load | lets the plugin read the `.db` file |
| **IDA plugin** (`frontend/`) | Python code inside IDA | the Timeline, Registers, Memory and Call tree windows |

The full chain is:

```
 your program ──(Frida)──> trace.py ──> trace.tenet ──> frinet.exe index ──> trace.db ──> IDA + plugin
                                    └─> trace.calls.txt ───────────────────────────────────> Call tree window
```

## 2. What this fork adds

**Tracer**
- **Call list** (`.calls.txt`): every trace also gets a list of all calls, API calls and returns,
  with the line number in the trace where each one happens. It can be recorded while tracing
  (no copy of the program needed) or built afterwards from a copy of the program (`-X`).
- **`child` mode**: trace a program that can only be started by another program
  (launchers, IDE runners, services).
- **`-C/--cwd`**: choose the working directory of the started program.
- Windows fixes: correct line endings in trace files, no crash when the program has already exited.

**Indexer**
- Windows fix: indexing no longer fails at the end with `os error 1224`.

**IDA plugin**
- **Call tree window**: the call list as a tree. Click to jump to functions and call sites.
  It follows the timeline (Sync). Shows **real names** (IDA names: PDB symbols, your renames)
  or **addresses** (`sub_14000E424`).
- **Graph window**: a **flame chart** (which function runs when, over the whole trace) and a
  **call graph** (who calls whom, how often), both following the timeline.
- **Story window**: a readable **narrative** of the trace ("opens file…, writes 16 bytes to
  handle 0x500…") where **every time and address is a link** you can click.
- **Memory window follows the trace**: it jumps by itself to the bytes the current instruction reads or writes.
- **Registers → Memory**: double-click a register value to show that address in the Memory window.
- **Works with programs at their normal address**: no need to rebase the IDA database to 0
  (needed for Windows `.exe`/`.dll` files, which load at addresses like `0x140000000`).
- Fix for the `libshiboken: Internal C++ object (...) already deleted` messages.

---

## 3. Installation on Windows

Follow the steps in order. Each step ends with a check: don't move on until it works.

### 3.1 What you need

| Software | Version | Why |
|---|---|---|
| Windows | 10 or 11, **64-bit** | |
| [IDA Pro](https://hex-rays.com/ida-pro) | **9.2 or newer** (the plugin is written for 9.3) | The plugin uses PySide6 (Qt 6), which IDA ships from 9.2 on. IDA 9.0/9.1 and older use PyQt5 and **will not work**. |
| [Python](https://www.python.org/downloads/windows/) | 3.10 or newer, **64-bit, from python.org** | Runs the tracer. It is also needed to *build* the Python module. **Don't use the Microsoft Store version**: it lacks the `python3.lib` file the build needs. |
| [Git](https://git-scm.com/download/win) | any | To download the source code |
| [Visual Studio Build Tools 2022](https://visualstudio.microsoft.com/visual-cpp-build-tools/) | with **"Desktop development with C++"** | Rust on Windows needs Microsoft's C++ linker (`link.exe`) and the Windows SDK |
| [Rust](https://rustup.rs) (`rustup`) | **1.85 or newer** (the code uses Rust edition 2024) | Builds the indexer and the Python module |

### 3.2 Install the tools

1. **Python**: run the python.org installer. On the first screen, tick **"Add python.exe to PATH"**.
   Keep the default options (they include the `libs` folder with `python3.lib`).
2. **Visual Studio Build Tools 2022**: in the installer, tick **"Desktop development with C++"**.
   On the right, make sure **"MSVC v143 – VS 2022 C++ x64/x86 build tools"** and a
   **"Windows 11 SDK"** (or Windows 10 SDK) are ticked. Install, then restart Windows.
3. **Rust**: run `rustup-init.exe` and accept the default (`x86_64-pc-windows-msvc`).
4. **Git**: run the installer with the default options.

**Check:** open a **new** Command Prompt (`cmd`) and run:

```bat
python --version
python -c "import struct; print(struct.calcsize('P')*8, 'bit')"
rustc --version
cargo --version
git --version
```

You should see Python 3.10+ and `64 bit`, Rust 1.85+, and a git version.
If `rustc` is older, run `rustup update`.

### 3.3 Make IDA use a Python it can work with

IDA uses a Python installation on your computer. Open IDA, then **File → Script command...**
(or the Python bar at the bottom of the Output window) and type:

```python
import sys; print(sys.version)
```

If no Python is found, or you want IDA to use the Python you just installed, run IDA's
**`idapyswitch.exe`** (in the IDA installation folder) and pick it.

IDA must use **Python 3.10 or newer, 64-bit** (the plugin code uses Python 3.10 syntax).
The Python module itself is built for Python's "stable ABI", so **one build works with any
Python 3.10+**. IDA's Python doesn't have to be the exact same version as the one used to build it.

### 3.4 Download and build

Pick a folder **without spaces or non-English letters** in the path, for example `C:\tools`.

```bat
cd C:\tools
git clone https://github.com/sigmantu1-design/frinet.git
cd frinet
cargo build --release -p frinet-python-bindings -p frinet-cli
```

The first build downloads the dependencies and takes a few minutes. Build **only these two
packages** (`-p ...`): a plain `cargo build --release` also builds the fuzzing tools, which need
extra setup and often fail on Windows.

**Check:** these two files must exist:

```
C:\tools\frinet\target\release\frinet.exe        <- the indexer
C:\tools\frinet\target\release\frinet_db.dll     <- the Python module (needs renaming, see below)
```

### 3.5 Install the IDA plugin

1. Open the folder `%APPDATA%\Hex-Rays\IDA Pro\plugins`. Paste that into the Explorer address
   bar; it is usually `C:\Users\<you>\AppData\Roaming\Hex-Rays\IDA Pro\plugins`.
   Create the `plugins` folder if it doesn't exist.
2. Create a folder named **`frinet`** inside it.
3. Copy **the contents** of `C:\tools\frinet\frontend` (all `.py` files and `ida-plugin.json`)
   into that `frinet` folder.
4. Copy `C:\tools\frinet\target\release\frinet_db.dll` into the same folder and
   **rename it to `frinet_db.pyd`**. Python only loads extension modules with the `.pyd` extension.

The result must look like this. The files sit **directly** in `frinet`, not in a subfolder:

```
%APPDATA%\Hex-Rays\IDA Pro\plugins\frinet\
    ida-plugin.json
    plugin.py
    core.py
    arch.py
    calls_model.py
    calltree_widget.py
    graph_widget.py
    story_gen.py
    story_widget.py
    memory_widget.py
    register_widget.py
    search_results_widget.py
    timeline_widget.py
    frinet_db.pyd
    frinet_db.pyi
```

> **Important:** IDA loads the plugin **only from this folder**, not from `C:\tools\frinet`.
> Whenever you update the source code, copy the files here again, then **restart IDA**.

**Check:** start IDA and open any 64-bit program. A **Frinet** menu appears in the menu bar
(before *Help*), and **File → Load file** shows **"Open Frinet DB..."** and **"Open Frinet call tree..."**. If they are missing, see
[Troubleshooting](#7-troubleshooting).

### 3.6 Install the tracer's Python packages

```bat
python -m pip install frida pefile capstone
```

`frida` is required. `pefile` and `capstone` are only needed for `-X` mode (building the call
list from a copy of the program).

**Check:**

```bat
python -c "import frida; print(frida.__version__)"
python C:\tools\frinet\tracer\trace.py --help
```

The second command must print the list of options.

### 3.7 Antivirus and administrator rights

- Frida works by **injecting code into another process**. Windows Defender or other antivirus
  software may block it or delete Frida's files. If tracing fails with access errors, or the
  target closes immediately, add an exclusion for your Python folder and your work folder.
- To trace a program that runs **as administrator** (or a service), run the Command Prompt
  **as administrator** too.

---

## 4. Your first trace, step by step

The example traces what Notepad does when you **save a file**. Use the same steps for your own targets.

### Step 1: Find the function to trace, in IDA

1. Copy the program into a work folder, so IDA can save its database next to it:
   `copy C:\Windows\System32\notepad.exe C:\work\`
   You still **run** the original. IDA only needs the **exact same build** of the file.
2. Open `C:\work\notepad.exe` in IDA and let the analysis finish.
3. Find the function you are interested in. Tips for finding it:
   - Look for the Windows API the feature must use (here `WriteFile` or `CreateFileW`) in the
     **Imports** window, then press **X** to see which functions call it.
   - Look for a text string the feature shows, then press **X** on it.
4. Note the function's address, for example `0x14000E5C8`.

### Step 2: Turn the address into an offset

The tracer needs the address **relative to the start of the program** (an *offset*), because
Windows loads the program at a different address every time (ASLR).

```
offset = function address in IDA - image base
       = 0x14000E5C8 - 0x140000000 = 0xE5C8
```

You can see the image base in IDA under **Edit → Segments → Rebase program...** (just read it
and press Cancel), or at the top of the disassembly listing.

### Step 3: Trace

Open a Command Prompt and run:

```bat
python C:\tools\frinet\tracer\trace.py -e spawn "C:/Windows/System32/notepad.exe" notepad.exe 0xE5C8
```

| Part | Meaning |
|---|---|
| `-e` | Follow only `notepad.exe`'s own code. Windows DLLs run at full speed and are not traced. **Recommended**, see section 5. Options go **before** `spawn`. |
| `spawn` | Start the program. `attach` connects to a program that is already running (give its name or PID). |
| `"C:/Windows/System32/notepad.exe"` | The program to start (use `/` or quote the path) |
| `notepad.exe` | The module (exe or dll) that contains the function |
| `0xE5C8` | The offset from step 2 |

Notepad opens. **Do the action that runs the function**, here *File → Save*.
In the Command Prompt you will see:

```
Entering function
Leaving function
INFO | Writing trace data of 57303 bytes...
INFO | Creating file ...\tracer\traces\notepad.exe_1795694481_20748.tenet
```

**`Leaving function` means the recording is complete.** By default the tracer records **one**
run of the function and then removes its hook (add `-m` to record every run).

Then:
1. Press **Ctrl+C** in the Command Prompt and wait for the script to finish.
2. Close the program **normally**.

> **Never kill the program (Task Manager) before `Leaving function` appears.** The tracer
> buffers about 1 MB in memory, and anything not yet written to the file is lost.

You now have two files in `C:\tools\frinet\tracer\traces\`:
- `notepad.exe_<time>_<thread>.tenet`: the trace
- `notepad.exe_<time>_<thread>.calls.txt`: the call list

### Step 4: Index the trace

```bat
C:\tools\frinet\target\release\frinet.exe index --format tenet C:\tools\frinet\tracer\traces\notepad.exe_1795694481_20748.tenet C:\work\save.db
```

A progress bar runs and `save.db` is created. Use a fast disk with enough free space:
the `.db` can be larger than the `.tenet`.

### Step 5: Open it in IDA

1. Open `C:\work\notepad.exe` in IDA (the same database as in step 1).
2. **File → Load file → Open Frinet DB...** and pick `C:\work\save.db`.
   The **Timeline** (a thin vertical bar at the far right of the IDA window), **Registers** and
   **Memory** windows open.
3. **File → Load file → Open Frinet call tree...** and pick the `.calls.txt` file.
   The **Call tree** window opens.
4. In the Call tree toolbar, click **Graph** and **Story** to open the other two views of the same
   call list. You can also open them from the **Frinet** menu.

> Open the **DB first**, then the call list: the Story reads API arguments from the DB.
> If you opened the Story before the DB, press **Regenerate** in it.

You can now explore. See section 6.

### Optional: build the call list from a copy of the program instead

If you pass `-X <copy of the exe>`, the call list is built after tracing from the trace and
the PE file (its import table and function table), instead of being recorded while tracing:

```bat
python C:\tools\frinet\tracer\trace.py -e -X C:\work\notepad.exe spawn "C:/Windows/System32/notepad.exe" notepad.exe 0xE5C8
```

You can also build it later for any existing trace:

```bat
python C:\tools\frinet\tracer\calltree.py C:\work\notepad.exe <trace.tenet> [out.calls.txt] [offset]
```

The file **must be the exact build that was traced**. A different version of the program
gives wrong function names.

---

## 5. How big should a trace be? Exceptions and how to avoid problems

### 5.1 What makes a trace big

Each executed instruction is one line in the `.tenet` file, about 30–150 bytes with its
registers and memory accesses. Real numbers from tracing Notepad:

| What was traced | Options | Result |
|---|---|---|
| The *Save* function only | `-e` | 1,577 instructions, **57 KB**, done instantly |
| Notepad's main function (`wWinMain`, the message loop), about 4 minutes of use | without `-e` | **1.9 GB**, and the trace broke at the first exception |

The difference comes from **which function you hook** and whether you **follow the Windows DLLs**:

- The trace starts when the hooked function is entered and stops **when it returns**.
  A function that runs for the whole life of the program (`main`/`WinMain`, a message loop,
  a thread procedure) only returns when the program exits. You record everything, and the
  trace may never finish properly.
- Without `-e`, the tracer also follows every call into Windows DLLs (user32, kernel32,
  COM, the shell...). A single *Save As* dialog runs millions of instructions inside them.

### 5.2 Recommendations

1. **Hook the smallest function that does what you want to study**, not the program's main
   function. In the Notepad example, the function that writes the file, not `wWinMain`.
2. **Use `-e`** (follow only the module you are studying). It is much faster and much smaller,
   and it avoids most exception problems (5.3).
   *Drawback:* memory written by the DLLs is not recorded. For example, the text buffer that
   `WideCharToMultiByte` fills shows as `??` in the Memory window. You still see the pointers
   in the registers, and everything the module's own code reads and writes.
3. **Add `-n`** (no memory tracking) if you only need the control flow. It is faster and smaller.
4. For a long function, record only part of it: **`-E <offset>`** stops at an address instead of
   at the function's return.
5. If the program crashes and you want the trace up to the crash, use **`-F <offset>`** to force
   the buffer to be written when that instruction runs.
6. As a rule of thumb, aim for **up to a few million instructions** (a few hundred MB). The
   indexer can handle up to about 2 billion instructions (~100 GB), but huge traces need a
   lot of disk space and time, and they are hard to explore.

### 5.3 Exceptions (`Got exception : flushing Stalker`)

While tracing you may see:

```
Got exception : flushing Stalker
```

**What it means:** the program raised a processor exception. Windows code does this all the
time on purpose (RPC and COM errors, C++ exceptions, the shell checking things) and handles
it itself. The tracer prints the message and writes its buffer, just in case.

- **If the message appears and the trace continues** (you still get `Leaving function`):
  harmless, ignore it.
- **If the trace stops after the message** (no new data, no `Leaving function`, even though
  the program keeps working): this is a known limitation of Frida's Stalker on 64-bit
  Windows. When an exception happens **inside code the tracer is following**, Windows can't
  unwind the stack through the tracer's rewritten code. The tracer loses the thread, or the
  program crashes. The trace up to that point is still usable.

**How to avoid it:**
- Use **`-e`**, so Windows DLLs, where most exceptions happen, run normally and are not followed.
- **Don't hook a function that opens dialogs** (file dialogs, message boxes) or heavy COM/shell
  code. Hook the function that runs **after** the dialog, which gets the result. In Notepad:
  not the *Save As* function, but the one that writes the file.
- Hook a function deeper in the call chain.

### 5.4 Ending a trace correctly

| Situation | What to do |
|---|---|
| `Leaving function` appeared | Done. Press **Ctrl+C**, then close the program normally. |
| Only `Entering function`, the function hasn't returned yet | Finish the action in the program (close the dialog, etc.) so it returns. Then follow the row above. |
| The hooked function only returns when the program exits (main function) | Close the program **normally** (File → Exit). Don't kill it. Next time, hook a smaller function. |
| The program exited, but the script keeps waiting | The script doesn't notice the exit. Press **Ctrl+C**. |

---

## 6. How the windows work together in IDA

### The Frinet menu

Everything Frinet can do is in the **Frinet** menu in IDA's menu bar:

| Item | What it does |
|---|---|
| Open Frinet DB... / Close trace | Open an indexed trace (`.db`) / close it and its windows |
| Open Frinet call tree... / call graph... / story... | Open a `.calls.txt` in one of the three views |
| Show Timeline / Registers / Memory | Bring back a window you closed |
| Search memory... | Search the whole trace's memory for text or a regular expression |
| Go to first / prev / next / last execution | For the address under the cursor in the disassembly (same as the right-click menu) |
| Help (guide)... | Open this guide |

Items that need an open trace are greyed out until you open one.

Everything in the plugin revolves around one thing: **the current time**. Time is the
position in the trace: instruction number 0, 1, 2... Some windows **change** the time, and all
windows **show the program state** at the current time.

```
             change the time                               show the state at that time
   ┌────────────────────────────────┐               ┌─────────────────────────────────────┐
   │ Timeline (drag / click)        │               │ Disassembly: colored trail          │
   │ Registers: time box, < > keys  │               │ Registers: values (red = changed)   │
   │ Disassembly: right-click →     │──> CURRENT ──>│ Memory: bytes; follows the accesses │
   │   Go to first/prev/next/last   │     TIME      │ Call tree: active call (Sync)       │
   │ Call tree: double-click        │               │ Timeline: position marker           │
   │ Graph: click / double-click    │               │ Graph: red cursor, active chain     │
   │ Story: click a link            │               │ Story: current paragraph            │
   │ Memory: Ctrl+wheel on a bp     │               └─────────────────────────────────────┘
   │ Search results: double-click   │
   └────────────────────────────────┘

Every window with a **Sync** checkbox follows the current time; untick it to keep the window still.
```

### Disassembly (IDA View)
- The **current instruction** is green. The few instructions executed **before** it are red,
  and the ones executed **after** it are blue. You see where execution came from and where it goes.
- **Right-click an instruction → Go to first / previous / next / last execution** jumps to the
  times this instruction ran. This is the fastest way to answer "when did this code run?"

### Timeline
- The thin vertical bar at the far right of the IDA window. Its full height is the whole trace.
- Click or drag on it to move in time.

### Registers
- Values at the current time. **Red** = changed by the current instruction, **gray** = not known yet.
- **`<` / `>`** next to a register: go to the previous/next time this register changes.
- Type a number in the **time box** at the top and press Enter to go to that time.
- **Double-click a value** (or right-click → *Show in Memory window*): shows that address in
  the Memory window. Useful for pointers: buffers, strings, structures.

### Memory
- Shows memory **as it was at the current time**. `??` means "the trace never saw this byte
  until now": the traced code didn't touch it, or it was written by an excluded DLL (`-e`).
- **Follow memory accesses** (on by default, toggle it with right-click): at each time, it
  jumps to and selects the bytes the current instruction reads or writes. Turn it off to keep
  watching one fixed address.
- **G** or right-click → **Go to address...**: show any address. The mouse wheel scrolls.
- **Select bytes → right-click → Break on reads / Break on writes**, then **Ctrl + mouse
  wheel** over them: jump to the previous/next time these bytes were read or written.
  This is how you answer "who wrote this value?"
- Right-click → **Search...**: search the whole trace's memory for text or a regular
  expression. Double-click a result to go to that time.

### Call tree (this fork)
- The calls made by the traced function, as a tree. **[API]** rows (magenta) are calls into
  untraced DLLs, for example `[API] KERNEL32!WriteFile`.
- **Double-click a function name**: jump to that function in IDA (for an API, its import entry).
- **Double-click "Called from"**: jump to the call instruction **and** move the time to that
  call. The registers then show the call's arguments (on x64 Windows: `rcx`, `rdx`, `r8`, `r9`).
- **Right-click → Go to return**: move the time to the moment that call returned
  (for a traced function, the return value is in `rax`; for an `[API]` call traced with `-e`,
  `rax` is not recorded, because the API itself runs untraced).
- **Names**: **Real names** shows the current IDA names: function names from PDB symbols, or
  the names **you give** functions in IDA (press **N** on a function). They update in every view as soon as you
  rename. **Addresses** shows `sub_<address>`, as in the call list file.
- **Graph** / **Story** buttons: open the other views of the same call list.
- **Sync** (on by default): when the time changes anywhere, the tree selects the function
  running at that moment, with the chain of active calls in **bold**.
- **Filter** box: show only branches containing a name (e.g. `WriteFile`). **APIs**: hide or show API calls.
- The **Line** / **Return line** columns are line numbers in the `.tenet` file.

### Graph (this fork)
Two tabs, both following the timeline:

- **Timeline (flame chart)**: time runs from left to right (the axis shows `.tenet` line
  numbers). Each call is a **bar** from the moment it is called to the moment it returns. Bars
  stacked below it are the calls it makes; thin magenta bars are API calls. The **red line** is
  the current time; the chain of calls active at that moment is **outlined in yellow**.
  - **Mouse wheel**: zoom in/out around the mouse. **Drag**: move left/right. **Fit**: show everything.
  - **Click**: move the trace time to that moment. **Double-click a bar**: go to that call (time + IDA).
  - Hover over a bar to see its name, lines and instruction count.
  - Use it to see the **shape** of the execution at a glance: long bars = where the time is spent,
    repeated patterns = loops.
- **Call graph**: one box per function, arrows = "calls", `×N` = how many times. The chain
  active at the current time is **yellow** with **red** arrows.
  - **Click a box**: jump to the **next** call of that function after the current time. Click
    again for the one after (it wraps around). This is how you step through the
    **transitions** between functions over time.
  - **Double-click a box**: go to the function in IDA. **Wheel**: zoom. **Drag**: move.
- **APIs** checkbox: hide API calls to see only the program's own functions.

### Story (this fork)
A generated account of the trace, written for people:

- **Summary**: how long the trace is, how many functions and API calls, the most used APIs.
- **Key events**: the important API calls grouped by topic (files, registry, processes, modules,
  network, cryptography, COM...), with their **decoded arguments**: file names, sizes, access
  rights, registry keys, window messages (`EM_GETHANDLE`, `SB_SETTEXTW`...), code pages...
- **Step by step**: what the traced function does, in order, with nested calls and what happens inside them.
- **Every `line N` and every function name is a link**: click it to move the trace time there
  (all windows follow) and show the code in IDA.
- **Sync**: the paragraph about the current time is highlighted as you move through the trace.
- **Save as...**: save the story as Markdown or HTML (the links are kept as text).
  **Open...**: open a story **written by hand** (by you, or by an AI assistant from the call
  list): any Markdown/HTML file whose links use `line:<n>`, `go:<line>:<address>`, `t:<time>` or
  `ea:<address>` becomes clickable in the same way.
- **Limits**: arguments come from registers `rcx`, `rdx`, `r8`, `r9` (the first four arguments
  on x64 Windows). A string shows as *"not in trace"* when its bytes were written by code that
  wasn't traced (for example by the Save dialog, with `-e`). API return values are not shown,
  because with `-e` they are not recorded.

### A typical workflow
1. Read the **Story** to get the big picture, and click the moment that interests you, or find
   the interesting API call in the **Call tree**, for example `WriteFile`.
2. Double-click its **Called from**. The time moves to the call, and IDA shows the call instruction.
3. In **Registers**, double-click `rdx` (the buffer argument). **Memory** shows the buffer.
4. Select the bytes, then **Break on writes** and **Ctrl+wheel up** to find the instruction that wrote them.
5. Use the colored trail and **Go to previous execution** to see how the program got there.

---

## 7. Troubleshooting

### Building

| Problem | Cause and fix |
|---|---|
| `error: linker 'link.exe' not found` | Visual Studio Build Tools with **"Desktop development with C++"** are missing (3.2). Install them, restart Windows, and use a **new** Command Prompt. |
| `LINK : fatal error LNK1181: cannot open input file 'python3.lib'` | The Python found during the build has no `libs` folder: usually the **Microsoft Store** Python. Install Python from **python.org**. If you have several Pythons, choose one: `set PYO3_PYTHON=C:\Path\To\python.exe`, then build again. |
| `feature 'edition2024' is required` / `edition 2024 is unstable` | Rust is too old. Run `rustup update`. |
| Errors in `libfuzzer-sys` / `frinet-fuzz` | You built the whole workspace. Build only the two needed packages: `cargo build --release -p frinet-python-bindings -p frinet-cli` |
| `frinet_db.dll` doesn't exist | Look in `target\release\`. If you built without `--release`, it's in `target\debug\`. |

### IDA plugin

| Problem | Cause and fix |
|---|---|
| No **Frinet** menu, and no "Open Frinet DB..." in File → Load file | Check that the files are **directly** in `%APPDATA%\Hex-Rays\IDA Pro\plugins\frinet\` and not in a subfolder (3.5). Restart IDA **completely**. Open **View → Output window** and look for errors mentioning `frinet`. |
| You changed the code but nothing changes | IDA uses its **own copy** in the plugins folder and reads it only at startup. Copy the files again and restart IDA. |
| `ModuleNotFoundError: No module named 'frinet_db'` | `frinet_db.pyd` is missing from the plugin folder, or it is still named `.dll`. |
| `ImportError: DLL load failed while importing frinet_db` | The module was built for a different architecture, or IDA's Python is 32-bit. Use a 64-bit Python 3.10+ in IDA (`idapyswitch`) and rebuild. |
| `TypeError: unsupported operand type(s) for \|` when IDA starts | IDA uses a Python older than 3.10. Switch it with `idapyswitch` (3.3). |
| `ModuleNotFoundError: No module named 'PySide6'` | IDA older than 9.2. The plugin needs IDA 9.2+. |
| Clicking in the Timeline or *Go to execution* jumps to a wrong address | The IDA database is not the program you traced (a different build or version), or an old plugin version without the image-base fix is installed. With an old version, rebasing the database to 0 is the workaround (**Edit → Segments → Rebase program**, value `0`). |
| `Open DB failed : ...` in the Output window | The `.db` file is damaged or incomplete: index the `.tenet` again. If the indexer itself failed with `os error 1224`, use this fork's indexer (Windows fix). |
| The Memory window shows only `??` | You are looking at an address the trace never touched (it starts at 0). Use **Follow memory accesses**, press **G**, or double-click a register. Bytes written by excluded DLLs (`-e`) are never known. |
| `libshiboken: Internal C++ object (...) already deleted` | Fixed in this fork. Install the current plugin files. |
| The Call tree can't jump to a function (`cannot resolve ...` in the Output window) | The IDA database is a different build than the traced program, or the API is not imported under that name. |
| **Real names** still show `sub_...` | IDA has no name for that function. Load the program's symbols (**File → Load file → PDB file...**; for Windows files IDA can download them from Microsoft), or rename the function yourself (**N**). |
| The Story shows `?` for API arguments | The Frinet DB was not open when the Story was created. Open the DB, then press **Regenerate**. |
| No **Graph** / **Story** buttons in the Call tree | Old plugin files: copy all `.py` files again (3.5), including `calls_model.py`, `graph_widget.py`, `story_gen.py`, `story_widget.py`, then restart IDA. |
| The Graph is slow to draw on a huge trace | Untick **APIs**, zoom into a part of the trace, or trace a smaller function (section 5). |

### Tracing

| Problem | Cause and fix |
|---|---|
| `Unable to find module. Module list: ...` | The module name is wrong, or the module (a DLL) isn't loaded yet when the hook is placed. Use the exact name from the printed list. For a DLL that loads later, start the program first, then use `attach`. |
| `Entering function` never appears | The function didn't run (do the action in the program), or the offset is wrong. Recalculate: offset = IDA address − image base (step 2). |
| `Got exception : flushing Stalker`, then the trace stops | See 5.3: use `-e` and hook a smaller function that doesn't open dialogs. |
| The trace file is huge | See 5.1–5.2: smaller function, `-e`, `-n`, `-E`. |
| The trace has no ending, or its last part is missing | The program was killed, or exited before the function returned. See 5.4. |
| `Failed to attach` / `access denied` | Run the Command Prompt **as administrator**, check the antivirus (3.7), and use a 64-bit Python for 64-bit targets. |
| The indexer fails with `ParseIntError(InvalidDigit)` | The trace has Windows (CRLF) line endings, for example after being edited in Notepad or written by the original tracer on Windows. This fork's tracer writes LF. Convert the file to LF, or trace again. |

---

## 8. Reference: tracer options

```
python tracer/trace.py [options] {spawn|attach|child} <process> <module> <offset>
```

Options go **before** `spawn`/`attach`/`child`.

| Option | Meaning |
|---|---|
| `spawn <path>` | Start the program and hook it before it runs |
| `attach <name or PID>` | Connect to a running program |
| `child <parent name or PID>` | Wait for the parent to start the target program, then hook it before it runs (`-c <exe name>` if it differs from `<module>`) |
| `-e, --exclude` | Follow only the traced module (recommended) |
| `-n, --nomem` | Don't record memory accesses |
| `-m, --multirun` | Record every run of the function, not just the first |
| `-E, --end <offset>` | Stop at this address instead of at the function's return (`-1` = never) |
| `-F, --flushaddr <offset>` | Write the buffer when this instruction runs (for crashes) |
| `-t, --traced-module <name>` | Record a different module than the hooked one; `*` = all modules |
| `-a, --args "a,b,c"` | Command-line arguments for `spawn` (including the program name) |
| `-C, --cwd <folder>` | Working directory for `spawn` (default: the program's folder) |
| `-X, --exe <file>` | Build the call list from this copy of the program after tracing |
| `--nocalls` | Don't record the call list while tracing |
| `-s, --slow` | Slower JavaScript tracer (for 32-bit and other architectures) |
| `-D` / `-U` / `-R` / `-H host` | Frida device: by ID / USB / remote / remote host |
| `-v` | Verbose output |

Indexer:

```
frinet.exe index --format tenet <trace.tenet> <out.db>
frinet.exe search <db> <regex>
```

Call list format (`.calls.txt`):

```
line=127          call sub_14000E424   @14000E8BC     <- call into traced code, from address 14000E8BC
line=185              api  KERNEL32!WriteFile   @14000E531   <- call into an untraced DLL
line=201          ret  -> 14000E8C1                 <- return (back to this address)
```

`line` is the line number in the `.tenet` file. Indentation shows the call depth.

---

## 9. Credits and license

- Frinet is developed by [Synacktiv](https://www.synacktiv.com): [synacktiv/frinet](https://github.com/synacktiv/frinet).
  It builds on [Tenet](https://github.com/gaasedelen/tenet) by Markus Gaasedelen and on [Frida](https://frida.re).
- This fork: Windows support, call list and call tree, memory follow and documentation.
  The changes are proposed upstream as a pull request.
- License: see [LICENSE](LICENSE) (same as the original project).

### Building on Linux and testing

See the [original README](https://github.com/synacktiv/frinet#readme). In short:

```bash
cargo build --release
cp target/release/libfrinet_db.so frontend/frinet_db.so
ln -s $PWD/frontend ~/.idapro/plugins/frinet
cargo test
```

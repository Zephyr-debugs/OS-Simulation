# OS Simulation: Multiprogramming Operating System (MOS)

![Language: C](https://img.shields.io/badge/language-C11-blue)
![Stages: 2](https://img.shields.io/badge/stages-I%20%2B%20II-success)
![Platform](https://img.shields.io/badge/platform-Windows%20%7C%20Linux%20%7C%20macOS-lightgrey)

A C simulation of a small teaching operating system. The program models a simple computer (a word-addressed memory, a few registers and a seven-instruction CPU) and an operating system layer on top of it. The OS reads jobs from a deck of card images, loads them into memory, runs them, handles the requests and errors they cause, and writes the results to an output file.

The project is built in two stages. Stage I is the basic machine and supervisor calls. Stage II adds paging, interrupts, a process control block and proper error handling.

## Contents

- [What each stage covers](#what-each-stage-covers)
- [How a job runs](#how-a-job-runs)
- [The machine](#the-machine)
- [Job card format](#job-card-format)
- [Quick start](#quick-start)
- [Examples](#examples)
- [Stage II in short](#stage-ii-in-short)
- [Sample jobs and testing](#sample-jobs-and-testing)
- [Design notes](#design-notes)
- [Limitations](#limitations)
- [Project structure](#project-structure)
- [OS concepts covered](#os-concepts-covered)

## What each stage covers

| | Stage I ([`phase1`](phase1)) | Stage II ([`phase2`](phase2)) |
| --- | --- | --- |
| Memory | 100 words, addressed directly | 300 words in 30 frames; jobs use 100 words of virtual memory |
| Addresses | Real addresses | Virtual addresses translated through a page table |
| Job information | Loose variables | Process control block (PCB) |
| Interrupts | `SI` only (supervisor call) | `SI`, `TI` (timer) and `PI` (program errors) |
| Errors | A message per problem | Error codes, valid and invalid page faults, and a report for every job |
| Limits | Time limit checked | Time limit and line limit both enforced |

Each folder has its own README with the full details of that stage.

## How a job runs

```
 input file (card deck)
        |
        v
     LOAD  -- reads $AMJ, sets up the job, stores the program cards
        |
        v
  $DTA card reached  -->  EXECUTEUSERPROGRAM
                             |
                             |   fetch -> decode -> execute -> update counters
                             |
                             |   GD / PD / H, a limit, or an error sets an interrupt
                             v
                            MOS()  -- READ, WRITE or TERMINATE
                             |
                             v
                        output file
```

Jobs are run one after another. When a job ends (normally or because of an error), the loader looks for the next `$AMJ` card.

## The machine

Every word is 4 characters. Instructions are one word: a 2-letter opcode followed by a 2-digit address, for example `GD20`.

| Instruction | Action |
| ----------- | ------ |
| `LR xx` | Load the word at address `xx` into register `R`. |
| `SR xx` | Store `R` into the word at address `xx`. |
| `CR xx` | Compare `R` with the word at `xx`. Toggle `C` is 1 if they are equal, 0 otherwise. |
| `BT xx` | If `C` is 1, jump to address `xx`. |
| `GD xx` | Get data: read the next data card into the 10 words starting at `xx`. |
| `PD xx` | Put data: write the 10 words starting at `xx` as one 40-character line. |
| `H` | Halt: end the job. |

`GD`, `PD` and `H` are the supervisor calls. They set the `SI` interrupt to 1, 2 or 3 and the operating system (`MOS`) carries out the request.

Registers: `IR` (instruction register), `IC` (instruction counter), `R` (general register) and `C` (toggle). Stage II adds `PTR` (page table register) and the interrupt flags `TI` and `PI`.

## Job card format

Both stages read the same kind of input file.

```
$AMJ<job id><time limit><line limit>      e.g. $AMJ010100100002
<program cards>                           up to 40 characters, 10 instructions per card
$DTA
<data cards>                              one card per GD
$END<job id>
```

The three fields of the `$AMJ` card are 4 digits each. `$AMJ010100100002` is job `0101` with a time limit of 10 and a line limit of 2. Several jobs can be placed one after another in the same file.

## Quick start

You need a C compiler (GCC or MinGW on Windows, GCC or Clang on Linux and macOS).

```bash
git clone https://github.com/Zephyr-debugs/OS-Simulation.git
cd OS-Simulation
```

**Windows (PowerShell)**

```powershell
# Stage I
cd phase1
gcc -std=c11 -Wall -Wextra -O2 main.c -o mos.exe
.\mos.exe input_Phase1.txt output.txt
Get-Content output.txt

# Stage II
cd ..\phase2
gcc -std=c11 -Wall -Wextra -O2 main.c -o mos.exe
.\mos.exe input_Phase2.txt output_Phase2.txt
Get-Content output_Phase2.txt
```

**Linux / macOS**

```bash
# Stage I
cd phase1
gcc -std=c11 -Wall -Wextra -O2 main.c -o mos
./mos input_Phase1.txt output.txt
cat output.txt

# Stage II
cd ../phase2
gcc -std=c11 -Wall -Wextra -O2 main.c -o mos
./mos input_Phase2.txt output_Phase2.txt
cat output_Phase2.txt
```

Both programs take `[input_file] [output_file]`. Stage II also accepts `-v`, which prints the page table of every job to the console.

## Examples

### Stage I

Input:

```
$AMJ010100040001
GD20PD20H
$DTA
HELLO WORLD
$END0101
```

Output (each line is 40 characters, followed by two blank lines when the job ends):

```
HELLO WORLD                             


```

`GD20` reads the data card into words 20 to 29, `PD20` writes them out, and `H` ends the job.

### Stage II

Input:

```
$AMJ010100100002
GD20PD20H
$DTA
HELLO PAGING WORLD
$END0101
```

Output:

```
HELLO PAGING WORLD                      

JOB ID : 0101
STATUS : NO ERROR
IC     : 03
IR     : H
TTC    : 04
LLC    : 01
```

Here `GD20` needs page 2, which does not exist yet. That is a valid page fault, so the OS gives the page a frame and the read continues. After the job, the report shows the instruction counter, the last instruction, the time used (`TTC`) and the lines written (`LLC`).

When something goes wrong, the report says why. This job has a time limit of 1 and an unknown instruction in the second step:

```
$AMJ010900010002
LR00XX00H
$DTA
NOT USED
$END0109
```

```
JOB ID : 0109
STATUS : TIME LIMIT EXCEEDED
         OPCODE ERROR
IC     : 02
IR     : XX00
TTC    : 02
LLC    : 00
```

## Stage II in short

**Paging.** A virtual address is split into a page number and an offset. The page table maps the page to a frame, and the real address is `frame * 10 + offset`. Program cards are loaded into randomly chosen frames. Data pages are created when they are first written.

```
page   = VA / 10
offset = VA % 10
real   = frame_of(page) * 10 + offset
```

**Page faults.** A write (`GD`, `SR`) to a page that is not in memory is a *valid* page fault: a frame is allocated and the instruction goes on. A fetch, `LR`, `CR` or `PD` on a missing page is an *invalid* page fault and ends the job.

**Interrupts.** After every instruction the program checks three flags and calls `MOS()` if one is set.

| Flag | Values |
| ---- | ------ |
| `SI` | 1 read, 2 write, 3 terminate |
| `TI` | 2 when the time limit is exceeded |
| `PI` | 1 opcode error, 2 operand error, 3 page fault |

**Errors.**

| Message | Cause |
| ------- | ----- |
| `NO ERROR` | The job ended with `H`. |
| `OUT OF DATA` | `GD` with no data card left. |
| `LINE LIMIT EXCEEDED` | `PD` after the line limit was reached. |
| `TIME LIMIT EXCEEDED` | The time counter went over the time limit. |
| `OPCODE ERROR` | Unknown instruction. |
| `OPERAND ERROR` | The address is not two digits, or a 10-word block would pass address 99. |
| `INVALID PAGE FAULT` | Fetch, `LR`, `CR` or `PD` used a page that is not in memory. |
| `MEMORY EXCEEDED` | The program has more than 10 cards (added in this project). |

The full interrupt servicing table, the PCB layout and the time costs are in [`phase2/README.md`](phase2/README.md).

## Sample jobs and testing

`phase2/input_Phase2.txt` holds nine short jobs, each one aimed at a particular behaviour. `phase2/output_Phase2.txt` is the output of a correct run.

| Job | What it checks | Result |
| --- | -------------- | ------ |
| 0101 | Valid page fault on `GD`, then print | NO ERROR |
| 0102 | Copy with `LR` / `SR` into a new page | NO ERROR |
| 0103 | `GD` without a data card | OUT OF DATA |
| 0104 | Two `PD` with a line limit of 1 | LINE LIMIT EXCEEDED |
| 0105 | Loop with `BT` and a time limit of 10 | TIME LIMIT EXCEEDED |
| 0106 | Unknown instruction | OPCODE ERROR |
| 0107 | Address that is not a number | OPERAND ERROR |
| 0108 | `LR` on a page that was never created | INVALID PAGE FAULT |
| 0109 | Time limit and opcode error together | TIME LIMIT EXCEEDED + OPCODE ERROR |

To check your build, run the sample and compare with the saved output.

```powershell
# PowerShell
.\mos.exe input_Phase2.txt my_output.txt
Compare-Object (Get-Content my_output.txt) (Get-Content output_Phase2.txt)
```

```bash
# Linux / macOS
./mos input_Phase2.txt my_output.txt && diff my_output.txt output_Phase2.txt && echo "same"
```

No output from `Compare-Object` (or "same" from `diff`) means the two files match.

## Design notes

- **Fixed random seed.** Stage II picks frames at random but with a fixed seed, so the same input always gives the same frames. The frames never change the normal output, only what `-v` prints.
- **Time cost.** In Stage II, `GD` and `SR` cost 2 time units and every other instruction costs 1. This lives in one function (`time_cost`) if a different rule is needed.
- **Line limit.** The line counter counts lines actually written. A `PD` that would go over the limit is refused and not counted.
- **Strict `H`.** The halt instruction must stand alone in its word. Something like `HALT` is reported as an opcode error.
- **Leftover cards.** Data cards that a job does not read are skipped when the loader looks for the next `$AMJ`.
- **Line endings.** Input files with Windows (CRLF) or Unix (LF) line endings give the same result.

## Limitations

- Only one job runs at a time. The paging and interrupt handling follow the Stage II design, but jobs are not scheduled against each other.
- A job with no time limit that loops forever will not stop.
- Only the first 40 characters of a card are used.
- In Stage I, the line limit on the `$AMJ` card is read but not enforced. Stage II enforces it.

## Project structure

```
OS-Simulation/
|-- phase1/
|   |-- main.c               Stage I simulator
|   |-- input_Phase1.txt     Sample job
|   |-- output.txt           Output of the sample job
|   `-- README.md            Stage I details
|-- phase2/
|   |-- main.c               Stage II simulator
|   |-- input_Phase2.txt     Nine sample jobs
|   |-- output_Phase2.txt    Output of the sample jobs
|   `-- README.md            Stage II details
|-- .gitignore               Keeps compiled programs out of the repository
`-- README.md                This file
```

## OS concepts covered

- Machine simulation: memory, registers, instruction fetch, decode and execute
- Supervisor calls and how a program asks the OS for input and output
- Virtual memory: pages, frames, page tables and address translation
- Page faults, and the difference between a valid fault (service it) and an invalid one (terminate)
- Interrupts: generation, priority between them, and servicing
- The process control block and per-job accounting (time and lines)
- Error handling and job termination reports

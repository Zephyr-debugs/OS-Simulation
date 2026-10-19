# MOS Phase 2 (Stage II)

Stage II of the Multiprogramming Operating System simulation. It builds on the Stage I machine (see [`../phase1`](../phase1)) and adds four things:

1. **Paging.** Jobs use virtual addresses. A page table maps each page to a frame of real memory, and frames are handed out at random.
2. **Process data structure.** A process control block (PCB) keeps the job id, the limits and the counters of the running job.
3. **Interrupt generation and servicing.** Three interrupt flags (`SI`, `TI`, `PI`) are raised while the program runs and handled by `MOS()`.
4. **Error handling.** Seven error messages, and a short report written for every job.

Jobs still run one at a time, as in Stage I.

## Machine model

| Component | Description |
| --------- | ----------- |
| Real memory `M` | 300 words of 4 characters. Divided into 30 frames of 10 words. |
| Virtual memory | 100 words per job, which is 10 pages of 10 words. |
| `IR` | Instruction register (4 characters). |
| `IC` | Instruction counter. This is a **virtual** address. |
| `R` | General purpose register (4 characters). |
| `C` | Toggle, set by `CR` and tested by `BT`. |
| `PTR` | Real address of the page table of the running job. |
| `SI` | Supervisor interrupt: 1 = read, 2 = write, 3 = terminate. |
| `TI` | Timer interrupt: 0 = none, 2 = time limit exceeded. |
| `PI` | Program interrupt: 1 = opcode error, 2 = operand error, 3 = page fault. |

The instruction set is the same as in Stage I: `LR`, `SR`, `CR`, `BT`, `GD`, `PD` and `H`. The address part of an instruction is now a virtual address from 00 to 99.

## Paging

**Page table.** The page table of a job sits in one frame of real memory, chosen when the job is loaded. It has 10 entries (one word each). An entry is `****` while the page is not in memory, otherwise it holds the frame number as 4 digits, for example `0017`.

**Address translation.** For a virtual address `VA`:

```
page   = VA / 10
offset = VA % 10
real address = (frame number of page) * 10 + offset
```

If the page table entry is `****`, a page fault (`PI = 3`) is raised.

**Loading.** Each program card goes into its own newly allocated frame, and the next page table entry is filled in. Card 1 becomes page 0, card 2 becomes page 1, and so on. A program can have at most 10 cards.

**Frame allocation.** `ALLOCATE()` picks a random frame number from 0 to 29 and repeats until it finds one that is free. The random generator uses a fixed seed, so the same input always gives the same frames. (Frames are only visible with `-v`; they never change the normal output.)

**Valid and invalid page faults.**

| Instruction that faulted | Kind | What MOS does |
| ------------------------ | ---- | ------------- |
| `GD`, `SR` (these write to memory) | Valid | Allocates a frame, fills the page table entry, and the instruction continues. |
| Instruction fetch, `LR`, `CR`, `PD` | Invalid | Job is terminated with `INVALID PAGE FAULT`. |

This is how data pages come into existence: they are not loaded with the program, only created when the first `GD` or `SR` writes to them. A `GD` or `PD` works on a block of 10 words, so a block that starts at `25` touches two pages and both are handled.

## Process control block

```c
typedef struct {
    char job_id[5];
    int ttl;    /* time limit */
    int tll;    /* line limit */
    int ttc;    /* total time counter */
    int llc;    /* line counter */
} PCB;
```

It is filled from the `$AMJ` card when a job starts. `ttc` and `llc` start at 0.

## Time and line limits

- **Time.** Each executed instruction adds to `ttc`: **2 units for `GD` and `SR`, 1 unit for everything else.** After the instruction, if `ttc` is greater than `ttl`, the timer interrupt `TI = 2` is raised.
- **Lines.** Before a `PD` writes, `llc` is compared with the line limit. If `llc` has already reached `tll`, the job ends with `LINE LIMIT EXCEEDED` and the line is not written. `llc` therefore always equals the number of lines actually written.
- If a limit field on the `$AMJ` card is missing or not numeric, that limit is not applied.

## Interrupt servicing

After every instruction the program checks `SI`, `TI` and `PI` and calls `MOS()` if any of them is set. `MOS()` decides what to do:

| TI | SI | PI | Action |
| -- | -- | -- | ------ |
| 0 | 1 | 0 | READ |
| 0 | 2 | 0 | WRITE |
| 0 | 3 | 0 | Terminate, no error |
| 0 | 0 | 1 | Terminate: `OPCODE ERROR` |
| 0 | 0 | 2 | Terminate: `OPERAND ERROR` |
| 0 | 0 | 3 | Valid page fault: allocate a frame and continue. Invalid: terminate with `INVALID PAGE FAULT` |
| 2 | 1 | 0 | Terminate: `TIME LIMIT EXCEEDED` |
| 2 | 2 | 0 | WRITE, then terminate: `TIME LIMIT EXCEEDED` |
| 2 | 3 | 0 | Terminate, no error |
| 2 | 0 | 0 | Terminate: `TIME LIMIT EXCEEDED` |
| 2 | any | 1 | Terminate: `TIME LIMIT EXCEEDED` and `OPCODE ERROR` |
| 2 | any | 2 | Terminate: `TIME LIMIT EXCEEDED` and `OPERAND ERROR` |
| 2 | any | 3 | Terminate: `TIME LIMIT EXCEEDED` |

Valid page faults are raised and serviced inside the instruction (for `SR` and `GD`) before the time check, so `TI = 2` never meets a valid page fault.

## Errors

| Code | Message | Cause |
| ---- | ------- | ----- |
| 0 | `NO ERROR` | Job ended with `H`. |
| 1 | `OUT OF DATA` | `GD` found no data card (end of file or `$END`). |
| 2 | `LINE LIMIT EXCEEDED` | `PD` after `tll` lines were already written. |
| 3 | `TIME LIMIT EXCEEDED` | `ttc` went over `ttl`. |
| 4 | `OPCODE ERROR` | The first two characters are not a known instruction. |
| 5 | `OPERAND ERROR` | The address is not two digits, or a `GD`/`PD` block would go past address 99 (address above 90). |
| 6 | `INVALID PAGE FAULT` | Fetch, `LR`, `CR` or `PD` used a page that is not in memory. |
| 7 | `MEMORY EXCEEDED` | The program has more than 10 cards. This one is an addition of this project. |

`H` must be written alone in its word (`H` followed by spaces). Something like `HALT` is an opcode error.

## Input format

Same card layout as Stage I:

```
$AMJ<job id><time limit><line limit>     e.g. $AMJ010100100002
<program cards>                          up to 40 characters, 10 instructions per card
$DTA
<data cards>
$END<job id>
```

Each field of the `$AMJ` card is 4 digits, so `$AMJ010100100002` means job `0101`, time limit 10, line limit 2. Several jobs can follow each other in one file. Data cards that a job does not read are skipped.

## Output format

- Every `PD` writes one line of 40 characters (padded with spaces).
- When a job ends, a report is written, followed by one blank line. If the job wrote any lines, one blank line also comes before the report.

```
JOB ID : 0101
STATUS : NO ERROR
IC     : 03
IR     : H
TTC    : 04
LLC    : 01
```

`IC` is the value of the instruction counter when the job stopped (it already points to the next instruction), `IR` is the last instruction fetched, `TTC` the time used and `LLC` the lines written. If the time limit and a program error happen together, both messages are listed under `STATUS`.

## Example

`input_Phase2.txt` contains nine small jobs, one for each situation. The result is in `output_Phase2.txt`.

| Job | What it does | STATUS |
| --- | ------------ | ------ |
| 0101 | Reads a card into page 2 (valid page fault) and prints it | NO ERROR |
| 0102 | Copies a word with `LR`/`SR` into page 3, then prints | NO ERROR |
| 0103 | `GD` with no data card | OUT OF DATA |
| 0104 | Two `PD` with a line limit of 1 | LINE LIMIT EXCEEDED |
| 0105 | Loop `LR`, `CR`, `BT` with a time limit of 10 | TIME LIMIT EXCEEDED |
| 0106 | Unknown instruction `XX20` | OPCODE ERROR |
| 0107 | Address `2A` is not a number | OPERAND ERROR |
| 0108 | `LR50` on a page that was never created | INVALID PAGE FAULT |
| 0109 | Time limit 1 and an unknown instruction in the same step | TIME LIMIT EXCEEDED + OPCODE ERROR |

Step by step for job 0101:

```
$AMJ010100100002
GD20PD20H
$DTA
HELLO PAGING WORLD
$END0101
```

1. The loader reads the `$AMJ` card, fills the PCB (time limit 10, line limit 2) and allocates a frame for the page table.
2. The card `GD20PD20H` is stored in a new frame and becomes page 0.
3. `GD20` needs page 2, which does not exist yet. This is a valid page fault, so a frame is allocated. Then the data card is read into it. Time used: 2.
4. `PD20` writes the line. Time used: 3, lines written: 1.
5. `H` ends the job. Time used: 4.

## Build and run

Any C11 compiler works (GCC, MinGW, Clang).

**Windows (PowerShell)**

```powershell
cd phase2
gcc -std=c11 -Wall -Wextra -O2 main.c -o mos.exe
.\mos.exe input_Phase2.txt output_Phase2.txt
Get-Content output_Phase2.txt
```

**macOS / Linux**

```bash
cd phase2
gcc -std=c11 -Wall -Wextra -O2 main.c -o mos
./mos input_Phase2.txt output_Phase2.txt
cat output_Phase2.txt
```

Usage:

```
mos [input_file] [output_file] [-v]
```

Without file names the program uses `input.txt` and `output.txt`. With `-v` it also prints the page table of every job to the console, which is useful for checking the paging:

```
job 0102: page table in frame 9
  page 0 -> frame 16
  page 2 -> frame 11
  page 3 -> frame 17
```

## Code overview

| Part | Functions |
| ---- | --------- |
| Setup and cards | `initMemory`, `init_pcb`, `trim_newline`, `trim_leading_spaces`, `is_control`, `read_limit` |
| Paging | `ALLOCATE`, `ADDRESSMAP`, `frame_of_page`, `set_page_entry`, `map_for_store`, `map_block_for_store`, `service_page_fault` |
| Instruction decoding | `get_opcode`, `get_operand`, `time_cost` |
| Services | `READ`, `WRITE`, `TERMINATE` |
| Interrupt handling | `MOS` |
| Running a job | `LOAD`, `store_card`, `STARTEXECUTION`, `EXECUTEUSERPROGRAM` |

## Limitations

- One job runs at a time. The paging and interrupt parts follow the Stage II design, but there is no real multiprogramming.
- Every instruction costs a fixed time (2 for `GD`/`SR`, 1 otherwise). Other costs would only need a change in `time_cost`.
- A program that has no time limit and loops forever will not stop.

## Project structure

```
phase2/
|-- main.c              Stage II simulator
|-- input_Phase2.txt    Nine sample jobs
|-- output_Phase2.txt   Output of the sample jobs
`-- README.md
```

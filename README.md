# MOS Phase 1 Simulation

A C simulation of Phase 1 of a teaching operating system (MOS). The program models a small computer (a 100-word memory, a few registers and a seven-instruction CPU) together with a minimal operating system layer. The OS reads jobs from a card-image input file, runs them one at a time, and writes the results to an output file.

This repository covers Phase 1 only. Jobs run one after another, so features such as paging, scheduling and time or line limits are not part of it.

## How it works

The input file is treated as a deck of punched cards. The loader stores each job's program in memory, the CPU runs it, and any request for input or output goes through the OS as a supervisor call.

```
input file -> loader -> memory -> CPU (fetch, decode, execute)
                                     |
                          GD / PD / H  (sets SI)
                                     |
                                   MOS()
                        READ / WRITE / TERMINATE
                                     |
                               output file
```

## Machine model

| Component | Description |
| --------- | ----------- |
| Memory `M` | 100 words, 4 characters each. Filled with spaces at the start of every job. |
| `IR` | Instruction register. Holds the instruction being executed (4 characters). |
| `IC` | Instruction counter. Address of the next instruction. |
| `R` | General purpose register (4 characters). |
| `C` | Toggle (0 or 1). Set by `CR`, tested by `BT`. |
| `SI` | Service interrupt code. Set by `GD`, `PD` and `H` before `MOS()` is called. |

## Instruction set

Every instruction fits in one word: a 2-character opcode followed by a 2-digit address (for example `LR20`). `H` has no address.

| Instruction | Action |
| ----------- | ------ |
| `LR xx` | Load the word at address `xx` into `R`. |
| `SR xx` | Store `R` into the word at address `xx`. |
| `CR xx` | Compare `R` with the word at `xx`. `C = 1` if equal, otherwise `C = 0`. |
| `BT xx` | If `C = 1`, set `IC = xx` (branch). |
| `GD xx` | Read the next data card into words `xx` to `xx+9` (`SI = 1`). |
| `PD xx` | Write words `xx` to `xx+9` to the output as one line (`SI = 2`). |
| `H` | Halt the job (`SI = 3`). |

Notes:

- `LR`, `SR` and `CR` work on a single word, which is 4 characters.
- `GD` and `PD` work on a block of 10 words (40 characters), so their address must be 90 or less.
- Any instruction word that starts with `H` is treated as a halt.

## Supervisor calls

| `SI` | Service | What it does |
| ---- | ------- | ------------ |
| 1 | READ | Reads one data card into memory. |
| 2 | WRITE | Writes one 40-character line to the output file. |
| 3 | TERMINATE | Writes two blank lines and ends the job. |

Any other value prints `INVALID SUPERVISOR CALL` and aborts the job.

## Input format

A job has this layout:

```
$AMJ<job id><time limit><line limit>
<program cards>
$DTA
<data cards>
$END<job id>
```

- **`$AMJ` card.** Starts a job and resets memory. The three 4-digit fields (job id, time limit, line limit) are read as part of the card but are not used in Phase 1.
- **Program cards.** Up to 40 characters each, so 10 instructions per card. Card 1 goes into words 00 to 09, card 2 into words 10 to 19, and so on. A job can have at most 10 program cards.
- **`$DTA` card.** Marks the end of the program and starts execution.
- **Data cards.** One card is consumed by each `GD`. Cards longer than 40 characters are cut to 40, and shorter ones are padded with spaces.
- **`$END` card.** Marks the end of the job.
- Several jobs can be placed one after another in the same file. Blank lines between jobs are ignored.

Choose data addresses that do not overlap the program. A one-card program uses words 00 to 09, which is why the sample job reads its data into word 20.

## Output format

- Each `PD` writes one line of exactly 40 characters, padded with spaces.
- Each job that ends with `H` is followed by two blank lines.
- If a job fails, an error message and one blank line are written, and the loader moves on to the next job.

### Error messages

| Message | Cause |
| ------- | ----- |
| `OUT OF DATA` | `GD` was executed but there was no data card left (end of file or `$END` reached). |
| `INVALID READ ADDRESS` / `INVALID WRITE ADDRESS` | `GD` or `PD` address is not two digits, or is greater than 90. |
| `INVALID LR ADDRESS` (also `SR`, `CR`, `BT`) | The address is not a valid two-digit number. |
| `INVALID OPERATION: xx` | The opcode is not one of the seven instructions. |
| `INVALID INSTRUCTION ADDRESS` | `IC` moved outside memory. |
| `INVALID SUPERVISOR CALL` | `SI` had a value other than 1, 2 or 3. |
| `MEMORY EXCEEDED` | The program has more than 10 cards. |

## Example

Input (`input_Phase1.txt`):

```
$AMJ010100040001
GD20PD20H
$DTA
HELLO WORLD
$END0101
```

Output (`output.txt`):

```
HELLO WORLD                             


```

What happens step by step:

1. The loader stores `GD20`, `PD20` and `H` in words 0, 1 and 2.
2. `GD20` reads `HELLO WORLD` into words 20 to 29.
3. `PD20` writes words 20 to 29 as one 40-character line.
4. `H` writes the two blank lines and ends the job.

Two jobs in one file work the same way. Each job gets its own output line and its own two blank lines.

## Build and run

Any C11 compiler will do (GCC, MinGW or Clang).

**Windows (PowerShell)**

```powershell
gcc -std=c11 -Wall -Wextra -O2 main.c -o mos.exe
.\mos.exe input_Phase1.txt output.txt
Get-Content output.txt
```

**macOS / Linux**

```bash
gcc -std=c11 -Wall -Wextra -O2 main.c -o mos
./mos input_Phase1.txt output.txt
cat output.txt
```

Usage:

```
mos [input_file] [output_file]
```

If the arguments are left out, the program uses `input.txt` and `output.txt`. When the run finishes it prints a short completion message. If a file cannot be opened, it prints an error and exits with code 1.

## Code overview

| Function | Role |
| -------- | ---- |
| `main` | Opens the input and output files and calls `LOAD`. |
| `LOAD` | Finds each `$AMJ`, resets memory, stores program cards, and starts execution at `$DTA`. |
| `EXECUTEUSERPROGRAM` | The fetch, decode and execute loop. |
| `MOS` | Looks at `SI` and calls the matching service. |
| `READ`, `WRITE`, `TERMINATE` | The three supervisor call handlers. |

## Limitations

- The time limit and line limit on the `$AMJ` card are not enforced. A program that loops forever, such as `GD20LR20CR20BT01H`, will never stop.
- Only one job runs at a time. There is no multiprogramming, paging or process scheduling.
- Only the first 40 characters of any card are used.
- Data cards that a job does not read are skipped when the loader searches for the next `$AMJ`.

## Project structure

```
MOS-Phase1-Simulation/
|-- main.c              Simulator source code
|-- input_Phase1.txt    Sample job
|-- output.txt          Output produced by the sample job
|-- mos.exe             Prebuilt Windows executable
`-- README.md
```

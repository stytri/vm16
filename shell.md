# Virtual Machine 16 Shell Program and Function Library
```msa;asm;c
;// MIT License
;//
;// Copyright (c) 2025 Tristan Styles
;//
;// Permission is hereby granted, free of charge, to any person obtaining a copy
;// of this software and atsociated documentation files (the "Software"), to deal
;// in the Software without restriction, including without limitation the rights
;// to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
;// copies of the Software, and to permit persons to whom the Software is
;// furnished to do so, subject to the following conditions:
;//
;// The above copyright notice and this permission notice shall be included in all
;// copies or substantial portions of the Software.
;//
;// THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
;// IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
;// FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
;// AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
;// LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
;// OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
;// SOFTWARE.
```
```msa;c
;// Version 0.1.0
```
## Description

A simple shell for the **vm16** emulator.

## Source Code

### Customised Assembler Instructions

#### CALL

Call a sub-routine whose address is in the register argument; saves the return address in RC
```msa
{{hidden}CALL}
CALL%r {
	$9 = $1,
	$5 = 1, $3 = PC, $2 = RC, $1 = ADD,
	$4 = 0xB, EMIT_ORXM_Y_INSTRUCTION,
	$3 = $9, $2 = PC, $1 = MOV,
	$4 = 0x4, EMIT_ORXM_INSTRUCTION
}
```
#### IF

If register is non-zero (any bit set)
```msa
{{hidden}IF}
IF%r {
	$2 = $1, $3 = 0xFFFF, $1 = IFB,
	EMIT_ORXM_L_INSTRUCTION
}
```
### Start Address and Interrupt Handler Assignments
```asm
0x0000:
	; Start Address
	_init
	; IRQ handler table - 15 entries
	_nul_irq_handler
	_nul_irq_handler
	_nul_irq_handler
	_nul_irq_handler
	_nul_irq_handler
	_nul_irq_handler
	_nul_irq_handler
	_nul_irq_handler
	_nul_irq_handler
	_nul_irq_handler
	_nul_irq_handler
	_nul_irq_handler
	_nul_irq_handler
	_nul_irq_handler
	_clk_irq_handler
```
### Set CODE and DATA Spaces
```asm
CODE 0x0010:                    ; set CODE address space
```
```asm
DATA 0x8000:                    ; set DATA address space
```
### Interrupts

#### Data for Interrupt Handlers
```asm
DATA:
```
```asm
_device_ready:
	0                           ; contains ready flag bits for devices
```
```asm
_clock_ticks:
	0                           ; count of clock ticks
```
#### Interrupt Handlers

```asm
CODE:
```
##### _init

system initialization
```asm
_init:
	MOV SP 0x0000               ; Stack Base - first stack entry will end up at 0xFFFF
	MOV R0 0xFFFF               ; all devices bit mask
	MOV @_device_ready R0       ; all devices ready
	JSR RC main                 ; call to main
	MOV R0 0xFFFF               ; select all devices
_init$1:
	IFN R0 @_device_ready       ; loop while any device ready bit not set
		BRA _init$1
	MOV R0 0                    ; exit code = 0
	HLT R0                      ; halt
```
##### _wait_device_ready

waits until a device bit is set signalling the receipt of the device interrupt
```asm
_wait_device_ready:
	; RB device number
	PSH R0
	MOV R0 1
	SHL R0 RB                   ; R0 = device ready bit mask
_wait_device_ready$1:
	IFC R0 @_device_ready       ; loop while device ready bit clear
		BRA _wait_device_ready$1
	POP R0
	RET RC
```
##### _wait_then_clear_device_ready

waits until a device bit is set signalling the receipt of the device interrupt; then clears it
```asm
_wait_then_clear_device_ready:
	; RB device number
	PSH R0
	MOV R0 1
	SHL R0 RB                   ; R0 = device ready bit mask
_wait_then_clear_device_ready$1:
	IFC R0 @_device_ready       ; loop while device ready bit clear
		BRA _wait_then_clear_device_ready$1
	XOR R0 0xFFFF               ; invert device ready bit mask
	AND @_device_ready R0       ; clear device ready bit
	POP R0
	RET RC
```
##### Clock Tick Interrupt

increments the clock tick count
```asm
_clk_irq_handler:
	; R0 irq number - not used
	MOV R0 @_clock_ticks        ;\
	ADD R0 1                    ; increment clock tick counter by 1
	MOV @_clock_ticks R0        ;/
	RFI
```
##### Device Command Complete Interrupt

sets the appropriate device ready bit
```asm
_nul_irq_handler:
	; R0 irq number
	PSH R1
	MOV R1 1
	SHL R1 R0                   ; R1 = device ready bit mask
	IOR @_device_ready R1       ; set device ready bit
	POP R1
	RFI
```
### Library Functions

#### I/O Functions

A set of device agnostic I/O functions
```asm
CODE:
```
##### io_eof

retrieves the end-of-file state
```asm
io_eof:
	; RB Device index
	PSH RC
	PSH R1
	MOV R0 0
	SHL R1 RB 12
	JSR RC _wait_then_clear_device_ready
	DEV IO_EOF 0 R0 R1
	JSR RC _wait_device_ready
	DEV IO_QUERY 0 R0           ; get EOF status
	POP R1
	; R0 eof
	RET
```
##### io_error

retrieves the last operation's error code
```asm
io_error:
	; RB Device index
	PSH RC
	PSH R1
	MOV R0 0
	SHL R1 RB 12
	JSR RC _wait_then_clear_device_ready
	DEV IO_ERROR 0 R0 R1
	JSR RC _wait_device_ready
	DEV IO_QUERY 0 R0           ; get Error code
	POP R1
	; R0 error
	RET
```
##### io_flush

flushes output
```asm
io_flush:
	; RB Device index
	PSH RC
	PSH R0
	PSH R1
	MOV R0 0
	SHL R1 RB 12
	JSR RC _wait_then_clear_device_ready
	DEV IO_SYNC 0 R0 R1
	JSR RC _wait_device_ready
	POP R1
	POP R0
	RET
```
##### io_putc

outputs a character
```asm
io_putc:
	; RB Device index
	; R0 character
	PSH RC
	PSH R1
	SHL R1 RB 12
	JSR RC _wait_then_clear_device_ready
	DEV IO_PUTC 0 R0 R1
	POP R1
	RET
```
##### io_puts

outputs a string
```asm
io_puts:
	; RB Device index
	; R0 string address
	; R1 string length
	PSH RC
	PSH R3
	SHL R3 RB 12
	IOR R1 R3
	JSR RC _wait_then_clear_device_ready
	DEV IO_PUTS 0 R0 R1
	POP R3
	RET
```
##### io_puti

outputs an integer value in decimal
```asm
io_puti:
	; RB Device index
	; R0 integer
	PSH RC
	PSH R1
	MOD R1 R0 10                ; R1 = R0 % 10
	DIV R0 10                   ; R0 = R0 / 10
	IFC SF Z                    ; if R0 non-zero
		JSR RC io_puti          ; recursive call self
	ADD R0 R1 0x30              ; R0 = R1 + '0'
	JSR RC putc                 ; call putc
	POP R1
	RET
```
##### io_getc

inputs a character
```asm
io_getc:
	; RB Device index
	PSH RC
	PSH R1
	MOV R0 0
	SHL R1 RB 12
	JSR RC _wait_then_clear_device_ready
	DEV IO_GETC 0 R0 R1
	JSR RC _wait_device_ready
	DEV IO_QUERY 0 R0           ; get read in character
	POP R1
	; R0 character
	RET
```
##### io_gets

inputs a string
```asm
io_gets:
	; RB Device index
	; R0 string address
	; R1 maximum string length
	PSH RC
	PSH R3
	SHL R3 RB 12
	IOR R1 R3
	JSR RC _wait_then_clear_device_ready
	DEV IO_GETS 0 R0 R1
	JSR RC _wait_device_ready
	DEV IO_QUERY 0 R0 R1        ; gets address + read in length
	POP R3
	; R0 string address
	; R1 string length
	RET
```
#### STDIO I/O Function Wrappers

A set of functions to wrap the `io_`... functions for the STDIO device.
```asm
CODE:
```
##### eof

determines if stdin has reached end-of-file
```asm
eof:
	PSH RC
	PSH RB
	MOV RB STDIO
	JSR RC io_eof
	POP RB
	; R0 eof
	RET
```
##### error

obtains the error code for the last operation
```asm
error:
	PSH RC
	PSH RB
	MOV RB STDIO
	JSR RC io_error
	POP RB
	; R0 error
	RET
```
##### flush

flush output on stdout
```asm
flush:
	PSH RC
	PSH RB
	MOV RB STDIO
	JSR RC io_flush
	POP RB
	RET
```
##### putc

writes a character to stdout
```asm
putc:
	; R0 character
	PSH RC
	PSH RB
	MOV RB STDIO
	JSR RC io_putc
	POP RB
	RET
```
##### puts

writes a string to stdout
```asm
puts:
	; R0 string address
	; R1 string length
	PSH RC
	PSH RB
	MOV RB STDIO
	JSR RC io_puts
	POP RB
	RET
```
##### puti

writes an integer in decimal to stdout
```asm
puti:
	; R0 integer
	PSH RC
	PSH RB
	MOV RB STDIO
	JSR RC io_puti
	POP RB
	RET
```
##### getc

reads a character from stdin
```asm
getc:
	PSH RC
	PSH RB
	MOV RB STDIO
	JSR RC io_getc
	POP RB
	; R0 character
	RET
```
##### gets

reads a string from stdin
```asm
gets:
	; R0 string address
	; R1 maximum string length
	PSH RC
	PSH RB
	MOV RB STDIO
	JSR RC io_gets
	POP RB
	; R0 string address
	; R1 string length
	RET
```
#### Character Functions
```asm
CODE:
```
##### _ascii_in_range

determines if a character is with an inclusive range of ascii codes; **must not** be called directly, only via the provided interface functions.
```asm
_ascii_in_range:
	; R0 character to check
	; R1 upper range character in high 8 bits
	;    lower range character in low 8 bits
	PSH R2
	PSH R3
	SHR R3 R1 8
	AND R1 255
	MOV R2 R0
	SUB R2 R1
	IFB SF N
		BRA _ascii_in_range$0
	SUB R0 R3
	IFC SF NZ
		BRA _ascii_in_range$0
	MOV R0 1
	BRA _ascii_in_range$1
_ascii_in_range$0:
	MOV R0 0
_ascii_in_range$1:
	POP R3
	POP R2
	POP R1
	RET RC
```
##### isgraph

determines if the character has a graphical representation
```asm
isgraph:
	PSH R1
	MOV R1 '~!'
	BRA _ascii_in_range
```
##### isspace

determines if the character has no graphical representation
```asm
isspace:
	PSH RC
	JSR RC isgraph
	XOR R0 1
	RET
```
##### isprint

determines if the character has a graphical representation, or is a blank space
```asm
isprint:
	PSH R1
	MOV R1 ' !'
	BRA _ascii_in_range
```
##### isupper

determines if the character is alphabetic uppercase
```asm
isupper:
	PSH R1
	MOV R1 'ZA'
	BRA _ascii_in_range
```
##### islower

determines if the character is alphabetic lowercase
```asm
islower:
	PSH R1
	MOV R1 'za'
	BRA _ascii_in_range
```
##### isdigit

determines if the character is a decimal digit
```asm
isdigit:
	PSH R1
	MOV R1 '90'
	BRA _ascii_in_range
```
##### isodigit

determines if the character is an octal digit
```asm
isodigit:
	PSH R1
	MOV R1 '70'
	BRA _ascii_in_range
```
##### isxdigit

determines if the character is a hexadecimal digit
```asm
isxdigit:
	PSH RC
	PSH R1
	MOV R1 R0
	JSR RC isdigit
	IF  R0
		BRA isxdigit$1
	MOV R0 R1
	JSR RC isupper
	IFC R0 0xFFFF
		BRA isxdigit$0
	ADD R1 0x20
isxdigit$0:
	MOV R0 R1
	MOV R1 'fa'
	BRA _ascii_in_range
isxdigit$1:
	MOV R0 1
	POP R1
	RET
```
##### isalpha

determines if the character is alphabetic
```asm
isalpha:
	PSH RC
	PSH R1
	MOV R1 R0
	JSR RC isupper
	IF  R0
		BRA isalpha$1
	MOV R0 R1
	JSR RC islower
	IF  R0
		BRA isalpha$1
isalpha$0:
	MOV R0 0
	POP R1
	RET
isalpha$1:
	MOV R0 1
	POP R1
	RET
```
##### isalnum

determines if the character is alphabetic or a decimal digit
```asm
isalnum:
	PSH RC
	PSH R1
	MOV R1 R0
	JSR RC isdigit
	IF  R0
		BRA isalnum$1
	MOV R0 R1
	JSR RC isupper
	IF  R0
		BRA isalnum$1
	MOV R0 R1
	JSR RC islower
	IF  R0
		BRA isalnum$1
isalnum$0:
	MOV R0 0
	POP R1
	RET
isalnum$1:
	MOV R0 1
	POP R1
	RET
```
##### tolower

converts an uppercase character to lowercase
```asm
tolower:
	PSH RC
	PSH R1
	MOV R1 R0
	JSR RC isupper
	IFC R0 0xFFFF
		BRA tolower$0
	ADD R1 0x20
tolower$0:
	MOV R0 R1
	POP R1
	RET
```
##### toupper

converts a lowercase character to uppercase
```asm
toupper:
	PSH RC
	PSH R1
	MOV R1 R0
	JSR RC islower
	IFC R0 0xFFFF
		BRA toupper$0
	SUB R1 0x20
toupper$0:
	MOV R0 R1
	POP R1
	RET
```
#### String Functions
```asm
CODE:
```
##### skip
skips characters that are not accepted by a character test function
```asm
skip:
	; R0 string address
	; R1 string length
	; R2 test function address
	PSH RC
	PSH R3
	MOV R3 R0
	BRA skip$2
skip$1:
	MOV R0 @R3
	CALL R2                     ; call function at address in R2
	IFC R0 0xFFFF               ; not a match
		BRA skip$3
	ADD R3 1
	SUB R1 1
skip$2:
	IF  R1
		BRA skip$1
skip$3:
	MOV R0 R3
	POP R3
	RET
```
##### skipws
skips white-space.
```asm
skipws:
	; R0 string address
	; R1 string length
	PSH RC
	PSH R2
	MOV R2 isspace
	JSR RC skip
	POP R2
	RET
```
##### skipns
skips non-space.
```asm
skipns:
	; R0 string address
	; R1 string length
	PSH R2
	MOV R2 isgraph
	JSR RC skip
	POP R2
	RET RC
```
##### equals
determines if are two strings equal up to the length of the 1st string
```asm
equals:
	; R0 1st string address -- NOT preserved -- returns result
	; R1 1st string length  -- NOT preserved -- unmatched character count
	; R2 2nd string address -- preserved
	; R3 2nd string length  -- preserved
	PSH R2
	PSH R3
	SUB R3 R1
	IFB SF N                    ; if 2nd string is shorter than 1st string
		BRA equals$0
equals$2:
	MOV R3 @R0++
	SUB R3 @R2++
	IFC SF Z                    ; if character mismatch
		BRA equals$0
	SUB R1 1
	IFC SF Z                    ; if characters remaining
		BRA equals$2
equals$1:
	MOV R0 1                    ; indicates equal
	POP R3
	POP R2
	RET RC
equals$0:
	MOV R0 0                    ; indicates not equal
	POP R3
	POP R2
	RET RC
```
##### modify
modify a string by calling a function for each character
```asm
modify:
	; R0 string address
	; R1 string length
	; R2 modify function address
	PSH RC
	PSH R0
	PSH R1
	PSH R3
	MOV R3 R0
	BRA modify$2
modify$1:
	MOV R0 @R3
	CALL R2                     ; call function at address in R2
	MOV @R3 R0
	ADD R3 1
	SUB R1 1
modify$2:
	IF  R1
		BRA modify$1
	POP R3
	POP R1
	POP R0
	RET
```
#### Shell Functions
```asm
CODE:
```
##### prompt
outputs the command prompt
```asm
prompt:
	PSH RC
	MOV R0 '('
	JSR RC putc
	MOV R0 @_clock_ticks
	JSR RC puti
	MOV R0 ')'
	JSR RC putc
	MOV R0 '>'
	JSR RC putc
	MOV R0 ' '
	JSR RC putc
	JSR RC flush
	RET
```
##### getcommand
command buffer
```
DATA:
```asm
ALIGN 128:
CmdString$len:
	OFF CmdString$ CmdString$end
CmdString$:
	RESERVE 127:
CmdString$end:
```
get the command
```asm
CODE:
getcommand:
	PSH RC
	MOV R0 CmdString$
	MOV R1 @CmdString$len
	JSR RC gets
	RET
```
##### do_command
names of supported commands
```asm
DATA:
```
```asm
Cmd$echo$len:
	OFF Cmd$echo$ Cmd$echo$end
Cmd$echo$:
	"echo"
Cmd$echo$end:

Cmd$quit$len:
	OFF Cmd$quit$ Cmd$quit$end
Cmd$quit$:
	"quit"
Cmd$quit$end:

Cmd$unknown$len:
	OFF Cmd$unknown$ Cmd$unknown$end
Cmd$unknown$:
	"unknown command: "
Cmd$unknown$end:
```
find and execute the entered command
```asm
CODE:
```
```asm
do_command:
	; R0 command string address
	; R1 command string length
	PSH RC
	PSH R2
	PSH R3

	JSR RC skipws               ; trim leading white-space
	MOV R2 R0
	MOV R3 R1

do_command$echo:
	MOV R0 Cmd$echo$
	MOV R1 @Cmd$echo$len
	JSR RC equals
	IFC R0 0xFFFF               ; R0 is zero, so no match
		BRA do_command$quit
	ADD R0 R2 @Cmd$echo$len
	SUB R1 R3 @Cmd$echo$len
	JSR RC skipws
	JSR RC puts
	MOV R0 '\n'
	JSR RC putc
	BRA do_command$1

do_command$quit:
	MOV R0 Cmd$quit$
	MOV R1 @Cmd$quit$len
	JSR RC equals
	IFC R0 0xFFFF               ; R0 is zero, so no match
		BRA do_command$unknown
	MOV R0 0                    ; terminate
	BRA do_command$0

do_command$unknown:
	MOV R0 Cmd$unknown$
	MOV R1 @Cmd$unknown$len
	JSR RC puts
	MOV R0 R2
	MOV R1 R3
	JSR RC puts
	MOV R0 '\n'
	JSR RC putc

do_command$1:
	MOV R0 1                    ; continue
do_command$0:
	POP R3
	POP R2
	RET
```
## main
```asm
CODE:
```
```asm
main:
	PSH RC
	main$1:
		JSR RC prompt
		JSR RC getcommand
		JSR RC do_command
		IF  R0                  ; loop while R0 is non-zero
			BRA main$1
	RET
```
## Build

Uses [m (v4.1.0)](https://github.com/stytri/m), [eb (v1.2.1)](https://github.com/stytri/eb), [cssc (v1.1.0)](https://github.com/stytri/cssc) and, [msa (V2.0.0)](https://github.com/stytri/msa) to compile this program.

### Build the Threaded Version of vm16

On the command line enter:
	`m -D threaded vm16.md`
to create the **msa** file, and compile the **vm16** emulator.

### Build the Shell Program

To compile the shell program enter:
	`m shell.md`
then enter:
	`vm16 shell.vm16`
to run it.

## m rules:
```
[comment]: # (::build)
[comment]: # (:+  eb -t msa -l -x .msa -o $!.msa $!)
[comment]: # (:&  eb -t asm -l -x .asm $! | cssc -c asm -p -s "," -n -o $!.asm)
[comment]: # (:&  msa $"* -a $!.sym -o $^.vm16 vm16.md.msa $!.msa $!.asm)
[comment]: # ()
```

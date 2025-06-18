# Virtual Machine 16 Shell
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
;// Version 0.0.1
```
## Description

A simple shell for the **vm16** emulator.

## Source Code

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
### Interrupt Handlers
```asm
DATA:
_device_ready:
	0                           ; contains ready flag bits for devices
```
```asm
DATA:
_clock_ticks:
	0                           ; count of clock ticks
```
```asm
CODE:
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
```asm
CODE:
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

_clk_irq_handler:
	; R0 irq number - not used
	MOV R0 @_clock_ticks        ;\
	ADD R0 1                    ; increment clock tick counter by 1
	MOV @_clock_ticks R0        ;/
	RFI

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

#### STDIO Functions
```asm
CODE:
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
	DEV IO_QUERY 0 R0 R1    ; gets address + read in length
	POP R3
	; R0 string address
	; R1 string length
	RET
```
```asm
CODE:
flush:
	PSH RC
	PSH RB
	MOV RB STDIO
	JSR RC io_flush
	POP RB
	RET

putc:
	; R0 character
	PSH RC
	PSH RB
	MOV RB STDIO
	JSR RC io_putc
	POP RB
	RET

puts:
	; R0 string address
	; R1 string length
	PSH RC
	PSH RB
	MOV RB STDIO
	JSR RC io_puts
	POP RB
	RET

puti:
	; R0 integer
	PSH RC
	PSH RB
	MOV RB STDIO
	JSR RC io_puti
	POP RB
	RET

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
#### String Functions

##### skipws
skips white-space; which is determined as being any character code less than 0x20
```asm
CODE:
skipws:
	; R0 string address
	; R1 string length
	PSH R2
	BRA skipws$2
skipws$1:
	MOV R2 @R0
	SUB R2 0x20
	IFC SF NZ
		BRA skipws$3
	ADD R0 1
	SUB R1 1
skipws$2:
	IFB R1 0xFFFF
		BRA skipws$1
skipws$3:
	POP R2
	RET RC
```
##### equals
determines if are two strings equal up to the length of the 1st string
```asm
CODE:
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
gets the command
```asm
DATA:
ALIGN 128:
CmdString$len:
	OFF CmdString$ CmdString$end
CmdString$:
	RESERVE 127:
CmdString$end:

CODE:
getcommand:
	PSH RC
	MOV R0 CmdString$
	MOV R1 @CmdString$len
	JSR RC gets
	RET
```
##### do_command
executes the entered command
```asm
DATA:
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

CODE:
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
main:
	PSH RC
	main$1:
		JSR RC prompt
		JSR RC getcommand
		JSR RC do_command
		IFB R0 0xFFFF           ; loop while R0 is non-zero
			BRA main$1
	RET
```

## Build

Uses [m (v4.1.0)](https://github.com/stytri/m), [eb (v1.2.1)](https://github.com/stytri/eb), [cssc (v1.1.0)](https://github.com/stytri/cssc) and, [msa (V1.4.1)](https://github.com/stytri/msa) to compile this program.

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
[comment]: # (:&  eb -t asm -l -x .asm  $! | cssc -c asm -p -s "," -n -o $!.asm)
[comment]: # (:&  msa $"* -o $^.vm16 vm16.md.msa $!.msa $!.asm)
[comment]: # ()
```

# Virtual Machine 16 Devices
```c
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
```c
;// Version 0.1.1
```
## Description

Implements devices for [vm16](https://github.com/stytri/vm16) running in separate threads.

## Implementation

Implemented in C[23](https://en.wikipedia.org/wiki/C23_(C_standard_revision)) + POSIX [Threads](https://en.wikipedia.org/wiki/Pthreads).

```c
#include <stdlib.h>
#include <string.h>
#include <stdint.h>
#include <stdio.h>
#include <errno.h>
#include <stdatomic.h>
#include <semaphore.h>
#include <pthread.h>
```
```c
#define THREADED_DEVICES 1
```
```c
typedef struct {
	sem_t                 command;
	atomic_bool           running;
	atomic_uint_fast16_t *irq_reg;
	uint16_t              irq_bit;
	uint16_t              a, b, c;
	void                 *context;
	uint16_t           *(*mem_at)(void *, uint16_t);
	void                 *memory;
	pthread_t             thread;
}
	device;
```
```c
static int new_device(
	atomic_uint_fast16_t *irq_reg,
	uint16_t              irq_bit,
	void               *(*handler)(void *context),
	void                 *context,
	uint16_t           *(*mem_at)(void *, uint16_t x),
	void                 *memory,
	device              **devp
) {
	int err = 0;
	device *dev = malloc(sizeof(*dev));
	if(!dev) {
		err = errno;
	} else {
		sem_init(&dev->command, 0, 0);
		dev->irq_reg = irq_reg;
		dev->irq_bit = irq_bit;
		dev->context = context;
		dev->mem_at  = mem_at;
		dev->memory  = memory;
		atomic_store(&dev->running, true);
		err = pthread_create(&dev->thread, nullptr, handler, dev);
		if(err != 0) {
			free(dev);
		}
	}
	*devp = dev;
	return err;
}
```
```c
static device *del_device(
	device *dev
) {
	pthread_join(dev->thread, nullptr);
	sem_destroy(&dev->command);
	free(dev);
	return nullptr;
}
```
### Device Commands
```msa
{DEVCMD#%i}
```
#### I/O Device Commands
```msa
{IO_QUERY:DEVCMD=0x00}
{IO_ERROR:DEVCMD}
{IO_EOF:DEVCMD}
{IO_SYNC:DEVCMD}
{IO_PUTC:DEVCMD}
{IO_GETC:DEVCMD}
{IO_PUTS:DEVCMD}
{IO_GETS:DEVCMD}
```
```c
#define IOERROR(IOERROR_prefered,IOERROR_stdc) ((IOERROR_prefered+0) ? (IOERROR_prefered) : (IOERROR_stdc))
```
### STDIO Devices

These have a pre-set i/o stream, and deal only with character based i/o.
```c
static void *dev_stdio(void *context) {
	device     *dev = context;
	void       *mem = dev->memory;
	uint16_t *(*mem_at)(void *, uint16_t) = dev->mem_at;
	for(int err;;) {
```
Wait for a command, abort if we are no longer running:
```c
		while(sem_wait(&dev->command))
			;
		if(!atomic_load(&dev->running)) {
			break;
		}
		switch(dev->c) {
```
Ignore unknown commands, but flag as error:
```c
		default:
			err = IOERROR(EINVAL,EILSEQ);
			continue;
```
We should not receive IO_QUERY, but we acknowledge it if we do:
```c
		case DEVCMD_IO_QUERY:
			atomic_fetch_or(dev->irq_reg, dev->irq_bit);
			continue;
```
Return error code from last command:
```c
		case DEVCMD_IO_ERROR:
			dev->a = err;
			atomic_fetch_or(dev->irq_reg, dev->irq_bit);
			continue;
```
Return End-Of-File status:
```c
		case DEVCMD_IO_EOF:
			dev->a = feof(dev->context);
			atomic_fetch_or(dev->irq_reg, dev->irq_bit);
			continue;
```
Synchronise - flush pending write:
```c
		case DEVCMD_IO_SYNC:
			fflush(dev->context);
			break;
```
Output single character:
```c
		case DEVCMD_IO_PUTC:
			dev->a = (uint16_t)fputc(dev->a & 255, dev->context);
			break;
```
Input single character:
```c
		case DEVCMD_IO_GETC:
			dev->a = (uint16_t)fgetc(dev->context);
			break;
```
Output character string:
```c
		case DEVCMD_IO_PUTS:
			{
				uint16_t const len = dev->b;
				uint16_t       addr = dev->a;
				for(dev->b = 0; dev->b < len; dev->b++) {
					int c = *mem_at(mem, addr++) & 255;
					dev->a = (uint16_t)(c = fputc(c, dev->context));
					if(c == EOF) break;
				}
			}
			break;
```
Input character string:
```c
		case DEVCMD_IO_GETS:
			{
				uint16_t const len = dev->b;
				uint16_t       addr = dev->a;
				for(dev->b = 0; dev->b < len; dev->b++) {
					int c = fgetc(dev->context);
					if(c == EOF) break;
					*mem_at(mem, addr++) = c & 255;
					if((c == '\n') || (c == '\0')) break;
				}
			}
			break;
```
End of i/o operation - save error code, and signal command complete:
```c
		}
		err = ferror(dev->context) ? 0 : errno;
		atomic_fetch_or(dev->irq_reg, dev->irq_bit);
	}
	return nullptr;
}
```

#### Block Device Commands
```msa
{IO_BLOCK_READ:DEVCMD}
{IO_BLOCK_WRITE:DEVCMD}
```
### Block Devices

These have a pre-opened binary file, and deal only with block based i/o.
```msa
{DEV_BLOCK_SIZE=512}
```
```c
struct dev_block {
	FILE    *io;
	uint16_t n_blocks;
	uint16_t buffer[DEV_BLOCK_SIZE];
};
```
```c
static void *dev_block(void *context) {
	device           *dev = context;
	void             *mem = dev->memory;
	uint16_t       *(*mem_at)(void *, uint16_t) = dev->mem_at;
	struct dev_block *blk = dev->context;
```
```c
	rewind(blk->io);
	if(fread(blk->buffer, sizeof(blk->buffer), 1, blk->io) != 1) {
		memset(blk->buffer, 0, 0);
		uint16_t n;
		for(n = 0; n < blk->n_blocks; n++) {
			if(fwrite(blk->buffer, sizeof(blk->buffer), 1, blk->io) != 1) {
				break;
			}
		}
		blk->n_blocks = n;
	}
```
```c
	for(int err = ferror(blk->io);;) {
```
Wait for a command, abort if we are no longer running:
```c
		while(sem_wait(&dev->command))
			;
		if(!atomic_load(&dev->running)) {
			break;
		}
		switch(dev->c) {
			uint16_t addr;
```
Ignore unknown commands, but flag as error:
```c
		default:
			err = IOERROR(EINVAL,EILSEQ);
			continue;
```
We should not receive IO_QUERY, but we acknowledge it if we do:
```c
		case DEVCMD_IO_QUERY:
			atomic_fetch_or(dev->irq_reg, dev->irq_bit);
			continue;
```
Return error code from last command:
```c
		case DEVCMD_IO_ERROR:
			dev->a = err;
			atomic_fetch_or(dev->irq_reg, dev->irq_bit);
			continue;
```
Return End-Of-File status:
```c
		case DEVCMD_IO_EOF:
			dev->a = feof(dev->context);
			atomic_fetch_or(dev->irq_reg, dev->irq_bit);
			continue;
```
Synchronise - flush pending write:
```c
		case DEVCMD_IO_SYNC:
			fflush(blk->io);
			break;
```
Read a block:
```c
		case DEVCMD_IO_BLOCK_READ:
			addr = dev->a;
			if((fseek(blk->io, dev->b * DEV_BLOCK_SIZE, SEEK_SET) == 0)
				&& (fread(blk->buffer, sizeof(blk->buffer), 1, blk->io) == 1)
			) {
#if BANKED_MEMORY
				if(((addr + DEV_BLOCK_SIZE) / MEMORY_BANK_SIZE) > 0) {
					uint16_t off = MEMORY_BANK_SIZE - (addr % MEMORY_BANK_SIZE);
					size_t   siz = off * sizeof(*blk->buffer);
					memcpy(mem_at(mem, addr), blk->buffer, siz);
					memcpy(mem_at(mem, addr + off), blk->buffer + off, sizeof(blk->buffer) - siz);
				} else
#endif
				{
					memcpy(mem_at(mem, addr), blk->buffer, sizeof(blk->buffer));
				}
			}
			break;
```
Write a block:
```c
		case DEVCMD_IO_BLOCK_WRITE:
			addr = dev->a;
#if BANKED_MEMORY
			if(((addr + DEV_BLOCK_SIZE) / MEMORY_BANK_SIZE) > 0) {
				uint16_t off = MEMORY_BANK_SIZE - (addr % MEMORY_BANK_SIZE);
				size_t   siz = off * sizeof(*blk->buffer);
				memcpy(blk->buffer, mem_at(mem, addr), siz);
				memcpy(blk->buffer + off, mem_at(mem, addr + off), sizeof(blk->buffer) - siz);
			} else
#endif
			{
				memcpy(blk->buffer, mem_at(mem, addr), sizeof(blk->buffer));
			}
			if((fseek(blk->io, dev->b * DEV_BLOCK_SIZE, SEEK_SET) == 0)
				&& (fwrite(blk->buffer, sizeof(blk->buffer), 1, blk->io) == 1)
			) {
				;
			}
			break;
```
End of i/o operation - save error code, and signal command complete:
```c
		}
		err = ferror(blk->io) ? 0 : errno;
		atomic_fetch_or(dev->irq_reg, dev->irq_bit);
	}
	return nullptr;
}
```

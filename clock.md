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

Implements a simple interval clock for [vm16](https://github.com/stytri/vm16) running in a separate thread.

## Implementation

Implemented in C[23](https://en.wikipedia.org/wiki/C23_(C_standard_revision)) + POSIX [Threads](https://en.wikipedia.org/wiki/Pthreads).

```c
#include <stdlib.h>
#include <stdint.h>
#include <time.h>
#include <errno.h>
#include <stdatomic.h>
#include <pthread.h>
```
```c
#define VM16_CLOCK 1
```
```c
typedef struct {
	atomic_bool           running;
	atomic_uint_fast16_t *irq_reg;
	uint16_t              irq_bit;
	long                  interval;
	struct timespec       next;
	pthread_t             thread;
}
	vm16clock;
```
```c
static void *vm16clock_handler(void *context) {
	vm16clock *clk = context;
	clock_gettime(CLOCK_REALTIME, &clk->next);
	for(;;) {
		if(!atomic_load(&clk->running)) {
			break;
		}
		clk->next.tv_nsec += clk->interval;
		while(clk->next.tv_nsec > 1'000'000'000l) {
			clk->next.tv_nsec -= 1'000'000'000l;
			clk->next.tv_sec  += 1;
		}
		clock_nanosleep(CLOCK_REALTIME, TIMER_ABSTIME, &clk->next, NULL);
		atomic_fetch_or(clk->irq_reg, clk->irq_bit);
	}
	return nullptr;
}
```
```c
static int new_vm16clock(
	atomic_uint_fast16_t *irq_reg,
	uint16_t              irq_bit,
	long                  interval,
	vm16clock           **clkp
) {
	int err = 0;
	vm16clock *clk = malloc(sizeof(*clk));
	if(!clk) {
		err = errno;
	} else {
		clk->irq_reg  = irq_reg;
		clk->irq_bit  = irq_bit;
		clk->interval = 1'000'000'000l / interval;
		atomic_store(&clk->running, true);
		err = pthread_create(&clk->thread, nullptr, vm16clock_handler, clk);
		if(err != 0) {
			free(clk);
		}
	}
	*clkp = clk;
	return err;
}
```
```c
static vm16clock *del_vm16clock(
	vm16clock *clk
) {
	pthread_join(clk->thread, nullptr);
	free(clk);
	return nullptr;
}
```

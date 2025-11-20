# Arduino_STM32-How-to-handle-Fault-with-extended-function-I
## 1. About default function of Arduino_STM32 Fault
It support only detect and display as LED fade blink the Fault occur or not .\
It is to less and not enough.
## 2. Next step
So, we want to add the way to know what type fault occur as below.
- nmi
- hardfault
- memmanage
- busfault
- usagefault

In reality, these decision already supported in ”exc.S” as vector table. 

`C:\Program Files (x86)\Arduino188\hardware\Arduino_STM32\STM32F1\cores\maple\libmaple\exc.S`

```
:
# On an exception, push a fake stack thread mode stack frame and redirect
# thread execution to a thread mode error handler

# From RM008:
# The SP is decremented by eight words by the completion of the stack push.
# Figure 5-1 shows the contents of the stack after an exception pre-empts the
# current program flow.
#
# Old SP--> <previous>
#           xPSR
#           PC
#           LR
#           r12
#           r3
#           r2
#           r1
#    SP-->  r0

.text
.globl __exc_nmi
.globl __exc_hardfault
.globl __exc_memmanage
.globl __exc_busfault
.globl __exc_usagefault

.code 16
.thumb_func
__exc_nmi:
    mov r0, #1
    b __default_exc

.thumb_func
__exc_hardfault:
    mov r0, #2
    b __default_exc

.thumb_func
__exc_memmanage:
    mov r0, #3
    b __default_exc

.thumb_func
__exc_busfault:
    mov r0, #4
    b __default_exc

.thumb_func
__exc_usagefault:
    mov r0, #5
    b __default_exc
  :
```

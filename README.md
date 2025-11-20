# Arduino_STM32-How-to-handle-Fault-with-extended-function-I
## 1. About default function of Arduino_STM32 Fault
It support only detect and display as LED fade blink the Fault occur or not .\
It is to less and not enough.
## 2. Preliminary knowledge
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
The source shows when fault occured, CPU set each fault number to r0 and branch __default_exc.\
So, we need only think the way of display or output to suitable device.\
The part of `__default_exc` is as below.
```
.thumb_func
__default_exc:
    ldr r2, NVIC_CCR            @ Enable returning to thread mode even if there are
    mov r1 ,#1                  @ pending exceptions. See flag NONEBASETHRDENA.
    str r1, [r2]
    cpsid i                     @ Disable global interrupts
    ldr r2, SYSTICK_CSR         @ Disable systick handler
    mov r1, #0
    str r1, [r2]
    ldr r1, CPSR_MASK           @ Set default CPSR
    push {r1}
    ldr r1, TARGET_PC           @ Set target pc
    push {r1}
    sub sp, sp, #24             @ Don't care
    ldr r1, EXC_RETURN          @ Return to thread mode
    mov lr, r1
    bx lr                       @ Exception exit

.align 4
CPSR_MASK:     .word 0x61000000
EXC_RETURN:    .word 0xFFFFFFF9
TARGET_PC:     .word __error
NVIC_CCR:      .word 0xE000ED14    @ NVIC configuration control register
SYSTICK_CSR:   .word 0xE000E010    @ Systick control register
```
There is no use and no destroy about r0 in this function.\
This function set some registers and branch `__error` which included in `util.c`
```
/* (Called from exc.S with global interrupts disabled.) */
__attribute__((noreturn)) void __error(void) {
    if (__lm_error) {
        __lm_error();
    }
    /* Reenable global interrupts */
    nvic_globalirq_enable();
    throb();
}
```
The `__lm_error` is not defined. Then skip.\
`nvic_globalirq_enable()` is macro define and equal to `asm volatile("cpsie 1");`.\
Then, call `throb()` which is main processing routine.
```
/* This was public as of v0.0.12, so we've got to keep it public. */
/**
 * @brief Fades the error LED on and off
 * @sideeffect Sets output push-pull on ERROR_LED_PIN.
 */
__attribute__((noreturn)) void throb(void) {
#ifdef HAVE_ERROR_LED
    int32  slope   = 1;
    uint32 CC      = 0x0000;
    uint32 TOP_CNT = 0x0200;
    uint32 i       = 0;

    gpio_set_mode(ERROR_LED_PORT, ERROR_LED_PIN, GPIO_MODE_OUTPUT);
    /* Error fade. */
    while (1) {
        if (CC == TOP_CNT)  {
            slope = -1;
        } else if (CC == 0) {
            slope = 1;
        }

        if (i == TOP_CNT)  {
            CC += slope;
            i = 0;
        }

        if (i < CC) {
            gpio_write_bit(ERROR_LED_PORT, ERROR_LED_PIN, 1);
        } else {
            gpio_write_bit(ERROR_LED_PORT, ERROR_LED_PIN, 0);
        }
        i++;
    }
#else
    /* No error LED is defined; do nothing. */
    while (1)
        ;
#endif
}
```
This is original source code and shows execute while(1) loop endless.\
if defined HAVE_ERROR_LED then brink at fade mode whwn Fault occur.
## 3. Extention for Fault code number display
For easy customizing, we try to display Fault code as counted blink.
```
#define USE_COUNTED_BLINK
#define DELAY_COUNT 12000
/*
 * Re-define ERROR_LED and ERROR_LED_PIN to PC13_LED for fit to BluePill
 */
#if defined(ERROR_LED_PORT) && defined(ERROR_LED_PIN)
#undef  ERROR_LED_PORT
#undef  ERROR_LED_PIN
#endif

#define ERROR_LED_PORT   GPIOC
#define ERROR_LED_PIN    13
  :
/* This was public as of v0.0.12, so we've got to keep it public. */
/**
 * @brief Fades the error LED on and off
 * @sideeffect Sets output push-pull on ERROR_LED_PIN.
 */
__attribute__((noreturn)) void throb(void) {

#ifdef HAVE_ERROR_LED
    int32  slope   = 1;
    uint32 CC      = 0x0000;
    uint32 TOP_CNT = 0x0200;
    uint32 i       = 0;

  #ifdef USE_COUNTED_BLINK
    uint32 j;
    uint32 cause;
    asm volatile (
        "mov %0, r0"   // assembly instruction: move r0 value to opernd 0 (%0)
        : "=r" (cause) // output operand: C variable cause
        :              // input  operand: none
        : "r0"         // may destroy register
    );
  #endif

    gpio_set_mode(ERROR_LED_PORT, ERROR_LED_PIN, GPIO_MODE_OUTPUT);

  #ifdef USE_COUNTED_BLINK
    /* Error blink */
    cause &= 0x07;     // limit max 7
    if (cause > 5 || cause < 1) {
        cause = 6;
    }
    while(1) {
        for(j=0; j<cause; j++) {
            gpio_write_bit(ERROR_LED_PORT, ERROR_LED_PIN, 0);   // LED ON
            i=DELAY_COUNT; while (i>0) {int k = sqrt(i); i--; }
            gpio_write_bit(ERROR_LED_PORT, ERROR_LED_PIN, 1);   // LED OFF
            i=DELAY_COUNT; while (i>0) {int k = sqrt(i); i--; }
        }
        for (j=cause; j<7; j++) {
            gpio_write_bit(ERROR_LED_PORT, ERROR_LED_PIN, 1);   // LED OFF
            i=DELAY_COUNT; while (i>0) {int k = sqrt(i); i--; }
            i=DELAY_COUNT; while (i>0) {int k = sqrt(i); i--; }
        }
    }
  #else	
    /* Error fade. */
    while (1) {
        if (CC == TOP_CNT)  {
            slope = -1;
        } else if (CC == 0) {
            slope = 1;
        }

        if (i == TOP_CNT)  {
            CC += slope;
            i = 0;
        }

        if (i < CC) {
            gpio_write_bit(ERROR_LED_PORT, ERROR_LED_PIN, 1);
        } else {
            gpio_write_bit(ERROR_LED_PORT, ERROR_LED_PIN, 0);
        }
        i++;
    }
  #endif
#else
    /* No error LED is defined; do nothing. */
	while (1)
        ;
#endif
}
```

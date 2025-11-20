# Arduino_STM32-How-to-handle-Fault-with-extended-function-I
## 1. About default function of Arduino_STM32 Fault
It support only detect and display the Fault occur or not.\
It is to less and not enough.
## 2. Next step
So, we want to add the way to know what type fault occur as below.
- nmi
- hardfault
- memmanage
- busfault
- usagefault
In reality, these decision already supported in ”exc.S” as vector table. 

C:\Program Files (x86)\Arduino188\hardware\Arduino_STM32\STM32F1\cores\maple\libmaple\exc.S

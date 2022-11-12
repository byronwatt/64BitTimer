# 64BitTimer
a lockless 64 bit timer using a 32 bit timer register

# typical way to extend a 32 bit timer to a 64 bit timer:

```
// disable interrupts
// read 32 timer
// if timer < last timer increment wrap count
// save last timer
// result = timer + wrap_count << 32
// enable interrupts
```

```c++
#include <inttypes.h>

extern uint32_t wrap_count;
extern uint32_t last_tick_count;

#define __STATIC_INLINE static inline
#define __ASM __asm
/**
  \brief   Get Priority Mask
  \details Returns the current state of the priority mask bit from the Priority Mask Register.
  \return               Priority Mask value
 */
__STATIC_INLINE uint32_t __get_PRIMASK(void)
{
  register uint32_t priMask;
   __asm volatile ( "mrs	%0, PRIMASK" : "=r" (priMask) );

  return(priMask);
}


/**
  \brief   Set Priority Mask
  \details Assigns the given value to the Priority Mask Register.
  \param [in]    priMask  Priority Mask
 */
__STATIC_INLINE void __set_PRIMASK(uint32_t priMask)
{
   __asm volatile ( "msr	PRIMASK, %0" :: "r" (priMask) );
}


static inline uint32_t arm_cpu_intr_disable()
{
    uint32_t primask = __get_PRIMASK();

    /* disable all interrupts of priority 0 and higher? */
    __set_PRIMASK(0);

    return primask;
}

static inline void arm_cpu_intr_restore(uint32_t primask)
{
        __set_PRIMASK(primask);
}

#define DWT_BASE            (0xE0001000UL)                            /*!< DWT Base Address */
#define DWT                 ((DWT_Type       *)     DWT_BASE      )   /*!< DWT configuration struct */
#define __IOM volatile

typedef struct
{
  __IOM uint32_t CTRL;                   /*!< Offset: 0x000 (R/W)  Control Register */
  __IOM uint32_t CYCCNT;                 /*!< Offset: 0x004 (R/W)  Cycle Count Register */
} DWT_Type;

static inline uint64_t read_tick64()
{
    /* Important function 1 */
    uint32_t prim;
    
    /* Read PRIMASK register, check interrupt status before you disable them */
    /* Returns 0 if they are enabled, or non-zero if disabled */
    prim = arm_cpu_intr_disable();
    uint32_t tick_count = DWT->CYCCNT;
    if (tick_count < last_tick_count)
    {
        wrap_count++;
    }
    last_tick_count = tick_count;
    uint64_t result = wrap_count;
    result <<= 32;
    result += tick_count;

    /* Enable interrupts back */
    arm_cpu_intr_restore(prim);
    return result;
}

uint64_t foo()
{
    return read_tick64();
}
```

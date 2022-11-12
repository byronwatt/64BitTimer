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
static inline uint64_t read_tick64()
{
    uint32_t primask;
    
    /* Read PRIMASK register, check interrupt status before you disable them */
    /* Returns 0 if they are enabled, or non-zero if disabled */
    primask = disable_interrupts();
    uint32_t tick_count = read_WDT_CYCCNT();
    if (tick_count < last_tick_count)
    {
        wrap_count++;
    }
    last_tick_count = tick_count;
    uint64_t result = wrap_count;
    result <<= 32;
    result += tick_count;

    restore_interrupts(primask);
    return result;
}

uint64_t foo()
{
    return read_tick64();
}
```


# lockless way:

```c++

extern volatile uint32_t wrap_count_zone[2];

extern uint32_t wrap_count;
extern uint32_t last_tick_count;

static inline uint64_t read_64_cyccnt()
{
    // note: this code fails if we stall more than 3 seconds between reading WDT_CYCCNT and reading the wrap_count
    // if this is important we could change the algorithm or disable interrupts.
    uint32_t cnt = read_WDT_CYCCNT();
    int zone = (cnt & 0x80000000) ? 1 : 0;
    uint32_t wrap_count = wrap_count_zone[zone];
    uint64_t result = wrap_count;
    result <<= 32;
    result |= cnt; // note |= seems to generate slightly better code than +=
    return result;
}
```

which assembles to:

```
foo:
        ldr     r2, .L11
        ldr     r3, .L11+4
        ldr     r0, [r2, #4]
        lsrs    r2, r0, #31
        ldr     r1, [r3, r2, lsl #2]
        bx      lr
.L11:
        .word   0xE0001000
        .word   wrap_count_zone
```

this requires that the zones counts are updated at least twice before the 32 counter wraps.  perhaps every operating system tick:

```c++
// two wrap counts so that we can use lockless coding.
// wrap_count_zone[0] is used if WDT_CYCCNT < 0x8000_0000
// wrap_count_zone[1] is used if WDT_CYCCNT >= 0x8000_0000
// we periodically update the wrap count that is not being used
volatile uint32_t wrap_count_zone[2];

static int last_zone = 0;

// whenever cyccnt crosses a zone, update the wrap count of the previous zone.
// to avoid a race condition we don't update the previous wrap count
// until we are quite a bit into the new zone.
//
// note: this must be called at least every .3 seconds otherwise we might not update the wrap count for
// each zone properly.
// this function could be called every freertos Tick.
void mt_time_update_64_bit_wrap_counts()
{
    uint32_t cnt = read_WDT_CYCCNT();
    // wait until we are almost finished with this zone...
    // this prevents a low priority thread from getting a bad
    // wrap count even it it swaps out while calling read_64_cyccnt
    // and doesn't get swapped back for 3 seconds!
    if ((cnt & 0x70000000) != 0x70000000)
        return;

    // cnt is now 0x70000000..0x7fffffff or 0xf0000000 .. 0xffffffff
    int zone = (cnt & 0x80000000) ? 1 : 0;

    // are we almost at the end of a new zone?
    // if so, update the wrap count of the previous zone.
    if (zone != last_zone)
    {
        // verify that the static variable hasn't been corrupted.
        MDX2_ASSERT(last_zone < 2,"last_zone %d",last_zone);
        wrap_count_zone[last_zone] += 1;
        last_zone = zone;
    }
    // this algorithm assumes one thread calls wrap_count_update at least 16 times every wrap period.
    // e.g. if the period is 8 seconds we need to call this at least every .5 seconds.
    // suggest we call this every tick.
    //
    // counter               last_zone  zone wrap_count[0] wrap_count[1]
    // 0x00000000..0x6fffffff      0     -       0              0       returns immediately.
    // 0x70000000..0x7fffffff      0     0       0              0       zone=last_zone we do nothing.
    // 0x80000000..0xefffffff      0     -       0              0       returns immediately.
    // 0xf0000000..0xffffffff      0     1       0              0       zone=1 => wrap_count_zone[0]++
    //     wrap_count_zone[0]++
    //     (e.g. we are nearing the end of zone 1,...
    //            get ready for zone 0 to be active)
    // 0xf0000000..0xffffffff      1     1       1              0       zone==last_zone do nothing
    // 0x00000000..0x6fffffff      1     -       1              0       returns immediately.
    // 0x70000000..0x7fffffff      1     1       1              0       zone=0 => wrap_count_zone[1]++
    //     wrap_count_zone[1]++
    //     (e.g. we are nearing the end of zone 0,...
    //            get ready for zone 1 to be active)
    // 0x70000000..0x7fffffff      1     0       1              1       zone=last_zone we do nothing.
    // 0x80000000..0xefffffff      0     -       1              1       returns immediately.
}
```


# cortex-M definitions:

```c++
#include <inttypes.h>

extern uint32_t wrap_count;
extern uint32_t last_tick_count;

/**
  \brief   Get Priority Mask
  \details Returns the current state of the priority mask bit from the Priority Mask Register.
  \return               Priority Mask value
 */
static inline uint32_t __get_PRIMASK(void)
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
static inline void __set_PRIMASK(uint32_t priMask)
{
   __asm volatile ( "msr	PRIMASK, %0" :: "r" (priMask) );
}

// set priority mask to 0 and return previous priority mask.
static inline uint32_t disable_interrupts()
{
    uint32_t primask = __get_PRIMASK();

    /* disable all interrupts */
    __set_PRIMASK(0);

    return primask;
}

static inline void restore_interrupts(uint32_t primask)
{
    __set_PRIMASK(primask);
}

#define DWT_BASE            (0xE0001000UL)                            /*!< DWT Base Address */
#define DWT                 ((DWT_Type       *)     DWT_BASE      )   /*!< DWT configuration struct */

typedef struct
{
  volatile uint32_t CTRL;                   /*!< Offset: 0x000 (R/W)  Control Register */
  volatile uint32_t CYCCNT;                 /*!< Offset: 0x004 (R/W)  Cycle Count Register */
} DWT_Type;

static inline uint32_t read_WDT_CYCCNT()
{
    return DWT->CYCCNT;
}
```

# efficiently extending a 32 bit counter to 64 bits

A 32 bit cycle counter wraps quite quickly, but disabling interrupts to check if the timer has wrapped is quite a few extra instructions.

The following code is a quick way to extend a 32 bit timer to 64 bits, and this light-weight approach means that it's only slightly more expensive to use 64 bit timers and not have to worry about the case where a 32 bit timer wraps.

If we have separate wrap counts two zones: the first half and second half of the 32 counter, then we have a long time to update the wrap counter for the zone that is not in use.

e.g. while the counter is counting from 0x0 .. 0x7fffffff we use wrap_count[0]

and while the counter is counting from 0x8000_0000 .. 0xffff_ffff we use wrap_count[1], and have a lot of time to update wrap_count[0] for when we eventually use wrap_count[0] again.

# typical way to extend a 32 bit timer to a 64 bit timer:

This is the typical way to extend a 32 bit timer to 64 bits, and it can be a little slow, and even slower for a multi-core approach.

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

# which assembles to:

```
foo:
        mrs     ip, PRIMASK
        movs    r3, #0
        msr     PRIMASK, r3
        ldr     r3, .L5
        ldr     r2, .L5+4
        ldr     r3, [r3, #4]
        ldr     r1, [r2]
        cmp     r3, r1
        bcc     .L2
        ldr     r1, .L5+8
        ldr     r1, [r1]
.L3:
        str     r3, [r2]
        movs    r0, #0
        msr     PRIMASK, ip
        adds    r0, r3, r0
        bx      lr
.L2:
        ldr     r0, .L5+8
        ldr     r1, [r0]
        adds    r1, r1, #1
        str     r1, [r0]
        b       .L3
.L5:
        .word   -536866816
        .word   last_tick_count
        .word   wrap_count
```

# lockless way:

full code at [godbolt](https://godbolt.org/#z:OYLghAFBqd5TKALEBjA9gEwKYFFMCWALugE4A0BIEAZgQDbYB2AhgLbYgDkAjF%2BTXRMiAZVQtGIHgBYBQogFUAztgAKAD24AGfgCsp5eiyagA%2BudTkVjVEQJDqzTAGF09AK5smIAKzknADIETNgAcp4ARtikIAAcPOQADuhKxPZMrh5evkkpaUJBIeFsUTHxVtg2dkIiRCykRJme3n7W2LbptfVEhWGR0XEJSnUNTdmtIz3BfSUD8QCUVujupKicXACkAEwAzMGoHjgA1Bs7zsFERACeidhKAHRIp7gbWgCC23tMB%2B7Hp84sJQqBqPZ6vN4AegAVFDwUcTj5nBFSARsDR4UcAOLYIhHVQosjEK5HACygIA1nCEc4cHUGEojgAlHErJgMohIbBHVArUjMXHDFhELnodEcrmJAko65HNgUo4RYhHGikdBsI7ivFSomk%2BXM4AEYbRe5UjaIvlEVkY602jH4%2BzS4lkpTko4ANwk7mwcKhEPBgrsqCOwXo0yO7guOy2plx5mAONMqkZAEkSW8RABpCBu9AETDzcEbADsACEqXyDUbSOHI9HcZKCM7KTsy%2B8MeZAeqc0Y7IwjhATlstmxSEpXgBObY%2BLTkPEptOZ7ZbI4gQdbU4AEVIS/7Dab8yOBZbhbbRwtrIge4pR9bHyLG5PH3e0Nhp7NSJRaIxIhxWodOqbU1EVpFh6SON4gQIYA2Q1TkjgNN1mHdT0uRIWCuXtQkZSbJlsErYVSBNN9EUSep2AREtgjNDdrSvF14Uwx1dRdH0/XeAMCCDEMwxzPMjnMFQiETed0yzCNhCjGMjjo8kC3eYtb3bUxO3dNwhQYLkByXNglG3LRxyTVNRNnKctB3EBVyXbch13FF90PU5b2LB95OfCEjkE6TtRlOV6LQrQjmMTAzxZUgmC87A3XsdwGQbLDiV88kiIgupA2DJhQxCGsJLrI5CCUFgIkYUwLmiUh3ESIglAgOS70UjFxKIST6xRRKTh2Gi4wTQyFyzG9H2taE8sNQq%2Bwkeh0oI8rKoZX0qXhATupEzMIC0frXLea1zzCryCESxzC3vR8OK4jKeNzYK%2BWGMhsBK4QyoqqqIEa5rdsS2qFPm/jTEE4SjJWht3oO%2BSjo2z4cDoLKNwAdQAFVMEt01wW1bVW9RcC0TGeExrQFACA8UcJomMWhMAwH%2BI4Ydho4S0BLk3kwTArtmtiPl2CGwyp4mUagKnTFhm4uVtKECYxPnEZEZGbVF0nybOSm4e5IQ6GAFZ1KEDyiHK2wjjmjbrluCHNe1ohDvq7t1L7F7cucWHGQCRzucJ2WKYAeRoGhBNXLR1Bx/tGQhaGCdcYRVQm/VDQI3XWfhC3ey5a2pOcABNZxnFCWHHadwaoTJt2Pa9o4fcx6R/cD4OrgOLlXHcYRcPw6Jo8Omi%2BYF25gafFL1NOzKE9rKS%2BRYTBTGhjd4ZTtOM5qs2vu28KqYAWmeCf08z48QZczvwQhdyiAAd3QI499IFhEiV2uqo8w%2BOSFI%2BuXEcKYq5eh0FQclGCBJXCBMZKd6Pk/EimAwBfUwAAvIQ3ofAljMj4GihpwwqGCgQdEo9x6p1Xu1ZwRd1CxBxqYHGZk3L/1PkA5YwgwEQLNCWHg1FgwMifkglBY9TArwzu1F4HVsG4MxvgnG293J7wlNEewhBxD0HoMSCqmAhSoTgsfU%2B586431xPApg6BcRRGCMABB2BMDgmwOoAi4U44aWyk1XK8jAHAPIeAkIVD1ywI7vowx0RH791xJY0hF9HZ/wqrcasUYFTEAZKKdCRwABspdFS4msQRZK/pUqcXSr3MxkSB7YCHqYNJqBK6oGENPEG9V4R/zUcKVcHJ4EYGODQUC9AGTILvprcasobqwWMEcHYHl2hCEwAyKI%2B9sBIUHt/bRqCWHoLYUFEKQ8tFhM8bEr6f8GkVPqfUtgyQGjGA8ffZY9BgqoCQMYeMYSJDACwkgdUZBhoFSKlyUqpBppVWStaROMS66bmmcPMZrDYbT3XptDEFwji2K5B8iAeTcTbHCVwghOMDynAAGJHB4CuIuHcXnuOIVYshkLOHzJxRQuxUCQXUXRQ1C4aTcRXXcPQXFNF8XeP%2BVtO4NLcXOH%2BB8qMZL4TUtpScIs7LOEQp8e5UpoKBUfJUNgHSGpD7xhCCfYUHlQzACQEQCRCocRRyqbI9p2wywdVnqFcKvLTb/Oco%2BfhHl2CJD7KEsRE1QnDKydIIBuThDgl4sFQQ6ACl1SNZk7J7qiB/KcqDTuJ1JpHCMMMQloLOFmXNUQvenIQiIWrDk1AELuSqiBHcQKwKIGzikTIuZADFG4lCZqSUkVooMhBb/Xeh8WBeoLSfNYSsmCEGqOFQReUhBgC4LiEtSrq18iissBkliK1WovgwRp9QuQAEcIxKpYEElRwhr5wRCHvQtIRG1WrFeUpA8C2AxQ0ffcaujAq4kYICXEkVojEnuJ0lQGAu0hPFKQPehouR9rYFBNVRxSnhkSNI0dcjy2xOVGQK1GSDn7olKqfxEjG2wXgTQWuHQNbAL2Zq7k17gpPtIMSFUgyGgpCOLDTiSVPUXVlEJOwHBTAjtumk6JpgGXCGqh9UsX1XncneZw513yJm/PWgC4pAjQLDuEPOvti7Ar0DYCkXEkNDScmCr%2BjkGH60QPuIZxZu9T2xXHfyBk66X57rikxDkwzlSqnVPGS4sz10RCHsZrFFajhPvCkqJUSg96nxCe4Dxp6%2BwOtmc6oNWaPWnmk4FLt/a7hMEHbiFzHlguJENgqFgb9YMBK6R%2B3pecEvBnRFAbNULsFFlhZjA8ecE3qDq/V2qhM55OPK3/bNqj0B7p9q12FhnBsezGx7I4VyfY0HqwFQz2Dxvja%2BkCkF7UaLgveVsaFPtuGwvhTsJFKLvZdak0cP%2BSnFMqbU7esJThJvonXbupDiKvMNKUOgYt4HS2amnTBqtcEa0Tpikh55gLKuraazRGNQkQXtYxJ9crg13LpuQcSZRYTI0ehRKNLkhylBpaHZqpCGAHmPV0aD20gJgQhuh3GzBRwtiScJtxmHlCoG05JbAwcBqaI0KZSjDnEC1vPf5/CC13WTPwNOec9UVPPD5qF/ZjJ%2BzxpToAV48hbGbv3uGMi6FzH80keJNO/xIiKdnfctge4wB7gVbCabrAdCjixGKz0qdXIQg3rQg6vT2uMm66N0ce4PhXefvN3/JQ7hgDxl132n3KzfPprR7R8PMcSbuViY3bO0aH106Q1ijXpsoEwPperhZUCaGOIl9g2bWgRvqHCYtibNoArWgXijVvTtO/MstGFNZHBCAyLQ15wbs369Fib%2BiFv0/bTd%2BJnP%2BEJKOqC6yn2zAh9Smnp/iPnBY/7g%2BzRE32fNp2/H%2BzgvkKveYJ7QHwQIfVxU8Lb39NyfZ%2BMQorf9zC/S%2BNwos3M8AvWJONKhEvfVMA6va0FnYA4vUlLYMsOArza0CAK3G3BdPkEDDJFEEwW7ZLUJVbBIQzR/bmTLYZMjK5VbAKNCKIQKDoRCOHRLabZ/dQSfKfa0D/d/FGdgr/FGH/TcFfLkdfEDdRLfYAHfWvevRvI/G0Lgo4U/aQ7PC/OefvXRO/YUYfavUfercfV/eQtgzghQngyhDqAKf/HYZGKAznahWA%2BAopC3G0CwtnKwxxBAhAiAjEZA63W3RTdAkIeoWZTUO7PAoXGcQgxA4mEg5XMg6sfA2VAjfLOwOgnfIbAhbQqQvQ60C/GQomLIkXDcfgxpQQzfLRR/HbJgw/RbT/WQ/Q7OHIpQ4MNgW/e/eJcNLgRYegbgHwfgbwLgHQcgdAbgLBd7FYDtT4PgcgIgbQVoxYckEAHYcJe4LYLQIsHYaQccLQeIOY6QWIHYQwbgaQfgNgEAcJWIe4IsWILYnYbY6cLQHwHwIsPwbo3o/orgfgJQEAGcCYno1o8gOAWAFAZYIgRIMLSgRwbtQkTAAwQQYQMQCQTgGQOQYQZQNQTQL48gfQBISxbgPgNojoroyYvo7gV2MLIEytdEN4RkEkeCLNZFHgBY%2B4FFCANREIA8CAVwdZDSasUY%2BYfgT4nQeYRYTkIeAYGqXYrgfY8gQ4nYWkrYHgHwHgcJeU1Y8caQaQLYO48gR4/gZ414948YyY/k8gGYnwLYe4WIHwLY8cM440rYM4440UnYPE1E7UvUr4xYX45ANANURIDkkE8FL0jktAeoNgHgHgGU8gHAKKNYAANVRD3ldluG6LGLoFpWiDeIgAiHxMVFYFIyxP4AwAaP5FdgyiuHxJwDlBMEkFRMID5FoLuHxIMXaDC3WDGNKnaNRNDGRHqCuFcBwFzPGNalzMWBoCMGACUBjOwDjITL7KhNEDEThNkBnKRI0HxPRMMCORAHMCAUMAIAiDeMgEWHQEqnSDeK4AXmhmaQXldi2FeMqHaB7UcC7TGG8ASECGmGKFKAMGSFSB7SfM/LyB7V6HfIGCGFvJwyYC6FGDcGaAMDaDAogqmCKH6BiCGEmF/JQu6EAqQqkEWCGNWDhJxK4E6I1PxOePAgpKpKDBDLpIZKZOwBZPwGICuS5J5P1OmJAGkCLHpOkDlK0BlI4qLBlKjFFPFM1IJJeKsF1N5KmNFOvOIqdO4BYtdMWHTVSAcGkCAA%3D)

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
        ldr     r2, .L11        // r2 = address of DWT (0xe0001000)
        ldr     r3, .L11+4      // r3 = address of wrap_count_zone
        ldr     r0, [r2, #4]    // r0 = read DWT->CYCCNT
        lsrs    r2, r0, #31     // r2 = zone (top bit of DWT->CYCCNT)
        ldr     r1, [r3, r2, lsl #2] // r1 = wrap_count_zone[zone]
        bx      lr              // 64 bit result returned in r0/r1
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

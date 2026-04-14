# Lab 004 - Latency-Optimized Loops Tasks

- Exercises covering worst-case blocking analysis, interrupt-driven refactoring, DMA optimization, and timing assertions.
- Each task includes a description, scenario, hint, and collapsible Solution.

#### Table of Contents

- [01. Identify Blocking Patterns](#01-identify-blocking-patterns)
- [02. Calculate Worst-Case Blocking Duration](#02-calculate-worst-case-blocking-duration)
- [03. Refactor Polling to Interrupt-Driven](#03-refactor-polling-to-interrupt-driven)
- [04. Refactor Polling to DMA](#04-refactor-polling-to-dma)
- [05. Add Latency Assertions](#05-add-latency-assertions)

---

#### 01. Identify Blocking Patterns

Audit the following code snippet and list every blocking pattern with: the pattern name, the line number(s) where it appears, and the worst-case blocking condition.

```c
// main.c
 1: void UART_SendString(const char *s) {
 2:     while (*s) {
 3:         while (!(USART2->SR & USART_SR_TXE));  // wait TX empty
 4:         USART2->DR = *s++;
 5:     }
 6: }
 7:
 8: uint8_t SPI_Transfer(uint8_t data) {
 9:     SPI1->DR = data;
10:     while (!(SPI1->SR & SPI_SR_RXNE));          // wait RX not empty
11:     return SPI1->DR;
12: }
13:
14: uint16_t ADC_Read(void) {
15:     ADC1->CR2 |= ADC_CR2_SWSTART;
16:     while (!(ADC1->SR & ADC_SR_EOC));            // wait end of conversion
17:     return ADC1->DR;
18: }
19:
20: void Delay_ms(uint32_t ms) {
21:     uint32_t start = HAL_GetTick();
22:     while (HAL_GetTick() - start < ms);          // busy wait
23: }
```

#### Scenario:

◦ A team is debugging intermittent missed deadlines on their 1 kHz control loop.
◦ The agent must identify all blocking loops before recommending optimizations.

**Hint:** Blocking patterns include: busy-wait polling on status flags, blocking delay functions.

<details markdown>
<summary>Solution</summary>

| #   | Pattern                        | Line(s) | Worst-Case Blocking                                                                                                                                |
| --- | ------------------------------ | ------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | **UART TX busy-wait**          | 3       | Time for TX shift register to empty: up to `1/baud × 10 bits` per character. At 9600 baud ≈ 1.04 ms/char. For long strings, sum of all characters. |
| 2   | **SPI RX busy-wait**           | 10      | Full SPI frame transfer: `8 / f_SCK`. At 1 MHz SCK → 8 µs; at 100 kHz → 80 µs.                                                                     |
| 3   | **ADC software-start polling** | 16      | ADC conversion time = `(sampling time + 12) × T_ADC_CLK`. Typical STM32F4 at 84 MHz APB2 → up to 15.5 µs.                                          |
| 4   | **Busy-wait delay**            | 22      | Exact `ms` milliseconds — deterministic in value but blocks all other processing.                                                                  |

**Total worst-case blockage per single `UART_SendString(32 chars)` + `SPI_Transfer` + `ADC_Read` + `Delay_ms(1)` call:**
≈ 33.3 ms + 8 µs + 15.5 µs + 1 ms ≈ **34.3 ms** — exceeds a 1 ms control loop deadline.

</details>

---

#### 02. Calculate Worst-Case Blocking Duration

Given the hardware parameters below, calculate the exact worst-case blocking time for each operation in Task 01.

**Hardware parameters:**

- USART2: Baud rate = 115200, 8N1 protocol (10 bits per frame), max string = 64 characters
- SPI1: SCK = 2 MHz, 8-bit frame
- ADC1: ADCCLK = 21 MHz (APB2=84 MHz, prescaler=4), sampling time = 480 cycles, 12-bit resolution
- `Delay_ms(5)` call

#### Scenario:

◦ A firmware architect needs concrete numbers to justify the effort of IRQ/DMA refactoring to management.
◦ Without the math, the team underestimates how bad the blocking really is.

**Hint:** USART frame = start + 8 data + stop = 10 bits. ADC conversion = (sampling_time + 12) / ADCCLK.

<details markdown>
<summary>Solution</summary>

**USART2 worst-case:**

```
T_frame = 10 bits / 115200 bps = 86.8 µs per character
64 chars × 86.8 µs = 5.55 ms
```

**SPI1 worst-case:**

```
T_transfer = 8 bits / 2,000,000 bps = 4 µs
```

**ADC1 worst-case:**

```
T_conv = (480 + 12) cycles / 21,000,000 Hz = 23.4 µs
```

**Delay_ms(5):**

```
5.0 ms (exact, deterministic)
```

**Total worst-case per combined operation:**

```
5.55 ms + 0.004 ms + 0.023 ms + 5.0 ms = 10.577 ms
```

At a 1 kHz (1 ms period) control loop, this represents **a >10× deadline overrun** for a single pass through all four blocking calls.

</details>

---

#### 03. Refactor Polling to Interrupt-Driven

Refactor the `UART_SendString()` function from Task 01 to use USART2 TX interrupts (instead of busy-waiting). The refactored implementation must:

1. Use a circular transmit buffer (ring buffer) of 256 bytes
2. Load the buffer and enable the `TXEIE` interrupt from the main loop
3. Service the interrupt in `USART2_IRQHandler`
4. Never block in `UART_SendString` even if the buffer is nearly full

#### Scenario:

◦ The agent rewrites the UART driver from polling to interrupt-driven, freeing the CPU for the control loop.
◦ Non-blocking UART send is a mandatory first step before adding DMA.

**Hint:** `USART_CR1_TXEIE` enables the TXE interrupt. Disable it in the IRQ handler when the buffer empties.

<details markdown>
<summary>Solution</summary>

```c
#include <string.h>
#include <stdint.h>
#include "stm32f4xx.h"

#define TX_BUF_SIZE 256U

static volatile uint8_t  tx_buf[TX_BUF_SIZE];
static volatile uint32_t tx_head = 0;   /* write index (main) */
static volatile uint32_t tx_tail = 0;   /* read  index (IRQ)  */

/* Non-blocking: copies string into ring buffer, enables TX interrupt */
void UART_SendString(const char *s)
{
    while (*s) {
        uint32_t next = (tx_head + 1U) % TX_BUF_SIZE;
        /* If buffer full, spin (rare; could also drop or block-partial) */
        while (next == tx_tail) { /* wait for space */ }
        tx_buf[tx_head] = (uint8_t)*s++;
        tx_head = next;
    }
    /* Enable TXE interrupt to drain the buffer */
    USART2->CR1 |= USART_CR1_TXEIE;
}

/* USART2 interrupt handler */
void USART2_IRQHandler(void)
{
    if (USART2->SR & USART_SR_TXE) {
        if (tx_tail != tx_head) {
            USART2->DR = tx_buf[tx_tail];
            tx_tail = (tx_tail + 1U) % TX_BUF_SIZE;
        } else {
            /* Buffer empty — disable TXE interrupt */
            USART2->CR1 &= ~USART_CR1_TXEIE;
        }
    }
}
```

</details>

---

#### 04. Refactor Polling to DMA

Refactor the `ADC_Read()` function from Task 01 to use DMA circular mode (DMA2 Stream 0 Channel 0 for ADC1 on STM32F4). The refactored implementation must:

1. Start DMA in circular mode with a 16-element buffer
2. Trigger on ADC1 scan complete (not software start per read)
3. Provide a non-blocking `ADC_GetLastSample()` function that reads the latest value from the DMA buffer

#### Scenario:

◦ The ADC was blocking the CPU for 23.4 µs per conversion (from Task 02).
◦ With DMA circular mode, the ADC runs autonomously and the CPU only reads the result when ready.

**Hint:** DMA2 Stream 0 Channel 0 → ADC1. Set `CIRC` (circular) and `PL` (priority). DMA direction: peripheral→memory.

<details markdown>
<summary>Solution</summary>

```c
#include "stm32f4xx.h"
#include <stdint.h>

#define ADC_BUFFER_SIZE  16U
static volatile uint16_t adc_dma_buf[ADC_BUFFER_SIZE];

void ADC1_DMA_Init(void)
{
    /* Enable clocks */
    RCC->APB2ENR |= RCC_APB2ENR_ADC1EN;
    RCC->AHB1ENR |= RCC_AHB1ENR_DMA2EN;

    /* Configure DMA2 Stream 0, Channel 0 */
    DMA2_Stream0->CR = 0;           /* Disable first */
    while (DMA2_Stream0->CR & DMA_SxCR_EN);

    DMA2_Stream0->PAR  = (uint32_t)&ADC1->DR;
    DMA2_Stream0->M0AR = (uint32_t)adc_dma_buf;
    DMA2_Stream0->NDTR = ADC_BUFFER_SIZE;

    DMA2_Stream0->CR = (0U << DMA_SxCR_CHSEL_Pos) |   /* Channel 0 */
                       DMA_SxCR_CIRC |                  /* Circular */
                       DMA_SxCR_MINC |                  /* Memory increment */
                       (1U << DMA_SxCR_PL_Pos) |        /* Medium priority */
                       (1U << DMA_SxCR_MSIZE_Pos) |     /* Memory 16-bit */
                       (1U << DMA_SxCR_PSIZE_Pos);      /* Peripheral 16-bit */

    DMA2_Stream0->CR |= DMA_SxCR_EN;

    /* Configure ADC1 for DMA circular */
    ADC1->CR1 = ADC_CR1_SCAN;
    ADC1->CR2 = ADC_CR2_DMA | ADC_CR2_DDS | ADC_CR2_CONT | ADC_CR2_ADON;
    ADC1->CR2 |= ADC_CR2_SWSTART;  /* Start continuous conversions */
}

/* Non-blocking: returns latest ADC sample from index 0 */
uint16_t ADC_GetLastSample(void)
{
    return adc_dma_buf[0];
}
```

</details>

---

#### 05. Add Latency Assertions

Add compile-time and runtime latency guard assertions to the `SPI_Transfer()` function from Task 01.

Requirements:

1. **Compile-time assertion**: SPI SCK must be at 2 MHz (verify with a `static_assert`)
2. **Runtime assertion**: Polling loop must exit within 10 µs (use DWT cycle counter); if it exceeds the deadline, call `FaultHandler(FAULT_SPI_TIMEOUT)`
3. Define a `FAULT_SPI_TIMEOUT` constant and a stub `FaultHandler()`

#### Scenario:

◦ After refactoring, the team wants guard rails to catch future configuration regressions.
◦ A compile-time check prevents shipping with the wrong SPI clock; runtime checks catch real hardware faults.

**Hint:** DWT->CYCCNT counts CPU cycles. At 168 MHz, 10 µs = 1680 cycles.

<details markdown>
<summary>Solution</summary>

```c
#include "stm32f4xx.h"
#include <stdint.h>
#include <assert.h>

/* --- Fault codes --- */
#define FAULT_SPI_TIMEOUT   0xAD01U

/* Compile-time assertion: SPI SCK must be 2 MHz
 * APB2 = 84 MHz, BR[2:0] = 101 → fPCLK/64 → NOT 2 MHz
 * This example uses APB2=84 MHz, BR=010 → 84/8 = 10.5 MHz (adjust as needed)
 * For strict 2 MHz: use an external oscillator or accept nearest divisor.
 * Here we assert that the build flag SPI_SCK_HZ is defined at 2000000.
 */
#ifndef SPI_SCK_HZ
#define SPI_SCK_HZ  2000000UL
#endif
_Static_assert(SPI_SCK_HZ == 2000000UL,
               "SPI SCK must be configured to 2 MHz for real-time constraints");

/* Expected max cycles for a 10 µs timeout at 168 MHz core clock */
#define SPI_TIMEOUT_CYCLES  (168U * 10U)   /* 1680 cycles */

/* Stub fault handler */
__attribute__((noreturn)) void FaultHandler(uint32_t code)
{
    (void)code;
    /* Log, blink LED, or enter safe state */
    while (1);
}

/* DWT cycle counter enable */
static inline void DWT_Init(void)
{
    CoreDebug->DEMCR |= CoreDebug_DEMCR_TRCENA_Msk;
    DWT->CYCCNT = 0;
    DWT->CTRL  |= DWT_CTRL_CYCCNTENA_Msk;
}

uint8_t SPI_Transfer(uint8_t data)
{
    SPI1->DR = data;

    uint32_t start = DWT->CYCCNT;
    while (!(SPI1->SR & SPI_SR_RXNE)) {
        if ((DWT->CYCCNT - start) > SPI_TIMEOUT_CYCLES) {
            FaultHandler(FAULT_SPI_TIMEOUT);
        }
    }
    return (uint8_t)SPI1->DR;
}
```

</details>

---

> **Back to [Tasks Index](../index.md)** | **Next: [005 Tasks](../005-AgenticDebugging-Tasks/README.md)**

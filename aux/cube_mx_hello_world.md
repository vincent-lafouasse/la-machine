# STM32F407G-DISC1 — DAC Sine Oscillator + LEDs (CubeMX Config)

Target: DAC1 (PA4) outputting a sine wave via TIM6-triggered, circular DMA, at a 48 kHz sample rate. Plus GPIO access to the 4 onboard LEDs for signaling.

---

## 1. Start the project

- New project → **MCU Selector** → `STM32F407VGTx` (not the Board Selector, which would also wire up the onboard LEDs/button/audio codec pins you don't need pre-configured).

## 2. Pinout & Configuration — DAC

- **Analog > DAC** → enable **OUT1** (Channel1). PA4 is auto-assigned as DAC1 output, analog mode.

## 3. Pinout & Configuration — TIM6

- **Timers > TIM6** → Activated.
- Trigger Event Selection **(TRGO)** → **Update Event**.
- Parameter Settings (set after clock config, see step 6):
  - Prescaler = `0`
  - Counter Period (ARR) = `1749`

## 4. Pinout & Configuration — DAC trigger + DMA

- Back in the DAC1 panel: **OUT1 Trigger** → **Timer 6 Trigger Out event**.
- DAC1 **DMA Settings** tab → Add → `DAC_CH1`:
  - Mode = **Circular** (required — Normal fires once and stops)
  - Data Width = **Half Word**, both Peripheral and Memory

## 5. Pinout & Configuration — LEDs

- Set **PD12, PD13, PD14, PD15** to **GPIO_Output** (LD4 green, LD3 orange, LD5 red, LD6 blue respectively).
- Optional: right-click each pin → Enter User Label (e.g. `LD4_GREEN`) so CubeMX generates named macros in `main.h`.

## 6. Clock Configuration

- **RCC** → HSE = **Crystal/Ceramic Resonator** (DISC1 has an 8 MHz crystal on board).
- Clock Configuration tab:
  - Set **HSE input frequency = 8** (MHz) — CubeMX defaults to 25, which is wrong for this board.
  - **System Clock Mux** → select **PLLCLK** (not HSI — this is what unlocks the SYSCLK box).
  - **PLL Source Mux** → HSE (should already be set).
  - Main PLL: **M = /8**, **N = x336**, **P = /2** → SYSCLK = 8 ÷ 8 × 336 ÷ 2 = **168 MHz**.
  - **APB1 Prescaler = /4** → PCLK1 = 42 MHz (at the max), **APB1 Timer clocks = 84 MHz**.
  - **APB2 Prescaler = /2** → PCLK2 = 84 MHz (at the max).
  - If anything is still magenta, click **Resolve Clock Issues**, then re-verify SYSCLK=168 and Timer clocks=84 (auto-resolve can pick a different valid combination than intended).
- This 84 MHz APB1 timer clock is why TIM6's PSC/ARR (step 3) are `0` / `1749`: `84,000,000 / 1750 = 48,000 Hz` sample rate, exact.

## 7. Project Manager

- Choose toolchain/IDE (CubeIDE, or Makefile).
- Keep **HAL** driver selected (LL available later if you want thinner abstraction).
- **Generate Code**.

## 8. Code — `main.c` USER CODE sections

```c
/* USER CODE BEGIN PV */
#define SINE_N 480          // samples/period → 48000/480 = 100 Hz tone
uint16_t sine_table[SINE_N];
/* USER CODE END PV */
```

```c
/* USER CODE BEGIN 2 */
for (int i = 0; i < SINE_N; i++) {
    float phase = (2.0f * 3.14159265f * i) / SINE_N;
    sine_table[i] = (uint16_t)(((sinf(phase) + 1.0f) * 0.5f) * 4095.0f);
}

HAL_TIM_Base_Start(&htim6);
HAL_DAC_Start_DMA(&hdac, DAC_CHANNEL_1, (uint32_t*)sine_table, SINE_N, DAC_ALIGN_12B_R);

HAL_GPIO_WritePin(GPIOD, GPIO_PIN_12, GPIO_PIN_SET); // LD4 on: DAC is live
/* USER CODE END 2 */
```

Notes:
- `HAL_DAC_Start_DMA` takes a `uint32_t*` even though the buffer is `uint16_t` — that's just the HAL signature; actual transfer width comes from the Half Word setting in step 4.
- Needs `-lm` linked (for `sinf`).
- `while(1)` body can stay empty — everything past setup is hardware-driven (TIM6 → DMA → DAC).

## Known limitation / next step

A fixed table played at a fixed DMA rate only produces **one frequency** (`fs / N`). If you want a tunable oscillator rather than one fixed test tone, that needs a phase-accumulator (DDS) approach instead of a static circular table — worth revisiting once this baseline is working.

---
layout: post
title: STM32F411 Discovery — Circular LED Chaser with Speed Ramp
date: 2025-09-07 00:40:00
description: Bare-metal GPIO example — LEDs PD12..PD15 chasing forward and reverse with slow→fast→slow pattern
tags: formatting links
categories: sample-posts
thumbnail: assets/img/blogs/STM32/stm32f411ve board.jpeg
published: true
---

This post shows how to control 4x user LEDs on the STM32F411 Discovery board **(PD12 = <font color="green">GREEN</font>, PD13 = <font color="orange">ORANGE</font>, PD14 = <font color="red">RED</font>, PD15 = <font color="blue">BLUE</font>) in a **circular pattern**:  
- forward (clockwise),  
- reverse (counter-clockwise),  
- with speed variation (slow → fast → slow).  



<!-- STM32F411 Disco LED ring demo (PD12=GREEN, PD13=ORANGE, PD14=RED, PD15=BLUE) -->
<!-- Works in Markdown renderers that allow inline HTML/CSS (e.g., GitHub Pages). -->
<style>
  .stm32-leds { font-family: system-ui, sans-serif; margin: 1rem 0; }
  .stm32-leds .ring { position: relative; width: 160px; height: 160px; margin: 0.5rem auto; }
  .stm32-leds .led {
    position: absolute; width: 26px; height: 26px; border-radius: 50%;
    opacity: .25; box-shadow: 0 0 0 currentColor; transform: scale(1.0);
    animation: blink var(--period,2.8s) infinite ease-in-out;
    animation-delay: calc(var(--i,0) * var(--step,.55s));
  }
  /* positions (top, right, bottom, left) */
  .pos0 { top: 0;             left: calc(50% - 13px); }
  .pos1 { top: calc(50% - 13px); right: 0; }
  .pos2 { bottom: 0;          left: calc(50% - 13px); }
  .pos3 { top: calc(50% - 13px); left: 0; }

  /* colors per PD pin */
  .pd12 { color:#00c853; background:#00c853; } /* GREEN  */
  .pd13 { color:#ff9800; background:#ff9800; } /* ORANGE */
  .pd14 { color:#e53935; background:#e53935; } /* RED    */
  .pd15 { color:#1e88e5; background:#1e88e5; } /* BLUE   */

  /* animation = “highlight one LED at a time” with slow→fast→slow feel via easing */
  @keyframes blink {
    0%   { opacity:.25; box-shadow:0 0 0 currentColor; transform:scale(1.00); }
    10%  { opacity:1.00; box-shadow:0 0 16px currentColor; transform:scale(1.18); }
    25%  { opacity:.25; box-shadow:0 0 0 currentColor; transform:scale(1.00); }
    100% { opacity:.25; box-shadow:0 0 0 currentColor; transform:scale(1.00); }
  }

  /* forward vs reverse = re-map delays */
  .stm32-leds.forward .pd12 { --i:0; }  /* GREEN  → first  */
  .stm32-leds.forward .pd13 { --i:1; }  /* ORANGE → second */
  .stm32-leds.forward .pd14 { --i:2; }  /* RED    → third  */
  .stm32-leds.forward .pd15 { --i:3; }  /* BLUE   → fourth */


  .stm32-leds.reverse .pd12 { --i:0; }  /* GREEN  → first  */
  .stm32-leds.reverse .pd15 { --i:1; }  /* BLUE   → second */
  .stm32-leds.reverse .pd14 { --i:2; }  /* RED    → third  */
  .stm32-leds.reverse .pd13 { --i:3; }  /* ORANGE → fourth */

  .legend { text-align:center; font-size:.9rem; opacity:.8; }
</style>

<div style="display:flex; gap:160px; align-items:flex-start; flex-wrap:wrap;">
  <!-- Forward (clockwise) -->
  <div class="stm32-leds forward" style="--period:1.8s; --step:.55s;">
    <div class="ring">
      <div class="led pd12 pos0"></div>
      <div class="led pd13 pos1"></div>
      <div class="led pd14 pos2"></div>
      <div class="led pd15 pos3"></div>
    </div>
    <div class="legend">Forward (clockwise)</div>
  </div>

  <!-- Reverse (counter-clockwise) -->
  <div class="stm32-leds reverse" style="--period:1.8s; --step:.55s;">
    <div class="ring">
      <div class="led pd12 pos0"></div>
      <div class="led pd13 pos1"></div>
      <div class="led pd14 pos2"></div>
      <div class="led pd15 pos3"></div>
    </div>
    <div class="legend">Reverse (counter-clockwise)</div>
  </div>
</div>

<div class="col-sm mt-8 mt-md-0">
  <div style="max-width:420px;margin:0;">
    {% include figure.liquid path="assets/img/blogs/STM32/stm32f411ve board.jpeg" class="img-fluid rounded z-depth-1" %}
  </div>
</div>


We use only **register-level CMSIS**, no **HAL Hardware Abstraction Layer**.
**What is CMSIS?**  
CMSIS (Cortex Microcontroller Software Interface Standard) is a set of low-level, vendor-independent C headers and functions for ARM Cortex microcontrollers. It lets you control hardware directly using registers.

**What is HAL?**  
HAL (Hardware Abstraction Layer) is a higher-level library provided by chip vendors (like STMicroelectronics). It makes hardware control easier by hiding register details, but can be less efficient than CMSIS.

[Learn more about CMSIS in the official ARM documentation.](https://www.keil.arm.com/cmsis)



---

## Full code

```c
#include "stm32f4xx.h"

/* simple blocking delay */
static void delay(volatile uint32_t t){ while(t--); }

static const uint8_t pins[4] = {12,13,14,15};   // PD12..PD15

/* blink one full ramp cycle in given direction: +1=FWD, -1=REV */
static void run_cycle(int dir){
  uint32_t t_max = 800000;   // slow
  uint32_t t_min =  80000;   // fast
  uint32_t step   =  40000;  // ramp step

  int idx = (dir>0)?0:3;

  /* SLOW -> FAST */
  for(uint32_t t=t_max; t>=t_min; t-=step){
    GPIOD->BSRRL = (1u<<pins[idx]);   // ON
    delay(t);
    GPIOD->BSRRH = (1u<<pins[idx]);   // OFF

    if(dir>0){ idx++; if(idx==4) idx=0; }
    else     { if(idx==0) idx=3; else idx--; }
  }

  /* FAST -> SLOW */
  for(uint32_t t=t_min; t<=t_max; t+=step){
    GPIOD->BSRRL = (1u<<pins[idx]);   // ON
    delay(t);
    GPIOD->BSRRH = (1u<<pins[idx]);   // OFF

    if(dir>0){ idx++; if(idx==4) idx=0; }
    else     { if(idx==0) idx=3; else idx--; }
  }
}

int main(void){
  /* 1) Enable GPIOD clock */
  RCC->AHB1ENR |= RCC_AHB1ENR_GPIODEN;

  /* 2) Configure PD12..PD15 as outputs (MODER bits = 01) */
  GPIOD->MODER &= ~((3u<<(12*2))|(3u<<(13*2))|(3u<<(14*2))|(3u<<(15*2)));
  GPIOD->MODER |=  ((1u<<(12*2))|(1u<<(13*2))|(1u<<(14*2))|(1u<<(15*2)));

  while(1){
    run_cycle(+1);   // forward: PD12→PD13→PD14→PD15
    run_cycle(-1);   // reverse: PD15→PD14→PD13→PD12
  }
}
```

---

## Explanation with bitwise examples

### Enable clock for port D
```c
RCC->AHB1ENR |= RCC_AHB1ENR_GPIODEN;
```
- `RCC_AHB1ENR_GPIODEN = (1<<3)`
- If `R = 0000 0000` (binary low 8 bits),  
  after operation: `R = 0000 1000`.

### Configure PD12..PD15 as outputs
```c
GPIOD->MODER &= ~((3u<<(12*2))|(3u<<(13*2))|(3u<<(14*2))|(3u<<(15*2)));
GPIOD->MODER |=  ((1u<<(12*2))|(1u<<(13*2))|(1u<<(14*2))|(1u<<(15*2)));
```
- Each pin uses 2 bits in `MODER`: `00=input`, `01=output`.  
- Example for PD12: bits `[25:24]`.  
- Binary clear: `..00..` then set: `..01..`.

### Turn LED ON
```c
GPIOD->BSRRL = (1u<<12);
```
- `BSRRL` low half sets bits = 1.  
- If `ODR=0000 0000 0000`, after: `ODR=0001 0000 0000 0000`.

### Turn LED OFF
```c
GPIOD->BSRRH = (1u<<12);
```
- `BSRRH` high half clears bits = 1.  
- If `ODR=0001 0000 0000 0000`, after: `ODR=0000 0000 0000 0000`.

### Delay
```c
delay(t);
```
- Simple busy-loop, `t` controls speed.  
- In our loop: decreases (`t_max→t_min`) for faster, then increases for slower.

---

## Behavior
1. All LEDs chase in a circle: Green → Orange → Red → Blue.  
2. Speed ramps from **slow → fast → slow**.  
3. Direction reverses: Blue → Red → Orange → Green.  
4. Pattern repeats infinitely.

---

> ⚠️ Note: If your CMSIS headers define unified `BSRR`, replace:  
> - ON → `GPIOD->BSRR = (1u<<pin);`  
> - OFF → `GPIOD->BSRR = (1u<<(pin+16));`

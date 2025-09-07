---
layout: post
title: STM32F411 Discovery — Circular LED Chaser with Speed Ramp
date: 2025-09-07 00:40:00
description: Bare-metal GPIO example — LEDs PD12..PD15 chasing forward and reverse with slow→fast→slow pattern
tags: formatting links
categories: sample-posts
published: true
---

This post shows how to control the **four user LEDs** on the STM32F411 Discovery board (PD12 = Green, PD13 = Orange, PD14 = Red, PD15 = Blue) in a **circular pattern**:  
- forward (clockwise),  
- reverse (counter-clockwise),  
- with speed variation (slow → fast → slow).  

We use only **register-level CMSIS**, no HAL.

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

<!-- docs/rp2350/leds-alarms.md -->
# Parpadeo de 4 LEDs con **System Timer** (RP2350 / Pico 2)

!!! tip "Resumen"
    Este programa usa las 4 *hardware alarms* del temporizador de sistema para hacer parpadear 4 LEDs de manera independiente, programando cada próximo “deadline” en microsegundos dentro de cada ISR (*interrupt service routine*). Es un enfoque sin bloqueo: todo el parpadeo ocurre en las ISRs; el `while(true)` queda libre.

---

## Objetivo

- Controlar **4 LEDs** con periodos distintos usando **ALARM0..ALARM3** del periférico `timer`.
- Trabajar en **microsegundos** (timebase de 1 MHz).
- Mantener precisión temporal reprogramando el siguiente disparo desde el *deadline* previo (**evita drift**).

---

## Plataforma y dependencias

- **MCU:** RP2350 (Raspberry Pi Pico 2)  
- **SDK:** Raspberry Pi Pico SDK  
- **Headers:** `pico/stdlib.h`, `hardware/irq.h`, `hardware/structs/timer.h`, `hardware/gpio.h`

---

## Mapa de pines y alarmas

| Recurso | Definición | Uso en código |
|---|---|---|
| LED0 | `LED0_PIN = 12` | Parpadeo con **ALARM0** |
| LED1 | `LED1_PIN = 13` | Parpadeo con **ALARM1** |
| LED2 | `LED2_PIN = 14` | Parpadeo con **ALARM2** |
| LED3 | `LED3_PIN = 15` | Parpadeo con **ALARM3** |
| ALARM0 | `ALARM0_NUM = 0` | IRQ `ALARM0_IRQ` → `on_alarm0_irq()` |
| ALARM1 | `ALARM1_NUM = 1` | IRQ `ALARM1_IRQ` → `on_alarm1_irq()` |
| ALARM2 | `ALARM2_NUM = 2` | IRQ `ALARM2_IRQ` → `on_alarm2_irq()` |
| ALARM3 | `ALARM3_NUM = 3` | IRQ `ALARM3_IRQ` → `on_alarm3_irq()` |

> Nota: Asegura que los comentarios de pines coinciden con los `#define` usados.

---

## Periodos y frecuencias

Los intervalos están en **µs**:

- `INTERVALO0_US = 250000` → 0.25 s → **4 Hz**
- `INTERVALO1_US = 500000` → 0.50 s → **2 Hz**
- `INTERVALO2_US = 750000` → 0.75 s → **1.33 Hz**
- `INTERVALO3_US = 1000000` → 1.00 s → **1 Hz**

Cambia estas constantes para ajustar la velocidad de cada LED.

---

## Flujo de ejecución

1. **Inicialización de GPIOs**: `gpio_init()`, `gpio_set_dir(..., GPIO_OUT)` y `gpio_put(..., 0)` por LED.  
2. **Base de tiempo**: `timer_hw->source = 0u;` (1 MHz → 1 µs por tick).  
3. **Captura “ahora”**: `now_us = timer_hw->timerawl;` (32 bits bajos).  
4. **Deadlines iniciales**: `nextX_us = now_us + INTERVALOX_US` (X=0..3).  
5. **Programación de alarmas**: `timer_hw->alarm[ALARMX_NUM] = nextX_us`.  
6. **ISRs registradas** y **IRQs habilitadas**.  
7. **Bucle principal**: `tight_loop_contents();` (sin trabajo, todo ocurre en ISRs).

---

## Rutinas de interrupción (ISRs)

Cada ISR:

- Limpia su *flag*: `hw_clear_bits(&timer_hw->intr, 1u << ALARMX_NUM);`
- Conmuta el pin: `sio_hw->gpio_togl = 1u << LEDX_PIN;`
- Reagenda **desde el deadline**: `nextX_us += INTERVALOX_US;`
- Reprograma su alarma: `timer_hw->alarm[ALARMX_NUM] = nextX_us;`

Reagendar desde el *deadline* previo evita el *drift* que ocurriría si se usara el “tiempo actual”.

---

## Consideraciones de temporización

- **Rollover 32 bit**: `timerawl`/`nextX_us` envuelven cada ≈ 71.58 min (2^32 µs). El esquema por *deadlines* lo maneja bien si todo se mantiene en 32 bits.  
- **SIO toggle**: Requiere GPIO en función SIO (por defecto tras `gpio_init()`). Es atómico y rápido.  
- **ISRs ligeras**: Evita trabajo pesado dentro de interrupciones.

---

## codigo

```c


#include "pico/stdlib.h"
#include "hardware/irq.h"
#include "hardware/structs/timer.h"
#include "hardware/gpio.h"

#define LED0_PIN     12   // LED integrado
#define LED1_PIN     13                      // LED externo en GPIO 0
#define LED2_PIN     14   // LED integrado
#define LED3_PIN     15


#define ALARM0_NUM   0
#define ALARM1_NUM   1
#define ALARM2_NUM   2
#define ALARM3_NUM   3


#define ALARM0_IRQ   timer_hardware_alarm_get_irq_num(timer_hw, ALARM0_NUM)
#define ALARM1_IRQ   timer_hardware_alarm_get_irq_num(timer_hw, ALARM1_NUM)
#define ALARM2_IRQ   timer_hardware_alarm_get_irq_num(timer_hw, ALARM2_NUM)
#define ALARM3_IRQ   timer_hardware_alarm_get_irq_num(timer_hw, ALARM3_NUM)

// Próximos "deadlines" (32 bits bajos en µs) y sus intervalos en µs
static volatile uint32_t next0_us, next1_us,next2_us,next3_us;
static const uint32_t INTERVALO0_US = 250000u;
static const uint32_t INTERVALO1_US = 500000u;
static const uint32_t INTERVALO2_US = 750000u;
static const uint32_t INTERVALO3_US = 1000000u;

// ISR para ALARM0
static void on_alarm0_irq(void) {
    hw_clear_bits(&timer_hw->intr, 1u << ALARM0_NUM);
    sio_hw->gpio_togl = 1u << LED0_PIN;
    next0_us += INTERVALO0_US;
    timer_hw->alarm[ALARM0_NUM] = next0_us;
}

// ISR para ALARM1
static void on_alarm1_irq(void) {
    hw_clear_bits(&timer_hw->intr, 1u << ALARM1_NUM);
    sio_hw->gpio_togl = 1u << LED1_PIN;
    next1_us += INTERVALO1_US;
    timer_hw->alarm[ALARM1_NUM] = next1_us;
}
static void on_alarm2_irq(void) {
    hw_clear_bits(&timer_hw->intr, 1u << ALARM2_NUM);
    sio_hw->gpio_togl = 1u << LED2_PIN;
    next2_us += INTERVALO2_US;
    timer_hw->alarm[ALARM2_NUM] = next2_us;
}
static void on_alarm3_irq(void) {
    hw_clear_bits(&timer_hw->intr, 1u << ALARM3_NUM);
    sio_hw->gpio_togl = 1u << LED3_PIN;
    next3_us += INTERVALO3_US;
    timer_hw->alarm[ALARM3_NUM] = next3_us;
}




int main() {

    gpio_init(LED0_PIN);
    gpio_set_dir(LED0_PIN, GPIO_OUT);
    gpio_put(LED0_PIN, 0);

    gpio_init(LED1_PIN);
    gpio_set_dir(LED1_PIN, GPIO_OUT);
    gpio_put(LED1_PIN, 0);

    
    gpio_init(LED2_PIN);
    gpio_set_dir(LED2_PIN, GPIO_OUT);
    gpio_put(LED2_PIN, 0);

    
    gpio_init(LED3_PIN);
    gpio_set_dir(LED3_PIN, GPIO_OUT);
    gpio_put(LED3_PIN, 0);

    // Timer de sistema en microsegundos (por defecto source = 0)
    timer_hw->source = 0u;

    uint32_t now_us = timer_hw->timerawl;

    // Primeros deadlines
    next0_us = now_us + INTERVALO0_US;
    next1_us = now_us + INTERVALO1_US;
    next2_us = now_us + INTERVALO2_US;
    next3_us = now_us + INTERVALO3_US;

    // Programa ambas alarmas
    timer_hw->alarm[ALARM0_NUM] = next0_us;
    timer_hw->alarm[ALARM1_NUM] = next1_us;
    timer_hw->alarm[ALARM0_NUM] = next2_us;
    timer_hw->alarm[ALARM1_NUM] = next3_us;

    // Limpia flags pendientes antes de habilitar
    hw_clear_bits(&timer_hw->intr, (1u << ALARM0_NUM) | (1u << ALARM1_NUM) | (1u << ALARM2_NUM) | (1u << ALARM3_NUM));

    // Registra handlers exclusivos para cada alarma
    irq_set_exclusive_handler(ALARM0_IRQ, on_alarm0_irq);
    irq_set_exclusive_handler(ALARM1_IRQ, on_alarm1_irq);
    irq_set_exclusive_handler(ALARM2_IRQ, on_alarm2_irq);
    irq_set_exclusive_handler(ALARM3_IRQ, on_alarm3_irq);


    // Habilita fuentes de interrupción en el periférico TIMER
    hw_set_bits(&timer_hw->inte, (1u << ALARM0_NUM) | (1u << ALARM1_NUM) | (1u << ALARM2_NUM) | (1u << ALARM3_NUM));

    // Habilita ambas IRQ en el NVIC
    irq_set_enabled(ALARM0_IRQ, true);
    irq_set_enabled(ALARM1_IRQ, true);
    irq_set_enabled(ALARM2_IRQ, true);
    irq_set_enabled(ALARM3_IRQ, true);

    // Bucle principal: todo el parpadeo ocurre en las ISRs
    while (true) {
        tight_loop_contents();
    }
}

```









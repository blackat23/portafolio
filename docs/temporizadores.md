# Parpadeo de 4 LEDs con múltiples alarmas del **System Timer** (RP2350 / Pico 2)

> **TL;DR:** Este programa usa las 4 *hardware alarms* del temporizador de sistema para hacer parpadear 4 LEDs de manera independiente, programando cada próximo “deadline” en microsegundos dentro de sus rutinas de interrupción (ISRs). Es un enfoque sin bloqueo: todo el parpadeo ocurre en las ISRs; el `while(true)` queda libre.

---

## Objetivo

- Controlar **4 LEDs** con periodos distintos usando **ALARM0..ALARM3** del periférico `timer`.
- Trabajar en **microsegundos** usando el **timebase a 1 MHz**.
- Mantener precisión temporal reprogramando el *siguiente* instante de disparo **relativo** (sumando el intervalo al último `deadline`), evitando *drift* por latencias.

---

## Plataforma y dependencias

- **MCU:** RP2350 (Raspberry Pi Pico 2)  
- **SDK:** Raspberry Pi Pico SDK (headers: `pico/stdlib.h`, `hardware/irq.h`, `hardware/structs/timer.h`, `hardware/gpio.h`)
- **Build:** CMake + GCC (estándar del SDK)

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

> **Nota:** En el comentario de `LED1_PIN` aparece “GPIO 0”, pero el `#define` es **13**. Ajusta el comentario o el valor para que coincidan.

---

## Periodos y frecuencias

Los intervalos están en **µs**:

- `INTERVALO0_US = 250000` → 0.25 s → **4 Hz**
- `INTERVALO1_US = 500000` → 0.50 s → **2 Hz**
- `INTERVALO2_US = 750000` → 0.75 s → **1.33̅ Hz**
- `INTERVALO3_US = 1000000` → 1.00 s → **1 Hz**

Cambia estos `#define` para ajustar la velocidad de cada LED de forma independiente.

---

## Flujo de ejecución

1. **Inicialización de GPIOs:**  
   `gpio_init()`, `gpio_set_dir(..., GPIO_OUT)` y `gpio_put(..., 0)` para cada LED.

2. **Base de tiempo a 1 MHz:**  
   `timer_hw->source = 0u;` selecciona el *source* por defecto (1 MHz → 1 µs por tick).

3. **Toma de “ahora” y primeros deadlines:**  
   `now_us = timer_hw->timerawl;` (parte baja a 32 bit).  
   Se calculan `nextX_us = now_us + INTERVALOX_US` para X = 0..3.

4. **Programación de alarmas:**  
   Se escribe `timer_hw->alarm[ALARMX_NUM] = nextX_us` para cada alarma.

5. **Registro de handlers e IRQs:**  
   `irq_set_exclusive_handler(ALARMX_IRQ, on_alarmX_irq);` y `irq_set_enabled(..., true)`.

6. **Bucle principal vacío:**  
   `tight_loop_contents();` mientras las ISRs manejan todos los toggles.

---

## Rutinas de interrupción (ISRs)

Cada ISR:

- Limpia su *flag* con `hw_clear_bits(&timer_hw->intr, 1u << ALARMX_NUM);`
- Conmuta el pin con **registro de toggle**: `sio_hw->gpio_togl = 1u << LEDX_PIN;`
- Agenda el **próximo deadline** sumando el intervalo: `nextX_us += INTERVALOX_US;`
- Reprograma la alarma: `timer_hw->alarm[ALARMX_NUM] = nextX_us;`

Este patrón es robusto porque evita que la latencia de ISR se acumule (no se usa “now” al reprogramar; se usa el *deadline* previo + intervalo).

---

## Compilación y carga (ejemplo)

1. En tu `CMakeLists.txt`, vincula contra `pico_stdlib` y habilita stdio (si hace falta).
2. `mkdir build && cd build`
3. `cmake ..`
4. `make -j`
5. Arrastra el `.uf2` al volumen del Pico en modo BOOTSEL.

---

## Consideraciones de temporización

- **Rollover de 32 bit:** `timerawl` y `nextX_us` son de 32 bit (µs). El *wrap* ocurre cada 2^32 µs ≈ **71.58 min**.  
  El esquema “deadline += intervalo” maneja bien el wrap si **alarm** y **nextX_us** permanecen en 32 bit (no mezclar con tiempos de 64 bit).  
  Si necesitas periodos por encima de ~1 h o marcar timestamps, considera `time_us_64()`.

- **SIO toggle:** `sio_hw->gpio_togl` requiere que el GPIO esté en función SIO (lo deja así `gpio_init()` por defecto). Es rápido y atómico.

- **Carga de ISR:** Evita trabajo pesado dentro de la ISR; aquí solo se hace toggle y reprogramación.

---

## Problemas/bugs detectados en el listado original

1. **Programación de alarmas 2 y 3 (error de índices):**  
   En la sección “Programa ambas alarmas” se reescriben índices de 0 y 1 en lugar de 2 y 3.  
   **Original:**
   ```c
   timer_hw->alarm[ALARM0_NUM] = next0_us;
   timer_hw->alarm[ALARM1_NUM] = next1_us;
   timer_hw->alarm[ALARM0_NUM] = next2_us;  // <-- debería ser ALARM2_NUM
   timer_hw->alarm[ALARM1_NUM] = next3_us;  // <-- debería ser ALARM3_NUM
   ```
   **Corrección:**
   ```c
   timer_hw->alarm[ALARM0_NUM] = next0_us;
   timer_hw->alarm[ALARM1_NUM] = next1_us;
   timer_hw->alarm[ALARM2_NUM] = next2_us;
   timer_hw->alarm[ALARM3_NUM] = next3_us;
   ```

2. **Comentario de pin no coincide:**  
   `LED1_PIN` está definido como **13**, pero el comentario dice “GPIO 0”. Corrige uno de los dos.

---

## Cómo ajustar velocidades

1. Modifica las constantes `INTERVALO*_US`.  
   Ejemplo para **LED2 a 5 Hz**: periodo = 1/5 s = 0.2 s = **200000 µs**  
   ```c
   #define INTERVALO2_US 200000u
   ```
2. Compila y carga de nuevo.

---

## Extensiones sugeridas

- **Duty cycle** con *PWM* en lugar de toggles si buscas brillo variable.
- **Sincronización** (p. ej., que uno sea divisor del otro) usando múltiplos de un base-interval.
- **Callback compartido** para varias alarmas con una tabla de contexto, reduciendo código repetido.
- **Protección contra rebotes/colisiones** si más lógica usa SIO toggle sobre los mismos pines.

---

## Fragmento compacto de inicialización (con correcciones aplicadas)

```c
// Base de tiempo y primeros deadlines
timer_hw->source = 0u; // 1 MHz
uint32_t now_us = timer_hw->timerawl;

next0_us = now_us + INTERVALO0_US;
next1_us = now_us + INTERVALO1_US;
next2_us = now_us + INTERVALO2_US;
next3_us = now_us + INTERVALO3_US;

// Programa cada alarma en su índice correcto
timer_hw->alarm[ALARM0_NUM] = next0_us;
timer_hw->alarm[ALARM1_NUM] = next1_us;
timer_hw->alarm[ALARM2_NUM] = next2_us;
timer_hw->alarm[ALARM3_NUM] = next3_us;
```

---

## Resumen

- 4 LEDs, 4 *hardware alarms*, temporización en µs, sin bloqueo.  
- ISR minimalista: limpia flag, *toggle*, reprograma deadline.  
- Evita *drift* sumando el intervalo al *deadline* previo.  
- **Corrige los índices** de `alarm[2]` y `alarm[3]` y alinea comentarios de pines.

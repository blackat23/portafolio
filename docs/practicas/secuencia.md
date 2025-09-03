## Dirección de Secuencia(Raspberry Pi Pico / Pico 2)
> Este proyecto implementa un **controlador de la dirección de secuencia** usando Raspberry Pi Pico.  
> El sistema utiliza 5 LEDs para representar la cancha y 2 LEDs para la puntuación.  
> La “pelota” se mueve automáticamente en una dirección mediante un **timer periódico**; al llegar a un extremo, el controlador detiene la secuencia y espera la intervención del jugador correspondiente a través de **interrupciones por botón**.  
> El controlador gestiona el cambio de dirección de la secuencia según la respuesta del jugador, reiniciando la pelota al centro si hay error y actualizando la puntuación con los LEDs.

---

## 1) Resumen

- **Nombre del proyecto:** _Dirección de Secuencia_  
- **Autor:** _Antonio Martínez_  
- **Curso / Asignatura:** _Sistemas Embebidos_  
- **Fecha:** _02/09/2025_  
- **Descripción breve:** _Dirección de Secuencia._

!!! tip "Información del proyecto"
    - **Lenguaje:** C (Pico SDK)  
    - **MCU:** Raspberry Pi Pico / Pico 2 (RP2040)  
    - **Librerías:** `pico/stdlib.h`, `hardware/gpio.h`, `hardware/timer.h`  
    - **Estrategia:** Timer cada 500 ms mueve la pelota; en los extremos, se detiene y espera la **IRQ del botón** correcto.

### Material utilizado
- Raspberry Pi Pico / Pico 2  
- 5 LEDs (cancha) + 2 LEDs (puntuación)  
- 7 resistencias 220–330 Ω (limitación de corriente)  
- 2 botones de pulso (Normally Open)  
- Cables, protoboard y cable USB  
- PC con VS Code + CMake + Pico SDK configurado

---

## 2) Objetivos

- **Timers y concurrencia simple:** Usar `repeating_timer` para lógica periódica.  
- **Interrupciones GPIO:** Capturar eventos de botón sin bloquear el bucle principal.  
- **Máquina de estados mínima:** “en marcha” vs “esperando respuesta en extremo”.  
- **Buenas prácticas de GPIO:** Pull-ups, flancos adecuados y actualización clara de LEDs.  
- **Retroalimentación visual:** Puntos con LEDs de score y reinicio al centro.

---

## 3) Circuito

- **Cancha (5 LEDs):** GPIO **9–13** (cada uno con su resistencia a GND si el pin maneja nivel alto para encender).  
- **Puntuación:** `LED_SCORE_P1` → **GPIO 7**, `LED_SCORE_P2` → **GPIO 8**.  
- **Botones:** `BTN_P1` → **GPIO 14**, `BTN_P2` → **GPIO 15**, ambos con **pull-up** interno; el otro terminal del botón a **GND**.  
- **Alimentación:** vía USB del Pico.

> Con pull-up, el botón en reposo lee **1** y al presionar baja a **0**. En este código las IRQ están configuradas por **flanco de subida (0x8u)**, es decir, al soltar.

---

## 4) Código

```c
    #include <stdio.h>
    #include "pico/stdlib.h"
    #include "hardware/timer.h"
    #include "hardware/gpio.h"

    // Pines de LEDs de la cancha
    #define LED0 9
    #define LED1 10
    #define LED2 11
    #define LED3 12
    #define LED4 13

    // LEDs de puntuación
    #define LED_SCORE_P1 7
    #define LED_SCORE_P2 8

    // Botones
    #define BTN_P1 14
    #define BTN_P2 15

    // Variables del juego
    volatile int pelota_pos = 2;       // posición inicial (LED del centro)
    volatile int direccion = 1;        // 1 → derecha, -1 → izquierda
    volatile bool esperando_respuesta = false;

    // Prototipos
    bool mover_pelota(struct repeating_timer *t);
    void actualizar_leds();
    void btn_callback(uint gpio, uint32_t events);

    int main() {
        stdio_init_all();

        // Configuración LEDs
        int leds[] = {LED0, LED1, LED2, LED3, LED4, LED_SCORE_P1, LED_SCORE_P2};
        for (int i = 0; i < 7; i++) {
            gpio_init(leds[i]);
            gpio_set_dir(leds[i], true);
            gpio_put(leds[i], 0);
        }

        // Configuración botones
        // Configuración botón jugador 1
        gpio_init(BTN_P1);
        gpio_set_dir(BTN_P1, false);
        gpio_pull_up(BTN_P1);

        // Configuración botón jugador 2
        gpio_init(BTN_P2);
        gpio_set_dir(BTN_P2, false);
        gpio_pull_up(BTN_P2);

        // Registrar el callback global UNA sola vez
        gpio_set_irq_enabled_with_callback(BTN_P1, 0x8u, true, &btn_callback);

        // Habilitar interrupción también en el otro botón, sin registrar callback otra vez
        gpio_set_irq_enabled(BTN_P2, 0x8u, true);

        // Timer para mover la pelota
        struct repeating_timer timer;
        add_repeating_timer_ms(500, mover_pelota, NULL, &timer);

        while (true) {
            tight_loop_contents(); // loop vacío, todo se maneja con interrupciones y timer
        }
    }

    // Timer: mueve la pelota
    bool mover_pelota(struct repeating_timer *t) {
        if (esperando_respuesta) return true; // espera al jugador

        // Apagar LEDs
        for (int i = LED0; i <= LED4; i++) gpio_put(i, 0);

        // Mover pelota
        pelota_pos += direccion;

        // Revisar si llegó al extremo
        if (pelota_pos <= 0) {
            esperando_respuesta = true; // espera botón del jugador 1
            pelota_pos = 0;
        } else if (pelota_pos >= 4) {
            esperando_respuesta = true; // espera botón del jugador 2
            pelota_pos = 4;
        }

        actualizar_leds();
        return true;
    }

    // Encender LED de la pelota
    void actualizar_leds() {
        gpio_put(LED0, pelota_pos == 0);
        gpio_put(LED1, pelota_pos == 1);
        gpio_put(LED2, pelota_pos == 2);
        gpio_put(LED3, pelota_pos == 3);
        gpio_put(LED4, pelota_pos == 4);
    }

    // Callback de botones (interrupciones)
    void btn_callback(uint gpio, uint32_t events) {
        if (!esperando_respuesta) return;

        if (gpio == BTN_P1 && pelota_pos == 0) {
            direccion = 1; // devuelve hacia la derecha
            esperando_respuesta = false;
        } 
        else if (gpio == BTN_P2 && pelota_pos == 4) {
            direccion = -1; // devuelve hacia la izquierda
            esperando_respuesta = false;
        } 
        else {
            // Falló → punto para el contrario
            if (gpio == BTN_P1) {
                gpio_put(LED_SCORE_P2, 1);
                sleep_ms(500);
                gpio_put(LED_SCORE_P2, 0);
            } else {
                gpio_put(LED_SCORE_P1, 1);
                sleep_ms(500);
                gpio_put(LED_SCORE_P1, 0);
            }
            // Reiniciar pelota
            pelota_pos = 2;
            direccion = (gpio == BTN_P1) ? 1 : -1;
            esperando_respuesta = false;
        }
        actualizar_leds();
    }
## 5) Explicación del programa

### a) Definiciones y mapeo de pines
- **Cancha (LEDs de juego):** `LED0..LED4` en GPIO **9–13**. Representan la posición de la pelota en la cancha.  
- **Puntuación:** `LED_SCORE_P1` (GPIO **7**) y `LED_SCORE_P2` (GPIO **8**). Se encienden brevemente cuando el jugador contrario gana un punto.  
- **Botones:** `BTN_P1` (GPIO **14**) y `BTN_P2` (GPIO **15**), configurados con **pull-up interno** (en reposo = 1, presionado = 0).  

---

### b) Variables clave
- **`pelota_pos ∈ {0..4}`:** indica qué LED de la cancha está encendido, es decir, la posición actual de la pelota.  
- **`direccion ∈ {+1, −1}`:** determina si la pelota se mueve hacia la derecha (+1) o hacia la izquierda (−1).  
- **`esperando_respuesta (bool)`:** cuando la pelota llega a un extremo, el avance se pausa hasta que el jugador correspondiente presione su botón.  

---

### c) Inicialización de GPIO y botones
- Los LEDs de la cancha y de puntuación se configuran como **salida** y se inicializan en **apagado** (`gpio_put(...,0)`).  
- Los botones se configuran como **entrada** con `gpio_pull_up()`.  
- Se registra un **callback global** con `gpio_set_irq_enabled_with_callback()` para BTN_P1.  
- Se habilita también la interrupción de BTN_P2 con `gpio_set_irq_enabled()`, pero sin registrar otro callback (se reutiliza el mismo).  
- Las interrupciones se configuran en **flanco de subida (0x8u)**, es decir, cuando el botón pasa de presionado a liberado.  

---

### d) Temporizador repetitivo (`repeating_timer`)
- Se crea un temporizador con `add_repeating_timer_ms(500, mover_pelota, ...)`, que ejecuta la función `mover_pelota` cada **500 ms**.  
- Si `esperando_respuesta = false`, la pelota avanza (`pelota_pos += direccion`).  
- Si llega a los extremos (`0` o `4`), se fija `esperando_respuesta = true` y el juego queda en pausa hasta que un jugador responda.  

---

### e) Función `actualizar_leds()`
- Se encarga de **reflejar la posición actual de la pelota** encendiendo únicamente el LED correspondiente (`LED0..LED4`) y apagando los demás.  

---

### f) Callback de botones (`btn_callback`)
- Solo actúa si `esperando_respuesta = true`.  
- **Caso correcto:**  
  - Si la pelota está en el extremo izquierdo (`pelota_pos==0`) y se presiona **BTN_P1**, se cambia la dirección a derecha (`direccion=+1`) y se reanuda el juego.  
  - Si la pelota está en el extremo derecho (`pelota_pos==4`) y se presiona **BTN_P2**, se cambia la dirección a izquierda (`direccion=-1`) y se reanuda el juego.  
- **Caso incorrecto (falla):**  
  - Si se presiona el botón equivocado, se enciende el LED de **puntuación del contrario** durante 500 ms.  
  - Después, la pelota se reinicia al centro (`pelota_pos=2`) y la dirección inicial se ajusta según quién falló.  

---

### g) Máquina de estados implícita
1. **RUNNING:** la pelota avanza automáticamente cada 500 ms.  
2. **WAIT_INPUT:** al llegar a un extremo, se detiene y espera la respuesta del jugador.  
3. **SCORE:** si el jugador falla, se muestra el punto en el LED correspondiente, se reinicia la pelota en el centro y se regresa a RUNNING.  

---
## Video de demostración
<div style="position:relative;padding-bottom:56.25%;height:0;overflow:hidden;">
  <iframe
    src="https://www.youtube.com/embed/twmNGeeP-nU"
    title="YouTube Short"
    allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
    allowfullscreen
    style="position:absolute;top:0;left:0;width:100%;height:100%;border:0;">
  </iframe>
</div>

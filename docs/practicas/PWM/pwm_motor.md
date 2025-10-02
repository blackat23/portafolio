# Control de motor DC por PWM (3 velocidades) con botones (Raspberry Pi Pico + TB6612FNG)

> Este proyecto controla la **velocidad de un motor DC** mediante **PWM** usando un Raspberry Pi Pico/Pico 2 y el driver **TB6612FNG**. Dos botones (SUBIR/BAJAR) cambian entre **tres niveles de velocidad** predefinidos. El PWM trabaja a **20 kHz** (fuera del rango audible).

---

## 1) Resumen

- **Nombre del proyecto:** _Motor DC PWM 3 velocidades_  
- **Equipo / Autor(es):** _Antonio Martínez_  
- **Curso / Asignatura:** _Sistemas Embebidos_  
- **Fecha:** _01/10/2025_  
- **Descripción breve:** _Control de un motor DC con TB6612FNG usando PWM (20 kHz) y dos botones con _pull-up_ interno para seleccionar 3 niveles de velocidad (70%, 80%, 90%)._  

!!! tip "Información del proyecto:"
    **Lenguaje/SDK:** C con Raspberry Pi Pico SDK (`pico/stdlib.h`, `hardware/pwm.h`).  
    **Técnicas clave:** PWM (20 kHz), cálculo de `clkdiv` y `wrap`, _debounce_ por software, lectura de GPIO con _pull-up_ y detección de flanco.  
    **Plataforma:** Raspberry Pi Pico / Pico 2.  

### Material utilizado
- Raspberry Pi Pico (o Pico 2) y cable USB  
- Driver **TB6612FNG**  
- **Motor DC** (compatible con la tensión de VM y corriente del driver)  
- **Fuente de VM** (ej. 5–12 V según motor)  
- 2 **botones** momentáneos (a GND, sin resistencias externas gracias al _pull-up_ interno)  
- Protoboard y jumpers

---

## 2) Objetivos

- Configurar **PWM** a 20 kHz sobre un pin GPIO para modular la velocidad del motor.  
- Implementar **tres velocidades** predefinidas con cambios mediante **dos botones** (subir/bajar).  
- Aplicar **_debounce_** por software y detección de **flanco 1→0**.  
- Fijar **dirección de giro** con pines AIN1/AIN2 (TB6612FNG).

---

## 3) Conexiones / Esquema

**Notas generales (TB6612FNG):**
- **PWMA** controla el canal A (ancho de pulso).  
- **AIN1/AIN2** fijan la dirección del canal A.  
- **AO1/AO2** salen al motor.  
- **VM**: alimentación del motor (p. ej. 6–12 V).  
- **VCC**: 3V3 lógicos desde el Pico.  
- **GND común** entre Pico, TB6612 y la fuente del motor.  
- **STBY**: mantener en alto (3V3) para habilitar el driver.

**Mapeo de pines (según el código):**

| Señal (Pico)  | GPIO | TB6612FNG | Descripción                      |
|---------------|------|-----------|----------------------------------|
| `PWM_PIN`     | 0    | PWMA      | PWM 20 kHz (control de velocidad)|
| `DIR1_PIN`    | 16   | AIN1      | Dirección (bit 1)                |
| `DIR2_PIN`    | 17   | AIN2      | Dirección (bit 2)                |
| `BTN_DOWN_PIN`| 2    | —         | Botón BAJAR (a GND, _pull-up_)   |
| `BTN_UP_PIN`  | 3    | —         | Botón SUBIR (a GND, _pull-up_)   |

**Botones (activos en bajo):**
- Un terminal del botón a **GND**; el otro a **GPIO 2** (BAJAR) o **GPIO 3** (SUBIR).  
- Se habilita `gpio_pull_up()`, por lo que al presionar: **0 lógico**.

**Dirección fija (ejemplo):**  
- `AIN1=1` y `AIN2=0` (definido en el código) → giro **adelante**.  
- Para invertir el giro: **intercambia** los valores o cablea al revés.

**Diagrama de conexión**  
![Conexión Pico + TB6612FNG](../recursos/imgs/motor_pwm_tb6612.png)

---

## 4) Código

```c
    // dc_motor_pwm.c — Control de motor DC con 3 velocidades (baja/media/alta)
    // Raspberry Pi Pico / Pico 2 + TB6612FNG (ejemplo)
    // PWM en GPIO15 → PWMA; Dirección: AIN1 (GPIO16), AIN2 (GPIO17)
    // Botones a GND con pull-up interno: BTN_DOWN(GPIO13), BTN_UP(GPIO14)

    #include "pico/stdlib.h"
    #include "hardware/pwm.h"

    // ----------------- Pines -----------------
    #define PWM_PIN      0       // PWMA (TB6612)
    #define DIR1_PIN     16     // AIN1
    #define DIR2_PIN     17     // AIN2
    #define BTN_DOWN_PIN 2     // Botón: bajar velocidad
    #define BTN_UP_PIN   3     // Botón: subir velocidad

    // ----------------- PWM -------------------
    #define F_PWM_HZ  20000     // 20 kHz: fuera de audio
    #define TOP       1023      // 10 bits de resolución (0..1023)

    // ----------------- Debounce --------------
    #define DEBOUNCE_MS 30

    int main() {
        stdio_init_all();

    // --- Dirección: adelante (AIN1=1, AIN2=0) ---
    gpio_init(DIR1_PIN); gpio_set_dir(DIR1_PIN, GPIO_OUT); gpio_put(DIR1_PIN, 1);
    gpio_init(DIR2_PIN); gpio_set_dir(DIR2_PIN, GPIO_OUT); gpio_put(DIR2_PIN, 0);

    // --- Botones con pull-up interno (activos en 0) ---
    gpio_init(BTN_DOWN_PIN); gpio_set_dir(BTN_DOWN_PIN, GPIO_IN); gpio_pull_up(BTN_DOWN_PIN);
    gpio_init(BTN_UP_PIN);   gpio_set_dir(BTN_UP_PIN,   GPIO_IN); gpio_pull_up(BTN_UP_PIN);

    // --- PWM en el pin de control de velocidad ---
    gpio_set_function(PWM_PIN, GPIO_FUNC_PWM);
    uint slice = pwm_gpio_to_slice_num(PWM_PIN);
    uint chan  = pwm_gpio_to_channel(PWM_PIN);

    float f_clk = 150000000.0f;                         // 125 MHz
    float div   = f_clk / (F_PWM_HZ * (TOP + 1));       // clkdiv flotante
    pwm_set_clkdiv(slice, div);
    pwm_set_wrap(slice, TOP);

    pwm_set_chan_level(slice, chan, 0);                 // arranque detenido
    pwm_set_enabled(slice, true);

    // --- Tabla de 3 velocidades (duty) ---
    // Baja=35%, Media=65%, Alta=90%
    uint16_t speed_levels[] = {
        (uint16_t)(TOP * 0.70f),
        (uint16_t)(TOP * 0.80f),
        (uint16_t)(TOP * 0.90f)
    };
    const int NUM_SPEEDS = 3;

    int idx = 0; // índice de velocidad actual (0=baja)
    uint32_t t_last_down = 0, t_last_up = 0;
    int last_down = 1, last_up = 1; // estados anteriores (con pull-up, reposo=1)

    // Aplica velocidad inicial
    pwm_set_chan_level(slice, chan, speed_levels[idx]);

    while (true) {
        // --- Leer botones (con edge + debounce) ---
        int cur_down = gpio_get(BTN_DOWN_PIN);
        int cur_up   = gpio_get(BTN_UP_PIN);
        uint32_t now = to_ms_since_boot(get_absolute_time());

        // Botón DOWN: flanco de 1->0
        if (last_down == 1 && cur_down == 0 && (now - t_last_down) > DEBOUNCE_MS) {
            if (idx > 0) idx--;
            pwm_set_chan_level(slice, chan, speed_levels[idx]);
            t_last_down = now;
        }
        // Botón UP: flanco de 1->0
        if (last_up == 1 && cur_up == 0 && (now - t_last_up) > DEBOUNCE_MS) {
            if (idx < (NUM_SPEEDS - 1)) idx++;
            pwm_set_chan_level(slice, chan, speed_levels[idx]);
            t_last_up = now;
        }

        last_down = cur_down;
        last_up   = cur_up;

        sleep_ms(5);
    }
}


## 5) Explicación del programa

### a) Configuración de PWM (20 kHz, 10 bits)

- `F_PWM_HZ = 20000` y `TOP = 1023` → resolución de **10 bits** (0..1023).
- Se calcula `clkdiv` con `div = f_clk / (F_PWM_HZ * (TOP + 1))`.
- **Nota:** En el código `f_clk = 150e6` con comentario “125 MHz”. Ajusta el valor si tu reloj es 125 MHz (`125000000.0f`) o deja 150 MHz si realmente configuraste el PLL a esa frecuencia.
- `pwm_set_wrap(slice, TOP)` y `pwm_set_clkdiv(slice, div)` fijan la frecuencia.
- `pwm_set_chan_level(slice, chan, duty)` actualiza el **ciclo de trabajo**.

### b) Dirección del motor (TB6612FNG)

- Se fuerza **adelante** con `AIN1=1`, `AIN2=0`.
- Para invertir el giro: `AIN1=0`, `AIN2=1`. No uses `0/0` (rueda libre) ni `1/1` (freno) salvo intencionalmente.

### c) Botones con *pull-up* y *debounce*

- `BTN_DOWN_PIN` (GPIO 2) y `BTN_UP_PIN` (GPIO 3) como **entradas** con `gpio_pull_up()`.
- Se detecta **flanco 1→0** (presión) comparando estado previo/actual.
- **Debounce:** `DEBOUNCE_MS = 30` ms por botón con marcas de tiempo (`to_ms_since_boot`).

### d) Tabla de velocidades

- Niveles en `speed_levels[]`: **70%**, **80%**, **90%** del `TOP`.
  - **Nota:** El comentario dice “35/65/90%”, pero el **código actual aplica 70/80/90%**. Cambia los coeficientes `0.70f, 0.80f, 0.90f` si deseas otros niveles (p. ej., `0.35f, 0.65f, 0.90f`).
- `idx` selecciona el nivel. Los botones **BAJAR/SUBIR** decrementan/incrementan `idx` dentro de `[0, NUM_SPEEDS-1]`.

### e) Bucle principal

- Lee botones, aplica *debounce*, actualiza `idx` si hay flanco válido y **reprograma el duty** con `pwm_set_chan_level`.
- `sleep_ms(5)` fija la **cadencia de sondeo** (~200 Hz), suficiente para la interfaz de botones.

---

## 6) Pruebas y comportamiento esperado

- **Arranque:** motor en marcha a la **velocidad 0 (70%)**.
- **Botón SUBIR (GPIO 3):** incrementa a **80%** y luego **90%**.
- **Botón BAJAR (GPIO 2):** decrementa hacia **80%** y **70%**.
- **Debounce estable:** pulsaciones rápidas no deben generar múltiples cambios espurios.
- **Silencio eléctrico:** PWM a **20 kHz** evita zumbidos audibles en la mayoría de motores.


<div style="position:relative;padding-bottom:56.25%;height:0;overflow:hidden;">
  <iframe
    src="https://www.youtube.com/embed/XNEPDBBVE14"
    title="YouTube video"
    allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
    allowfullscreen
    style="position:absolute;top:0;left:0;width:100%;height:100%;border:0;">
  </iframe>
</div>

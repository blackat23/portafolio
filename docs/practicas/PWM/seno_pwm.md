# Senoide aproximada de 60 Hz por PWM (Raspberry Pi Pico)

> Este proyecto genera una **señal senoidal “promediada” de 60 Hz** modulando el **duty cycle** de una portadora **PWM a 40 kHz** (10 bits). Se actualiza el duty en **128 muestras por ciclo**, logrando un valor medio (tras filtrado o por inercia de la carga) que sigue una senoide. Útil para pruebas de filtrado RC, control de potencia (baja tensión) o caracterización de etapas H-bridge (aisladas). **No conectar a la red eléctrica.**

---

## 1) Resumen

- **Nombre del proyecto:** _PWM → senoide de 60 Hz_  
- **Equipo / Autor(es):** _Antonio Martínez_  
- **Curso / Asignatura:** _Sistemas Embebidos_  
- **Fecha:** _01/10/2025_  
- **Descripción breve:** _Síntesis de una senoide de 60 Hz mediante modulación del duty de un PWM a 40 kHz con 10 bits y 128 muestras por ciclo. Timing con `sleep_until()` para reducir jitter._  

!!! tip "Información del proyecto:"
    **Lenguaje/SDK:** C con Raspberry Pi Pico SDK (`pico/stdlib.h`, `pico/time.h`, `hardware/pwm.h`).  
    **Técnicas clave:** PWM de alta frecuencia, tabla/muestreo senoidal, temporización absoluta (bajo jitter), cuantización 10 bits.  
    **Plataforma:** Raspberry Pi Pico / Pico 2.  

### Material utilizado
- Raspberry Pi Pico (o Pico 2) y cable USB  
- Protoboard y jumpers  
- **(Opcional)** Filtro **RC** pasabajas (p. ej., R=1–4.7 kΩ, C=4.7–10 µF) para recuperar la senoide promedio  
- **(Opcional)** Osciloscopio para visualizar portadora y señal filtrada  
- **(Si se usa potencia)** Etapa H-bridge/línea aislada, **nunca** a red AC

---

## 2) Objetivos

- Configurar **PWM** a **40 kHz** con **10 bits** de resolución.  
- Generar una **senoide de 60 Hz** modulando el duty con **128 muestras/ciclo**.  
- Usar **temporización absoluta** (`sleep_until`) para minimizar **jitter** de muestreo.  
- Controlar **amplitud** (`AMP_PCT`) y **offset DC** (`DC_OFFSET`) de la señal promediada.

---

## 3) Conexiones / Esquema

**Mapeo de pines (según el código):**

| Señal (Pico) | GPIO | Descripción                                  |
|--------------|------|----------------------------------------------|
| `PWM_PIN`    | 15   | Salida PWM (portadora 40 kHz, duty variable) |
| `GND`        | —    | Referencia común                             |

**Notas de conexión:**
- Para observar la **senoide promedio**, conecta `PWM_PIN` a un **filtro RC** pasabajas y mide en la salida del filtro respecto a GND.  
- Ajusta el corte del RC (p. ej., 300–800 Hz) para atenuar la portadora (40 kHz) preservando 60 Hz.

**Diagrama de conexión**  
![PWM + RC pasabajas](../recursos/imgs/pwm_sine_rc.png)

---

## 4) Código

    ```c
    // pwm_sine_60hz.c — Senoide aproximada de 60 Hz modulando el duty del PWM
    #include "pico/stdlib.h"
    #include "pico/time.h"
    #include "hardware/pwm.h"
    #include <math.h>

    #define PWM_PIN             15      // GPIO de salida PWM
    #define F_PWM_HZ            40000   // Portadora PWM (40 kHz, fuera de audio)
    #define TOP                 1023    // 10 bits de resolución

    #define SINE_FREQ_HZ        60      // Frecuencia de la senoide “promediada”
    #define SAMPLES_PER_CYCLE   128     // Muestras por ciclo de seno
    #define AMP_PCT             0.98f   // Amplitud relativa (0..1)
    #define DC_OFFSET           0.50f   // Offset DC relativo (0.5 = centrado)

    int main() {
        stdio_init_all();

        // --- Configurar PWM en el pin ---
        gpio_set_function(PWM_PIN, GPIO_FUNC_PWM);
        uint slice = pwm_gpio_to_slice_num(PWM_PIN);
        uint chan  = pwm_gpio_to_channel(PWM_PIN);

        // Calcular divisor para la portadora
        float f_clk = 125000000.0f;                    // 125 MHz
        float div   = f_clk / (F_PWM_HZ * (TOP + 1));  // clkdiv flotante
        pwm_set_clkdiv(slice, div);
        pwm_set_wrap(slice, TOP);

        pwm_set_chan_level(slice, chan, 0);
        pwm_set_enabled(slice, true);

        // --- Generación de seno por software (duty = sin()) ---
        const float fs = (float)SINE_FREQ_HZ * (float)SAMPLES_PER_CYCLE;   // tasa de actualización
        const uint32_t Ts_us = (uint32_t)lrintf(1000000.0f / fs);          // periodo de muestreo en us

        float phase = 0.0f;
        const float phase_step = 2.0f * (float)M_PI / (float)SAMPLES_PER_CYCLE;

        // Reloj base para minimizar jitter
        absolute_time_t t = delayed_by_us(get_absolute_time(), Ts_us);

        while (true) {
            // s ∈ [-1,1] → u ∈ [0,1] con offset y amplitud
            float s = sinf(phase);
            float u = DC_OFFSET + 0.5f * AMP_PCT * s;
            if (u < 0.0f) u = 0.0f; if (u > 1.0f) u = 1.0f;

            uint16_t level = (uint16_t)lrintf(u * (float)TOP);
            pwm_set_chan_level(slice, chan, level);

            // Avanzar fase y temporización precisa
            phase += phase_step;
            if (phase >= 2.0f * (float)M_PI) phase -= 2.0f * (float)M_PI;

            sleep_until(t);
            t = delayed_by_us(t, Ts_us);
        }
    }
    
## 5) Explicación del programa

### a) Portadora PWM (40 kHz, 10 bits)

- `F_PWM_HZ = 40000` y `TOP = 1023` → **10 bits** (niveles 0..1023).
- Frecuencia de PWM mediante `div = f_clk / (F_PWM_HZ*(TOP+1))` con `f_clk = 125 MHz`.
- `pwm_set_wrap(slice, TOP)` define el conteo y `pwm_set_clkdiv(slice, div)` ajusta el preescalador.
- El **duty** se actualiza con `pwm_set_chan_level(slice, chan, level)`.

### b) Senoide “promedio” de 60 Hz (muestreo a 7.68 kHz)

- `SINE_FREQ_HZ = 60`, `SAMPLES_PER_CYCLE = 128` → `fs = 60×128 = 7680 Hz`.
- Periodo de muestra: `Ts_us ≈ 1e6 / 7680 ≈ 130.21 µs` (redondeado con `lrintf`).
- Se incrementa la **fase** en `phase_step = 2π/128` por muestra y se calcula `u = DC_OFFSET + 0.5·AMP_PCT·sin(phase)` limitado a `[0,1]`.
- `u` se mapea a 10 bits: `level = round(u·TOP)`.

### c) Temporización precisa (bajo jitter)

- Se emplea un **reloj absoluto**: `sleep_until(t); t = delayed_by_us(t, Ts_us);` en vez de `sleep_us(Ts_us)`.
- Esto mantiene un **periodo constante** de actualización de duty, reduciendo el jitter de muestreo.

### d) Control de amplitud y offset

- `AMP_PCT` (0..1) escala la amplitud **pico a pico** de la senoide promedio (aquí 98%).
- `DC_OFFSET = 0.5` centra la señal en **50%** de duty (salida bipolar tras filtro/etapa diferencial).
- Para una salida puramente **unipolar** filtrada, mantiene `DC_OFFSET ≥ AMP_PCT/2`.

### e) Recuperación de la senoide (filtro)

- Para observar una senoide limpia, usa un **filtro pasabajas** (p. ej., RC con `fc≈300–800 Hz`).
- La portadora a **40 kHz** queda muy atenuada; se conserva la envolvente de **60 Hz**.

---

## 6) Pruebas y comportamiento esperado

- **Osciloscopio (PWM crudo):** tren a 40 kHz cuyo duty va oscilando suavemente.
- **Tras RC:** señal aproximada a **senoide 60 Hz**; amplitud depende de `AMP_PCT`, offset según `DC_OFFSET`.
- **Estabilidad temporal:** la frecuencia de 60 Hz es estable por el uso de `sleep_until`.
- **Ajustes rápidos:** cambiar `SAMPLES_PER_CYCLE` mejora la suavidad (más muestras) a costa de mayor tasa de actualización.


<div style="position:relative;padding-bottom:56.25%;height:0;overflow:hidden;">
  <iframe
    src="https://www.youtube.com/embed/2d4McjVFlx8"
    title="YouTube video"
    allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
    allowfullscreen
    style="position:absolute;top:0;left:0;width:100%;height:100%;border:0;">
  </iframe>
</div>


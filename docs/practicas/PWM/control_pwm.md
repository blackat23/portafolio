#  Control de PWM 

: Periféricos, Memoria, Ecosistema, Costos, Arquitectura y Velocidad de Trabajo


---

## Informacion
- **Nombre del proyecto:** _Control de pwm con Raspberry Pico_
- **Equipo / Autor(es):** _Antonio Martínez_
- **Curso / Asignatura:** _Sistemas Embebidos_
- **Fecha:** _23/08/2025_
- **Descripción breve:** _Implementación de un sistema de generación de audio con buzzer pasivo usando control PWM en el microcontrolador RP2040 (Raspberry Pi Pico). Se documentan los periféricos utilizados y se ubica al RP2040 en una comparativa con otros MCUs._


---

## Introducción: Control PWM  

En este proyecto se emplea el **módulo PWM (Pulse Width Modulation)** del microcontrolador **RASPBERRY PICO** para generar señales de audio en un buzzer pasivo.  
El principio consiste en variar la frecuencia de la señal cuadrada enviada al buzzer, de manera que éste reproduzca diferentes notas musicales.  

El código define una lista de notas (frecuencias en Hz) y figuras rítmicas (divisiones de tiempo), que son enviadas al periférico PWM del RP2040 para reproducir una melodía.  

Este enfoque demuestra cómo los periféricos de un microcontrolador pueden ser usados no solo para control de motores o regulación de voltaje, sino también para la generación de sonidos y música digital.


---

## Periféricos  

- **RASPBERRY PICO**  
    Se utiliza la Raspberry Pi Pico porque integra el microcontrolador RP2040, que cuenta con periféricos PWM de hardware dedicados. Esto permite generar señales de modulación por ancho de pulso de manera precisa y eficiente, sin sobrecargar el procesador principal. Además, la Pico es económica, fácil de programar y ampliamente soportada por la comunidad, lo que facilita el desarrollo y la depuración de proyectos de control de señales como este.

---
  
## Ecosistema  

El **RP2040** cuenta con un ecosistema creciente y robusto:  

- **Lenguajes soportados:** C/C++ (usando SDK oficial) y MicroPython.  
- **IDE y entornos comunes:** VSCode, Thonny, Arduino IDE (con soporte agregado).  

---




## Velocidad de Trabajo  

- La Pico opera a **150 MHz**, lo cual resulta sobrado para generar PWM de audio.  
- El cálculo del divisor para el periférico PWM (`pwm_set_clkdiv_int_frac()`) se hace rápidamente gracias a la alta frecuencia de reloj.  
- En este proyecto, la frecuencia máxima usada fue ~2 kHz (notas musicales), muy por debajo del límite real del microcontrolador.  


---



## Codigo del buzzer 


```c

    #include <stdio.h>
    #include "pico/stdlib.h"
    #include "hardware/pwm.h"


    #define BUZZER_PIN 15
    #define BPM 200           
    #define GAP_MS 12          

    // notes en hz
    #define NOTE_B0  31
    #define NOTE_C1  33
    #define NOTE_CS1 35
    #define NOTE_D1  37
    #define NOTE_DS1 39
    #define NOTE_E1  41
    #define NOTE_F1  44
    #define NOTE_FS1 46
    #define NOTE_G1  49
    #define NOTE_GS1 52
    #define NOTE_A1  55
    #define NOTE_AS1 58
    #define NOTE_B1  62
    #define NOTE_C2  65
    #define NOTE_CS2 69
    #define NOTE_D2  73
    #define NOTE_DS2 78
    #define NOTE_E2  82
    #define NOTE_F2  87
    #define NOTE_FS2 93
    #define NOTE_G2  98
    #define NOTE_GS2 104
    #define NOTE_A2  110
    #define NOTE_AS2 117
    #define NOTE_B2  123
    #define NOTE_C3  131
    #define NOTE_CS3 139
    #define NOTE_D3  147
    #define NOTE_DS3 156
    #define NOTE_E3  165
    #define NOTE_F3  175
    #define NOTE_FS3 185
    #define NOTE_G3  196
    #define NOTE_GS3 208
    #define NOTE_A3  220
    #define NOTE_AS3 233
    #define NOTE_B3  247
    #define NOTE_C4  262
    #define NOTE_CS4 277
    #define NOTE_D4  294
    #define NOTE_DS4 311
    #define NOTE_E4  330
    #define NOTE_F4  349
    #define NOTE_FS4 370
    #define NOTE_G4  392
    #define NOTE_GS4 415
    #define NOTE_A4  440
    #define NOTE_AS4 466
    #define NOTE_B4  494
    #define NOTE_C5  523
    #define NOTE_CS5 554
    #define NOTE_D5  587
    #define NOTE_DS5 622
    #define NOTE_E5  659
    #define NOTE_F5  698
    #define NOTE_FS5 740
    #define NOTE_G5  784
    #define NOTE_GS5 831
    #define NOTE_A5  880
    #define NOTE_AS5 932
    #define NOTE_B5  988
    #define NOTE_C6  1047
    #define NOTE_CS6 1109
    #define NOTE_D6  1175
    #define NOTE_DS6 1245
    #define NOTE_E6  1319
    #define NOTE_F6  1397
    #define NOTE_FS6 1480
    #define NOTE_G6  1568
    #define NOTE_GS6 1661
    #define NOTE_A6  1760
    #define NOTE_AS6 1865
    #define NOTE_B6  1976

    #define REST 0

    // ====== Figuras (divisor del pulso) ======
    // Duración real (ms) = (60000 / BPM) * 4 / FIG
    #define W   1   // redonda (4 tiempos)
    #define H   2   // blanca   (2 tiempos)
    #define Q   4   // negra    (1 tiempo)
    #define E   8   // corchea  (1/2)
    #define S   16  // semicor. (1/4)
    #define T32 32  // fusa

    // ====== PWM helper ======
    static void pwm_set_freq_duty(uint pin, uint32_t freq, float duty_percent) {
        gpio_set_function(pin, GPIO_FUNC_PWM); // seteamos la funcion del pin a PWM
        uint slice = pwm_gpio_to_slice_num(pin); //Cambiar la funcioin de GPIO a PWM
        uint chan  = pwm_gpio_to_channel(pin); //Asignamos el canal

        if (freq == 0) {
            pwm_set_enabled(slice, false);
            return;
        }

        //Aqui empezamos a calcular el divisor

        uint32_t f_sys = 150000000u; // la frecuencia del pico
        uint32_t top = 4095; // aqui pensamos esto para usar 12 bits, pues tenemos la formula log2(top +1)
        float div = (float)f_sys / (freq * (top + 1));

        //esto es solo por si se sale de rano el divisor
        if (div < 1.0f) div = 1.0f;          // límite inferior ( pues solo tiene un rango el pwm de 1 a 255)
        if (div > 255.0f) div = 255.0f;      // límite superior

        uint32_t div_int  = (uint32_t)div; //dividimos div en su parte entera
        uint32_t div_frac = (uint32_t)((div - div_int) * 16.0f); // y esta es su parte fraccionaria
        //basicamente se define div que se usará


        pwm_set_clkdiv_int_frac(slice, div_int, div_frac); //seteamos el divisor, esta es como la funcion pwm_set_clkdiv pero con parte fraccionaria
        pwm_set_wrap(slice, top);//seteamos el top

        uint32_t level = (uint32_t)(duty_percent * 0.01f * (top + 1));
        pwm_set_chan_level(slice, chan, level);
        pwm_set_enabled(slice, true);
    }

    static inline uint32_t ms_per_whole(void) {
        // para la duracion de una redonda 
        return (60000u / BPM) * 4u;
    }

    static void play(uint32_t freq, uint32_t figure_div) {
        uint32_t dur_ms = ms_per_whole() / figure_div;
        if (freq == REST) {
            pwm_set_freq_duty(BUZZER_PIN, 0, 0);
            sleep_ms(dur_ms);
        } else {
            pwm_set_freq_duty(BUZZER_PIN, freq, 50.0f);
            
            sleep_ms(dur_ms);
            pwm_set_freq_duty(BUZZER_PIN, 0, 0);
            
        }
    }


    static const uint16_t melody[] = {
        // Frase A
        NOTE_D4, NOTE_D4, NOTE_D5,
        NOTE_A4, REST, NOTE_GS4,
        REST, NOTE_G4, NOTE_F4, 
        NOTE_F4, NOTE_D4, NOTE_F4, NOTE_G4,
        //B
        NOTE_C4, NOTE_C4, NOTE_D5,
        NOTE_A4, REST, NOTE_GS4,
        REST, NOTE_G4, NOTE_F4,
        NOTE_F4, NOTE_D4, NOTE_F4, NOTE_G4,
        //C
        NOTE_A3, NOTE_A3, NOTE_D5,
        NOTE_A4, REST, NOTE_GS4,
        REST, NOTE_G4, NOTE_F4,
        NOTE_F4, NOTE_D4, NOTE_F4, NOTE_G4,
        //D 
        NOTE_AS3, NOTE_A3, NOTE_D5,
        NOTE_A4, REST, NOTE_GS4,
        REST, NOTE_G4, NOTE_F4,
        NOTE_F4, NOTE_D4, NOTE_F4, NOTE_G4,

        //REPEAT 2
        NOTE_D4, NOTE_D4, NOTE_D5,
        NOTE_A4, REST, NOTE_GS4,
        REST, NOTE_G4, NOTE_F4, 
        NOTE_F4, NOTE_D4, NOTE_F4, NOTE_G4,
        //B
        NOTE_C4, NOTE_C4, NOTE_D5,
        NOTE_A4, REST, NOTE_GS4,
        REST, NOTE_G4, NOTE_F4,
        NOTE_F4, NOTE_D4, NOTE_F4, NOTE_G4,
        //C
        NOTE_A3, NOTE_A3, NOTE_D5,
        NOTE_A4, REST, NOTE_GS4,
        REST, NOTE_G4, NOTE_F4,
        NOTE_F4, NOTE_D4, NOTE_F4, NOTE_G4,
        //D 
        NOTE_AS3, NOTE_A3, NOTE_D5,
        NOTE_A4, REST, NOTE_GS4,
        REST, NOTE_G4, NOTE_F4,
        NOTE_F4, NOTE_D4, NOTE_F4, NOTE_G4,

        //siguiente f   
        NOTE_AS3, NOTE_A3, NOTE_D5,
        NOTE_A4, REST, NOTE_GS4,
        REST, NOTE_G4, NOTE_F4,
        NOTE_F4, NOTE_D4, NOTE_F4, NOTE_G4,
        // G
        NOTE_C5, NOTE_C5, NOTE_C6,
        NOTE_A4, REST, NOTE_GS6,REST,
        NOTE_G5, NOTE_F5,
        NOTE_F5,NOTE_C5,NOTE_F5,NOTE_G5,
        //H
        NOTE_C5, NOTE_C5, NOTE_C6,
        NOTE_A4, REST, NOTE_GS6,REST,
        NOTE_G5, NOTE_F5,
        NOTE_F5,NOTE_C5,NOTE_F5,NOTE_G5,
        //x2
        NOTE_C5, NOTE_C5, NOTE_C6,
        NOTE_A4, REST, NOTE_GS6,REST,
        NOTE_G5, NOTE_F5,
        NOTE_F5,NOTE_C5,NOTE_F5,NOTE_G5,
        //H
        NOTE_C5, NOTE_C5, NOTE_C6,
        NOTE_A4, REST, NOTE_GS6,REST,
        NOTE_G5, NOTE_F5,
        NOTE_F5,NOTE_C5,NOTE_F5,NOTE_G5,





    };

    static const uint8_t rhythm[] = {
        // Frase A
        E, E, Q,
        Q, E, E, 
        E,Q,E,
        E,E,E,E,
        /// Frase B
        E, E, Q,
        Q, E, E, 
        E,Q,E,
        E,E,E,E,
        //FRASE C
        E, E, Q,
        Q, E, E, 
        E,Q,E,
        E,E,E,E,
        //FRASE D
        E, E, Q,
        Q, E, E, 
        E,Q,E,
        E,E,E,E,

        //REPEAT 2
        E, E, Q,
        Q, E, E, 
        E,Q,E,
        E,E,E,E,
        /// Frase B
        E, E, Q,
        Q, E, E, 
        E,Q,E,
        E,E,E,E,
        //FRASE C
        E, E, Q,
        Q, E, E, 
        E,Q,E,
        E,E,E,E,
        //FRASE D
        E, E, Q,
        Q, E, E, 
        E,Q,E,
        E,E,E,E,
        //siguiente f   
        E, E, Q,  
        E, E, Q,
        Q, E, E, 
        E,Q,E,
        E,E,E,E,
        // G
        E,E,Q,
        Q,E,E,E,
        Q,E,
        E,E,E,E,
        //H
        E,E,Q,
        Q,E,E,E,
        Q,E,
        E,E,E,E,
        //x2
        E,E,Q,
        Q,E,E,E,
        Q,E,
        E,E,E,E,
        //H
        E,E,Q,
        Q,E,E,E,
        Q,E,
        E,E,E,E


        
        

    };


    // ====== main ======
    int main() {
        stdio_init_all();
        gpio_init(BUZZER_PIN);
        gpio_set_dir(BUZZER_PIN, true);

        const int notes = sizeof(melody) / sizeof(melody[0]);

        while (true) {
            for (int i = 0; i < notes; ++i) {
                play(melody[i], rhythm[i]);
            }
            sleep_ms(800);  // breve pausa antes de repetir
        }
        return 0;
    }





---


## Explicación

# ¿Qué hace `play(freq, figure_div)`?

Función que **toca una nota** (o un silencio) en un buzzer mediante PWM durante el tiempo correspondiente a la figura rítmica.

## Parámetros
- `freq`: frecuencia de la nota en Hz. Si es `REST`, se reproduce **silencio**.
- `figure_div`: divisor de la figura musical (1=redonda, 2=blanca, 4=negra, 8=corchea, etc.).

## Flujo de ejecución
1. **Duración**: calcula `dur_ms = ms_per_whole() / figure_div`, usando el BPM para obtener la duración de una redonda y dividirla por la figura indicada.
2. **Silencio (REST)**:
   - Si `freq == REST`, apaga el PWM (`pwm_set_freq_duty(..., 0, 0)`) y espera `dur_ms`.
3. **Nota**:
   - Enciende el PWM con la **frecuencia `freq`** y **duty 50%** (`pwm_set_freq_duty(BUZZER_PIN, freq, 50.0f)`), generando una onda cuadrada simétrica.
   - Mantiene la nota con `sleep_ms(dur_ms)`.
   - **Apaga** el PWM al final para garantizar silencio.

## Claves
- La **frecuencia** define el **pitch** (altura de la nota).
- El **duty 50%** ofrece buen equilibrio de volumen/timbre para un buzzer pasivo.
- Implementación **bloqueante**: `sleep_ms(...)` detiene la CPU durante la nota/silencio.
- Versión **mínima**: no incluye micro-silencios de articulación ni envolventes (ataque/decay).

## Referencias  

## Video

<div style="position:relative;padding-bottom:56.25%;height:0;overflow:hidden;">
  <iframe
    src="https://www.youtube.com/embed/bLRwLpEDBk8"
    title="YouTube video"
    allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
    allowfullscreen
    style="position:absolute;top:0;left:0;width:100%;height:100%;border:0;">
  </iframe>
</div>


- Raspberry Pi Ltd. (2025). *RP2040 Datasheet*. Disponible en: [https://www.raspberrypi.com/documentation/microcontrollers/rp2040.html](https://www.raspberrypi.com/documentation/microcontrollers/rp2040.html)  
- Código de práctica en C (PWM con buzzer), desarrollado por el autor (2025).  
- Espressif Systems, STMicroelectronics, NXP, Microchip, Texas Instruments – Datasheets oficiales de referencia.  

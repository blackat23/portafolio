# Control de Servomotor con Lista de Ángulos y 3 Modos (Raspberry Pi Pico / Pico 2)

> Programa para **grabar**, **reproducir** y **navegar** una lista de ángulos (0–180°) para un **servomotor** usando PWM en el RP2040.  
> La interfaz se realiza por **terminal USB** (stdio), **botones** y **UART0** configurado (listo para depuración/expansión).

---

## 1) Resumen

- **Nombre del proyecto:** _Servo List Runner (3 modos)_  
- **Autor:** _Antonio Martínez_  
- **Curso / Asignatura:** _Sistemas Embebidos_  
- **Fecha:** _22/10/2025_  
- **Descripción breve:** Control de servo por PWM a ~50 Hz; comandos por consola para llenar una lista de ángulos y dos botones para navegar entre elementos.

!!! tip "Información del proyecto"
    - **Lenguaje:** C++ (Pico SDK, usa `std::string`)
    - **MCU:** Raspberry Pi Pico / Pico 2 (RP2040)
    - **Librerías:** `pico/stdlib.h`, `hardware/pwm.h`, `hardware/uart.h`, `<string>`, `stdio`
    - **Estrategia general:**
      - **Modo 0 (Escritura):** desde la consola, comando `Escribir,` seguido de ángulos separados por comas y terminados con `;` o `Enter`.  
      - **Modo 1 (Lectura):** recorre y mueve el servo por todos los ángulos almacenados, con pausa visible.  
      - **Modo 2 (Navegación):** mueve el servo **manualmente** por la lista con **btn_next** / **btn_prev**.  
      - **btn_mode:** interrupción para alternar de **0 → 1 → 2 → 0** (con mensaje por `printf`).

### Material utilizado
- Raspberry Pi Pico / Pico 2  
- 1 **servomotor** (estándar) conectado a **GPIO 15** (señal), Vcc 5 V (o 3.3 V según servo) y GND común  
- 3 **botones** (NO) con **pull-up interno**:  
  - `btn_mode` → **GPIO 16** (cambio de modo)  
  - `btn_next` → **GPIO 17** (siguiente)  
  - `btn_prev` → **GPIO 18** (anterior)  
- Fuente y cableado adecuados  
- PC con Pico SDK (compilación CMake)

---

## 2) Objetivos

- Configurar **PWM** en el RP2040 para generar pulsos de **1–2 ms** dentro de un periodo de **20 ms** (~50 Hz) para servos.  
- Implementar una **lista de ángulos** con **validación (0–180)**.  
- Diseñar **tres modos de operación** con cambio por **interrupción GPIO**.  
- Practicar **parsing de comandos** vía consola y **navegación por botones** con detección de flanco.

---

## 3) Circuito

- **Servo:** señal en **GPIO 15**, GND común con el Pico, alimentación del servo acorde a su especificación (idealmente fuente separada con tierras comunes).  
- **Botones:** `btn_mode=GP16`, `btn_next=GP17`, `btn_prev=GP18` → configurar como entrada con `gpio_pull_up()`. El otro terminal de cada botón a **GND**.  
- **UART0 (para depuración/expansión):** `TX=GP0`, `RX=GP1` (cruzado si conectas a otro dispositivo).

> Con **pull-up**, el botón en reposo lee **1** y al presionar baja a **0**. En este proyecto, `btn_mode` usa **IRQ por flanco de bajada**.

---

## 4) Código (referencia)



    #include "pico/stdlib.h"
    #include "hardware/uart.h"
    #include <stdio.h>
    #include <string>
    #include "hardware/pwm.h"
    #define UART_ID uart0
    #define BAUD_RATE 115200
    #define TX_PIN 0
    #define RX_PIN 1
    #define btn_mode 16
    #define btn_next 17
    #define btn_prev 18
    #define PWM_PIN 0
    #define SERVO_PIN 15
    using namespace std;
    #define list 10

    volatile int programa = 0;

    uint16_t angle_to_pwm(float angle) {
        float pulse_ms = 1.0f + (angle / 180.0f);   // 1.0 → 2.0 ms
        float duty = (pulse_ms / 20.0f) * 39062.0f; // 20 ms periodo total
        return (uint16_t)duty;
    }

    // --- Rutina de interrupción ---
    static void button_isr(uint gpio, uint32_t events) {
        if (gpio == btn_mode && (events & GPIO_IRQ_EDGE_FALL)) { // flanco de bajada (presionado)
            programa++;
            if (programa > 2) { programa = 0; }
            if (programa == 0)      printf("Modo de Escritura Activado\n");
            else if (programa == 1) printf("Modo de Lectura Activado\n");
            else if (programa == 2) printf("Modo de Navegación Activado\n");
            sleep_ms(500); // (anti-rebote básico, ojo: dentro de ISR)
        }
    }

    int main() {
        int pos = 0;
        stdio_init_all();
        uart_init(UART_ID, BAUD_RATE);
        uart_set_format(UART_ID, 8, 1, UART_PARITY_NONE);

        gpio_set_function(SERVO_PIN, GPIO_FUNC_PWM);
        uint slice = pwm_gpio_to_slice_num(SERVO_PIN);
        pwm_set_clkdiv(slice, 64.0f);
        pwm_set_wrap(slice, 39062);
        pwm_set_enabled(slice, true);

        int next1 = 0;
        int prev1 = 0;

        // --- Configuración de botones ---
        gpio_init(btn_mode); gpio_set_dir(btn_mode, GPIO_IN); gpio_pull_up(btn_mode);
        gpio_init(btn_next); gpio_set_dir(btn_next, GPIO_IN); gpio_pull_up(btn_next);
        gpio_init(btn_prev); gpio_set_dir(btn_prev, GPIO_IN); gpio_pull_up(btn_prev);

        // --- Interrupción ---
        gpio_set_irq_enabled_with_callback(btn_mode, GPIO_IRQ_EDGE_FALL, true, &button_isr);

        string p1 = "";
        int lista[list] = {720,720,720,720,720,720,720,720,720,720}; // 720 = vacío
        int i = -1;

        while (true) {
            int pos = 0;

            if (programa == 0) { // --- Escritura ---
                int ch = getchar_timeout_us(0);
                if (ch != PICO_ERROR_TIMEOUT) {
                    if (ch == ',') {
                        if (p1 == "Escribir" || p1 == "write" || p1 == "WRITE" ||
                            p1 == "escribir" || p1 == "Write" || p1 == "ESCRIBIR") {
                            p1 = "";
                            int contador = 0;
                            for(int k=0;k<list;k++) if (lista[k]!=720) contador++;
                            if (contador == list) {
                                printf("Lista llena, no se pueden agregar más elementos.\n");
                                p1 = "";
                                continue;
                            } else {
                                i++;
                                while (ch != ';' && ch != '\n') {
                                    ch = getchar_timeout_us(0);
                                    if (ch != PICO_ERROR_TIMEOUT) {
                                        if (ch == ',') {
                                            printf("Eco: %s\n", p1.c_str());
                                            if(stoi(p1)>180 || stoi(p1)<0){
                                                printf("Valor fuera de rango (0-180).\n");
                                                i--;
                                            } else {
                                                lista[i] = stoi(p1);
                                                i++;
                                                p1 = "";
                                            }
                                        } else if (ch == ';' || ch == '\n') {
                                            if(stoi(p1)>180 || stoi(p1)<0){
                                                printf("Valor fuera de rango (0-180).\n");
                                                i--;
                                            } else {
                                                lista[i] = stoi(p1);
                                                p1 = "";
                                            }
                                        } else {
                                            p1 += (char)ch;
                                        }
                                        // imprimir lista actual
                                        printf("Lista: ");
                                        for (int k=0;k<=i;++k){ printf("%d%s", lista[k], (k+1<=i)?", ":""); }
                                        printf("\n");
                                    }
                                }
                            }
                        } else if (p1 == "Remplazar" || p1 == "replace" || p1 == "REPLACE" ||
                                p1 == "remplazar" || p1 == "REMPLAZAR") {
                            p1 = ""; // (reservado para extensión)
                        }
                    }

                    if (ch == ';' || ch == '\n') {
                        if (p1 == "Borrar" || p1 == "delete" || p1 == "DELETE" ||
                            p1 == "borrar" || p1 == "BORRAR") {
                            for (int k=0;k<list;k++) lista[k]=720;
                            i = -1;
                            printf("Ok, Lista borrada\n");
                        }
                        p1 = "";
                    } else {
                        p1 += (char)ch;
                    }
                }
            } 
            else if (programa == 1) { // --- Lectura secuencial ---
                while(programa == 1){
                    int contador = 0;
                    for(int k=0;k<list;k++) if (lista[k]!=720) contador++;
                    if (contador == 0) {
                        printf("La lista está vacía. Cambie al modo de escritura para agregar elementos.\n");
                        sleep_ms(1500);
                    } else {
                        int j = 0;
                        while(programa == 1 && j < contador){
                            printf("posición: %d , valor: %d\n", j, lista[j]);
                            pwm_set_gpio_level(SERVO_PIN, angle_to_pwm(lista[j]));
                            printf("Ángulo: %.1f°\n", lista[j]);
                            sleep_ms(1500);
                            j++;
                        }
                    }
                }          
            } 
            else if (programa == 2) { // --- Navegación manual ---
                while (programa == 2) {
                    if (lista[pos] == 720) pos--;
                    int contador = 0;
                    for(int k=0;k<list;k++) if (lista[k]!=720) contador++;

                    // botón siguiente (detección de flanco)
                    if (next1 == 0 && gpio_get(btn_next) == 1){
                        if (contador == 0) {
                            printf("Lista Vacia, no se puede avanzar.\n");
                            sleep_ms(100);
                        } else {
                            pos++;
                            printf("siguiente \n");
                            if (lista[pos] == 720) { printf("Valor no válido, regresando a %d\n", pos-1); pos--; }
                            if (pos == list) pos = 0;
                            printf("posición: %d , valor: %d\n", pos, lista[pos]);
                            pwm_set_gpio_level(SERVO_PIN, angle_to_pwm(lista[pos]));
                            printf("Ángulo: %.1f°\n", lista[pos]);
                            sleep_ms(200);
                        }
                    } 
                    next1 = gpio_get(btn_next);

                    // botón anterior (detección de flanco)
                    if (prev1 == 0 && gpio_get(btn_prev) == 1) {
                        if (contador == 0) {
                            printf("Lista Vacia, no se puede avanzar.\n");
                            sleep_ms(100);
                        } else {
                            pos--;
                            printf("anterior \n");
                            if (pos == -1) { printf("Valor no válido, regresando a %d\n", pos+1); pos++; }
                            if (pos == list) pos = 0;
                            printf("posición: %d , valor: %d\n", pos, lista[pos]);
                            pwm_set_gpio_level(SERVO_PIN, angle_to_pwm(lista[pos]));
                            printf("Ángulo: %.1f°\n", lista[pos]);
                            sleep_ms(200);
                        }
                    }
                    prev1 = gpio_get(btn_prev);
                }
            }
        }
        return 0;
    }

## 5) Explicación del programa

### a) Definiciones y mapeo de pines
- **Servo PWM:** `SERVO_PIN = GP15` (función PWM)
- **Botones:** `btn_mode = GP16`, `btn_next = GP17`, `btn_prev = GP18` con **pull-up**
- **UART0:** `TX = GP0`, `RX = GP1`, `BAUD_RATE = 115200`, formato **8N1**

### b) PWM para servomotor
Se configura el *slice* correspondiente a **GP15** con:
- `clkdiv = 64.0f`
- `wrap = 39062` → periodo ≈ **20 ms** (**50 Hz**)

`angle_to_pwm(angle)` mapea **0–180°** a **1.0–2.0 ms** de pulso:
- `pulse_ms = 1.0 + angle/180.0`
- `duty = (pulse_ms / 20.0) * 39062`

### c) Estructuras de datos
- `lista[10]` almacena hasta **10 ángulos** (enteros).
- **Marcador de vacío:** `720`.
- Índice `i` lleva el **último índice válido** (inicia en `-1`).

### d) Máquina de estados (modos)
**Modo 0 – Escritura (`programa == 0`)**
- Comando de entrada: `Escribir,` (admite variantes de mayúsculas/minúsculas).
- Luego, números separados por **comas** y terminados con `;` o **Enter**.
- Cada número se **valida (0–180)** antes de guardarse; si es inválido, se descarta y se ajusta `i`.
- `Borrar;` limpia la lista (rellena con `720`).

**Modo 1 – Lectura**
- Recorre automáticamente los elementos **válidos** de `lista` y mueve el servo; pausa de **1.5 s** entre pasos.

**Modo 2 – Navegación**
- Avance/retroceso manual con **`btn_next`** y **`btn_prev`** usando **detección de flanco** (variables `next1`/`prev1`).
- Muestra **posición** y **valor**, aplica el ángulo al servo y espera **200 ms**.

## 6) Video Demostracion:

<div style="position:relative;padding-bottom:56.25%;height:0;overflow:hidden;">
  <iframe
    src="https://www.youtube.com/embed/-4OjMs1xZ-A"
    title="YouTube Short"
    allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
    allowfullscreen
    style="position:absolute;top:0;left:0;width:100%;height:100%;border:0;">
  </iframe>
</div>

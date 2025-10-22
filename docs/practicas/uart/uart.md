## UART Eco y Control LED entre dos Pico (Raspberry Pi Pico / Pico 2)
> Este proyecto implementa **comunicación serie UART** entre una Raspberry Pi Pico y otro dispositivo (otro Pico, PC con adaptador USB–TTL, etc.).  
> El programa **eco** reconstruye texto recibido por USB (stdio) y lo reenvía por **UART0** cuando detecta fin de línea (enter o `.`).  
> Además, **recibe comandos por UART** para controlar un LED (`LEDON` / `LEDOFF`) y **envía “LEDON”** cuando detecta la pulsación de un botón.

---

## 1) Resumen

- **Nombre del proyecto:** _UART Eco + LED Control_  
- **Autor:** _Antonio Martínez_  
- **Curso / Asignatura:** _Sistemas Embebidos_  
- **Fecha:** _21/10/2025_  
- **Descripción breve:** Comunicación UART a 115200 baudios con eco, reconstrucción de mensajes y control de LED mediante comandos de texto.

!!! tip "Información del proyecto"
    - **Lenguaje:** C++ (Pico SDK, usa `std::string`)  
    - **MCU:** Raspberry Pi Pico / Pico 2 (RP2040)  
    - **Librerías:** `pico/stdlib.h`, `hardware/uart.h`, `<string>`, `stdio`  
    - **Estrategia:**  
      - USB stdio → reconstruye `p` y envía por **UART0** al recibir `.` o `\n`.  
      - **UART0 RX** → reconstruye `c` y, al cerrar línea, interpreta: `LEDON` / `LEDOFF` / inválido.  
      - **Botón** → al presionar, transmite `"LEDON\n"` por UART0.

### Material utilizado
- Raspberry Pi Pico / Pico 2  
- 1 LED con resistencia **220–330 Ω** en **GPIO 16**  
- 1 botón (NO) en **GPIO 17** con **pull-up interno**  
- Cableado y protoboard  
- **Si conectas dos Picos por UART:** GP0↔GP1 cruzado (TX→RX, RX→TX) y GND común

---

## 2) Objetivos

- **Configurar UART0** en RP2040 (115200-8N1).  
- **Distinguir flujos**: USB stdio (teclado/terminal) vs UART hardware.  
- **Reconstruir mensajes** por caracteres y delimitar por `.` o `\n`.  
- **Implementar un mini-protocolo** de texto (`LEDON`, `LEDOFF`).  
- **Leer GPIO** (botón con pull-up) y **enviar comandos** por UART en respuesta.  

---

## 3) Circuito

- **UART0:**  
  - `TX_PIN` → **GPIO 0 (TX0)**  
  - `RX_PIN` → **GPIO 1 (RX0)**  
  - Conecta **TX de un dispositivo al RX del otro** y **GND común**.  
- **LED de usuario:** **GPIO 16** (con resistencia en serie a GND si activas con nivel alto).  
- **Botón:** **GPIO 17** con `gpio_pull_up()`; el otro terminal a **GND**.  
- **Alimentación:** USB del Pico.

---

## 4) Código

    ```cpp
    #include "pico/stdlib.h"
    #include "hardware/uart.h"
    #include <stdio.h>
    #include <string>

    #define UART_ID uart0
    #define BAUD_RATE 115200
    #define TX_PIN 0
    #define RX_PIN 1
    #define button_pin 17
    #define led_PIN 16
    using namespace std; // USO DE STRING en la terminal 

    int main() {
        stdio_init_all();

        gpio_set_function(TX_PIN, GPIO_FUNC_UART); // DEFINE TX Y RX
        gpio_set_function(RX_PIN, GPIO_FUNC_UART);

        uart_init(UART_ID, BAUD_RATE); // VELOCIDAD DE TRANSMISIÓN
        uart_set_format(UART_ID, 8, 1, UART_PARITY_NONE); // 8N1

        gpio_init(button_pin);
        gpio_set_dir(button_pin, GPIO_IN);
        gpio_pull_up(button_pin);
        gpio_init(led_PIN);
        gpio_set_dir(led_PIN, GPIO_OUT);

        string c = ""; // RECONSTRUCCIÓN de la palabra recibida por UART
        string p = ""; // RECONSTRUCCIÓN de la palabra recibida por USB stdio

        // Estado previo del botón para detectar flanco
        int prev = 1;

        while (true){
            // ——— Entrada por USB stdio ———
            int ch = getchar_timeout_us(0); // LEE UN CARÁCTER (no bloqueante)
            if (ch != PICO_ERROR_TIMEOUT) { 
                printf("Eco: %c\n", (char)ch); 
                p += (char)ch; // reconstruye palabra

                // Al cerrar línea, envía por UART
                if (ch == '.' || ch == '\n'){
                    uart_puts(UART_ID, p.c_str()); // envía a UART (otro Pico)
                    p.clear();
                }
            }

            // ——— Botón: envío de comando por UART (detección de flanco) ———
            int now = gpio_get(button_pin);
            if (prev == 1 && now == 0) { // transición alto->bajo
                printf("Button pressed!\n");
                uart_puts(UART_ID, "LEDON\n");
                sleep_ms(200); // antirrebote básico
            }
            prev = now;

            // ——— Entrada por UART hardware ———
            if (uart_is_readable(UART_ID)) {
                char character = uart_getc(UART_ID); // LEE UN CARÁCTER
                printf("%c\n", character); // imprime recibido
            
                if (character == '\n' || character == '.'){
                    if (c == "LEDON"){
                        gpio_put(led_PIN, 1);
                        printf("LED is ON\n");
                    }
                    else if (c == "LEDOFF"){
                        gpio_put(led_PIN, 0);
                        printf("LED is OFF\n");
                    } 
                    else{
                        uart_puts(UART_ID, "Invalid Command\n");
                    }
                    c.clear();
                } else {
                    c += character;
                }
            }
        }
    }
## 5) Explicación del programa

### a) Definiciones y mapeo de pines
- **UART0:** `TX_PIN=GP0`, `RX_PIN=GP1`, `BAUD_RATE=115200`, formato **8N1**.
- **LED:** `led_PIN=GP16` como salida.
- **Botón:** `button_pin=GP17` con **pull-up** interno (reposo = 1, presionado = 0).

### b) Variables clave
- `std::string p`: reconstruye el mensaje proveniente de **USB stdio** (teclado/terminal).
- `std::string c`: reconstruye el mensaje recibido por **UART0**.
- `prev/now`: detección de **flanco** de botón (alto → bajo).

### c) Flujo por USB stdio → UART
1. Se lee un carácter con `getchar_timeout_us(0)` (no bloqueante).
2. Se hace “eco” (`printf("Eco: %c\n", ...)`) y se concatena a `p`.
3. Si el carácter es `.` o `\n`, se considera **fin de mensaje** y se envía `p` por **UART0** con `uart_puts`. Luego `p` se limpia.

### d) Flujo por UART → acciones
1. `uart_is_readable()` indica si hay datos en UART0.
2. Se lee un carácter con `uart_getc()` y se concatena a `c`.
3. Al recibir `.` o `\n`, se interpreta `c`:
   - `LEDON` → enciende **GP16**.
   - `LEDOFF` → apaga **GP16**.
   - Otro texto → responde `"Invalid Command\n"` por UART.  
   Se limpia `c`.

### e) Botón
- Se detecta **flanco alto → bajo** (pull-up).
- Al presionar, se transmite por UART: `"LEDON\n"`.
- `sleep_ms(200)` añade **antirrebote** básico.

---

## 6) Procedimiento de ejecución
1. Compilar como **C++** (usa `<string>`). Asegúrate de que el archivo tenga extensión `.cpp` y que tu `CMakeLists.txt` agregue `pico_stdlib` y `hardware_uart`.
2. Habilita **USB stdio** si usarás la consola por USB.
3. **Flashea** el binario en el Pico.
4. **Conecta UART** si usarás dos Picos: cruza **TX↔RX** y comparte **GND**.
5. Abre un **terminal USB** (ej. `115200-8N1`) y escribe texto: al poner `.` o Enter, se enviará por UART.
6. Envía por **UART** desde el otro dispositivo: escribe `LEDON\n` o `LEDOFF\n` para controlar el LED del Pico.

---

## 7) Resultados esperados
- Al teclear en el terminal USB, verás líneas tipo `Eco: X` y, al finalizar con `.` o Enter, el Pico enviará esa cadena por UART.
- Si por UART llega `LEDON` (terminado en `.` o `\n`), se enciende el LED en **GP16** y se imprime `LED is ON`.
- Si llega `LEDOFF`, se apaga y se imprime `LED is OFF`.
- Cualquier otro comando retorna por UART `"Invalid Command\n"`.

---

## 8) Problemas comunes y mejoras recomendadas
- **Antirrebote y flanco:** usar estado previo `prev` y no variables sin inicializar.
- **Formateo de impresión:** usar `printf("%c\n", character);` en lugar de sumar punteros.
- **Consistencia con `UART_ID`:** usar `UART_ID` en todas las llamadas.
- **Terminadores:** asegurarse de finalizar mensajes con `\n` o `.` para disparar el parseo.

---

## 9) Conclusiones
- Se implementó una **comunicación UART** con separación clara **USB/UART**.
- Un **protocolo textual mínimo** permite controlar hardware (LED).
- Con mejoras de flancos y antirrebote, el sistema queda robusto y extensible.

---

## 10) Preguntas de control
1. ¿Por qué es necesario terminar los mensajes con `\n` o `.`?
2. ¿Qué diferencia hay entre **USB stdio** y **UART hardware** en el Pico?
3. ¿Cómo se cruzan correctamente las líneas **TX/RX** entre dos dispositivos UART?
4. ¿Cómo detectar correctamente el **flanco** de un botón con pull-up?
5. ¿Qué ocurriría si no inicializas una variable antes de usarla en una condición?

---


## 11) Video de demostración



<div style="position:relative;padding-bottom:56.25%;height:0;overflow:hidden;">
  <iframe
    src="https://www.youtube.com/embed/6N8PV1zc3Wk"
    title="YouTube Short"
    allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
    allowfullscreen
    style="position:absolute;top:0;left:0;width:100%;height:100%;border:0;">
  </iframe>
</div>

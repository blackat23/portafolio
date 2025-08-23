#  Comparativa de Microcontroladores

: Periféricos, Memoria, Ecosistema, Costos, Arquitectura y Velocidad de Trabajo


---

## Informacion
- **Nombre del proyecto:** _Comparativa de Microcontroladores_
- **Equipo / Autor(es):** _Antonio Martínez_
- **Curso / Asignatura:** _Sistemas Embebidos_
- **Fecha:** _23/08/2025_
- **Descripción breve:** _Análisis comparativo de microcontroladores de distintas marcas considerando periféricos, memoria, ecosistema, costos, arquitectura y velocidad de trabajo._


---
## Introducion  
# Sobre los microcontroladores:

En el desarrollo de sistemas embebidos y proyectos electrónicos, la elección del microcontrolador adecuado es un paso crítico.  
Cada microcontrolador se diferencia en varios aspectos técnicos y prácticos que determinan su rendimiento, facilidad de uso y costo.  
A continuación, se explican los criterios utilizados en la tabla comparativa:

- **Periféricos:** Son los módulos de hardware integrados que permiten la interacción del microcontrolador con el entorno. Incluyen interfaces de comunicación (UART, SPI, I²C, USB, CAN), convertidores analógico-digital (ADC), generadores de PWM, temporizadores y otros. Su disponibilidad define qué tan versátil puede ser el dispositivo en una aplicación específica.

- **Memoria:** Se refiere a la capacidad de almacenamiento integrada.  
  - **Flash:** Guarda el programa principal (firmware).  
  - **SRAM:** Es la memoria de trabajo usada durante la ejecución.  
  - **EEPROM o Flash externa:** Permite guardar datos de manera permanente, incluso sin energía.  

- **Ecosistema:** Hace referencia al conjunto de herramientas de software y librerías disponibles para programar y depurar el microcontrolador. Un ecosistema robusto (IDE, SDK, soporte en la comunidad) facilita el desarrollo, reduce la curva de aprendizaje y mejora la productividad.

- **Costos:** Indican el precio aproximado por unidad en volúmenes pequeños. El costo influye directamente en la viabilidad de un proyecto, sobre todo en aplicaciones de bajo presupuesto o producción masiva.

- **Arquitectura:** Describe el tipo de núcleo del procesador, ya sea de 8, 16 o 32 bits, y la familia a la que pertenece (por ejemplo, AVR, Cortex-M, Xtensa). La arquitectura determina las capacidades de procesamiento, eficiencia energética y compatibilidad con herramientas de desarrollo.

- **Velocidad de trabajo:** Corresponde a la frecuencia de reloj máxima (MHz) a la que puede operar el microcontrolador. A mayor velocidad, más operaciones por segundo, aunque también puede aumentar el consumo de energía.

Esta comparativa permite visualizar de manera clara las diferencias entre distintas marcas y familias de microcontroladores, ayudando a seleccionar el dispositivo más apropiado según las necesidades del proyecto.


##  tabla comparativa 

# Comparativa de Microcontroladores
| MCU | Perif. | Memoria | Ecosistema | $USD | Arq. | Vel. |
|-----|--------|---------|------------|------|------|------|
| ATmega328P | ADC10, UART, I²C, SPI, Timers, PWM | 32KB / 2KB / 1KB EEPROM | Arduino, MPLAB | ~2.7 | AVR 8-bit | 20 MHz |
| STM32F103C8 | ADC12, USART, I²C, SPI, USB, CAN | 64KB / 20KB | STM32CubeIDE | ~6 | Cortex-M3 | 72 MHz |
| LPC1768 | Eth, USB OTG, CAN, UART, I²C, SPI, I²S, ADC, DAC | 512KB / 64KB | MCUXpresso | ~14 | Cortex-M3 | 100 MHz |
| MSP430G2553 | ADC10, UART, SPI, I²C, Timers | 16KB / 512B | CCS | ~3.5 | MSP430 16-bit | 16 MHz |
| ESP32-WROOM | Wi-Fi, BT, ADC12, DAC, UART, SPI, I²C, PWM | 520KB SRAM / 4MB flash | ESP-IDF, Arduino | ~3–7 | Xtensa LX6 | 240 MHz |
| RP2040 | USB, UART, SPI, I²C, PWM, ADC, PIO | 264KB SRAM / 2MB flash | C/C++ SDK, MicroPython | ~4 | Cortex-M0+ | 133 MHz |



##  Resumen

La revisión comparativa de diferentes microcontroladores muestra que no existe un “mejor” dispositivo de manera absoluta, sino que cada uno responde a necesidades específicas de diseño.  

- Los **ATmega328P** y **MSP430** destacan por su simplicidad, bajo consumo y facilidad de aprendizaje, ideales para proyectos educativos o aplicaciones sencillas.  
- Los **STM32F103** y **LPC1768** ofrecen mayor potencia de procesamiento y una amplia variedad de periféricos, adecuados para aplicaciones industriales o que requieren comunicaciones avanzadas.  
- El **ESP32** resalta por integrar conectividad inalámbrica (Wi-Fi y Bluetooth) a bajo costo, siendo muy utilizado en proyectos de IoT.  
- El **RP2040** de Raspberry Pi se ha consolidado como una alternativa flexible y económica, con un ecosistema creciente que lo hace atractivo para prototipado y desarrollo educativo.  

En conclusión, la selección del microcontrolador debe basarse en un balance entre costo, recursos disponibles, complejidad del proyecto y facilidad de desarrollo. Conocer las diferencias en periféricos, memoria, ecosistema, arquitectura y velocidad de trabajo permite tomar decisiones informadas que optimicen tanto el desempeño como la viabilidad económica de un proyecto embebido.






##  Referencias


- Microchip Technology Inc. (2025). *ATmega328P Datasheet*. Disponible en: [https://www.microchip.com/en-us/product/ATmega328P](https://www.microchip.com/en-us/product/ATmega328P)  
- STMicroelectronics (2025). *STM32F103x8 Datasheet*. Disponible en: [https://www.st.com/en/microcontrollers-microprocessors/stm32f103.html](https://www.st.com/en/microcontrollers-microprocessors/stm32f103.html)  
- NXP Semiconductors (2025). *LPC1768 Product Data Sheet*. Disponible en: [https://www.nxp.com/part/LPC1768FBD100](https://www.nxp.com/part/LPC1768FBD100)  
- Texas Instruments (2025). *MSP430G2553 Datasheet*. Disponible en: [https://www.ti.com/product/MSP430G2553](https://www.ti.com/product/MSP430G2553)  
- Espressif Systems (2025). *ESP32-WROOM-32 Datasheet*. Disponible en: [https://www.espressif.com/en/products/modules/esp32](https://www.espressif.com/en/products/modules/esp32)  
- Raspberry Pi Ltd. (2025). *RP2040 Datasheet*. Disponible en: [https://www.raspberrypi.com/documentation/microcontrollers/rp2040.html](https://www.raspberrypi.com/documentation/microcontrollers/rp2040.html)  

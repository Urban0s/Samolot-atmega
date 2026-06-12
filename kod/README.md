# 💻 Oprogramowanie ATmega328P (Sterownik Samolotu)

Ten folder zawiera pełny kod źródłowy napisany w języku C dla mikrokontrolera ATmega328P. Każdy plik został opisany i wydzielony osobno, co ułatwia przeglądanie struktury projektu.

---

## 🛠️ Kod źródłowy plików

### 1. `main.c`
Serce całego oprogramowania. Odpowiada za inicjalizację systemów pokładowych oraz nadrzędną pętlę sterowania urządzeniami wykonawczymi.

<details>
<summary>👁️ Kliknij, aby zobaczyć kod main.c</summary>

```c
#include <avr/interrupt.h>
#include "uart.h"
#include "pwm.h"

int main(void) {
    uart_init();
    pwm_init();
    sei();

    while(1) {
        if (frame_ready) {
            frame_ready = 0;

            // Zbieranie i przeliczanie danych
            uint16_t aileron_1_ticks = rc_channels[0] * 2;
            uint16_t elevator_ticks  = rc_channels[1] * 2;
            uint16_t aileron_2_ticks = (3000 - rc_channels[0]) * 2;
            uint16_t rudder_ticks    = rc_channels[3] * 2;

            // Wysyłanie do modułu PWM
            pwm_set_hardware(aileron_1_ticks, elevator_ticks);
            pwm_set_software(aileron_2_ticks, rudder_ticks);
        }

        // Utrzymywanie programowego PWM przy życiu w pętli
        pwm_process_software();
    }
    
    return 0;
}
```
</details>

---

### 2. `makefile`
Skrypt konfiguracyjny dla narzędzia `make`, który automatyzuje proces kompilacji kodu źródłowego oraz budowania pliku wynikowego `.hex` dla programatora.

<details>
<summary>👁️ Kliknij, aby zobaczyć zawartość makefile</summary>

```makefile
NAME = main
SERIAL = COM3
ARDUINO = -F -V -c arduino -P $(SERIAL) -b 115200
USBASP = avrdude -c usbasp
AVRDUDE = avrdude $(ARDUINO)

DEF = -DF_CPU=16000000UL -D__AVR_ATmega328P__

SRC = \
main.c uart.c pwm.c

INC = \
-I.

.PHONY : build flash flase run erase

.DEFAULT_GOAL := run

run :
	@$(MAKE) build
	@$(MAKE) flash

build : $(NAME).elf
	avr-objcopy -j .text -j .data -O ihex $< $(NAME).hex

flash :
	$(AVRDUDE) -p ATMEGA328P -U flash:w:$(NAME).hex:i

flase : flash

erase :
	$(AVRDUDE) -p ATMEGA328P -e

$(NAME).elf : $(NAME).o
	avr-gcc -Os $(DEF) -mmcu=atmega328p $(SRC) $(INC) -o $@

$(NAME).o : $(SRC)
	avr-gcc -g -Os $(DEF) -mmcu=atmega328p $^ $(INC) -o $@

include assets/fetch.mk
```
</details>

---

### 3. `pwm.h`
Plik nagłówkowy modułu sterowania sygnałem PWM. Zawiera deklaracje funkcji oraz struktur potrzebnych do zarządzania silnikiem i serwomechanizmami.

<details>
<summary>👁️ Kliknij, aby zobaczyć kod pwm.h</summary>

```c
#ifndef PWM_H
#define PWM_H

#include <stdint.h>

void pwm_init(void);
void pwm_set_hardware(uint16_t ch1_ticks, uint16_t ch2_ticks);
void pwm_set_software(uint16_t ail2_ticks, uint16_t rudder_ticks);
void pwm_process_software(void);

#endif
```
</details>

---

### 4. `pwm.c`
Implementacja modułu PWM. Zawiera bezpośrednią konfigurację rejestrów sprzętowego licznika Timer1 w tryb Fast PWM (50 Hz) do obsługi ESC oraz serw SG90.

<details>
<summary>👁️ Kliknij, aby zobaczyć kod pwm.c</summary>

```c
#include "pwm.h"
#include <avr/io.h>

static volatile uint16_t ail2_ticks = 3000;
static volatile uint16_t rudder_ticks = 3000;

void pwm_init(void) {
    // Piny pod PWM sprzętowy: PB1 (9), PB2 (10)
    // Piny pod PWM programowy: PB3 (11), PD6 (6)
    DDRB |= (1 << PB1) | (1 << PB2) | (1 << PB3);
    DDRD |= (1 << PD6);

    TCCR1A = (1 << COM1A1) | (1 << COM1B1) | (1 << WGM11);
    TCCR1B = (1 << WGM13) | (1 << WGM12) | (1 << CS11);

    ICR1 = 39999; 
    OCR1A = 3000; 
    OCR1B = 3000;
}

void pwm_set_hardware(uint16_t ch1_ticks, uint16_t ch2_ticks) {
    OCR1A = ch1_ticks;
    OCR1B = ch2_ticks;
}

void pwm_set_software(uint16_t ail_t, uint16_t rud_t) {
    ail2_ticks = ail_t;
    rudder_ticks = rud_t;
}

// Funkcja odpowiedzialna za przełączanie pinów 11 i 6
void pwm_process_software(void) {
    uint16_t current_ticks = TCNT1;

    if (current_ticks < ail2_ticks) {
        PORTB |= (1 << PB3);
    } else {
        PORTB &= ~(1 << PB3);
    }

    if (current_ticks < rudder_ticks) {
        PORTD |= (1 << PD6);
    } else {
        PORTD &= ~(1 << PD6);
    }
}
```
</details>

---

### 5. `uart.h`
Plik nagłówkowy odpowiedzialny za konfigurację interfejsu szeregowego UART. Definiuje prędkość transmisji danych (Baudrate) oraz prototypy funkcji.

<details>
<summary>👁️ Kliknij, aby zobaczyć kod uart.h</summary>

```c
#ifndef UART_H_
#define UART_H_

#define BAUDRATE 9600
#define MYUBRR (((F_CPU / (BAUDRATE * 16UL))) - 1)

void uart_init(void);
void uart_putc(char data);
void uart_puts(const char* s);

#endif
```
</details>

---

### 6. `uart.c`
Implementacja modułu UART. Zawiera kod odpowiedzialny za inicjalizację rejestrów transmisji szeregowej oraz wysyłanie pojedynczych znaków i całych ciągów tekstowych na komputer.

<details>
<summary>👁️ Kliknij, aby zobaczyć kod uart.c</summary>

```c
#include <avr/io.h>
#include "uart.h"

void uart_init(void) {
    // Ustawienie prędkości transmisji Baudrate
    UBRR0H = (uint8_t)(MYUBRR >> 8);
    UBRR0L = (uint8_t)MYUBRR;
    
    // Aktywacja nadajnika (TX)
    UCSR0B |= (1 << TXEN0); 
    
    // Ustawienie formatu ramki: 8 bitów danych, 1 bit stopu
    UCSR0C |= (1 << UCSZ01) | (1 << UCSZ00); 
}

void uart_putc(char data) {
    // Oczekiwanie na zwolnienie bufora nadawczego
    while (!(UCSR0A & (1 << UDRE0))); 
    UDR0 = data;
}

void uart_puts(const char* s) {
    // Wysyłanie pełnego ciągu znaków
    while (*s) {
        uart_putc(*s++);
    }
}
```
</details>

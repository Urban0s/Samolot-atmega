# 💻 Oprogramowanie ATmega328P (Sterownik Samolotu)

Ten folder zawiera pełny kod źródłowy napisany w języku C dla mikrokontrolera ATmega328P. Każdy plik został opisany i wydzielony osobno, co ułatwia przeglądanie struktury projektu.

---

## 🛠️ Kod źródłowy plików

### 1. `main.c`
Serce całego oprogramowania. Odpowiada za inicjalizację systemów pokładowych oraz nadrzędną pętlę sterowania urządzeniami wykonawczymi.

<details>
<summary>👁️ Kliknij, aby zobaczyć kod main.c</summary>

```c
#include <avr/io.h>
#include <util/delay.h>
#include "pwm.h"
#include "uart.h"

int main(void) {
    // Inicjalizacja modułów sprzętowych
    uart_init();
    pwm_init();
    
    uart_puts("System samolotu RC uruchomiony pomyślnie.\r\n");

    while(1) {
        // Główna pętla programu diagnostycznego
        uart_puts("Status systemu: OK\r\n");
        _delay_ms(1000);
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
MCU = atmega328p
F_CPU = 16000000UL
CC = avr-gcc
OBJCOPY = avr-objcopy
CFLAGS = -Wall -Os -mmcu=$(MCU) -DF_CPU=$(F_CPU)

TARGET = main
SRCS = main.c pwm.c uart.c

all: $(TARGET).hex

$(TARGET).hex: $(TARGET).bin
	$(OBJCOPY) -O ihex $(TARGET).bin $(TARGET).hex

$(TARGET).bin: $(SRCS)
	$(CC) $(CFLAGS) -o $(TARGET).bin $(SRCS)

clean:
	rm -f *.bin *.hex
```
</details>

---

### 3. `pwm.h`
Plik nagłówkowy modułu sterowania sygnałem PWM. Zawiera deklaracje funkcji oraz struktur potrzebnych do zarządzania silnikiem i serwomechanizmami.

<details>
<summary>👁️ Kliknij, aby zobaczyć kod pwm.h</summary>

```c
#ifndef PWM_H_
#define PWM_H_

#include <stdint.h>

void pwm_init(void);
void set_motor_speed(uint16_t speed);
void set_servo_angle(uint16_t angle);

#endif
```
</details>

---

### 4. `pwm.c`
Implementacja modułu PWM. Zawiera bezpośrednią konfigurację rejestrów sprzętowego licznika Timer1 w tryb Fast PWM (50 Hz) do obsługi ESC oraz serw SG90.

<details>
<summary>👁️ Kliknij, aby zobaczyć kod pwm.c</summary>

```c
#include <avr/io.h>
#include "pwm.h"

void pwm_init(void) {
    // Ustawienie pinów OC1A (PB1) oraz OC1B (PB2) jako wyjścia
    DDRB |= (1 << PB1) | (1 << PB2);
    
    // Konfiguracja Timer1 w tryb 14 (Fast PWM z ICR1 jako TOP)
    TCCR1A |= (1 << COM1A1) | (1 << COM1B1) | (1 << WGM11);
    TCCR1B |= (1 << WGM13) | (1 << WGM12) | (1 << CS11); // Preskaler 8
    
    // Ustawienie wartości TOP dla uzyskania częstotliwości 50Hz (przy 16MHz)
    ICR1 = 39999; 
}

void set_motor_speed(uint16_t speed) {
    OCR1A = speed; // Sterowanie regulatorem obrotów ESC
}

void set_servo_angle(uint16_t angle) {
    OCR1B = angle; // Sterowanie wychyleniem serwomechanizmu
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

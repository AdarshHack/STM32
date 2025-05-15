#define F_CPU 16000000UL
#include <avr/io.h>
#include <avr/interrupt.h>
#include <util/delay.h>
#include <stdbool.h>

//Step sequence for full-stepping
const uint8_t step_sequence[4] = {
    0b00001001,
    0b00000011,
    0b00000110,
    0b00001100
};

// Pin Definitions
#define BUTTON_START_STOP  PC0
#define BUTTON_SPEED_DIR   PC1

// Global State
volatile uint8_t step_index = 0;
volatile uint16_t step_count = 0;
volatile uint8_t revolution_count = 0;
volatile bool motor_running = false;
volatile bool motor_direction = true;  // true = forward
volatile uint8_t speed_level = 0;      // 0=slow, 1=medium, 2=fast

// Timer overflow counter for timing steps
volatile uint8_t timer_counter = 0;
volatile uint8_t speed_thresholds[3] = {10, 5, 2}; // Speed control thresholds

// Long press variable
uint16_t long_press_count = 0;

// Step motor function
void step_motor(uint8_t index) {
    PORTD = (PORTD & 0xF0) | (step_sequence[index] & 0x0F);
}

// Check if button is pressed 
bool is_button_pressed(uint8_t pin) {
    return !(PINC & (1 << pin));
}

// Timer0 Overflow Interrupt (~1ms)
ISR(TIMER0_OVF_vect) {
    if (!motor_running) return;

    timer_counter++;
    if (timer_counter >= speed_thresholds[speed_level]) {
        timer_counter = 0;

        step_motor(step_index);
        if (motor_direction)
            step_index = (step_index + 1) % 4;
        else
            step_index = (step_index == 0) ? 3 : step_index - 1;

        step_count++;
        if (step_count >= 2000) {
            step_count = 0;
            revolution_count++;
            if (revolution_count > 15) revolution_count = 0;
        }
    }
}

// Setup Timer0 for overflow interrupt
void setup_timer0() {
    TCCR0A = 0x00;
    TCCR0B = (1 << CS01) | (1 << CS00); // Prescaler 64
    TIMSK0 = (1 << TOIE0);              // Enable overflow interrupt
    TCNT0 = 0;
}

int main(void) {
    // Stepper: PD0–PD3 output
    DDRD |= 0x0F;

    // Buttons: PC0 and PC1 input with pull-up
    DDRC &= ~((1 << BUTTON_START_STOP) | (1 << BUTTON_SPEED_DIR));
    PORTC |= (1 << BUTTON_START_STOP) | (1 << BUTTON_SPEED_DIR);

    sei();             // Enable global interrupts
    setup_timer0();    // Initialize Timer0

    while (1) {
        // ---- Button 1: Start/Stop ----
        if (is_button_pressed(BUTTON_START_STOP)) {
            _delay_ms(50); // debounce
            if (is_button_pressed(BUTTON_START_STOP)) {
                motor_running = !motor_running;

                // Wait until release (debounced)
                while (is_button_pressed(BUTTON_START_STOP));
                _delay_ms(50);
            }
        }

        // ---- Button 2: Speed or Direction ----
        if (is_button_pressed(BUTTON_SPEED_DIR)) {
            long_press_count = 0;

            // Count how long button is held
            while (is_button_pressed(BUTTON_SPEED_DIR)) {
                _delay_ms(50);
                long_press_count += 50;
                if (long_press_count >= 1000) { // Long press
                    motor_direction = !motor_direction;

                    while (is_button_pressed(BUTTON_SPEED_DIR));
                    _delay_ms(50);
                   // break;
                }
            }

            if (long_press_count < 1000) { // Short press
                speed_level = (speed_level + 1) % 3;

                while (is_button_pressed(BUTTON_SPEED_DIR));
                _delay_ms(50);
            }
        }
    }
}

# Sleep-moode
#include <LiquidCrystal.h>
#include "Adafruit_ZeroFFT.h"

// Configuración para el LCD
LiquidCrystal lcd(12, 14, 27, 26, 25, 33);  // Pines para el LCD

// Configuración de FFT
#define DATA_SIZE 256  // El tamaño de los datos debe ser potencia de 2 entre 16 y 2048
#define FS 8000        // Frecuencia de muestreo (Hz)

volatile unsigned long pulseCount = 0;  // Contador de pulsos
unsigned long previousMillis = 0;        // Tiempo previo para medir la frecuencia
int16_t fftInput[DATA_SIZE];              // Array para almacenar las muestras de FFT

// Configuración del botón
const int buttonPin = 35;  // Pin para el botón
volatile bool sleepMode = false;  // Estado del modo de suspensión

// Interrupción para contar los pulsos
void IRAM_ATTR countPulse() {
  pulseCount++;
}

// Interrupción para alternar el modo de suspensión
void IRAM_ATTR toggleSleepMode() {
  sleepMode = !sleepMode;  // Alternar el estado del modo de suspensión
}

void setup() {
  lcd.begin(16, 2);  // Inicializar el LCD
  lcd.print("Frecuencia:");

  pinMode(34, INPUT);  // Configurar el pin para la entrada de pulsos
  attachInterrupt(digitalPinToInterrupt(34), countPulse, RISING);  // Interrupción por flanco ascendente

  pinMode(buttonPin, INPUT_PULLUP);  // Configurar el pin del botón con resistencia pull-up
  attachInterrupt(digitalPinToInterrupt(buttonPin), toggleSleepMode, FALLING);  // Interrupción por flanco descendente

  Serial.begin(9600); // Iniciar comunicación serial para depuración
}

void loop() {
  if (sleepMode) {
    // Si está en modo de suspensión, apaga la pantalla LCD y entra en modo de bajo consumo
    lcd.noDisplay();  // Apagar la pantalla LCD
    delay(1000);  // Espera un segundo antes de "dormir"
    return;  // Salir del loop para "dormir"
  } else {
    lcd.display();  // Asegurar que la pantalla LCD esté encendida

    unsigned long currentMillis = millis();

    // Medir la frecuencia cada segundo (1000 ms)
    if (currentMillis - previousMillis >= 1000) {
      previousMillis = currentMillis;

      // Leer el contador de pulsos
      noInterrupts(); // Deshabilitar temporalmente las interrupciones
      unsigned long frequency = pulseCount;  // Calcular la frecuencia (pulsos por segundo)
      pulseCount = 0;  // Reiniciar el contador de pulsos
      interrupts();  // Volver a habilitar las interrupciones

      // Mostrar la frecuencia en la LCD
      lcd.setCursor(0, 1);  // Mover el cursor a la segunda línea
      lcd.print(frequency);
      lcd.print(" Hz     ");  // Sobrescribir cualquier valor previo

      // Información de depuración
      Serial.print("Contador de Pulsos: ");
      Serial.println(frequency);

      // Muestrear la señal analógica
      for (int i = 0; i < DATA_SIZE; i++) {
        fftInput[i] = analogRead(A0);  // Leer de la entrada analógica A0
        delayMicroseconds(125);         // Ajustar el retardo para la frecuencia de muestreo
      }

      // Preparar la señal para el cálculo de la FFT
      ZeroFFT(fftInput, DATA_SIZE);  // Realizar la FFT sobre la señal

      // Encontrar el bin de frecuencia con la mayor magnitud
      float maxMagnitude = 0;
      int maxIndex = 0;

      for (int i = 0; i < DATA_SIZE / 2; i++) {
        float magnitude = sqrt(fftInput[i] * fftInput[i] + fftInput[i + DATA_SIZE / 2] * fftInput[i + DATA_SIZE / 2]);
        if (magnitude > maxMagnitude) {
          maxMagnitude = magnitude;
          maxIndex = i;
        }
      }

      // Calcular la frecuencia de la magnitud máxima
      float frequencyFromFFT = (maxIndex * FS) / DATA_SIZE;

      // Imprimir la frecuencia y el resultado de la FFT en la consola serial
      Serial.print("Frecuencia (FFT): ");
      Serial.print(frequencyFromFFT);
      Serial.println(" Hz");
    }
  }
}

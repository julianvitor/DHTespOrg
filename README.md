# DHTesp (ESP32 / ESP32-S3 Fix)

Biblioteca para sensores de temperatura e umidade **DHT11 / DHT22 (AM2302)** em placas **ESP32, ESP32-S3 e ESP8266** (Arduino Framework + FreeRTOS).

Este repositório é um fork atualizado da biblioteca original `beegee-tokyo/DHTesp` (que foi arquivada) com correções críticas para a família **ESP32 / ESP32-S3**.

---

## O que foi corrigido neste fork?

1. **Preservação do Pull-Up Interno no ESP32 / ESP32-S3 (`INPUT_PULLUP`):**
   - A biblioteca original executava `pinMode(pin, INPUT); digitalWrite(pin, HIGH);` após enviar o pulso de *start* (um padrão antigo de Arduino AVR).
   - No core do **ESP32 / ESP32-S3** (`esp32-hal-gpio.c`), `pinMode(pin, INPUT)` executa `GPIO_PULLUP_DISABLE`, desligando o resistor de pull-up interno do pino e causando erro de `TIMEOUT` nas leituras seguintes.
   - **Correção:** Uso explícito de `pinMode(pin, INPUT_PULLUP)` no `setup()` e na leitura do barramento.

2. **Proteção de Seção Crítica no FreeRTOS (`portENTER_CRITICAL`):**
   - Na biblioteca original, o barramento era liberado para resposta do sensor **antes** de entrar em `portENTER_CRITICAL(&mux)`. Com Wi-Fi e BLE ativos, trocas de contexto do FreeRTOS faziam o microcontrolador perder o pulso de *handshake* (`20–40 µs`) do sensor.
   - **Correção:** Entrada na seção crítica **antes** da transição da linha para recepção.

3. **Tolerância de pulso (`200 µs`) e correção de precedência:**
   - Ajuste do limite de timeout por borda de `90 µs` para `200 µs` (evitando falsos timeouts em sensores DHT22/AM2302 reais) e correção de precedência do operador ternário na leitura dos bits.

---

## Instalação (PlatformIO)

No seu arquivo `platformio.ini`, adicione em `lib_deps`:

```ini
lib_deps =
    https://github.com/julianvitor/DHTespOrg.git
```

---

## Exemplo Simples de Uso

```cpp
#include <Arduino.h>
#include <DHTesp.h>

DHTesp dht;
const int PIN_DHT = 12;

void setup() {
  Serial.begin(115200);
  dht.setup(PIN_DHT, DHTesp::DHT22);
  delay(2000); // Aguarda estabilizacao inicial do DHT22
}

void loop() {
  TempAndHumidity data = dht.getTempAndHumidity();

  if (dht.getStatus() == DHTesp::ERROR_NONE) {
    Serial.printf("Temperatura: %.1f C | Umidade: %.1f %%\n", data.temperature, data.humidity);
  } else {
    Serial.printf("Erro na leitura: %s\n", dht.getStatusString());
  }

  delay(2500); // Respeita o intervalo minimo de 2s do DHT22
}
```

# Painel BLE — Pressão e Vazão

## O que tem aqui
- `index.html` — app principal (2 manômetros analógicos em Canvas + Bluetooth)
- `manifest.json`, `sw.js`, `icon-192.png`, `icon-512.png` — necessários para instalar como app no Android

## Comportamento
1. Ao abrir, os dois ponteiros fazem um autoteste: sobem até o fundo de escala e voltam a zero em **3 segundos** no total.
2. Depois disso ficam em zero com o status "Aguardando dados".
3. Ao tocar em **Conectar**, abre o seletor de dispositivos Bluetooth do Android.
4. Assim que a característica BLE notificar, os ponteiros passam a seguir os valores em tempo real (status "Ao vivo").
5. Se a conexão cair, os ponteiros congelam no último valor e o status muda para "Desconectado".

## IMPORTANTE: por que precisa de HTTPS
O Web Bluetooth só funciona em **contexto seguro** (HTTPS ou `localhost`). Abrir o `index.html` direto do armazenamento do celular (`file://`) ou de um servidor HTTP comum **não funciona** — o Bluetooth fica bloqueado pelo navegador.

### Forma mais simples de publicar (grátis, HTTPS automático)
1. Crie um repositório novo no GitHub e suba estes arquivos.
2. Vá em **Settings → Pages**, escolha a branch principal, salve.
3. O GitHub te dá uma URL tipo `https://seu-usuario.github.io/seu-repo/`.
4. Abra essa URL no **Chrome do Android**.
5. Toque no menu (⋮) → **"Instalar aplicativo"** (ou vai aparecer um banner automático) — isso cria o ícone na tela do celular, abrindo em tela cheia como um app nativo.

Alternativas igualmente válidas: Netlify Drop (arrasta a pasta e já publica com HTTPS) ou Vercel.

## Estrutura BLE esperada (GATT)

```
Service UUID:        a55d1a00-1000-4000-8000-00805f9b34fb
Characteristic UUID: a55d1a01-1000-4000-8000-00805f9b34fb
Propriedade:          NOTIFY
Payload:              8 bytes
  bytes 0-3  -> float32 little-endian = pressão (mmHg)
  bytes 4-7  -> float32 little-endian = vazão   (L/min)
```

Ajuste os UUIDs no topo do `index.html` (`SERVICE_UUID` / `CHAR_UUID`) se preferir usar outros.

## Exemplo de firmware ESP32 (Arduino, biblioteca BLE nativa)

```cpp
#include <BLEDevice.h>
#include <BLEServer.h>
#include <BLEUtils.h>
#include <BLE2902.h>

#define SERVICE_UUID       "a55d1a00-1000-4000-8000-00805f9b34fb"
#define CHARACTERISTIC_UUID "a55d1a01-1000-4000-8000-00805f9b34fb"

BLECharacteristic *pChar;

void setup() {
  BLEDevice::init("Painel-CO2");
  BLEServer *pServer = BLEDevice::createServer();
  BLEService *pService = pServer->createService(SERVICE_UUID);

  pChar = pService->createCharacteristic(
      CHARACTERISTIC_UUID,
      BLECharacteristic::PROPERTY_NOTIFY
  );
  pChar->addDescriptor(new BLE2902());

  pService->start();
  pServer->getAdvertising()->start();
}

void loop() {
  float pressao = lerPressaoSensor();   // mmHg
  float vazao    = lerVazaoSensor();    // L/min

  uint8_t payload[8];
  memcpy(payload,     &pressao, 4);
  memcpy(payload + 4, &vazao,   4);

  pChar->setValue(payload, 8);
  pChar->notify();

  delay(20); // ~50 Hz — ajuste conforme a estabilidade da conexão
}
```

## Sobre a velocidade de atualização
O teto físico prático do BLE fica em torno de 30–60 Hz (connection interval de 15–30 ms). O app usa `requestAnimationFrame` com suavização exponencial, então o ponteiro se move fluido mesmo que os pacotes cheguem de forma um pouco irregular.

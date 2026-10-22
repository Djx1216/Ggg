ychhhh

#include <Arduino.h>
#include <WiFi.h>
#include <esp_now.h>
#include <driver/i2s.h>

// I2S Microphone Pins (INMP441)
#define MIC_WS    25
#define MIC_SCK   26
#define MIC_SD    33

// I2S DAC Pins (MAX98357A)
#define DAC_LRC   22
#define DAC_BCLK  19
#define DAC_DIN   21

// Audio Configurations
#define SAMPLE_RATE     16000
#define BUFFER_SIZE     256
#define ESP_NOW_MAX_PAYLOAD 250 // ESP-NOW limit is 250 bytes

// Broadcast MAC Address (Sends to any ESP32 listening on the same channel)
uint8_t broadcastAddress[] = {0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF};

// Audio buffers
int16_t txBuffer[ESP_NOW_MAX_PAYLOAD / 2]; 
int16_t rxBuffer[ESP_NOW_MAX_PAYLOAD / 2];

// Callback when data is received over ESP-NOW
void OnDataRecv(const uint8_t * mac, const uint8_t *incomingData, int len) {
    size_t bytesWritten;
    // Play the incoming audio directly through the DAC speaker
    i2s_write(I2S_NUM_0, incomingData, len, &bytesWritten, portMAX_DELAY);
}

void setup() {
    Serial.begin(115200);

    // 1. Initialize Wi-Fi in Station Mode
    WiFi.mode(WIFI_STA);
    WiFi.disconnect();

    // 2. Initialize ESP-NOW
    if (esp_now_init() != ESP_OK) {
        Serial.println("Error initializing ESP-NOW");
        return;
    }

    // Register receive callback
    esp_now_register_recv_cb(esp_now_recv_cb_t(OnDataRecv));

    // Register peer (Broadcast)
    esp_now_peer_info_t peerInfo = {};
    memcpy(peerInfo.peer_addr, broadcastAddress, 6);
    peerInfo.channel = 1;  
    peerInfo.encrypt = false;
    
    if (esp_now_add_peer(&peerInfo) != ESP_OK){
        Serial.println("Failed to add peer");
        return;
    }

    // 3. Configure I2S for Speaker Output (I2S_NUM_0)
    i2s_config_t dacConfig = {
        .mode = (i2s_mode_t)(I2S_MODE_MASTER | I2S_MODE_TX),
        .sample_rate = SAMPLE_RATE,
        .bits_per_sample = I2S_BITS_PER_SAMPLE_16BIT,
        .channel_format = I2S_CHANNEL_FMT_ONLY_LEFT,
        .communication_format = I2S_COMM_FORMAT_STAND_I2S,
        .intr_alloc_flags = ESP_INTR_FLAG_LEVEL1,
        .dma_buf_count = 8,
        .dma_buf_len = BUFFER_SIZE,
        .use_apll = false
    };
    
    i2s_pin_config_t dacPins = {
        .bck_io_num = DAC_BCLK,
        .ws_io_num = DAC_LRC,
        .data_out_num = DAC_DIN,
        .data_in_num = I2S_PIN_NO_CHANGE
    };
    
    i2s_driver_install(I2S_NUM_0, &dacConfig, 0, NULL);
    i2s_set_pin(I2S_NUM_0, &dacPins);

    // 4. Configure I2S for Microphone Input (I2S_NUM_1)
    i2s_config_t micConfig = {
        .mode = (i2s_mode_t)(I2S_MODE_MASTER | I2S_MODE_RX),
        .sample_rate = SAMPLE_RATE,
        .bits_per_sample = I2S_BITS_PER_SAMPLE_16BIT,
        .channel_format = I2S_CHANNEL_FMT_ONLY_LEFT,
        .communication_format = I2S_COMM_FORMAT_STAND_I2S,
        .intr_alloc_flags = ESP_INTR_FLAG_LEVEL1,
        .dma_buf_count = 8,
        .dma_buf_len = BUFFER_SIZE,
        .use_apll = false
    };

    i2s_pin_config_t micPins = {
        .bck_io_num = MIC_SCK,
        .ws_io_num = MIC_WS,
        .data_out_num = I2S_PIN_NO_CHANGE,
        .data_in_num = MIC_SD
    };

    i2s_driver_install(I2S_NUM_1, &micConfig, 0, NULL);
    i2s_set_pin(I2S_NUM_1, &micPins);
}

void loop() {
    size_t bytesRead = 0;
    
    // Read audio data from the microphone
    i2s_read(I2S_NUM_1, &txBuffer, sizeof(txBuffer), &bytesRead, portMAX_DELAY);

    if (bytesRead > 0) {
        // Broadcast the audio chunk instantly to the other ESP32
        esp_now_send(broadcastAddress, (uint8_t *)txBuffer, bytesRead);
    }
}


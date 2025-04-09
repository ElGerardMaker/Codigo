código de los sensores: import network
import socket
import time
import json
from machine import ADC, Pin
import math

# Configuración WiFi
SSID = 'INFINITUMCE2E'
PASSWORD = 'PGJu36qKFu'
SERVER_IP = "192.168.1.65"
SERVER_PORT = 80

# Calibración basada en pruebas reales
NOISE_FLOOR = 38         # Valor mínimo (silencio)
MAX_DB = 120             # Valor máximo aceptado
CALIBRATION_M = 32.33    # Pendiente ajustada (voltaje a dB)
CALIBRATION_B = -1.0     # Offset ajustado

# Inicializar ADC
adc = ADC(Pin(26))
if hasattr(adc, 'atten'):
    adc.atten(ADC.ATTN_11DB)

# Conexión WiFi
wlan = network.WLAN(network.STA_IF)
wlan.active(True)
wlan.connect(SSID, PASSWORD)

while not wlan.isconnected():
    print("Conectando a WiFi...")
    time.sleep(1)

print("Conectado a WiFi - IP:", wlan.ifconfig()[0])

def read_peak_sound_level(samples=200, interval_us=100):
    """
    Lee una serie de muestras ADC para calcular el nivel de sonido en dB.
    """
    values = []
    for _ in range(samples):
        values.append(adc.read_u16())
        time.sleep_us(interval_us)

    max_val = max(values)
    min_val = min(values)
    p2p_voltage = (max_val - min_val) / 65535 * 3.3

    if p2p_voltage < 0.01:
        return NOISE_FLOOR

    dB = CALIBRATION_M * p2p_voltage + CALIBRATION_B
    return min(MAX_DB, max(NOISE_FLOOR, dB))

def send_data():
    try:
        # Medir y promediar
        readings = [read_peak_sound_level() for _ in range(3)]
        dB = sum(readings) / len(readings)

        print(f"dB promedio: {dB:.2f}")

        data = json.dumps({"sensor1": [dB]})
        
        headers = "POST /update HTTP/1.1\r\n"
        headers += f"Host: {SERVER_IP}\r\n"
        headers += "Content-Type: application/json\r\n"
        headers += f"Content-Length: {len(data)}\r\n\r\n"

        addr = socket.getaddrinfo(SERVER_IP, SERVER_PORT)[0][-1]
        s = socket.socket()
        s.connect(addr)
        s.send(headers.encode() + data.encode())
        s.close()

    except Exception as e:
        print("Error:", e)

# Bucle principal
while True:
    send_data()
    time.sleep(1)

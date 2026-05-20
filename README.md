# Week11_IoT_Thermostat_Valve

## 프로젝트 영상

https://github.com/user-attachments/assets/a1321ff8-60d2-4039-8729-db3cd1c56a13

## 프로젝트 소개
이 프로젝트는 ESP32 보드와 MQTT 통신을 이용하여 실내 온도를 실시간으로 측정하고, Cloud Intelligence을 통해 설정된 목표 온도에 맞춰 자동으로 난방 밸브(릴레이)를 개폐하는 스마트 난방 제어 시스템입니다.

Python 애플리케이션 서버(io7app)에서 각 기기들을 제어하는 IoT 플랫폼의 아키텍처가 구현 되어있습니다.

## 시스템 주요 기능
* **온도 측정 (Thermostat):** DHT22 센서를 통해 현재 온도를 측정하여 주기적으로 MQTT 브로커로 Publish합니다. 벨브 조절 시 기준 온도 조절 가능합니다.
* **자동 밸브 제어 (Valve):** 클라우드 서버의 명령을 수신하여 릴레이 모듈을 구동해 난방 밸브를 제어합니다.
* **안전 제어:** 잦은 밸브 작동을 방지하기 위해 목표 온도 기준 ±0.5도의 여유 구간을 두어 제어하는 로직이 Python 서버에 구현 되어있습니다.

## 하드웨어 구성 (Hardware Setup)
### 1. Thermostat (온도조절기) 보드
* **Microcontroller:** ESP32_LilyGo
* **Sensor:** DHT22
* **Pin Connection:** DHT22_PIN = 17

### 2. Valve (난방 구동기) 보드
* **Microcontroller:** ESP32
* **Actuator:** 5V Relay 모듈
* **Pin Connection:** RELAY_PIN = 15

## 소프트웨어 및 플랫폼
* **Firmware:** C++ (PlatformIO 기반)
* **IoT Platform** (Docker)
  * **MQTT Broker:** Eclipse Mosquitto (디바이스 간 메시지 중계)
  * **App Server:** Python io7app 프레임워크 (제어 로직 담당)
  * **DB & Monitoring:** InfluxDB (데이터 저장) & Grafana (데이터 시각화)

## 앱 서버 제어 로직
`thermo1`에서 올라오는 온도(temperature) 데이터를 분석하여 목표 온도(target)와 비교한 뒤, valve1에 개폐 명령(on/off)을 내립니다.

```python
from io7app import App

app = App()
state = {"valve": "off"}

@app.on_event("thermo1", "status")
def thermostat(data):
    temp = data["temperature"]
    target = data.get("target", 22.0)
    
    desired = (
        "on" if temp < target - 0.5
        else "off" if temp > target + 0.5
        else state["valve"]
    )
    
    if desired != state["valve"]:
        state["valve"] = desired
        app.send_cmd("valve1", "valve", {"valve": desired})

if __name__ == "__main__":
    app.run()

```

### 실행 및 테스트
1. Docker Compose를 통해 io7-platform-cloud 시스템을 구동합니다 (docker compose up -d).
2. ESP32 보드를 팩토리 리셋(Boot 버튼 5초) 후 Captive Portal에서 기기 아이디(thermo1, valve1)와 PC 브로커 IP를 등록합니다.
3. 파이썬 앱 서버를 실행하고 로그를 모니터링합니다 (docker logs -f io7app).
4. 온도 센서(DHT22)에 온도를 상승시키면 목표 온도 도달 시 릴레이 모듈이 차단(off)되는 것을 확인할 수 있습니다.

---
layout: post
title: "Home Assistant OS (HAOS) 설치 가이드"
subtitle: "라즈베리파이 5 (SD 카드 기반) 설치 가이드"
date: 2025-12-06
author: kgi0412
# cover-img:
# thumbnail-img:
# share-img:
categories: [wiki, blog]
tags: [ChatGPT, RaspberryPi, HomeAssistant, HAOS]
language: ko
comments: true
---

## 💾 Home Assistant OS (HAOS) 설치 가이드 - 라즈베리파이 5 (SD 카드 기반)

### 📌 1단계: 준비물 확인 및 설치 도구 준비

SD 카드만으로 라즈베리파이 5에 HAOS를 설치하기 위한 필수 준비물과 설치 도구를 확인합니다.

| 준비물 | 권장 사양 / 설명 |
| :--- | :--- |
| **라즈베리파이 5** | 본체 |
| **SD 카드** | **32GB 이상, A2 등급** 권장 (성능 및 내구성 확보) |
| **전원 어댑터** | 라즈베리파이 5용 **27W USB-C PD 어댑터** (안정적인 전원 공급) |
| **LAN 케이블** | **유선 연결 필수** (초기 설치의 안정성을 위해 권장) |
| **컴퓨터 및 카드 리더기** | 설치 파일 쓰기 작업에 필요 |

#### 설치 도구 다운로드
* **[Raspberry Pi Imager](https://www.raspberrypi.com/software/)**: HAOS 이미지를 SD 카드에 쉽고 정확하게 복사해주는 공식 도구입니다.

---

### 📝 2단계: SD 카드에 HAOS 이미지 쓰기 (Flashing)

Raspberry Pi Imager를 사용하여 SD 카드에 운영체제 이미지를 기록하는 과정입니다.

```markdown
1. **Imager 실행 및 SD 카드 연결:**
   * 컴퓨터에서 **Raspberry Pi Imager**를 실행하고, SD 카드를 카드 리더기를 통해 컴퓨터에 연결합니다.
2. **운영 체제(OS) 선택:**
   * `운영 체제 선택 (CHOOSE OS)` 클릭
   * `다른 특정 목적의 OS (Other specific-purpose OS)` > `Home assistants and home automation` > `Home Assistant`로 이동합니다.
   * **반드시** 라즈베리파이 5에 맞는 이미지 (`Raspberry Pi 5 (64-bit)` 등)를 선택합니다.
3. **저장 공간 선택:**
   * `저장 공간 선택 (CHOOSE STORAGE)`을 클릭하여 연결한 **SD 카드**를 정확히 선택합니다.
   * ⚠️ **경고:** SD 카드 내의 모든 데이터는 이 과정에서 영구적으로 삭제됩니다.
4. **쓰기 시작:**
   * `다음 (Next)`을 클릭하고 쓰기(Writing) 및 검증(Verifying) 작업이 완료될 때까지 기다립니다.
   * 작업 완료 후, **SD 카드를 안전하게 분리**합니다.
```

---

### 🔌 3단계: 최초 부팅 및 Home Assistant 접근

SD 카드를 라즈베리파이에 삽입하고 전원을 켜서 초기 설치를 완료합니다.

```markdown
1. **하드웨어 연결:**
   * SD 카드를 라즈베리파이 5 슬롯에 삽입합니다.
   * **LAN 케이블**을 공유기/허브에 연결합니다.
   * **전원 어댑터**를 연결하여 라즈베리파이를 켭니다.
2. **초기 설치 대기:**
   * 전원을 켜면 라즈베리파이가 부팅을 시작하고, **HAOS가 초기 설정 및 설치를 완료하는 데 약 5~10분 정도 소요**될 수 있습니다. 설치 중에는 LED가 깜빡입니다.
3. **웹 접속 및 초기 설정:**
   * 같은 네트워크에 연결된 컴퓨터나 스마트폰의 웹 브라우저를 열고 다음 주소 중 하나로 접속합니다.
   * **권장 주소:** `http://homeassistant.local:8123`
   * **대체 주소:** 공유기에서 할당된 **IP 주소**를 확인하여 `http://[IP 주소]:8123` 로 접속합니다.
4. **온보딩(Onboarding) 완료:**
   * 접속에 성공하면 Home Assistant의 **환영 화면**이 나타납니다.
   * 안내에 따라 **사용자 계정 생성, 위치 및 시간대 설정** 등 초기 설정을 완료하면 HAOS 사용을 시작할 수 있습니다.
```

---
### 💡 추가 참고: SSD 마이그레이션 계획
현재 SD 카드로 사용하다가 나중에 **SSD를 구매해도 처음부터 다시 설치할 필요가 없습니다.**

* **설정 유지:** 현재 SD 카드에 저장된 모든 Home Assistant 설정, 자동화, 데이터는 그대로 SSD로 옮겨지므로 끊김 없이 이어서 사용 가능합니다.
* **마이그레이션 방법:** HAOS의 **백업/복원 기능**을 사용하거나, SD 카드의 전체 이미지를 복제하는 도구를 사용하여 손쉽게 시스템을 SSD로 이전할 수 있습니다.
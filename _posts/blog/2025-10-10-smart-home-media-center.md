---
layout: post
title: "Raspberry Pi and HomeAssistant OS"
subtitle: "라즈베리파이 HAOS + NAS 및 LibreELEC(Kodi) PC 통합 시스템"
date: 2025-10-10
author: kgi0412
# cover-img:
# thumbnail-img:
# share-img:
categories: [wiki, blog]
tags: [ChatGPT, RaspberryPi, HomeAssistant]
language: ko
comments: true
---

# 라즈베리파이 HAOS + NAS 및 LibreELEC(Kodi) PC 통합 시스템

## 개요
이 문서는 라즈베리파이에 **Home Assistant OS (HAOS)**와 **NAS(Network Attached Storage)** 기능을 설치하고, 별도 PC에 **LibreELEC**(Kodi 기반 미디어 센터 OS)을 설치해 Home Assistant로 전원 제어 및 미디어 스트리밍을 구현하는 시스템의 설정, 전력 소모, 효율성을 정리합니다. 이 구성은 스마트 홈 제어와 미디어 재생을 통합하며, 저전력 및 고효율 운영을 목표로 합니다.

## 시스템 구성
- **라즈베리파이 (HAOS + NAS)**:
  - **역할**: 스마트 홈 제어(Home Assistant) 및 파일 공유(NAS).
  - **OS**: Home Assistant OS.
  - **NAS 구현**: Samba 애드온 또는 Docker 기반 OpenMediaVault(OMV).
  - **하드웨어**: Raspberry Pi 4/5 (4GB 이상 RAM 권장), SSD 외장 하드 추천.
- **PC (LibreELEC)**:
  - **역할**: Kodi 미디어 센터로 NAS 파일 스트리밍 및 재생.
  - **OS**: LibreELEC (Kodi 전용 경량 리눅스).
  - **하드웨어**: 저전력 미니 PC(예: Intel NUC, Beelink) 또는 일반 데스크톱.
- **연동**:
  - Home Assistant의 **Kodi Integration**으로 LibreELEC 제어(재생, 볼륨 등).
  - **Wake-on-LAN(WoL)**으로 PC 전원 제어.
  - NAS에서 미디어 파일(SMB 프로토콜) 접근.

## 구현 가능성
### 설정 절차
1. **라즈베리파이 설정**:
   - **HAOS 설치**: [Raspberry Pi Imager](https://www.raspberrypi.com/software/)로 HAOS 이미지를 SD 카드에 기록.
   - **NAS 설정**:
     - HAOS Add-on Store에서 Samba 애드온 설치 → 외장 하드 공유.
     - 또는 Docker로 [OpenMediaVault](https://www.openmediavault.org/) 설치(고급 NAS 기능).
   - **항상 켜짐**: 24시간 365일 운영, 저전력 특성 활용.
2. **PC (LibreELEC) 설정**:
   - [LibreELEC 공식 사이트](https://libreelec.tv/)에서 이미지 다운로드, USB/SSD에 설치.
   - Kodi에서 NAS(SMB) 추가: HAOS의 IP 주소 입력 → 공유 폴더 연결.
   - Kodi 웹 서버 활성화(설정 > 서비스 > 제어 > 웹 서버 허용, 포트 8080).
3. **Home Assistant 연동**:
   - **Kodi Integration**: HAOS 설정 > 통합 > Kodi 추가 → PC IP/포트 입력.
   - **WoL 설정**: HAOS에서 Wake-on-LAN 통합 추가 → PC MAC 주소 입력.
   - **자동화 예시**: “영화 시작” 버튼 → PC WoL로 켜기 → Kodi 특정 플레이리스트 재생.

### 주요 기능
- **미디어 스트리밍**: NAS에서 영화, 음악 등 파일을 LibreELEC으로 스트리밍.
- **스마트 홈 제어**: HAOS로 조명, TV 등 연동 가능.
- **원격 전원 관리**: WoL로 PC 전원 효율적 제어.

## 전력 소모

| 장치 | 상태 | 전력 소모 | 월간 전력 (24시간 365일 또는 2시간/일) | 월 전기료 (한국 기준, 약 200원/kWh) |
|------|------|-----------|---------------------------------------|-------------------------------------|
| Raspberry Pi 4/5 (HAOS + NAS) | 항상 켜짐 | 유휴: 3-5W, NAS 포함: 10-15W | 7-10 kWh | 1,400-2,000원 |
| 미니 PC (LibreELEC) | 사용 시 | 유휴: 10-20W, 재생: 20-40W | 1.5-3 kWh (2시간/일) | 300-600원 |
| 일반 데스크톱 (LibreELEC) | 사용 시 | 유휴: 50-100W, 재생: 100-200W | 5-15 kWh (2시간/일) | 1,000-3,000원 |

- **총 전기료**: Pi + 미니 PC 조합은 월 1,700-2,600원, 일반 PC 사용 시 2,400-5,000원.
- **절약 팁**:
  - SSD 사용으로 NAS 전력 감소.
  - WoL로 PC 대기 전력 최소화.
  - 저전력 미니 PC 채택.

## 효율성 평가
### 장점
- **전력 효율**: Pi의 저전력(10-15W)으로 24시간 365일 NAS+스마트 홈 운영 적합. WoL로 PC 전력 낭비 최소화.
- **기능 통합**: HAOS로 스마트 홈, NAS, 미디어 제어 통합. LibreELEC은 미디어 재생에 최적화.
- **확장성**: NAS 용량 확장, HA로 추가 스마트 기기 연동 가능.
- **안정성**: 독립된 장치(HAOS와 LibreELEC)로 충돌 없음.

### 단점
- **설정 복잡성**: Samba/OMV 및 WoL 설정은 초보자에게 다소 복잡.
- **네트워크 의존**: 미디어 스트리밍은 네트워크 속도에 의존(기가비트 LAN 추천).
- **PC 전력**: 일반 데스크톱은 전력 소모↑, 미니 PC 권장.

### 대안 비교
- **단일 Pi (HAOS + Kodi)**: 전력 낮음(15-20W), 하지만 성능 저하 및 설정 복잡성↑.
- **전용 NAS (예: Synology)**: 전력↑(20-50W), 비용↑, 설정 간단.
- **결론**: Pi + 저전력 PC 조합은 전력과 기능 면에서 균형 잡힘.

## 추천
- **하드웨어**:
  - Pi: 4GB 이상 Pi 4/5, SSD 외장 하드.
  - PC: Intel NUC, Beelink 등 저전력 미니 PC.
- **설정 최적화**:
  - HAOS Samba로 간단한 NAS 구현.
  - LibreELEC에서 NAS 자동 마운트.
  - HA 자동화로 “영화 모드” 설정(조명, PC 전원, Kodi 재생 연동).
- **문제 해결**:
  - [Home Assistant 커뮤니티](https://community.home-assistant.io/) 및 [LibreELEC 포럼](https://forum.libreelec.tv/) 참고.
  - Reddit: [r/homeassistant](https://www.reddit.com/r/homeassistant/), [r/libreelec](https://www.reddit.com/r/libreelec/).

## 결론
이 시스템은 **저전력(월 1,700-5,000원)**, **고효율**, **확장 가능**한 스마트 홈 및 미디어 센터 솔루션입니다. 라즈베리파이(HAOS + NAS)와 저전력 PC(LibreELEC)를 활용해 스마트 홈 제어, 파일 공유, 미디어 스트리밍을 통합 관리할 수 있습니다. 설정 초기 복잡성을 극복하면 안정적이고 편리한 환경을 제공합니다.

---

## 추가 노트
- **확장 가능**: “고급 설정” 섹션 추가 가능(예: OMV 고급 NAS 설정, HA 자동화 스크립트).
- **업데이트**: 최신 HAOS/LibreELEC 버전 호환성 주기적 확인 권장.
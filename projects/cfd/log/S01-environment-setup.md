---
layout: page
title: "환경 구축: Windows에서 WSL2 + Ubuntu + OpenFOAM 설치 & 튜토리얼 테스트"
permalink: /projects/cfd/log/environment-setup/
---

## TL;DR
- **목표**: Windows에서 WSL2 기반 OpenFOAM 실행 환경 구축 + 튜토리얼 1개 정상 실행 확인
- **오늘 한 것**: WSL2/Ubuntu 설치 → OpenFOAM 설치/환경변수 확인 → 튜토리얼 실행 → paraFoam(ParaView) 확인
- **현재 상태**: (✅성공 / ⚠️부분성공 / ❌실패 중 택1로 나중에 체크)

---

## 1) 내 환경
- Host OS: Windows (버전: ___)
- WSL: WSL2
- Ubuntu: ___ (예: 22.04 LTS)
- OpenFOAM: ___ (예: OpenFOAM 13 / v2312 등)
- Post-processing: ParaView / paraFoam

---

## 2) WSL2 + Ubuntu 설치/확인
### 2.1 설치/버전 확인
```bash
wsl -l -v

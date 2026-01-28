---
layout: page
title: CFD (OpenFOAM)
permalink: /projects/cfd/
---
## 프로젝트 전체 목표
- OpenFOAM으로 대표 벤치마크 2개를 '재현 가능한 형태'로 수행하고, (A) 난류 분리/재부착의 V&V + (B) 압축성 충격파 구조의 기본 재현을 통해 CFD 워크플로우·검증·해석 능력 향상
  
## 2D BFS (Backward-Facing Step) V&V
- 문제정의: BFS에서 재부착 길이 x/H와 속도/난류 프로파일이 격자·벽처리·난류모델에 얼마나 민감한가? ‘재현 가능한 안정 세팅’은 무엇인가?
- 도구: OpenFOAM
- 성공 기준: NASA TMR과 비교
- 최종 정리: [2D BFS V&V]
  
## Logs
- [환경 구축: Windows에서 WSL2 + Ubuntu + OpenFOAM 설치](/projects/cfd/log/environment-setup/)
- [튜토리얼: OpenFOAM 간단한 예제 코드 + paraFoam 실행](/projects/cfd/log/tutorial-simulation/)

# JOO's CFD Study — personalized starter for `1001wngud`

이 폴더는 **Minimal Mistakes + GitHub Pages** 기반으로, 네 GitHub 아이디와 프로젝트 정보를 반영해서 바로 **로컬 폴더에서 수정 → git push** 할 수 있게 정리한 버전이다.

현재 반영된 프로젝트는 아래 두 개다.

- **pitzDailySteady BFS (Backward-Facing Step)**
  - repo: `https://github.com/1001wngud/CFD-Study-BFS`
- **CFD Study of IGE (In-Ground Effect)**
  - repo: `https://github.com/1001wngud/CFD-Study-IGE`

---

## 너의 작업 방식 기준으로 쓰는 법

너는 이미 **VS Code 로컬 폴더 → git add / commit / push → 원격 반영** 흐름이 준비되어 있으니까,
이 템플릿도 GitHub 웹에서 새로 만지는 방식보다 **네 로컬 폴더 안에서 파일 교체/수정**하는 기준으로 보면 된다.

### 추천 순서

1. 현재 네 **웹사이트용 로컬 폴더**를 연다.
2. 이 폴더의 파일들을 그 로컬 폴더에 **복사해서 덮어쓴다**.
3. `_config.yml`만 먼저 확인한다.
4. `assets/images/`에 대표 이미지 넣는다.
5. BFS / IGE md 파일을 네가 이미 작성한 문서로 교체한다.
6. VS Code에서 commit 후 push 한다.

---

## 가장 먼저 확인할 파일

### 1) `_config.yml`

현재는 아래 기준으로 들어가 있다.

```yml
url: "https://1001wngud.github.io"
baseurl: ""
repository: "1001wngud/1001wngud.github.io"
```

이 설정은 **사이트 저장소 이름이 `1001wngud.github.io`일 때** 그대로 쓰면 된다.

만약 네 사이트 저장소 이름이 예를 들어 `joo-cfd-study` 라면 아래처럼 바꿔야 한다.

```yml
url: "https://1001wngud.github.io"
baseurl: "/joo-cfd-study"
repository: "1001wngud/joo-cfd-study"
```

### 2) `index.md`

홈 화면 문구와 프로젝트 카드가 들어 있다.
지금은 네 프로젝트 제목과 짧은 설명이 이미 반영되어 있다.

### 3) `_portfolio/project-1.md`, `_portfolio/project-2.md`

각 프로젝트 소개 페이지다.
이미 아래 내용이 반영되어 있다.

- 프로젝트 실제 제목
- repo 링크
- 요약 문구
- 핵심 결과 요약
- 상세 로그 링크

### 4) `bfs/`, `ige/`

여기가 **상세 markdown 페이지**다.
네가 이미 작성한 md를 여기에 붙여 넣거나 파일명을 유지한 채 내용만 바꾸면 된다.

---

## BFS 쪽은 네 현재 구조를 반영해 둠

네가 말한 것처럼 BFS는 레포 README 말고도 **상세 md 파일 4개** 구조를 쓰는 걸 반영해서,
사이트 템플릿도 BFS 프로젝트 페이지 아래 로그를 **4개**로 맞춰뒀다.

현재 템플릿 기준 BFS 상세 페이지는 다음 흐름이다.

1. `bfs/01-tracka-readme.md`
2. `bfs/02-project-flow.md`
3. `bfs/03-day12-report.md`
4. `bfs/04-lab-contact-summary.md`

즉 네 로컬 폴더에서 아래 식으로 바꾸면 된다.

- `trackA/README.md` → `bfs/01-tracka-readme.md`
- `trackA/deliverables/PROJECT_FLOW.md` → `bfs/02-project-flow.md`
- `trackA/day12_package/report_Day12.md` → `bfs/03-day12-report.md`
- `trackA/deliverables/LAB_CONTACT_1PAGE.md` → `bfs/04-lab-contact-summary.md`

파일명을 꼭 그대로 유지할 필요는 없고,
**페이지 안 front matter의 permalink만 유지**되면 링크는 계속 동작한다.

예시:

```md
---
title: "BFS · 02. Project flow"
permalink: /bfs/02-project-flow/
---
```

---

## IGE 쪽은 지금 3개 예시 페이지로 둠

IGE는 일단 아래 3개 구조로 넣어 두었다.

1. overview
2. setup
3. results

나중에 네 md 구조에 맞춰 4개나 5개로 늘려도 된다.

---

## 이미지 바꾸는 곳

아래 파일을 네 실제 그림으로 바꾸면 사이트 완성도가 훨씬 올라간다.

- `assets/images/hero-cfd.svg`
- `assets/images/project-1-cover.svg`
- `assets/images/project-2-cover.svg`

처음에는 png/jpg로 그냥 덮어쓰고, 파일 경로만 맞춰도 충분하다.

---

## push 하기 전 최소 체크리스트

- `_config.yml` 저장소 이름 확인
- 프로젝트 repo 버튼 링크 확인
- BFS 4개 상세 링크 확인
- IGE 상세 링크 확인
- 이미지 경로 확인

그 다음에는 그냥 네 원래 하던 대로 VS Code에서 commit / push 하면 된다.

---

## 추가로 추천하는 운영 방식

교수님이 보는 사이트는 보통 오래 읽기보다 빠르게 훑는다.
그래서 각 프로젝트 페이지는 아래 순서를 유지하는 게 좋다.

1. 문제 정의
2. 사용한 방법
3. 핵심 결과
4. GitHub repo
5. 상세 md 로그

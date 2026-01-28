---
layout: page
title: "환경 구축: Windows에서 WSL2 + Ubuntu + OpenFOAM 설치"
permalink: /projects/cfd/log/environment-setup/
---

## List
- **목표**: Windows에서 WSL2 기반 OpenFOAM 실행 환경 구축
- **현재 상태**: (✅성공)

---

## 1) 내 환경
- Host OS: Windows 11 Pro

OpenFOAM은 리눅스기반 프로그램이기에 Windows 11 Pro OS를 사용하는 나는 가상 환경을 구축하기로 결정했다.

<figure>
  <img src="/assets/images/setting1image.png" alt="WSL 설치 화면" />
  <figcaption>그림 1) WSL 설치 후 Ubuntu 계정 생성 화면</figcaption>
</figure>

먼저 Linux용 Windows 하위 시스템을 깔기 위해 PowerShell을 관리자 권한으로 실행시키고 아래 명령어를 먼저 쳐줬다.  
(이건 **Windows PowerShell**에서 실행)

```
wsl.exe --install
```

wsl 중에서도 Ubuntu 라는 기본 프로그램을 바탕으로 설치하였다.  
그 후 리눅스 계정 생성 화면이 뜨는데 사용자명과 비밀번호를 적어 리눅스 계정을 먼저 생성했다.

이때 비밀번호 입력할 때 비밀번호가 화면에 아무것도 입력 안되는 것처럼 안보이는데 blind typing이라고 정상이다.

이제 Ubuntu라는 리눅스형 cmd? 에서 기본적인 세팅을 먼저 하였다.

---

일단 wsl1 버전과 wsl2버전이 있는데 버전 체크를 해준다.

```
wsl -l -v
```

2026.01 기준 wsl2 버전이 최신이기에 이버전을 사용하였다.

처음에는 command not found가 떠서 놀라는 경우도 있었는데 지금 cmd에 떠있는 상태 즉 명령 내리는 위치가 리눅스인지 윈도우인지 잘 확인해보고 명령해야한다.

command not found가 뜬 이유는 Ubuntu(WSL 리눅스)안에 들어가 있는 상태라서, `wsl -l -v`를 치면 리눅스에는 wsl이라는 명령어가 없어서 저렇게 뜨는 거다.  
`wsl -l -v`에서의 wsl 명령어는 애초에 윈도우 기반이니 윈도우 파워쉘에서 명령을 해야한다.

---

이후 이제 환경 세팅을 할 건데 리눅스환경에서 아래를 입력해서 저장소 URIs 주소를 찾을 것이다.

```
sudo nano /etc/apt/sources.list.d/ubuntu.sources
```

목적은 나는 한국에서 실행하고 있기 때문에 한국 지역에 맞는 주소로 바꿔서 프로그램을 더빠르게 최적화 시키는 것이 목적이였다.

따라서 들어가면 보통

```
URIs: http://archive.ubuntu.com/ubuntu/   (일반패키지)
security.ubuntu.com                      (보안업데이트)
```

이 두 주소가 있고, 이 두 주소를 한국에 있는 미러 `mirror.kakao.com`으로 전부 바꾸면 다운로드가 더 빠르거나 안정적일 수 있기 때문에 바꿀 것이다.

꼭 필요한 세팅은 아니지만 앞으로 OpenFOAM 같은 큰 패키지를 다운로드 받는다면 속도차이가 체감될 수 있기 때문에 다운로드 하였다.

이후 apt update와 apt upgrade를 하여 서버에서 패키지 목록(아까 mirror.kakao.com으로 최신화한 목록)을 가져와서 로컬 목록을 갱신하고, 지금 설치된 것들을 실제로 최신 버전으로 교체하였다.

```
sudo apt update
sudo apt upgrade
```

여기서 나는 https://를 지우고 mirror.kakao.com/ubuntu/로 설정을 했었는데 오류가 떴다.  
URIs는 반드시 http:// 또는 https://로 시작하게 만들어야하기 때문에 정해진 주소 이외에는 건들지 않는 것이 좋다.

---

그리고 개발/빌드 기본 패키지 세트를 다운로드 하였다. OpenFOAM을 패키지 설치만 할 것이라도 나중에 튜토리얼 따라가다 보면 필요할 것들이라 기본적으로 깔아두는게 편하다.

명령어는

```
sudo apt install -y build-essential git cmake python3 python3-pip \
  vim nano htop tmux
```

라고 나와서 다운로드 해봤지만...?

```
joo@JOO-DESKTOP:~$ sudo apt install -y build-essential git cmake python3 python3-pip \ vim nano htop tmux
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
E: Unable to locate package build-essential
E: Unable to locate package cmake
E: Unable to locate package python3-pip
E: Unable to locate package htop
```

바로 에러가 났다. 에러를 읽어보면 패키지를 못찾는다고 뜨길래 APT 저장소 설정이 잘못되어 있는 것 같아서 차근차근 다시 확인해봤다.

- `sudo apt update` : APT업데이트가 에러 없이 되는지
- `cat /etc/apt/sources.list.d/ubuntu.sources` : 여기에서 Components:에 main restricted universe multiverse가 있는지

확인해봤는데 딱히 문제가 없어서 애매하게 고치기보다 그냥 정상 템플릿으로 백업하고 다시 덮어쓰는 방법을 선택했다.

백업

```
sudo cp /etc/apt/sources.list.d/ubuntu.sources ~/ubuntu.sources.bak
```

Ubuntu 코드명 확인

```
lsb_release -cs
```

여기서 noble이 뜬다면

```
sudo tee /etc/apt/sources.list.d/ubuntu.sources >/dev/null <<'EOF'
Types: deb
URIs: http://mirror.kakao.com/ubuntu/
Suites: noble noble-updates noble-backports noble-security
Components: main restricted universe multiverse
Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg
EOF
```

이걸로 덮어씌워준다.

덮어씌웠으니 다시 apt update

```
sudo apt update
sudo apt install -y build-essential git cmake python3 python3-pip vim nano htop tmux
```

---

또 사전 세팅으로 WSL2 메모리/스왑 제한 걸기가 필요하다. 이 작업은 나중에 컴파일 혹은 메싱할 때 체감이 크다.  
OpenFOAM 빌드/대형 메쉬를 돌리게 되면 Vmmem이 RAM을 갑자기 확 잡아 먹을 수 있기에, Windows에 .wslconfig로 제한을 걸어두었다.  
제한을 걸기위해서는 window 사용자 폴더에 \.wslconfig를 만들어야한다.

방법은 간단한데

1. 메모장 열기  
2. 아래 내용 붙여넣기 -> 나는 램이 16GB이기 때문에 Windows가 죽지 않게 최소 4~6GB는 여유분으로 남겨놓았다.

```
[wsl2]
memory=8GB
processors=4
swap=4GB
```

3. 파일 > 다른 이름으로 저장  
4. 파일 이름: .wslconfig  
5. 파일 형식: 모든 파일(.) 로 선택  
6. 저장 위치: C:\Users\PC(나의 윈도우계정)\ (바탕화면 말고 “사용자 폴더”에 넣어야한다.)  
7. 저장

(저장 후 적용하려면 WSL을 껐다 켜야 해서 PowerShell에서 아래를 한 번 실행했다.)

```
wsl --shutdown
```

---

ParaView(후처리)는 OpenFOAM 설치 시 같이 설치된다. Windows 11이면 리눅스 gui 띄우는것도 가능하기에 매우 유용하다.

케이스 폴더는 Windows 드라이브에서 작업하게 되면 파일 I/O가 느려져서 메쉬 후처리/로그 저장할 때 답답해질 수 있기에 리눅스 파일시스템에 케이스를 두었다.

```
mkdir -p ~/cfd/run
cd ~/cfd/run
```

---

**OpenFOAM 설치

나는 OpenFOAM v13을 설치할 것이기에

```
sudo apt update
sudo apt -y install openfoam13
```

이렇게 다운로드 하려고 했다.

(그리고 설치되면 바로 환경설정도 잡으려고 했다.)

```
source /opt/openfoam13/etc/bashrc
```

```
echo 'source /opt/openfoam13/etc/bashrc' >> ~/.bashrc
```

다운로드 과정중

```
joo@JOO-DESKTOP:~/cfd/run$ sudo apt update
sudo apt -y install openfoam13
[sudo] password for joo:
Hit:1 http://mirror.kakao.com/ubuntu noble InRelease
Hit:2 http://mirror.kakao.com/ubuntu noble-updates InRelease
Hit:3 http://mirror.kakao.com/ubuntu noble-backports InRelease
Hit:4 http://mirror.kakao.com/ubuntu noble-security InRelease
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
104 packages can be upgraded. Run 'apt list --upgradable' to see them.
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
E: Unable to locate package openfoam13
```

이런 오류가 뜨게 됐는데...

OpenFOAM13은 Ubuntu 기본 저장소(mirror.kakao.com = Ubuntu 저장소 미러)에 있는 패키지가 아니라, OpenFOAM Foundation(openfoam.org)의 전용 APT 저장소(dl.openfoam.org)를 추가해야 설치되는 pack이여서 OpenFOAM v13 저장소를 새롭게 추가하고 설치해야한다.

1. add-apt-repository 도구가 없을 수 있으니 먼저 준비

```
sudo apt update
sudo apt install -y wget ca-certificates gnupg software-properties-common
```

2. openfoam.org 저장소 GPG 키 추가 + 저장소 추가

```
sudo sh -c "wget -O - https://dl.openfoam.org/gpg.key > /etc/apt/trusted.gpg.d/openfoam.asc"
sudo add-apt-repository http://dl.openfoam.org/ubuntu
```

3. 목록 갱신 후 다시 설치해준다.

```
sudo apt update
sudo apt -y install openfoam13
```

---

마지막으로 설치됐는지 확인

```
echo '. /opt/openfoam13/etc/bashrc' >> ~/.bashrc
. ~/.bashrc
foamRun -help
```

이렇게 해서 에러가 뜨지 않는다면 드디어 기다리고 기다리던 OpenFOAM을 설치완료 한 것이다.

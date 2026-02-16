---
layout: single
title: "pitzDailySteady BFS(Backward-Facing Step) 2D"
mathjax: true
---

## 재부착 길이 추출, 파이프라인 구축, 난류모델 민감도 측정

### 요약

OpenFOAM 튜토리얼 케이스인 pitzDailySteady의 2D BFS(Backward-Facing Step) 유동에서 재부착 길이를 라인 샘플링을 통해 추출하고, kEpsilon과 kOmegaSST로 난류모델 민감도를 비교했으며, 결과 신뢰도를 위해 taper 구간 배제, 시간안정성, y오프셋 민감도를 함께 검토했다.

### 프로젝트 동기

OpenFOAM을 처음 접했기 때문에 가장 먼저 공식 tutorial 케이스를 실행하며 기본 사용법과 작업 흐름을 익혔다. 하지만 이번 프로젝트의 목표는 단순히 튜토리얼을 그대로 재현했다에서 멈추는 것이 아닌 실제 연구에서는 해석을 돌리는 것 자체보다, 그 결과로부터 **관심량(QoI)을 정의하고 추출한 뒤**, 계산·후처리·조건 변화에 대해 **검증하고**, 동일한 절차로 언제든 다시 얻을 수 있도록 **재현성을 확보하는 과정**이 반복적으로 요구된다. 그래서 나는 규모가 작고 통제 가능한 튜토리얼 케이스를 기반으로 **‘QoI 추출 → 교차 검증 → 재현 가능한 절차화’** 흐름을 처음부터 끝까지 한 번 완성해보는 것을 프로젝트의 핵심 동기로 삼았다.

### CASE & SETTING

**해석**
Steady-state RANS
`constant/momentumTransport`: `simulationType RAS`

**난류모델**
Baseline(00_pitzDailySteady): `model kEpsilon`
Variant(11_kOmegaSST): `model kOmegaSST`
`turbulence on`
`viscosityModel Newtonian`

**점성계수(동점도)**
`constant/physicalProperties`: `nu 1e-05;`  $\nu = 1\times10^{-5}$ $m^2/s$ 

**경계조건**

- 유속 $U$
inlet: fixedValue uniform: `uniform (10 0 0)`  → 유입속도 $U_{in}$ $=10$  $m/s$
outlet: `zeroGradient`
upperWall/lowerWall: `noSlip`
frontAndBack: `empty` (2D 처리)
- 압력 $p$
inlet: `zeroGradient`
outlet: fixedValue `uniform 0`
upperWall/lowerWall: `zeroGradient`
frontAndBack: `empty`

**기하/메쉬(blockMeshDict)**

- 단위 변환: `convertToMeters 0.001` 
z 두께: vertices에서 z = -0.5 ~ +0.5 ($mm$) → 2D 케이스(앞/뒤 empty)
- 주요 x 좌표(단위 $mm$ → $m$)
upstream 포함: $-20.6$ $mm$
step 근방: $x=0$
taper 기준점:  $x=206$ $mm$ $(=0.206$ $m)$
outlet: $x=290$ $mm$ $(=0.29$ $m)$
- y 스케일(단위 $mm$)
$25.4$ $mm$
- 프로젝트에서 Step 높이
$𝐻=0.0254$ $m$

[이미지1]`                                     [이미지3 아마 파라뷰 들어갈 듯]

[이미지2]

**Reynolds 수**

$Re=\frac{\rho UL}{\mu}=\frac{UL}{\nu}$ 

$U=10$  $m/s$
$L(H)=0.0254$ $m$
$\nu = 1\times10^{-5}$ $m^2/s$

$Re_{inlet} = \frac{UH}{\nu} = \frac{10 \cdot 0.0254}{1\times 10^{-5}}= 25400$
$Re_{inlet}≈ 25400$ 

### 수치해석 및 솔버 세팅

**실행 출력(controlDict)**

- `solver incompressibleFluid;`
- `startTime 0; endTime 2000; deltaT 1;`
- `writeInterval 100;` (timeStep 기준)

OpenFOAM v13 user guide에서 `fvSolution` 예시는 `incompressibleFluid` 모듈 솔버 케이스를 기준으로 하였다.

**이산화(fvSchemes)**

baseline, SST 공통

- `ddtSchemes`: `steadyState`
- `div(phi,U) bounded Gauss linearUpwind grad(U);`
- `laplacianSchemes Gauss linear corrected;`
- `snGradSchemes corrected;`
- `wallDist method meshWave;`

baseline, SST 차이

- baseline(00_pitzDailySteady): `gradSchemes default Gauss linear;`
- SST(11_kOmegaSST): `grad(U) cellLimited Gauss linear 1;` 추가됨

**선형솔버 및 알고리즘(fvSolution)**

- p: `GAMG`, tolerance 1e-06, relTol 0.1, smoother GaussSeidel
- pcorr: `GAMG`, tolerance 1e-06, relTol 0, smoother GaussSeidel
- (U|k|epsilon|omega|f|v2): `smoothSolver`, symGaussSeidel, tolerance 1e-05, relTol 0.1
- SIMPLE
    - `nNonOrthogonalCorrectors 0; consistent yes;`
    - residualControl: p 1e-2, U 1e-3, turbulence fields 1e-3
- relaxationFactors: U 0.9, ".*" 0.9

## 환경 구축

내 컴퓨터 OS는 window다. Linux OS에서만 돌아가는 OpenFOAM을 실행하기 위해 wsl Ubuntu를 설치하였고 VScode 편집기를 이용해 진행하였다. 

OpenFOAM 버전에 따라 명령어가 다른데 나는 OpenFOAM-13에서 진행하였다.

### 0. OpenFOAM 버전 확인

```bash
foamVersion
echo $WM_PROJECT_VERSION
```

### 1. 프로젝트 폴더 뼈대 생성

```bash
echo $FOAM_RUN
mkdir -p $FOAM_RUN/portfolio_openfoam/trackA
cd $FOAM_RUN/portfolio_openfoam/trackA
pwd
```

튜토리얼 원본은 백업 파일용으로 두고, 튜토리얼 케이스를 내 작업 디렉토리로 복사해서 설정 변경과 결과를 독립적으로 기록할 수 있게 했다.

### 2. pitzDailySteady 케이스 복사 및 1회 실행

```bash
cd $FOAM_RUN/portfolio_openfoam/trackA

# 혹시 같은 폴더가 이미 있으면 충돌 나니까, 없을 때만 복사
cp -r $FOAM_TUTORIALS/incompressibleFluid/pitzDailySteady 00_pitzDailySteady
cd 00_pitzDailySteady

blockMesh | tee log.blockMesh
checkMesh | tee log.checkMesh
foamRun   | tee log.foamRun
```

여기서 나온 `blocMesh`와 `checkMesh`, `foamRun`은 각각 격자(메쉬) 생성, 메쉬 체크, 솔버 실행이다.

- `blockMesh`는  `system/blockMeshDict`알에 적힌 설정대로 격자(mesh)를 생성하는 도구이다. 케이스에서 해석 공간을 쪼개는 작업이라고 생각하면 된다.
즉, `system/blockMeshDict`파일을 읽어서 **메쉬를 만들고** `constant/polyMesh`에 저장한다.
- `checkMesh`는 셀의 비틀림/찌그러짐 같은 품질 지표(Non-orthogonality, skewness, aspect ratio 등)를 검사해서 **“해석이 터질 확률이 높은 메쉬인지” 사전에 걸러준다.** 
`checkMesh`에서 `non-orth`/`skewness`가 중요하다.
non-orthogonality(비직교성)가 크면, 이산화(유한체적)에서 **면 방향/셀 중심 방향이 어긋나** 보정이 많이 필요하고 오차·불안정이 커진다.
skewness가 크면, 보간/기울기 계산이 꼬여서 수렴이 느리거나 발산할 가능성이 커지게 된다.
밑에 나의 로그를 보면 알겠지만 나와 같은 경우는 `Mesh OK.` 가 떠서 해석 돌려도 될 정도의 품질이라고 판단하였다.
- `foamRun`은 케이스 설정(controlDict 등)에 맞춰 해석을 돌리고 로그를 남기는 실행 흐름이다.
pitzDailySteady(Backward-facing step 튜토리얼)는 **SIMPLE 알고리즘에서 residualControl 기준을 만족하면 종료**하는데, 내 케이스는 **285 iterations에서 수렴 종료하였다.**

```bash
tail -n 25 log.checkMesh
tail -n 35 log.foamRun
ls -al
```

위에서 저장된 `checkMesh`와 `foamRun`의 log 뒷부분을 확인하여 수렴했는지 어디서 오류가 생겼는지 확인할 수 있었다.

```bash
joo@JOO-DESKTOP:~/OpenFOAM/joo-13/run/portfolio_openfoam/trackA/00_pitzDailySteady$ tail -n 25 log.checkMesh
tail -n 35 log.foamRun
ls -al
    inlet               30       62       ok (non-closed singly connected)  
    outlet              57       116      ok (non-closed singly connected)  
    upperWall           223      448      ok (non-closed singly connected)  
    lowerWall           250      502      ok (non-closed singly connected)  
    frontAndBack        24450    25012    ok (non-closed singly connected)  

Checking geometry...
    Overall domain bounding box (-0.0206 -0.0254 -0.0005) (0.29 0.0254 0.0005)
    Mesh has 2 geometric (non-empty/wedge) directions (1 1 0)
    Mesh has 2 solution (non-empty) directions (1 1 0)
    All edges aligned with or perpendicular to non-empty directions.
    Max cell openness = 2.21044e-16 OK.
    Max aspect ratio = 8.1407 OK.
    Minimum face area = 1.68589e-07. Maximum face area = 5.14451e-06.  Face area magnitudes OK.
    Min volume = 1.6902e-10. Max volume = 3.83579e-09.  Total volume = 1.4516e-05.  Cell volumes OK.
    Mesh non-orthogonality Max: 5.95045 average: 1.63034
    Non-orthogonality check OK.
    Face pyramids OK.
    Max skewness = 0.260575 OK.
    Coupled point location match (average 0) OK.

Mesh OK.

End

GAMG:  Solving for p, Initial residual = 0.00156359, Final residual = 0.000143814, No Iterations 3
time step continuity errors : sum local = 0.00609788, global = -0.000523682, cumulative = 0.556781
smoothSolver:  Solving for epsilon, Initial residual = 0.000142769, Final residual = 9.0675e-06, No Iterations 3
smoothSolver:  Solving for k, Initial residual = 0.000232675, Final residual = 1.42623e-05, No Iterations 4
ExecutionTime = 5.57087 s  ClockTime = 6 s

Time = 284s

smoothSolver:  Solving for Ux, Initial residual = 0.000127676, Final residual = 1.26058e-05, No Iterations 5
smoothSolver:  Solving for Uy, Initial residual = 0.0010319, Final residual = 6.981e-05, No Iterations 6
GAMG:  Solving for p, Initial residual = 0.00131556, Final residual = 0.00012283, No Iterations 4
time step continuity errors : sum local = 0.00520864, global = 0.000614958, cumulative = 0.557396
smoothSolver:  Solving for epsilon, Initial residual = 0.00014159, Final residual = 8.97856e-06, No Iterations 3
smoothSolver:  Solving for k, Initial residual = 0.000227002, Final residual = 1.38911e-05, No Iterations 4
ExecutionTime = 5.58845 s  ClockTime = 6 s

Time = 285s

smoothSolver:  Solving for Ux, Initial residual = 0.000122524, Final residual = 1.21333e-05, No Iterations 5
smoothSolver:  Solving for Uy, Initial residual = 0.000992942, Final residual = 6.75593e-05, No Iterations 6
GAMG:  Solving for p, Initial residual = 0.00123295, Final residual = 0.000112114, No Iterations 4
time step continuity errors : sum local = 0.0047489, global = 0.000564888, cumulative = 0.55796
smoothSolver:  Solving for epsilon, Initial residual = 0.000139397, Final residual = 8.97324e-06, No Iterations 3
smoothSolver:  Solving for k, Initial residual = 0.000221974, Final residual = 1.35975e-05, No Iterations 4
ExecutionTime = 5.60688 s  ClockTime = 6 s

SIMPLE solution converged in 285 iterations

streamlines streamlines write:
    Seeded 10 particles
    Sampled 12040 locations

End

total 244
drwxr-xr-x 9 joo joo   4096 Jan 29 12:38 .
drwxr-xr-x 3 joo joo   4096 Jan 29 12:37 ..
drwxr-xr-x 2 joo joo   4096 Jan 29 12:37 0
drwxr-xr-x 3 joo joo   4096 Jan 29 12:38 100
drwxr-xr-x 3 joo joo   4096 Jan 29 12:38 200
drwxr-xr-x 3 joo joo   4096 Jan 29 12:38 285
-rwxr-xr-x 1 joo joo    316 Jan 29 12:37 Allrun
drwxr-xr-x 3 joo joo   4096 Jan 29 12:38 constant
-rw-r--r-- 1 joo joo   3108 Jan 29 12:38 log.blockMesh
-rw-r--r-- 1 joo joo   2939 Jan 29 12:38 log.checkMesh
-rw-r--r-- 1 joo joo 196701 Jan 29 12:38 log.foamRun
drwxr-xr-x 3 joo joo   4096 Jan 29 12:38 postProcessing
drwxr-xr-x 2 joo joo   4096 Jan 29 12:37 system
```

내 로그를 살펴보면 메쉬 품질, 해석 수렴, 증거 파일 생성이 잘 실행됐는지 확인할 수 있었다.

**메쉬 품질**
`Mesh OK.` 가 뜨고 `Mesh has 2 solution directions (1 1 0)` → 사실상 2D로 잘 잡힌 것을 확인할 수 있었다.

또한 

- `Mesh non-orthogonality Max: 5.95045, average: 1.63034`
- `Max skewness = 0.260575`
    
    checkMesh는 기본적으로 **non-orthogonality 경고 임계값 70°**, **skewness 임계값 4**을 사용한다. 
    non-orth: 5.95°는 기본 70°의 **약 8.5%**
    skewness: 0.26은 기본 4의 **약 6.5%**
    
    OpenFOAM checkMesh 기본 기준과 비교했을 때 경고 임계값의 8.5%, 6.5% 정도의 수치가 나와 격자 품질 문제로 인한 발산/불안정 가능성이 낮다고 판단되었다. 
    

**해석수렴**
해석 로그에 `SIMPLE solution converged in 285 iterations`가 출력되었으며, 여기서 285는 물리적 시간(sec)이 아니라 정상상태(SIMPLE) 해석의 외부 반복(iteration) 종료 지점을 의미한다.
또한 pitzDailySteady(backward-facing step) 튜토리얼에서도 계산 결과가 100, 200, 285의 time directory로 저장되며, 최종 해를 285에서 확인하도록 안내하고 있어 본 케이스의 종료 메시지를 정상 수렴으로 해석하였다.

**수렴판정기준(residual / residualControl)**
위 종료가 단순히 최대 반복 한도(`endTime`)에 도달해서 발생한 것이 아니라, 수렴 기준을 만족했기 때문에 종료된 것임을 명확히 하기 위해 `fvSolution`의 `SIMPLE.residualControl`을 확인하였다. OpenFOAM에서는 각 방정식을 풀기 전 initial residual을 평가하며, SIMPLE 기반 솔버의 경우 `residualControl`에 지정한 임계값 아래로 residual이 내려가면 계산을 종료(terminate)하도록 설정할 수 있다. 따라서 케이스의 종료는 `residualControl`로 정의한 수렴 기준을 충족한 결과로 정리하였다.
또한 본 케이스의 임계값(p 1e-2, U 1e-3, 난류 변수 1e-3)은 pitzDaily 계열 튜토리얼에서 사용하는 레퍼런스 수렴 기준을 베이스라인으로 잡고 진행하였다.

**증거 파일 생성**
시간 폴더 `0, 100, 200, 285` 가 생성되었고, `postProcessing/` 이 생성되었다.
이 출력 주기는 `system/controlDict`의 **Time control** 및 **Data writing** 설정으로 결정된다. `stopAt endTime;`과 `endTime <값>;`은 계산의 종료 시점을 지정하고, `deltaT`는 time step 크기를 정의한다. 또한 `writeControl`은 “언제 결과를 쓸지”를 정하며, `writeControl timeStep;`일 때 `writeInterval`은 몇 time step마다 저장할지를 의미한다. 예를 들어 writeInterval 100; 이면 100스텝마다 저장한다는 뜻이다. 
따라서 이 케이스폴더에서 `0/`는 초기조건(initial time directory)이고, `100/`, `200/`은 `writeControl=timeStep` + `writeInterval=100` 규칙에 의해 중간 결과가 저장되며 생성된 폴더이고 마지막 `285/`는 정상상태 SIMPLE 해석이 285 iterations에서 수렴 종료되면서 해당 시점의 결과가 기록된 최종 time directory이다.
마지막으로 `postProcessing/`는 function object(샘플링/그래프/후처리 계산 등)의 출력이 저장되는 표준 위치로, 후처리 결과가 time 디렉토리 구조로 하위에 누적된다.

## yPlus 출력

yPlus는 벽에서 첫 번째 셀 중심까지 거리(y)를 벽마찰 속도/점성 기준으로 무차원화한 값인데, 벽 근처 격자가 ‘벽함수로 해도 되는 거리’에 있는지, 아니면 점성층까지 해상해야하는 거리(y+≈1)에 있는지 판단하는 지표이다.
즉, yPlus는 OpenFOAM function object로 계산하고, 벽 근처 격자/벽함수 타당성 확인에 쓰는 지표이다. 

기본 형태는 

$y^+=\frac{u_{\tau}y}{\nu}$

$u_{\tau}= \sqrt{\tau_{\omega}/\rho}$

$y=$벽~첫 셀 중심 거리, $\nu=$동점성계수, $\tau_{\omega}=$벽전단응력

같은 RANS라도 벽 근처를 어떻게 처리했는지가 틀리면 박리, 재부착 길이, 벽전단 같은 핵심 결과가 다 틀어질 수 있기 때문에 yPlus를 먼저 뽑아보고, 내가 의도한 벽처리가 가능한 격자인지 확인하는 것이다.

### 벽 패치 이름 확정

[이미지1]

이전에 생성된 log.checkMesh를 살펴보면 upperWall과 lowerwall이 벽이다. (사진 참고)

yPlus를 전체 벽이 아니라 upperWall과 lowerWall만 계산하게 지정했다. 그 이유는 전체 벽으로 할 때보다  훨씬 더 계산이 수월하고 해석이 깔끔해지기 때문이다.

```bash
grep -n "upperWall\|lowerWall\|frontAndBack" -n constant/polyMesh/boundary
```

이렇게 실행 시켰을 때 나는 아래오 같이 출력되었다.

```bash
joo@JOO-DESKTOP:~/OpenFOAM/joo-13/run/portfolio_openfoam/trackA/00_pitzDailySteady$ grep -n "upperWall|lowerWall|frontAndBack" -n constant/polyMesh/boundary
31:    upperWall
38:    lowerWall
45:    frontAndBack
```

출력된 `upperWall`, `lowerWall`, `frontAndBack` 은 이 케이스에서 벽(패치) 이름이 무엇인지 확정했다는 뜻이다. 이게 중요한 이유는, yPlus를 계산할 때 어느 벽에서 yPlus를 볼 건지 패치 이름으로 지정하기 때문이다.

### system/controlDict에 yPlus function object 추가

밑의 코드를 system/controlDict에 붙여넣기하여 저장하였다.
controlDict에 yPlus 계산기를 달아주는 과정이다.

```bash
functions
{
    yPlus1
    {
        type        yPlus;
        libs        ("libfieldFunctionObjects.so");

        patches     (upperWall lowerWall);

        writeControl writeTime;
    }
}
```

- `type yPlus;` : yPlus 계산기
- `libs ("libfieldFunctionObjects.so");` : yPlus가 들어있는 functionObjects 라이브러리 로드
- `patches (upperWall lowerWall)` : 어떤 벽에서 yPlus를 낼지 지정
- `writeControl writeTime;` : 솔버가 결과를 쓰는 타이밍에 맞춰 같이 저장

### yPlus 계산 실행

환경마다  `postProcess -func yPlus`가 바로 될 때도 있는데, 가장 안정적인 루트는 솔버를 -postProcess로 실행하는 것이다.

일단 먼저 어떤 솔버를 썼는지 다시 확인해준다.

```bash
head -n 30 log.foamRun | grep -E "Application|Solver|Exec"
```

나와같은 경우는 모듈 솔버(incompressibleFluid)에 foamRun 흐름이기에 아래와 같이 진행하였다.

```bash
foamPostProcess -solver incompressibleFluid -func yPlus | tee log.yPlus
```

만약 다른 솔버라면 그 이름으로 바꿔서 실행시키면 된다.

실행 결과

```bash
joo@JOO-DESKTOP:~/OpenFOAM/joo-13/run/portfolio_openfoam/trackA/00_pitzDailySteady$ cd ~/OpenFOAM/joo-13/run/portfolio_openfoam/trackA/00_pitzDailySteady

# yPlus 계산 (모듈 솔버 정보까지 같이 로딩)
foamPostProcess -solver incompressibleFluid -func yPlus | tee log.yPlus
/*---------------------------------------------------------------------------*\
  =========                 |
  \\      /  F ield         | OpenFOAM: The Open Source CFD Toolbox
   \\    /   O peration     | Website:  https://openfoam.org
    \\  /    A nd           | Version:  13
     \\/     M anipulation  |
\*---------------------------------------------------------------------------*/
Build  : 13-cde978a97c93
Exec   : foamPostProcess -solver incompressibleFluid -func yPlus
Date   : Jan 30 2026
Time   : 09:41:39
Host   : "JOO-DESKTOP"
PID    : 16316
I/O    : uncollated
Case   : /home/joo/OpenFOAM/joo-13/run/portfolio_openfoam/trackA/00_pitzDailySteady
nProcs : 1
sigFpe : Enabling floating point exception trapping (FOAM_SIGFPE).
fileModificationChecking : Monitoring run-time modified files using timeStampMaster (fileModificationSkew 10)
allowSystemOperations : Allowing user-supplied system call operations

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //
Create time

Create mesh for time = 0

Time = 0s
Selecting solver incompressibleFluid
Selecting viscosity model constant
Selecting turbulence model type RAS
    Selecting RAS turbulence model kEpsilon
        Selecting generalised Newtonian model Newtonian
No MRF models present

yPlus yPlus write:
    writing object yPlus
    patch upperWall y+ : min = 2.85758, max = 7.0481, average = 5.81862
    patch lowerWall y+ : min = 5.34712, max = 18.9199, average = 15.3925

Time = 100s
Selecting solver incompressibleFluid
Selecting viscosity model constant
Selecting turbulence model type RAS
    Selecting RAS turbulence model kEpsilon
        Selecting generalised Newtonian model Newtonian
No MRF models present

yPlus yPlus write:
    writing object yPlus
    patch upperWall y+ : min = 2.85041, max = 7.62261, average = 6.09754
    patch lowerWall y+ : min = 1.59872, max = 33.571, average = 20.1757

Time = 200s
Selecting solver incompressibleFluid
Selecting viscosity model constant
Selecting turbulence model type RAS
    Selecting RAS turbulence model kEpsilon
        Selecting generalised Newtonian model Newtonian
No MRF models present

yPlus yPlus write:
    writing object yPlus
    patch upperWall y+ : min = 3.02753, max = 7.25318, average = 6.07561
    patch lowerWall y+ : min = 0.56582, max = 27.4072, average = 16.9044

Time = 285s
Selecting solver incompressibleFluid
Selecting viscosity model constant
Selecting turbulence model type RAS
    Selecting RAS turbulence model kEpsilon
        Selecting generalised Newtonian model Newtonian
No MRF models present

yPlus yPlus write:
    writing object yPlus
    patch upperWall y+ : min = 2.81958, max = 7.24192, average = 6.08872
    patch lowerWall y+ : min = 0.338914, max = 26.5171, average = 16.0754

End
```

- **Solver 모듈:** `incompressibleFluid`.
- **난류:** `RAS kEpsilon`
- **yPlus (최종 시간 = 285s 디렉토리 기준)**
    - `upperWall`: min **2.82**, max **7.24**, avg **6.09**
    - `lowerWall`: min **0.339**, max **26.52**, avg **16.08**
    
    그리고 100/200/285에서 평균이 거의 비슷하기 때문에, 수렴 이후 yPlus 통계가 안정적인 상태라고 판단했다.
    

### 결과 해석

 OpenFOAM 교육자료에서 쓰는 near-wall 기준은 크게 3개가 있다.

- 표준 wall function(고Re): **30 < y+ < 300**
- yPlus 둔감(= yPlus insensitive) wall function: **1 < y+ < 300**
- 경계층 해상(저Re, 벽까지 resolving): 대략 **y+ < 6**

이 기준으로 내 결과를 봤을 때

- **upperWall 평균 6.09** → 경계층 해상(y+<6) 에 걸쳐있음
- **lowerWall 평균 16.08 / max 26.5** → (5~30 근처) 버퍼 영역 쪽에 많이 걸쳐있음

즉, 지금 y+ 분포는 **표준 wall function(로그층 전제)을 깔끔하게 만족하는 값(>30)**도 아니고, **완전한 벽-해상(y+<6)으로 일관되게 가는 값**도 아니라서, **어느 전제를 깔고 계산 중인지가 애매한 구간**이다.

**따라서 여기서 핵심 질문은 “y+가 낮다/높다”가 아니라, 내 케이스가 어떤 벽처리(near-wall model/BC)를 전제로 하는가였다**.

왜냐하면 같은 OpenFOAM v13 backward step 튜토리얼에서는 표준 wall function(nutkWallFunction)을 사용한다고 문서에 명시돼 있기 때문이다.

만약 내 케이스(pitzDailySteady)도 동일하게 **표준 wall function 전제**라면, 원칙적으로는 y+가 주로 30 이상에 놓이는 메쉬가 자연스럽고, 지금처럼 0~26대가 넓게 깔리는 분포는 전제와 어긋나게 된다.

하지만 그렇다고 여기서 바로 “메쉬가 나쁘다”로 결론 내리진 않았다.

왜냐하면 Ansys Flow separation and Reattachment Lesson 자료에 따르면 박리(Separation)가 발생하는 구간에서는 벽 전단응력(τw)이 0에 가까워지는 지점이 존재하고(박리점 정의 자체가 τw=0), 그 주변에서는 유동이 벽 근처에서 약해지거나 역류가 생기면서 마찰속도 uτ가 작아질 수 있다.

$y+∝u_τ$ 이므로, $τ_w$↓ → $u_τ$↓ →  y+↓가 되어 **박리/재부착 문제에서는 특정 구간의 y+가 작아지는 것이 정상일 수 있다.**
그래서 BFS처럼 **박리가 본질적으로 포함된 문제**는 모든 벽에서 y+를 일정하게(예: 전 구간 30~300) 맞추는 것이 애초에 구조적으로 어려울 수 있고, 특히 **분리 버블 주변**에서는 y+가 낮아지는 현상이 나타날 수 있다.

결론적으로 y+ 값만으로 “좋다/나쁘다”를 단정하는 것이 아니라 먼저 **내 케이스가 표준 wall function(30<y+<300), y+ 둔감(1<y+<300), 벽해상(y+<6) 중 어떤 벽처리를 전제로 하는지**를 확인하고, 그 전제에서 **현재 y+ 분포가(특히 박리·재부착 구간에서 τw↓로 y+가 낮아질 수 있다는 점까지 포함해) 납득 가능한지**를 판단해야 한다.

### 벽함수 종류 확인

내 케이스(pitzDailySteady)가 표준 wall function인지, yPlus 둔감인지 내 파일에서 직접 확인할 것이다.

따라서 폴더 확인을 먼저 해준다.

```bash
# turbulent viscosity (nut) 벽 BC 확인
sed -n '1,200p' 0/nut | sed -n '1,200p'
grep -n "upperWall\|lowerWall" -n 0/nut

# k, epsilon 벽 BC 확인
grep -n "upperWall\|lowerWall" -n 0/k
grep -n "upperWall\|lowerWall" -n 0/epsilon

# 난류모델 설정 확인
ls constant
grep -RIn "RAS\|kEpsilon\|kOmega\|SST" constant 2>/dev/null | head -n 50
```

이렇게 실행시켰을 때

```bash
joo@JOO-DESKTOP:~/OpenFOAM/joo-13/run/portfolio_openfoam/trackA/00_pitzDailySteady$ # turbulent viscosity (nut) 벽 BC 확인
sed -n '1,200p' 0/nut | sed -n '1,200p'
grep -n "upperWall\|lowerWall" -n 0/nut

# k, epsilon 벽 BC 확인
grep -n "upperWall\|lowerWall" -n 0/k
grep -n "upperWall\|lowerWall" -n 0/epsilon

# 난류모델 설정 확인
ls constant
grep -RIn "RAS\|kEpsilon\|kOmega\|SST" constant 2>/dev/null | head -n 50
/*--------------------------------*- C++ -*----------------------------------*\
  =========                 |
  \\      /  F ield         | OpenFOAM: The Open Source CFD Toolbox
   \\    /   O peration     | Website:  https://openfoam.org
    \\  /    A nd           | Version:  13
     \\/     M anipulation  |
\*---------------------------------------------------------------------------*/
FoamFile
{
    format      ascii;
    class       volScalarField;
    location    "0";
    object      nut;
}
// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

dimensions      [0 2 -1 0 0 0 0];

internalField   uniform 0;

boundaryField
{
    inlet
    {
        type            calculated;
        value           uniform 0;
    }
    outlet
    {
        type            calculated;
        value           uniform 0;
    }
    upperWall
    {
        type            nutkWallFunction;
        value           uniform 0;
    }
    lowerWall
    {
        type            nutkWallFunction;
        value           uniform 0;
    }
    frontAndBack
    {
        type            empty;
    }
}

// ************************************************************************* //
33:    upperWall
38:    lowerWall
32:    upperWall
37:    lowerWall
32:    upperWall
37:    lowerWall
momentumTransport  physicalProperties  polyMesh
constant/momentumTransport:17:simulationType RAS;
constant/momentumTransport:19:RAS
constant/momentumTransport:21:    // Tested with kEpsilon, realizableKE, kOmega, kOmega2006, kOmegaSST, v2f,
constant/momentumTransport:23:    model           kEpsilon;
```

upperWall과 lowerWall type을 확인했을때 표준 wall function(`nutkWallFunction`)인 것을 확인할 수 있었다.

내 케이스는 표준 wall function을 쓰는 것이 파일로 확인됐지만 y+가 권장 범위보다 낮게 나오는 구간이 있다.

따라서 나는 ****낮은 yPlus가 “박리/재부착 구간(τw≈0)”과 대응되는지 τw로 교차검증을 하여 **전제 위반인지 유동 특성으로 인한 y+ 저하인지 확인**해볼 것이다.

```bash
foamPostProcess -solver incompressibleFluid -func wallShearStress -time 285 | tee log.wallShearStress285
tail -n 60 log.wallShearStress285
```

 285 수렴된 해에서 벽 전단응력(τw)을 계산해서, y+ 해석이 박리/재부착 때문에 낮아진 건지 확인해봤다.

```bash
joo@JOO-DESKTOP:~/OpenFOAM/joo-13/run/portfolio_openfoam/trackA/00_pitzDailySteady$ tail -n 60 log.wallShearStress285
/*---------------------------------------------------------------------------*\
  =========                 |
  \\      /  F ield         | OpenFOAM: The Open Source CFD Toolbox
   \\    /   O peration     | Website:  https://openfoam.org
    \\  /    A nd           | Version:  13
     \\/     M anipulation  |
\*---------------------------------------------------------------------------*/
Build  : 13-cde978a97c93
Exec   : foamPostProcess -solver incompressibleFluid -func wallShearStress -time 285
Date   : Feb 11 2026
Time   : 18:52:36
Host   : "JOO-DESKTOP"
PID    : 6260
I/O    : uncollated
Case   : /home/joo/OpenFOAM/joo-13/run/portfolio_openfoam/trackA/00_pitzDailySteady
nProcs : 1
sigFpe : Enabling floating point exception trapping (FOAM_SIGFPE).
fileModificationChecking : Monitoring run-time modified files using timeStampMaster (fileModificationSkew 10)
allowSystemOperations : Allowing user-supplied system call operations

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //
Create time

Create mesh for time = 285

wallShearStress wallShearStress:
    processing all wall patches

Time = 285s
Selecting solver incompressibleFluid
Selecting viscosity model constant
Selecting turbulence model type RAS
    Selecting RAS turbulence model kEpsilon
        Selecting generalised Newtonian model Newtonian
No MRF models present

wallShearStress wallShearStress write:
    writing object wallShearStress
    min/max(upperWall) = (-0.469971 -0.0436448 -2.5536e-17), (-0.156361 0.0424164 1.5731e-17)
    min/max(lowerWall) = (-0.6122 -0.129635 -5.77782e-18), (0.0915311 0.0738913 4.4785e-18)

End

```

`foamPostProcess ... wallShearStress ... -time 285`가 성공적으로 실행됐고

`writing object wallShearStress`가 찍혔으니 285 시간 디렉토리에 `wallShearStress` 필드가 생성된 게 정상이다.

로그에서 보이는 `min/max(lowerWall/upperWall) = ( ... ), ( ... )`는 그 patch 전체에서의 벡터값 최솟값/최댓값이다.

여기서 중요한 건 내 로그에는 lowerWall의 min/max에 양수와 음수의 데이터들이 모두 있다는 것이다.

`lowerWall`의 x성분(첫 번째 값)을 보면 min은 **-0.6122** (음수), max는 **+0.0915** (양수)

`upperWall`는 반면 upperWall은 x성분이 둘 다 음수이기에 전체적으로 한 방향인 상태이다.

벽 전단은 **재순환(역류) 구간에서는 부호가 바뀔 수 있고** 벽 전단의 부호가 변하는 지점을 통해 separation/reattachment 위치를 잡는다.

즉, lowerWall에서 어딘가에 0 교차점(부호 반전)이 존재할 가능성이 높다.

time=285에서 lowerWall의 wallShearStress가 patch 내에서 부호가 바뀌는(min<0, max>0) 것이 확인되어, 박리/재부착으로 인해 τw가 0에 가까워지는 구간이 존재한다는 것을 알 수 있었다. 

**따라서 전제위반이 아닌 해당 구간에서 uτ 감소로 y+가 낮아졌다고 할 수 있다.**

이제 다음 단계로 넘어가서 나중에 NASA TMR 같은 기준 데이터가 제공하는 비교 위치(x/H=1,4,6,10)에 맞춰 비교하기위해 **BFS(Backstep)에서 특정 x 위치에서의 속도 프로파일 U(y)** 같은 걸 뽑을 것이다.

## graphUniform

`graphUniform`은 OpenFOAM의 샘플링(후처리) function object 중 하나인데
도메인 안에 내가 지정한 선(line)을 하나 긋고, 그 선 위를 일정 간격(nPoints)으로 찍어서 U, p 같은 필드 값을 뽑아 `.xy` 파일로 저장해준다.

- `start (x y z);` : 선의 시작점
- `end (x y z);` : 선의 끝점
- `nPoints 100;` : 선 위에 균일 간격으로 찍을 점 개수
- `fields (U p);` : 뽑을 필드들
- `axis distance;` : 출력 파일의 x축(독립변수)을 무엇으로 할지 (x/y/z/xyz/distance 중 선택)

```bash
cd ~/OpenFOAM/joo-13/run/portfolio_openfoam/trackA/00_pitzDailySteady
cp system/graphUniform system/graphUniform.bak

#  graphUniform 설정 파일을 system/ 으로 복사
foamGet graphUniform

#  파일 열어서 start/end만 바꾸면 됨
sed -n '1,120p' system/graphUniform
```

```bash
foamPostProcess -solver incompressibleFluid | tee log.graphUniform
```

이렇게 하면 결과가 `postProcessing/graphUniform/<time>/line_U.xy` 생성 된다.

유저가이드가 pitzDaily(BFS)에서 예시로 든 라인을 참고해 `system/graphUniform`을 아래와 같이 바꿨다.

```bash
start           (0.01 -0.025 0);
end             (0.01  0.025 0);
nPoints         100;

fields          (U p);

axis            distance;

#includeEtc "caseDicts/functions/graphs/graphUniform.cfg"
```

그 다음 후처리만 다시 돌려줬다.

```bash
foamPostProcess -solver incompressibleFluid -func graphUniform -latestTime | tee log.graphUniform
```

### **x/H=1,4,6,10 으로 확장**

내 케이스의 H(스텝 높이)를 기준으로 x를 찍었다.

튜토리얼 지오메트리(유저가이드에 나온 blockMeshDict 기준)에서는 스텝이 x=0 부근에 있고 치수 스케일이 들어가 있다.

- `system/blockMeshDict`에서 **스텝 높이 H가 정확히 얼마인지** 확인
- `x = (x/H)*H`로 x 위치 계산
- `system/graphUniform`의 start/end의 x만 바꿔서
    - x/H=1 → 한번 실행해서 line_U.xy 저장(파일명 바꿔 보관)
    - x/H=4 → 또 실행 … 이런 식으로 4개 만들었다.

```bash
cp system/graphUniform system/graphUniform_xH1
cp system/graphUniform system/graphUniform_xH4
cp system/graphUniform system/graphUniform_xH6
cp system/graphUniform system/graphUniform_xH10
```

그리고 각 파일에서 `start/end`의 x만 바꾼 뒤

```bash
foamPostProcess -solver incompressibleFluid -func graphUniform_xH1  -latestTime
```

이렇게 나눠서 뽑아 파일이 섞이지 않도록 하였다.

**H와 x좌표기준 확정**

내 케이스(pitzDailySteady)는 좌표가 아래와 같이 정의되어 있다.

- step edge가 **x=0**
- step 높이 **H = 25.4mm = 0.0254m**
- `convertToMeters 0.001`(즉 blockMeshDict의 숫자는 mm 단위로 적고, m로 바꿈)
- 이건 OpenFOAM v13 유저가이드에 pitzDaily(backwardStep) `blockMeshDict` vertices가 그대로 실려 있어서 x=0에서 y가 0 → -25.4로 꺾이는 step을 확인할 수 있다.

그래서 비교 지점 x/H는 그냥 x 좌표(m)로 바꾸면:

- x/H=1 → x = 0.0254
- x/H=4 → x = 0.1016
- x/H=6 → x = 0.1524
- x/H=10 → x = 0.2540

### (A) xH1: `system/graphUniform_xH1`

```c
start   (0.0254 -0.0254 0);
end     (0.0254 0.0254 0);
nPoints 200;

fields  (U);// 일단 U만 뽑자 (헷갈림 최소화)
axis    y;// y좌표를 x축(독립변수)로 쓰게 해서 해석 쉽게
```

### (B) xH4: `system/graphUniform_xH4`

```c
start   (0.1016 -0.0254 0);
end     (0.1016 0.0254 0);
nPoints 200;

fields  (U);
axis    y;
```

### (C) xH6: `system/graphUniform_xH6`

```c
start   (0.1524 -0.0254 0);
end     (0.1524 0.0254 0);
nPoints 200;

fields  (U);
axis    y;
```

### (D) xH10: `system/graphUniform_xH10`

이 케이스는 x가 뒤로 가면 상/하 벽이 **taper** 가 되기 때문에 높이가 줄어들게 된다. 그래서  x=0.254에서 y를 ±0.0254로 잡으면 선의 일부가 벽 밖으로 나갈 수 있기 때문에 주의하여야한다.

```c
start   (0.2540 -0.02037 0);
end     (0.2540 0.02037 0);
nPoints 200;

fields  (U);
axis    y;
```

- 내가 y를 0.02037 잡은 이유는 유저가이드의 vertices를 보면 x=206mm에서 ±25.4mm, x=290mm에서 ±16.6mm로 taper고, 그 사이를 선형으로 보면 x=254mm에서 대략 ±20.37mm이다.

```bash
cd ~/OpenFOAM/joo-13/run/portfolio_openfoam/trackA/00_pitzDailySteady

foamPostProcess -solver incompressibleFluid -func graphUniform_xH1  -latestTime | tee log.graphUniform_xH1
foamPostProcess -solver incompressibleFluid -func graphUniform_xH4  -latestTime | tee log.graphUniform_xH4
foamPostProcess -solver incompressibleFluid -func graphUniform_xH6  -latestTime | tee log.graphUniform_xH6
foamPostProcess -solver incompressibleFluid -func graphUniform_xH10 -latestTime | tee log.graphUniform_xH10
```

4개를 모두 실행시키고 gnuplot로 4개를 모두 겹쳐 그렸다.

```bash
sudo apt update && sudo apt install -y gnuplot
gnuplot
```

gnuplot 안에서

```bash
set style data linespoints
plot \
"postProcessing/graphUniform_xH1/285/line.xy"  u 2:1 title "x/H=1", \
"postProcessing/graphUniform_xH4/285/line.xy"  u 2:1 title "x/H=4", \
"postProcessing/graphUniform_xH6/285/line.xy"  u 2:1 title "x/H=6", \
"postProcessing/graphUniform_xH10/285/line.xy" u 2:1 title "x/H=10"
```

실행시키면

[이미지5]

Backward-Facing Step(BFS)에서는 스텝 직후에 유동이 분리(separation)되고, 아래쪽에 **재순환(역류) 영역**이 생겼다가, 아래로 갈수록 재부착(reattachment)하고 다시 회복한다.

- 어떤 y 구간에서 Ux가 음수(왼쪽으로 튐) → 그 구간은 역류/재순환이 있다는 뜻
- x/H가 커질수록(1→4→6→10) → 보통 역류 구간이 줄고 프로파일이 “정상적인” 모양으로 회복됨
- 위쪽(upper wall 쪽)과 아래쪽(lower wall 쪽) 프로파일이 다르게 휘는 건 벽 영향 + 분리/전단층 발달이 주된 원인이라 판단함

[BFS 이미지 하나 더 해줘도 될 듯]

## 재부착 점 추출

재부착 위치를 구하는 단계에 들어가면서, 먼저 재부착을 무엇으로 정의할지부터 분명히 했다. 이 문제에서 가장 엄밀한 정의는 **벽 전단응력 τw**가 **0이 되는 지점**이며, 따라서 최종 재부착 위치는 τw=0에서 결정하는 것이 원칙이다. 다만 τw는 후처리 과정(계산, 샘플링, 보간 방식)에 따라 값이 민감하게 달라질 수 있고, 내가 얻은 결과가 정의에 맞게 제대로 나온 것인지를 한 가지 지표만으로 확신하기 어렵다. 그래서 τw=0 지점을 바로 확정하기 전에, **서로 독립적으로 교차 검증할 수 있는 기준을 하나 더 세우기로 했다.**

그 대안으로 선택한 것이 **벽 바로 위(near-wall)의 Ux** 부호 변화다. 후방계단 유동에서는 벽 근처에 재순환 영역이 형성되기 때문에 재부착 이전에는 Ux<0 (역류)이고, 재부착 이후에는 Ux>0 (정방향)으로 바뀐다. 즉, 벽 바로 위에서 **Ux가 음수에서 양수로 전환되는 x위치**는 물리적으로 재순환이 끝나고 다시 주유동이 벽을 따라 흐르기 시작하는 지점을 가리키므로, 재부착 위치의 합리적인 **대체 추정값**이 된다. 나는 먼저 이 Ux 기반 위치를 구해 재부착 위치를 먼저 구해서 기준을 만든 뒤, 그 다음에 정의인 τw=0지점을 계산해 두 결과를 비교했다. 이렇게 하면 τw=0으로 얻은 재부착 위치가 Ux부호 전환과 같은 물리적 징후와 일관되는지 확인할 수 있고, 두 값이 잘 맞아떨어질수록 내가 수행한 후처리와 판단이 타당하다는 검증 근거를 확보할 수 있다.

### 벽 유속의 방향 변화를 이용해 재부착 길이 추출

먼저 near-wall 라인 샘플용 graphUniform을 만들었다.

파일: `system/graphUniform_reattach`
내용은 기존 `system/graphUniform` 복사해서 내용만 아래처럼 바꿨다.

```cpp
start   (0.0   -0.025399 0);
end     (0.29  -0.025399 0);
nPoints 800;

fields  (U);
axis    x;

#includeEtc "caseDicts/functions/graphs/graphUniform.cfg"
```

`y=-0.0254`가 lower wall이지만 벽 위에 라인을 샘플링 할 것이기 때문에 `-0.025399` 로 살짝 올렸다.

그 후 실행!

```bash
foamPostProcess -solver incompressibleFluid -func graphUniform_reattach -latestTime | tee log.graphUniform_reattach
```

음수에서 양수 지점을 선형 보간해서 데이터를 출력했다.

```bash
f="postProcessing/graphUniform_reattach/285/line.xy"

awk '!/^#/ && NF>0{
  x=$1; ux=$2;
  if(!init){px=x; pu=ux; init=1; next}
  if(pu<0 && ux>=0){
    xr = px + (0-pu)*(x-px)/(ux-pu);
    printf("Reattachment estimate: x ~= %.6g m (between %.6g and %.6g)\n", xr, px, x);
    exit
  }
  px=x; pu=ux
}' "$f"
```

내 결과는 다음과 같았다.

```bash
joo@JOO-DESKTOP:~/OpenFOAM/joo-13/run/portfolio_openfoam/trackA/00_pitzDailySteady$ cd ~/OpenFOAM/joo-13/run/portfolio_openfoam/trackA/00_pitzDailySteady

foamPostProcess -solver incompressibleFluid -func graphUniform_reattach -latestTime | tee log.graphUniform_reattach
/*---------------------------------------------------------------------------*\
  =========                 |
  \\      /  F ield         | OpenFOAM: The Open Source CFD Toolbox
   \\    /   O peration     | Website:  https://openfoam.org
    \\  /    A nd           | Version:  13
     \\/     M anipulation  |
\*---------------------------------------------------------------------------*/
Build  : 13-cde978a97c93
Exec   : foamPostProcess -solver incompressibleFluid -func graphUniform_reattach -latestTime
Date   : Feb 02 2026
Time   : 11:43:31
Host   : "JOO-DESKTOP"
PID    : 24956
I/O    : uncollated
Case   : /home/joo/OpenFOAM/joo-13/run/portfolio_openfoam/trackA/00_pitzDailySteady
nProcs : 1
sigFpe : Enabling floating point exception trapping (FOAM_SIGFPE).
fileModificationChecking : Monitoring run-time modified files using timeStampMaster (fileModificationSkew 10)
allowSystemOperations : Allowing user-supplied system call operations

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //
Create time

Create mesh for time = 285

Reading set description:
    line

Time = 285s
Selecting solver incompressibleFluid
Selecting viscosity model constant
Selecting turbulence model type RAS
    Selecting RAS turbulence model kEpsilon
        Selecting generalised Newtonian model Newtonian
No MRF models present

End

joo@JOO-DESKTOP:~/OpenFOAM/joo-13/run/portfolio_openfoam/trackA/00_pitzDailySteady$ f="postProcessing/graphUniform_reattach/285/line.xy"

awk '!/^#/ && NF>0{
  x=$1; ux=$2;
  if(!init){px=x; pu=ux; init=1; next}
  if(pu<0 && ux>=0){
    xr = px + (0-pu)*(x-px)/(ux-pu);
    printf("Reattachment estimate: x ~= %.6g m (between %.6g and %.6g)\n", xr, px, x);
    exit
  }
  px=x; pu=ux
}' "$f"
Reattachment estimate: x ~= 0.172121 m (between 0.17204 and 0.172403)

```

내가 만든 near_wall에서 부호변화가 일어난 구간이 0.17204~0.172403 m 였다. 이 사이를 선형 보간으로 계산하여 재부착 위치 x = 0.172121 m의 값을 추출할 수 있었다.

대부분의 재부착 길이는 $x_{r/H}$ 무원 재부착 길이로 표현한다.
따라서 내 케이스에서 재부착 길이를 무차원화 해서 계산하였다.

```bash
cd ~/OpenFOAM/joo-13/run/portfolio_openfoam/trackA/00_pitzDailySteady

# graphUniform_xH1의 start (x y z)에서 x만 뽑아서 H로 사용
H=$(awk '/^start/{gsub(/[();]/,""); print $2}' system/graphUniform_xH1)

xr=0.172121

awk -v xr="$xr" -v H="$H" 'BEGIN{
  printf("H = %.8g m\nxr = %.8g m\nxr/H = %.8g\n", H, xr, xr/H)
}'
```

또한 최대 최소를 확인하여 음수에서 양수로 바뀌는 구간이 맞는지 확인하였다.

```bash
f="postProcessing/graphUniform_reattach/285/line.xy"

# Ux 최소/최대 확인 (2열이 Ux라는 가정)
awk '!/^#/ && NF>0{ if(NR==1){min=$2;max=$2} if($2<min)min=$2; if($2>max)max=$2 }
END{printf("Ux min=%g, max=%g\n",min,max)}' "$f"
```

내 결과는 다음과 같이 나왔다.

```bash
joo@JOO-DESKTOP:~/OpenFOAM/joo-13/run/portfolio_openfoam/trackA/00_pitzDailySteady$           
H=$(awk '/^start/{gsub(/[();]/,""); print $2}' system/graphUniform_xH1)
                                                                       
xr=0.172121                                                            
                                                                       
awk -v xr="$xr" -v H="$H" 'BEGIN{                                           
  printf("H = %.8g m\nxr = %.8g m\nxr/H = %.8g\n", H, xr, xr/H)
}'                                                             
H = 0.0254 m
xr = 0.172121 m
xr/H = 6.7764173

joo@JOO-DESKTOP:~/OpenFOAM/joo-13/run/portfolio_openfoam/trackA/00_pitzDailySteady$ f="postProcessing/graphUniform_reattach/285/line.xy"

awk '!/^#/ && NF>0{ if(NR==1){min=$2;max=$2} if($2<min)min=$2; if($2>max)max=$2 }
END{printf("Ux min=%g, max=%g\n",min,max)}' "$f"
Ux min=-0.0048397, max=0.00189621
```

**재부착 길이 `xr/H = 6.776`이 나왔고, Ux의 최소 최대 값을 비교해 보았을 때, `Ux min=-0.0048397, max=0.00189621` 의 값이 나온 것을 보아 음수에서 양수로 바뀌는 구간이 맞다고 판단하였다.**

### 재부착점 민감도

위에서 나온수치의 신뢰도를 올리기 위해 y 오프셋을 바꿔도 xr이 크게 안흔들리는지 확인을 해볼 것이다.

벽에서 y를 0.5 mm, 1.0 mm씩 올린 재부착선 2개를 추가로 만들어서, xr/H가 안정적인지 확인했다.

```bash
cp system/graphUniform_reattach system/graphUniform_reattach_y05mm
cp system/graphUniform_reattach system/graphUniform_reattach_y10mm
```

새로운 폴더를 만들어주고 이전에 만든 `system/graphUniform_reattach` 를 복사해서 VScode로 y만 수정해주었다.

- `system/graphUniform_reattach_y05mm` : y를 `-0.0254 + 0.0005 = -0.0249`
- `system/graphUniform_reattach_y10mm` : y를 `0.0254 + 0.0010 = -0.0244`

즉 start/end가 

- `(0 -0.0249 0)` → `(0.29 -0.0249 0)`  → y05mm
- `(0 -0.0244 0)` → `(0.29 -0.0244 0)`  → y10mm

그 이외의 axis x와 fields 는 U로 유지 하였다.

**실행, xr 다시 뽑기**

```bash
for name in graphUniform_reattach graphUniform_reattach_y05mm graphUniform_reattach_y10mm; do
  foamPostProcess -solver incompressibleFluid -func $name -latestTime > log.$name

  f="postProcessing/$name/285/line.xy"
  echo "==== $name ===="
  awk '!/^#/ && NF>0{
    x=$1; ux=$2;
    if(!init){px=x; pu=ux; init=1; next}
    if(pu<0 && ux>=0){
      xr = px + (0-pu)*(x-px)/(ux-pu);
      printf("xr = %.6g m (between %.6g and %.6g)\n", xr, px, x);
      exit
    }
    px=x; pu=ux
  }' "$f"
done
```

내 결과는 아래와 같이 나왔다.

```bash
joo@JOO-DESKTOP:~/OpenFOAM/joo-13/run/portfolio_openfoam/trackA/00_pitzDailySteady$ for name in graphUniform_reattach graphUniform_reattach_y05mm graphUniform_reattach_y10mm; do
  foamPostProcess -solver incompressibleFluid -func $name -latestTime > log.$name

  f="postProcessing/$name/285/line.xy"
  echo "==== $name ===="
  awk '!/^#/ && NF>0{
    x=$1; ux=$2;
    if(!init){px=x; pu=ux; init=1; next}
    if(pu<0 && ux>=0){
      xr = px + (0-pu)*(x-px)/(ux-pu);
      printf("xr = %.6g m (between %.6g and %.6g)\n", xr, px, x);
      exit
    }
    px=x; pu=ux
  }' "$f"
done
==== graphUniform_reattach ====
xr = 0.172121 m (between 0.17204 and 0.172403)
==== graphUniform_reattach_y05mm ====
xr = 0.170107 m (between 0.169862 and 0.170225)
==== graphUniform_reattach_y10mm ====
xr = 0.168657 m (between 0.168411 and 0.168773)
```

**결과 정리**

- H = **0.0254 m**
- (y 오프셋 0.001 m = 1mm)

**재부착 추청값**

- y = -0.025399
    - **xr = 0.172121 m → xr/H = 6.7764**
- y = -0.024899 (**+0.5 mm**)
    - **xr = 0.170107 m → xr/H = 6.6971**
- y = -0.024399 (**+1.0 mm**)
    - **xr = 0.168657 m → xr/H = 6.6400**

**민감도 변화량**

- 벽에서 **0.5 mm** 더 떼면: xr이 **약 2.014 mm** upstream(작아짐) → xr/H **0.0793**
- 벽에서 **1.0 mm** 더 떼면: xr이 **약 3.464 mm** upstream → xr/H **0.1364**
- 상대 변화(기준 대비): **약 1.17% ~ 2.01%**

**near-wall Ux=0-crossing은 y 위치가 증가함에 따라 xr가 감소하는 것을 확인할 수 있었다.**

**의문**

**왜 y를 위로 올릴수록 xr이 작아질까 의문이 생겼다.**

내가 내린 결론은 이 방법이 일단 정의가 아니고, 벽 그 자체에서의 재부착이 아니라 **벽에서 y만큼 떨어진 선에서 유동의 방향이 바뀌는 위치, 즉 근사를 해서 그렇다고 판단되었다.**

재순환 버블은 보통 벽 근처가 더 오래(더 downstream까지) 역류가 남고, 위로 갈수록 역류 두께가 줄어들기 때문에 y를 위로 올리면 순방향으로 바뀌는 xr이 조금씩 감소한다.

## 벽 전단응력으로  재부착점 추출

이번에는 재부착을 정의 기반으로 잡아서 추출 해볼 것이다.
재부착점 : lowerWall에서 벽 전단응력 또는 skin-friction(Cf)이 0으로 바뀌는 지점 
graphUniform_tauLowerWall의 입력값으로는 `wallShearStress` 필드가 필요하다.
따라서, `wallShearStress` 를 먼저 계산하고 같은 foamPostProcess 실행 안에서 이어서 graphUniform을 돌릴 것이다. 
따로 실행하게 된다면 “cannot find required object wallShearStress”문구의 오류가 떠서 같이 실행하였다.

```bash
rm -rf postProcessing/graphUniform_tauLowerWall

cat > system/graphUniform_tauLowerWall <<'EOF'
FoamFile
{
    format  ascii;
    class   dictionary;
    object  graphUniform_tauLowerWall;
}

start   (0       -0.0254  0);
end     (0.29    -0.0254  0);
nPoints 800;

fields  (wallShearStress);
axis    x;

#includeEtc "caseDicts/postProcessing/graphs/graphUniform.cfg"
EOF

# 같은 실행에서 wallShearStress 생성 -> 즉시 graphUniform에서 사용
foamPostProcess -solver incompressibleFluid -latestTime \
  -funcs '(wallShearStress graphUniform_tauLowerWall)' \
  | tee log.day5_tauLowerWall

# 결과 생성 확인

t=$(ls postProcessing/graphUniform_tauLowerWall | sort -n | tail -1)
echo "latest sampled time = $t"
ls -al postProcessing/graphUniform_tauLowerWall/$t
head -n 5 postProcessing/graphUniform_tauLowerWall/$t/line.xy

# wallShearStress가 time 폴더에 써졌는지도 확인(정상이라면 존재)
find . -maxdepth 2 -type f -name "wallShearStress" -print

# reattachment x_r/H 계산
# 코너 영향 제거: x/H > 1 (x > H) 이후만 사용
# 재부착을 "x/H>1 이후의 마지막 sign-change"로 잡는 방식

H=0.0254
x0=$H
f=postProcessing/graphUniform_tauLowerWall/$t/line.xy

awk -v H=$H -v x0=$x0 '
!/^#/ && NF==4{
  x=$1; tau=$2;

  # x/H > 1 구간 시작 전은 스킵(코너/초반 영향 컷)
  if(x < x0){px=x; pt=tau; have=1; next}

  # sign-change가 있으면 xr 갱신 (끝까지 가서 "마지막" sign-change가 남게 됨)
  if(have && (pt*tau)<0){
    xr = px + (0-pt)*(x-px)/(tau-pt);
  }

  px=x; pt=tau; have=1;
}
END{
  if(xr=="") print "NO_CROSS_AFTER_x0";
  else printf("x_r = %.8g  |  x_r/H = %.3f\n", xr, xr/H);
}' "$f"
```

- 내 케이스에서는 `line.xy`에서 **x/H>1 이후 sign-change가 딱 1번 나타나고**,
    
    그 값인 **xr≈0.17169281** 에서 **xr/H=6.760** 나왔다.
    
- **xr 이후 구간에서 tau_x의 부호가 바뀌지 않고** 한쪽 부호로 유지된 것으로 보아 재부착 점이라고 판단할 수 있었다. `AFTER xr: tau_x min=-0.0305978, max=-0.000395067`

**그리고 이제 비교 하는 거 이미지 쫙넣으면 됨 day6,7, 10←이거 확인하면 또 선형 보간으로 구했으니까 이것도 10확인해서 할 것**

비교 하는 결론글 하나 넣어주면 될 듯

## kOmegaSST로 난류 모델 교체 후 측정

Backward-facing step(BFS)은 박리–재부착이 핵심이라, 재부착 길이 같은 결과가 **난류모델에 따라 달라질 수** 있다. 그래서 baseline(k-ε 등) 한 번으로 끝내지 않고, 박리/불리한 압력구배에서 성능이 좋다고 널리 쓰이는 k-ω SST(kOmegaSST)로 모델을 바꿔 **같은 방식으로 재부착 길이를 다시 측정**했다. 이렇게 하면 내 결과가 모델 선택에 얼마나 민감한지를 확인할 수 있고, 어떤 모델을 기준으로 삼을 지 비교할 수 있기 때문이다.

내 케이스는RANS(RAS)로 평균 유동(속도/압력)을 풀고 있다. 평균 방정식 자체는 그대로지만, 평균 방정식에 추가로 생기는 난류 응력(Reynolds stress)을 어떤 모델로 닫을지는 사용자가 선택해야 한다. 

이번에 `kEpsilon → kOmegaSST`로 바꾸면서 실질적으로 달라진 점은 다음 3가지다.

**두 번째 난류 변수가 바뀜**
k-ε는 `k`와 `ε`를 풀고, k-ω(SST)는 `k`와 `ω`를 푼다. 그래서 SST로 바꾸면 `0/omega` 초기화가 필요하다. 튜토리얼에서도 `0/omega`의 내부/입구값으로 **440.2** 같은 초기화 예시를 명시한다.

**벽 근처(near-wall) 처리의 중심 변수**가 바뀜
SST는 벽에서 `ω`에 대한 제약(예: `omegaWallFunction`)을 사용한다. 즉 “벽 근처에서 무엇을 어떻게 고정/제약하느냐”가 k-ε와 다르게 구성된다.

**수렴/안정성 거동이 달라질 수 있음**
OpenFOAM backward step 예시에서는 SST에서 2차 와류(secondary vortex)가 미세하게 흔들리며, 경우에 따라 endTime까지 가는 현상을 설명하고, 이를 안정화하기 위해 `grad(U)`에 `cellLimited`를 적용하는 예시를 제시한다.

**왜 이걸 굳이 했냐면, BFS처럼 박리–재부착이 핵심인 문제에서 재부착 길이가 난류모델/벽처리 선택에 민감할 수 있어서, 모델을 바꿔도 결과가 얼마나 변하는지(=모델 민감도)를 확인하려는 목적이다.**

### 케이스 준비(복제)

난류의 종류가 바뀌었기 때문에 새로운 케이스 폴더에 실행을 할 것이다.

```bash
cd ~/OpenFOAM/joo-13/run/portfolio_openfoam/trackA

cp -r 00_pitzDailySteady 11_kOmegaSST
cd 11_kOmegaSST
```

### 난류모델을 kOmegaSST로 교체

kOmegaSST 는 뭔지 그리고 기존에 쓰던 k입실론과 무슨 차이가 있는지 설명

```bash
# 교체 전 확인
grep -n "simulationType\|model" constant/momentumTransport

# model 줄만 kOmegaSST로 교체
sed -i 's/^\(\s*model\s*\).*/\1kOmegaSST;/' constant/momentumTransport

# 교체 후 확인
grep -n "simulationType\|model" constant/momentumTransport
```

### 초기상태폴더(0폴더) 확인

kOmegaSST는 `k`, `omega`, `nut`를 쓰기 때문에 추가가 됐는지 확인해주었다. 이전에 했던 kEpsilon 케이스의 `epsilon`값은 남아있어도 보통 안 쓰는 필드로 남을 뿐이라 있어도 자동으로 무시되기에 따로 지우지는 않았다.

```bash
ls 0 | egrep 'k$|epsilon$|omega$|nut$' || true
```

그리고 `0/omega` 값이 튜토리얼 권장 계열인지 확인했다. 나 같은 경우는 internalField≈440.15, inlet value가 $internalField로 일관되어 있었다. OpenFOAM v13 backwardStep 튜토리얼에서도 `0/omega`에 440.2를 internalField와 inlet에 같이 쓰는 예를 든다.

```bash
grep -n "internalField|inlet" 0/omega
```

### 이전 결과 삭제

지금 SST를 실행하기전 kEpsilon때의 결과들이 이번 실험값에  영향을 줄 수 있기 때문에 깨끗한 재실행을 위해 삭제해주었다.

```bash
foamListTimes -rm
rm -rf postProcessing
```

### 실행 후 수렴확인

```bash
 foamRun > log.foamRun 2>&1
```

```bash
# 수렴/종료 메시지 확인(대표 라인 확인용)
grep -n "SIMPLE solution converged\|End" log.foamRun | tail
```

실행했을 때 나는 685 iterations에서 종료되었다.

```bash
Time = 685s
  SIMPLE solution converged in 685 iterations
  End
```

### 후처리용 샘플링 실행

나는 이미 `graphUniform_reattach`, `graphUniform_reattach_y05mm`, `graphUniform_reattach_y10mm`를 functionObject로 갖고 있었고, 결과가 `postProcessing/<name>/<time>/line.xy`로 떨어지는 구조였다.

```bash
T=$(foamListTimes -latestTime)

for name in graphUniform_reattach graphUniform_reattach_y05mm graphUniform_reattach_y10mm; do
  foamPostProcess -solver incompressibleFluid -func $name -time $T > log.$name.t$T 2>&1
done
```

**line.xy 포맷 확인** 

NF=4, 즉 컬럼이 `x Ux Uy Uz` 가 나와야한다.

```bash
head -n 5 postProcessing/graphUniform_reattach/$T/line.xy
awk 'NF && $1!~/#/{print "NF=",NF,"(expect 4)"; exit}' postProcessing/graphUniform_reattach/$T/line.xy
```

### Ux=0 교차점 계산

 난류의 종류만 바뀌었기 때문에 이전에 사용했던 graphUniform들(`graphUniform_reattach, y05mm, y10mm`)은 그대로 사용하여 전과 비교할 수 있게 했다.

벽 바로 위에서 추출한 Ux 데이터를 따라가며 재부착 위치를 잡을 때는, 단순히 처음 0을 넘는 지점이나 마지막 교차점만 고르면 작은 국소 포켓(국부적인 역류/재순환)에 걸려 재부착을 잘못 판정할 위험이 있다. 그래서 나는 Ux의 부호가 바뀌는 구간을 **역류 구간(segment)** 으로 정의하고, 그중에서도 **가장 길이가 긴 역류 구간**이 주 재순환(primary recirculation)을 대표한다고 보고 그 끝점을 재부착 후보로 선택하는 방식으로 절차를 설계했다.

구체적으로는 x 방향으로 데이터를 순차적으로 훑으면서 Ux가 **양수에서 음수로 바뀌는 순간**을 역류 구간의 시작점(segStart)으로 기록하고, 이후 Ux가 **음수에서 양수로 다시 바뀌는 순간**을 역류 구간의 끝점(segEnd)으로 기록한다. 이때 segEnd는 격자 점 사이에서 정확히 0을 찍지 않을 수 있으므로, 부호가 바뀌는 두 점 (px,pu), (x,ux) 사이에서 Ux=0이 되는 위치를 **선형 보간**으로 계산해 끝점을 정밀화한다(예: $segEnd=p_x+(0-p_u)(x-p_x)/(u_x-p_u)$). 이렇게 얻은 각 역류 구간에 대해 길이 $segLen=segEnd-segStart$를 계산한 뒤, 전체 역류 구간들 중에서 segLen이 **가장 큰 구간**을 선택하고, 그 구간의 끝점 segEnd를 bestX로 두었다. 마지막으로 bestX를 재부착 위치 xr로 출력하고, 외부에서 $H=0.0254$를 사용해 xr/H형태로 무차원화 값을 계산했다.

이 접근을 택한 이유는 명확하다. 단일 교차점 기준은 데이터의 잡음이나 국소 재순환에 민감해 “진짜 재부착”이 아니라 작은 포켓의 경계를 재부착으로 착각할 수 있다. 반면 **“최장 역류 구간의 끝점”** 규칙은 주 재순환 영역의 끝을 잡는 데 더 안정적이며, 결과적으로 τw=0 기반 재부착 위치와 비교했을 때도 더 일관된 교차 검증 기준으로 기능한다.

```bash
H=0.0254
xMax=999 # 기본은 제한 없음

for name in graphUniform_reattach graphUniform_reattach_y05mm graphUniform_reattach_y10mm; do
  f="postProcessing/$name/$T/line.xy"

  xr=$(awk -v xMax="$xMax" '
  !/^#/ && NF>0{
    x=$1; ux=$2;
    if(x > xMax) exit

    if(!init){px=x; pu=ux; init=1; next}

    if(!inNeg && pu>=0 && ux<0){ inNeg=1; segStart=px }
    if(inNeg && pu<0 && ux>=0){
      segEnd = px + (0-pu)*(x-px)/(ux-pu)
      segLen = segEnd - segStart
      if(segLen > bestLen){bestLen=segLen; bestX=segEnd}
      inNeg=0
    }
    px=x; pu=ux
  }
  END{ if(bestLen>0) printf("%.12g\n", bestX); }' "$f")

  printf "%-30s xr=%.6g m  xr/H=%.6f (T=%s)\n" \
    "$name" "$xr" "$(awk -v xr="$xr" -v H="$H" 'BEGIN{print xr/H}')" "$T"
done
```

**taper 영향 분리 검증**

**taper 영향 분리 검증**을 따로 한 이유는, 내가 사용하는 pitzDailySteady 케이스의 메쉬가 일반적인 BFS(backward-facing step) 형상과 완전히 동일하지 않기 때문이다. 메쉬를 보면 outlet 쪽에 **taper(단면이 점진적으로 변하는) 구간**이 존재하는데, 이 구간이 생기면 출구 경계 조건이나 압력 회복 과정이 달라지면서 하류 유동이 미세하게 변형될 가능성이 있다. 만약 그 영향이 상류까지 미친다면, 내가 구하려는 **재부착 길이 xr** 자체가 taper 때문에 달라졌다고 오해할 수도 있다. 그래서 재부착 길이가 제대로 된 값인지 확인하려면, taper의 영향을 **계산에서 분리**해 검증하는 과정이 필요하다고 판단했다.

이를 위해 후처리에서 x 범위를 제한했다. taper 구간의 시작 위치가  $x=0.206 m$이므로, x가 이 값을 넘는 순간부터는 데이터를 더 이상 사용하지 않고 **계산을 중단**하도록 설정했다. 즉, taper 구간에 들어가기 전까지의 영역만으로 재부착 위치를 계산한 값과, 기존처럼 전체 구간을 사용해 계산한 값을 비교한 것이다. 두 계산 결과가 동일하게 나오면, 재부착 위치가 taper 구간의 존재와 무관하게 이미 그 이전에서 결정된다는 의미가 되고, 따라서 **taper가 재부착 길이에 영향을 주지 않았다**고 결론 내릴 수 있다. 반대로 두 값의 차이가 크다면, taper가 후류 구조에 영향을 주었을 가능성을 열어두고 추가 점검이 필요하다고 생각했다.

```bash
xMax=0.206

for name in graphUniform_reattach graphUniform_reattach_y05mm graphUniform_reattach_y10mm; do
  f="postProcessing/$name/$T/line.xy"

  xr=$(awk -v xMax="$xMax" '
  !/^#/ && NF>0{
    x=$1; ux=$2;
    if(x > xMax) exit

    if(!init){px=x; pu=ux; init=1; next}
    if(!inNeg && pu>=0 && ux<0){ inNeg=1; segStart=px }
    if(inNeg && pu<0 && ux>=0){
      segEnd = px + (0-pu)*(x-px)/(ux-pu)
      segLen = segEnd - segStart
      if(segLen > bestLen){bestLen=segLen; bestX=segEnd}
      inNeg=0
    }
    px=x; pu=ux
  }
  END{ if(bestLen>0) printf("%.12g\n", bestX); }' "$f")

  printf "%-30s xr=%.6g m  xr/H=%.6f (x<=%.3f)\n" \
    "$name" "$xr" "$(awk -v xr="$xr" -v H="$H" 'BEGIN{print xr/H}')" "$xMax"
done
```

### 시간 안정성 검증

이후 내가 최종적으로 보고하려는 관심량(QoI)인 **재부착 길이 xr/H**가 **시간적으로 충분히 안정** 되었는지를 검증해야 했다. 이전에 수렴판단을 내릴때 residual 기준을 만족해서 계산이 종료되었는지 확인하는 “해석 종료 확인”을 했다면, 지금은은 종료 시점의 xr/H가 우연히 찍힌 값이 아니라 수렴된 값인지를 확인하는 “QoI 수렴 확인”이라고 할 수 있다.

비교 시점으로 **t=600과 t=685**를 선택했다. 이 케이스는 controlDict의 **writeInterval이 100**이라 중간 저장이 100, 200, …, 600처럼 떨어져 기록되고, 계산이 종료되면서 **마지막 저장 시점이 685**로 남는다. 즉, “종료 직전 값(685)”이 얼마나 안정적인지 판단하려면, 그 직전의 충분히 늦은 저장 지점인 “600”과 비교하는 것이 가장 자연스럽다. 이를 위해 동일한 조건을 유지한 채로 시간만 바꿔서, `foamPostProcess -time 600`과 `foamPostProcess -time 685`로 **같은 라인 데이터**를 다시 생성했고, 두 시점에서 계산된 xr/H를 비교하여 변화율을 산출했다. 여기서 핵심은 **조건 고정**이다. 시간만 다르고 나머지(라인 위치, 추출 방식, 재부착 판정 알고리즘, x 제한, H 값)가 모두 동일해야만, 두 값의 차이를 제대로 해석할 수 있다.

```bash
 T1=600
T2=$T   # 보통 685

for TT in $T1 $T2; do
  for name in graphUniform_reattach graphUniform_reattach_y05mm graphUniform_reattach_y10mm; do
    foamPostProcess -solver incompressibleFluid -func $name -time $TT > log.$name.t$TT 2>&1
  done
done

xMax=0.206  # (선택) taper 전 제한까지 같이 적용해도 좋음
for TT in $T1 $T2; do
  echo "---- Time $TT ----"
  for name in graphUniform_reattach graphUniform_reattach_y05mm graphUniform_reattach_y10mm; do
    f="postProcessing/$name/$TT/line.xy"
    xr=$(awk -v xMax="$xMax" '
    !/^#/ && NF>0{
      x=$1; ux=$2;
      if(x > xMax) exit
      if(!init){px=x; pu=ux; init=1; next}
      if(!inNeg && pu>=0 && ux<0){ inNeg=1; segStart=px }
      if(inNeg && pu<0 && ux>=0){
        segEnd = px + (0-pu)*(x-px)/(ux-pu)
        segLen = segEnd - segStart
        if(segLen > bestLen){bestLen=segLen; bestX=segEnd}
        inNeg=0
      }
      px=x; pu=ux
    }
    END{ if(bestLen>0) printf("%.12g\n", bestX); }' "$f")
    printf "%-30s xr/H=%.6f\n" \
      "$name" "$(awk -v xr="$xr" -v H="$H" 'BEGIN{print xr/H}')" 
  done
done
```

실제로 비교 결과는 **y05mm에서 7.855078→7.849779로 -0.07%, y10mm에서 7.782667→7.749403로 -0.43% 수준의 미세한 변화**만 나타났다. 
특히 대표 라인(예: y=0, y=0.5mm)에서는 값이 거의 고정된 수준으로 관찰되었기 때문에, t=685에서 보고한 $x_r/H≈7.7∼7.9$는 종료 직전의 일시적인 값이 아니라 **안정값**이라고 볼 수 있다.

이 시간 안정성 검증을 한 이유는 residual 기준만으로는 잡히지 않는 리스크들을 걸러내기 때문이다. 계산이 너무 빨리 종료돼 QoI가 아직 이동 중이거나, 후반부에 진동이 남아 있는데 residual만 우연히 기준을 만족했거나, 혹은 후처리 규칙/추출 조건이 달라져 값이 흔들리는 경우가 있을 수 있다. 

**즉, $x_r/H$의 시간 안정성을 t=600과 t=685에서 동일 알고리즘으로 비교했으며, 변화율이 0.07~0.43%로 작아 최종 값이 안정된 상태임을 확인할 수 있었다.**

### kOmegaSST 재부착 길이

 (t=685, x≤0.206 제한에서도 동일)

- y=0 라인: **xr/H = 7.936760**
- y=0.5mm: **xr/H = 7.849780**
- y=10mm: **xr/H = 7.749400**

**→ 대표값으론 벽 바로 위 y=0.5mm 의 재부착 길이인 xr/H=7.8498로 결정하였다.** 

**의문**

**왜 차이가 났을까**

![image.png]({{ '/assets/posts/2026-02-16-pitzDailySteady-BFS/1.png' | relative_url }})
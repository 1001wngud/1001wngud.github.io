---
title: "BFS · 01. Track A README"
permalink: /bfs/01-tracka-readme/
---

[← Back to BFS Project]({{ "/portfolio/pitzdailysteady-bfs/" | relative_url }})

## 재부착 길이 추출, 파이프라인 구축, 난류모델 민감도 측정

![BFS]({{ '/assets/posts/2026-02-16-pitzDailySteady-BFS-1/image03.png' | relative_url }})

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

<div class="img-grid-2">
 <img src="{{ '/assets/posts/2026-02-16-pitzDailySteady-BFS-1/image01.png' | relative_url }}" alt="image01">
 <img src="{{ '/assets/posts/2026-02-16-pitzDailySteady-BFS-1/image02.png' | relative_url }}" alt="image02">
</div>

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
- (U\|k\|epsilon\|omega\|f\|v2): `smoothSolver`, symGaussSeidel, tolerance 1e-05, relTol 0.1
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

**출력 결과**
```bash                         
[Mesh]                                                                                
Mesh has 2 solution (non-empty) directions (1 1 0)
Mesh non-orthogonality Max: 5.95045 average: 1.63034
Max skewness = 0.260575 OK.                                                                                   
Mesh OK.                                                                                 
```
```bash
[Convergence]
SIMPLE solution converged in 285 iterations
```
```bash
[Evidence files]
0/  100/  200/  285/  postProcessing/  
```

<details markdown="block">
<summary>로그 원문(클릭)</summary>

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
</details> 


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

![Mesh1]({{ '/assets/posts/2026-02-16-pitzDailySteady-BFS-1/image01.png' | relative_url }})

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
[Command]
foamPostProcess -solver incompressibleFluid -func yPlus | tee log.yPlus
```
```bash
[Model check]
Selecting solver incompressibleFluid
Selecting RAS turbulence model kEpsilon
```
```bash
[Final (Time = 285s)]
patch upperWall y+ : min = 2.81958, max = 7.24192, average = 6.08872
patch lowerWall y+ : min = 0.338914, max = 26.5171, average = 16.0754
```
```bash
[Stability evidence]
Time = 100s ... upper avg = 6.09754, lower avg = 20.1757
Time = 200s ... upper avg = 6.07561, lower avg = 16.9044
Time = 285s ... upper avg = 6.08872, lower avg = 16.0754
```

<details markdown="block">
<summary>로그 원문(클릭)</summary>

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

</details>


- **Solver 모듈:** `incompressibleFluid`.
- **난류:** `RAS kEpsilon`
- **yPlus (최종 시간 = 285s 디렉토리 기준)**
    - `upperWall`: min **2.82**, max **7.24**, avg **6.09**
    - `lowerWall`: min **0.339**, max **26.52**, avg **16.08**
    
    그리고 100/200/285에서 평균이 거의 비슷하기 때문에, 수렴 이후 yPlus 통계가 안정적인 상태라고 판단했다.
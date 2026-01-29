---
layout: page
title: "튜토리얼: OpenFOAM 간단한 예제 코드 + paraFoam 실행"
permalink: /projects/cfd/log/tutorial-simulation/
---

## 튜토리얼 (pitzDailySteady)

환경 구축 이후 **pitzDailySteady**라는 국룰 예제를 한 번 튜토리얼 삼아 돌려볼 것이다.

내가 돌릴 pitzDailySteady는 **BFS(Backward-Facing Step)** 유동 튜토리얼이다. 단차(step) 뒤에서 유동이 **박리(separation)** 됐다가 다시 **재부착(reattachment)** 되는, 검증(V&V)에서 자주 나오는 대표 케이스다.

---

### 튜토리얼 돌리려고 내가 처음에 입력한 것

```
source /opt/openfoam13/etc/bashrc

mkdir -p $FOAM_RUN
cd $FOAM_RUN

cp -r $FOAM_TUTORIALS/modules/incompressibleFluid/pitzDailySteady .
cd pitzDailySteady

blockMesh
foamRun
```

여기까지 하면 계산이 돌고(steady라 반복하면서) 수렴하면 멈출 거라고 생각했다.

근데... 바로 실패했다.

---

### 에러 분석 및 해결

```
joo@JOO-DESKTOP:~/OpenFOAM/joo-13/run$ source /opt/openfoam13/etc/bashrc

mkdir -p $FOAM_RUN
cd $FOAM_RUN

cp -r $FOAM_TUTORIALS/modules/incompressibleFluid/pitzDailySteady .
cd pitzDailySteady

blockMesh
foamRun
cp: cannot stat '/opt/openfoam13/tutorials/modules/incompressibleFluid/pitzDailySteady': No such file or directory
-bash: cd: pitzDailySteady: No such file or directory
/*---------------------------------------------------------------------------*  =========                 |
  \      /  F ield         | OpenFOAM: The Open Source CFD Toolbox
   \    /   O peration     | Website:  https://openfoam.org
    \  /    A nd           | Version:  13
     \/     M anipulation  |
\*---------------------------------------------------------------------------*/
Build  : 13-cde978a97c93
Exec   : blockMesh
Date   : Jan 27 2026
Time   : 21:49:04
Host   : "JOO-DESKTOP"
PID    : 9252
I/O    : uncollated
Case   : /home/joo/OpenFOAM/joo-13/run
nProcs : 1
sigFpe : Enabling floating point exception trapping (FOAM_SIGFPE).
fileModificationChecking : Monitoring run-time modified files using timeStampMaster (fileModificationSkew 10)
allowSystemOperations : Allowing user-supplied system call operations

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //
Create time

--> FOAM FATAL ERROR:
cannot find file "/home/joo/OpenFOAM/joo-13/run/system/controlDict"

    From function virtual Foam::autoPtr<Foam::ISstream> Foam::fileOperations::uncollatedFileOperation::readStream(Foam::regIOobject&, const Foam::fileName&, const Foam::word&, bool) const
    in file global/fileOperations/uncollatedFileOperation/uncollatedFileOperation.C at line 538.

FOAM exiting

/*---------------------------------------------------------------------------*  =========                 |
  \      /  F ield         | OpenFOAM: The Open Source CFD Toolbox
   \    /   O peration     | Website:  https://openfoam.org
    \  /    A nd           | Version:  13
     \/     M anipulation  |
\*---------------------------------------------------------------------------*/
Build  : 13-cde978a97c93
Exec   : foamRun
Date   : Jan 27 2026
Time   : 21:49:05
Host   : "JOO-DESKTOP"
PID    : 9253
I/O    : uncollated
Case   : /home/joo/OpenFOAM/joo-13/run
nProcs : 1
sigFpe : Enabling floating point exception trapping (FOAM_SIGFPE).
fileModificationChecking : Monitoring run-time modified files using timeStampMaster (fileModificationSkew 10)
allowSystemOperations : Allowing user-supplied system call operations

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //
Create time

--> FOAM FATAL ERROR:
cannot find file "/home/joo/OpenFOAM/joo-13/run/system/controlDict"

    From function virtual Foam::autoPtr<Foam::ISstream> Foam::fileOperations::uncollatedFileOperation::readStream(Foam::regIOobject&, const Foam::fileName&, const Foam::word&, bool) const
    in file global/fileOperations/uncollatedFileOperation/uncollatedFileOperation.C at line 538.

FOAM exiting
```

에러를 딱 뜯어보면 원인은 단순했다.

- `cp: cannot stat ... No such file or directory`  
  → 내가 쓴 튜토리얼 경로가 실제로 없어서 복사가 안 됨.

- 복사가 안 됐으니 `cd pitzDailySteady`도 당연히 실패.

- 그런데도 `blockMesh`, `foamRun`은 실행이 됐고, 실행 위치가 여기로 찍혔다:  
  `Case : /home/joo/OpenFOAM/joo-13/run`  
  → 즉, 케이스 폴더가 아니라 **run 폴더에서 그냥 실행해버린 것**.

그래서 OpenFOAM이 케이스 기본 파일을 못 찾는다는 에러로 이어졌다.

- `cannot find file ".../run/system/controlDict"`  
  → 정상 케이스는 `system/controlDict`가 있어야 돌아가는데, run 폴더에는 그런 게 없다.

결론: **튜토리얼이 실제로 어디 있는지부터 찾고**, 그걸 **케이스 폴더로 복사한 뒤** 그 안에서 실행해야 한다.

---

### 튜토리얼이 어디 있는지 찾기 (내가 한 방식)

OpenFOAM은 버전/패키징에 따라 튜토리얼 경로 표기가 가끔 다르게 보일 때가 있어서(내가 딱 그 케이스), 그냥 찾는게 제일 빠르다.

1) 튜토리얼 루트 확인

```
source /opt/openfoam13/etc/bashrc
echo $FOAM_TUTORIALS
ls -1 $FOAM_TUTORIALS | head
```

2) pitzDailySteady 폴더 검색

```
find "$FOAM_TUTORIALS" -maxdepth 5 -type d -name "pitzDailySteady"
```

내 출력은 아래와 같았다. (그대로)

```
joo@JOO-DESKTOP:~/OpenFOAM/joo-13/run$ source /opt/openfoam13/etc/bashrc
echo $FOAM_TUTORIALS
ls -1 $FOAM_TUTORIALS | head
/opt/openfoam13/tutorials
Allclean
Allrun
Alltest
XiFluid
compressibleMultiphaseVoF
compressibleVoF
film
fluid
incompressibleDenseParticleFluid
incompressibleDriftFlux
joo@JOO-DESKTOP:~/OpenFOAM/joo-13/run$ find "$FOAM_TUTORIALS" -maxdepth 5 -type d -name "pitzDailySteady"
/opt/openfoam13/tutorials/incompressibleFluid/pitzDailySteady
```

즉, 내 환경에서는 이게 정답이었다.

- `$FOAM_TUTORIALS/incompressibleFluid/pitzDailySteady`

그래서 아까처럼 `modules/...`로 잡았던 게 틀렸던 거고, 실제론 `incompressibleFluid/...` 쪽이었다.

---

### 올바른 경로로 복사해서 실행

이제 제대로 복사하고 케이스 폴더 안에서 실행한다.

```
source /opt/openfoam13/etc/bashrc

cd $FOAM_RUN
cp -r "$FOAM_TUTORIALS/incompressibleFluid/pitzDailySteady" .
cd pitzDailySteady
```

튜토리얼 폴더에는 보통 `Allrun` 스크립트가 같이 들어있고, 이걸 쓰면 블록메시 → 솔버 → (필요시) 후처리 준비까지 정해진 순서로 돌아가서 실수 확률이 줄어든다.

```
./Allrun
```

---

### 케이스 폴더가 제대로 만들어졌는지 확인

난 실행 후에 `pwd`, `ls`로 구조가 정상인지 확인했다.

```
pwd
ls
```

내 출력은 아래와 같다. (그대로)

```
joo@JOO-DESKTOP:~/OpenFOAM/joo-13/run$ source /opt/openfoam13/etc/bashrc

cd $FOAM_RUN
cp -r "$FOAM_TUTORIALS/incompressibleFluid/pitzDailySteady" .
cd pitzDailySteady
joo@JOO-DESKTOP:~/OpenFOAM/joo-13/run/pitzDailySteady$ ./Allrun
Running blockMesh on /home/joo/OpenFOAM/joo-13/run/pitzDailySteady
Running foamRun on /home/joo/OpenFOAM/joo-13/run/pitzDailySteady
joo@JOO-DESKTOP:~/OpenFOAM/joo-13/run/pitzDailySteady$ cd pitzDailySteady
-bash: cd: pitzDailySteady: No such file or directory
joo@JOO-DESKTOP:~/OpenFOAM/joo-13/run/pitzDailySteady$ pwd
ls
/home/joo/OpenFOAM/joo-13/run/pitzDailySteady
0  100  200  285  Allrun  constant  log.blockMesh  log.foamRun  postProcessing  system
joo@JOO-DESKTOP:~/OpenFOAM/joo-13/run/pitzDailySteady$
```

여기서 보면:

- `0/`, `constant/`, `system/` → 케이스 폴더 기본 3종이 생김  
- `log.blockMesh`, `log.foamRun` → Allrun이 남긴 실행 로그가 생김  
- `100`, `200`, `285` → 계산 진행하면서 time 디렉토리 생성됨  
- `postProcessing/` → 후처리 결과 폴더 생성됨  

정상적으로 돌아갔다고 봐도 된다.

(중간에 내가 `cd pitzDailySteady`를 한 번 더 쳤는데, 이미 그 폴더 안이었으니까 당연히 “없다”고 뜬 것. 이건 그냥 내가 실수로 한 번 더 친 거라 무시해도 된다.)

---

### 계산이 진짜 끝났는지, 메쉬는 괜찮은지 확인

수렴/종료 확인은 로그 tail로 봤고:

```
tail -n 30 log.foamRun
```

메쉬 품질은 checkMesh로 봤다:

```
checkMesh | tee log.checkMesh
```

내 출력은 아래와 같다. (그대로)

```
joo@JOO-DESKTOP:~/OpenFOAM/joo-13/run/pitzDailySteady$ tail -n 30 log.foamRun

Time = 284s

smoothSolver:  Solving for Ux, Initial residual = 0.000127676, Final residual = 1.26058e-05, No Iterations 5
smoothSolver:  Solving for Uy, Initial residual = 0.0010319, Final residual = 6.981e-05, No Iterations 6
GAMG:  Solving for p, Initial residual = 0.00131556, Final residual = 0.00012283, No Iterations 4
time step continuity errors : sum local = 0.00520864, global = 0.000614958, cumulative = 0.557396
smoothSolver:  Solving for epsilon, Initial residual = 0.00014159, Final residual = 8.97856e-06, No Iterations 3
smoothSolver:  Solving for k, Initial residual = 0.000227002, Final residual = 1.38911e-05, No Iterations 4
ExecutionTime = 4.63522 s  ClockTime = 5 s

Time = 285s

smoothSolver:  Solving for Ux, Initial residual = 0.000122524, Final residual = 1.21333e-05, No Iterations 5
smoothSolver:  Solving for Uy, Initial residual = 0.000992942, Final residual = 6.75593e-05, No Iterations 6
GAMG:  Solving for p, Initial residual = 0.00123295, Final residual = 0.000112114, No Iterations 4
time step continuity errors : sum local = 0.0047489, global = 0.000564888, cumulative = 0.55796
smoothSolver:  Solving for epsilon, Initial residual = 0.000139397, Final residual = 8.97324e-06, No Iterations 3
smoothSolver:  Solving for k, Initial residual = 0.000221974, Final residual = 1.35975e-05, No Iterations 4
ExecutionTime = 4.64914 s  ClockTime = 5 s


SIMPLE solution converged in 285 iterations

streamlines streamlines write:
    Seeded 10 particles
    Sampled 12040 locations

End
```

`SIMPLE solution converged in 285 iterations` + `End`가 찍혔으니 정상 종료로 보면 된다.

그리고 `checkMesh`는 아래처럼 `Mesh OK.`로 통과했다.

```
joo@JOO-DESKTOP:~/OpenFOAM/joo-13/run/pitzDailySteady$ checkMesh | tee log.checkMesh
/*---------------------------------------------------------------------------*  =========                 |
  \      /  F ield         | OpenFOAM: The Open Source CFD Toolbox
   \    /   O peration     | Website:  https://openfoam.org
    \  /    A nd           | Version:  13
     \/     M anipulation  |
\*---------------------------------------------------------------------------*/
Build  : 13-cde978a97c93
Exec   : checkMesh
Date   : Jan 27 2026
Time   : 21:57:45
Host   : "JOO-DESKTOP"
PID    : 10415
I/O    : uncollated
Case   : /home/joo/OpenFOAM/joo-13/run/pitzDailySteady
nProcs : 1
sigFpe : Enabling floating point exception trapping (FOAM_SIGFPE).
fileModificationChecking : Monitoring run-time modified files using timeStampMaster (fileModificationSkew 10)
allowSystemOperations : Allowing user-supplied system call operations

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //
Create time

Create mesh for time = 0

Time = 0s

Mesh stats
    points:           25012
    internal points:  0
    faces:            49180
    internal faces:   24170
    cells:            12225
    faces per cell:   6
    boundary patches: 5
    point zones:      0
    face zones:       0
    cell zones:       0

Overall number of cells of each type:
    hexahedra:     12225
    prisms:        0
    wedges:        0
    pyramids:      0
    tet wedges:    0
    tetrahedra:    0
    polyhedra:     0

Checking topology...
    Boundary definition OK.
    Cell to face addressing OK.
    Point usage OK.
    Upper triangular ordering OK.
    Face vertices OK.
    Number of regions: 1 (OK).

Checking patch topology for multiply connected surfaces...
    Patch               Faces    Points   Surface topology
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
```

여기까지로 “튜토리얼 실행” 자체는 완료.

---

### 이제 기다리고 기다리던 후처리 작업

사실 이 후처리 작업이 제일 다가가기 쉽고 예쁘니 기대를 안 할 수가 없었다.

케이스 폴더에서 `paraFoam`을 출력하면 ParaView가 떠야 한다.

```
paraFoam
```

내 터미널 출력은 아래처럼 나왔다.

<figure>
  <img src="/assets/images/tutorial1.png" alt="ParaView 실행" />
  <figcaption>그림 1) ParaView 실행 </figcaption>
</figure>

```
joo@JOO-DESKTOP:~/OpenFOAM/joo-13/run/pitzDailySteady$ paraFoam
Created temporary 'pitzDailySteady.OpenFOAM'
I/O    : uncollated
```


<figure>
  <img src="/assets/images/tutorial2.png" alt="ParaView" />
  <figcaption>그림 2) ParaView </figcaption>
</figure>

ParaView 창은 이런식으로 나왔다.

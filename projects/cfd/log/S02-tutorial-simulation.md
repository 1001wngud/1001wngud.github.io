---
layout: page
title: "튜토리얼: OpenFOAM 간단한 예제 코드 + paraFoam 실행"
permalink: /projects/cfd/log/tutorial-simulation/
---

##튜토리얼

환경 구축 이후 pitzDailySteady라는 국룰 예제를 한 번 튜토리얼 삼아 돌려볼 것이다.

내가 돌릴 pitzDailySteady는 BFS(Backward-Facing Step) 유동 튜토리얼이다. 난류가 단차 뒤에서 박리(separation)→재부착(reattachment) 되는 전형적인 검증 문제다. 

튜토리얼을 돌리기 위해 아래 사항들을 출력했다.

```
source /opt/openfoam13/etc/bashrc

mkdir -p $FOAM_RUN
cd $FOAM_RUN

cp -r $FOAM_TUTORIALS/modules/incompressibleFluid/pitzDailySteady .
cd pitzDailySteady

blockMesh
foamRun
```

여기까지 하면 계산이 돌고 steady라 반복하다 멈출 것이다. 

근데...?

```
joo@JOO-DESKTOP:~/OpenFOAM/joo-13/run$ source /opt/openfoam13/etc/bashrc mkdir -p $FOAM_RUN cd $FOAM_RUN cp -r $FOAM_TUTORIALS/modules/incompressibleFluid/pitzDailySteady . cd pitzDailySteady blockMesh foamRun cp: cannot stat '/opt/openfoam13/tutorials/modules/incompressibleFluid/pitzDailySteady': No such file or directory -bash: cd: pitzDailySteady: No such file or directory /*---------------------------------------------------------------------------*\ ========= | \\ / F ield | OpenFOAM: The Open Source CFD Toolbox \\ / O peration | Website: https://openfoam.org \\ / A nd | Version: 13 \\/ M anipulation | \*---------------------------------------------------------------------------*/ Build : 13-cde978a97c93 Exec : blockMesh Date : Jan 27 2026 Time : 21:49:04 Host : "JOO-DESKTOP" PID : 9252 I/O : uncollated Case : /home/joo/OpenFOAM/joo-13/run nProcs : 1 sigFpe : Enabling floating point exception trapping (FOAM_SIGFPE). fileModificationChecking : Monitoring run-time modified files using timeStampMaster (fileModificationSkew 10) allowSystemOperations : Allowing user-supplied system call operations // * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * // Create time --> FOAM FATAL ERROR: cannot find file "/home/joo/OpenFOAM/joo-13/run/system/controlDict" From function virtual Foam::autoPtr<Foam::ISstream> Foam::fileOperations::uncollatedFileOperation::readStream(Foam::regIOobject&, const Foam::fileName&, const Foam::word&, bool) const in file global/fileOperations/uncollatedFileOperation/uncollatedFileOperation.C at line 538. FOAM exiting /*---------------------------------------------------------------------------*\ ========= | \\ / F ield | OpenFOAM: The Open Source CFD Toolbox \\ / O peration | Website: https://openfoam.org \\ / A nd | Version: 13 \\/ M anipulation | \*---------------------------------------------------------------------------*/ Build : 13-cde978a97c93 Exec : foamRun Date : Jan 27 2026 Time : 21:49:05 Host : "JOO-DESKTOP" PID : 9253 I/O : uncollated Case : /home/joo/OpenFOAM/joo-13/run nProcs : 1 sigFpe : Enabling floating point exception trapping (FOAM_SIGFPE). fileModificationChecking : Monitoring run-time modified files using timeStampMaster (fileModificationSkew 10) allowSystemOperations : Allowing user-supplied system call operations // * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * // Create time --> FOAM FATAL ERROR: cannot find file "/home/joo/OpenFOAM/joo-13/run/system/controlDict" From function virtual Foam::autoPtr<Foam::ISstream> Foam::fileOperations::uncollatedFileOperation::readStream(Foam::regIOobject&, const Foam::fileName&, const Foam::word&, bool) const in file global/fileOperations/uncollatedFileOperation/uncollatedFileOperation.C at line 538. FOAM exiting
```

또 튜토리얼을 실행시키는데 실패했다.

로그 상태를 보니

cp: cannot stat ... pitzDailySteady: No such file or directory
→ 내가 지정한 튜토리얼 경로가 실제로 존재하지 않아서 복사가 안 됨.

cd: pitzDailySteady: No such file or directory
→ 당연히 복사가 안 됐으니 폴더도 없음.

그 다음 blockMesh가 실행된 위치가
Case : /home/joo/OpenFOAM/joo-13/run
→ 즉, 케이스 폴더가 아니라 run 폴더에서 바로 실행됨.

그래서 system/controlDict가 없다고 뜬 것이다.
cannot find file ".../run/system/controlDict"
→ OpenFOAM 케이스는 system/controlDict가 있어야 돌아가는데, run 폴더엔 그게 없음. (튜토리얼 케이스 폴더 안에 있어야 함)

즉, 튜토리얼이 실제로 어디 있는지 찾고, 그것을 케이스 폴더 안에서 돌려야 돌아갈 것이다.

1) 튜토리얼 위치 확인
```
source /opt/openfoam13/etc/bashrc
echo $FOAM_TUTORIALS
ls -1 $FOAM_TUTORIALS | head
```

2) pitzDailySteady의 경로 검색
```
find "$FOAM_TUTORIALS" -maxdepth 5 -type d -name "pitzDailySteady"
```
경로를 검색 시켰더니

```
joo@JOO-DESKTOP:~/OpenFOAM/joo-13/run$ source /opt/openfoam13/etc/bashrc echo $FOAM_TUTORIALS ls -1 $FOAM_TUTORIALS | head /opt/openfoam13/tutorials Allclean Allrun Alltest XiFluid compressibleMultiphaseVoF compressibleVoF film fluid incompressibleDenseParticleFluid incompressibleDriftFlux joo@JOO-DESKTOP:~/OpenFOAM/joo-13/run$ find "$FOAM_TUTORIALS" -maxdepth 5 -type d -name "pitzDailySteady" /opt/openfoam13/tutorials/incompressibleFluid/pitzDailySteady
```
출력된 결과를 확인하면 내 경로가 $FOAM_TUTORIALS/incompressibleFluid/pitzDailySteady 인 것을 알 수 있다.

출력된 경로를 복사해서 케이스 폴더로 들어가준다.
```
source /opt/openfoam13/etc/bashrc

cd $FOAM_RUN
cp -r "$FOAM_TUTORIALS/incompressibleFluid/pitzDailySteady" .
cd pitzDailySteady
```

튜토리얼 폴더에는 보통 Allrun 스크립트가 있는데 이걸 쓴다면 블록메시, 솔버, 후처리 준비까지 정해진 순서로 돌아가서 오류가 날 확률이 적다. 따라서 입력해 주었다.

```
./Allrun
```

마지막으로 cd pitzDailySteady 한 상태에서 pwd ls 명령어를 쳐서 system/ constant/0/ 같은 폴더가 생긴것을 확인해주면 된다.

```
pwd
ls
```

이제 출력된 내 커멘드를 보여주면,

```
joo@JOO-DESKTOP:~/OpenFOAM/joo-13/run$ source /opt/openfoam13/etc/bashrc cd $FOAM_RUN cp -r "$FOAM_TUTORIALS/incompressibleFluid/pitzDailySteady" . cd pitzDailySteady joo@JOO-DESKTOP:~/OpenFOAM/joo-13/run/pitzDailySteady$ ./Allrun Running blockMesh on /home/joo/OpenFOAM/joo-13/run/pitzDailySteady Running foamRun on /home/joo/OpenFOAM/joo-13/run/pitzDailySteady joo@JOO-DESKTOP:~/OpenFOAM/joo-13/run/pitzDailySteady$ cd pitzDailySteady -bash: cd: pitzDailySteady: No such file or directory joo@JOO-DESKTOP:~/OpenFOAM/joo-13/run/pitzDailySteady$ pwd ls /home/joo/OpenFOAM/joo-13/run/pitzDailySteady 0 100 200 285 Allrun constant log.blockMesh log.foamRun postProcessing system joo@JOO-DESKTOP:~/OpenFOAM/joo-13/run/pitzDailySteady$
```

- 0/ constant/ system/ -> 케이스 폴더의 필수 3종 확인.
- log.blockMesh, log.foamRun -> Allrun이 남긴 실행로그 확인.
- 100 200 285 -> time 디렉토리 생성 확인
- postProcessing/ -> 후처리 결과 폴더 생성 확인

정상적으로 마친 것을 확인하였다.

이제 케이스 폴더 설정을 마쳤으니 아까 오류났던 국룰 코드를 다시 확인해보기 위
```
tail -n 30 log.foamRun
```
를 출력해보고, 이후에 

```
checkMesh | tee log.checkMesh
```
메쉬 품질도 한 번 체크를 해보았다.


```
joo@JOO-DESKTOP:~/OpenFOAM/joo-13/run/pitzDailySteady$ tail -n 30 log.foamRun Time = 284s smoothSolver: Solving for Ux, Initial residual = 0.000127676, Final residual = 1.26058e-05, No Iterations 5 smoothSolver: Solving for Uy, Initial residual = 0.0010319, Final residual = 6.981e-05, No Iterations 6 GAMG: Solving for p, Initial residual = 0.00131556, Final residual = 0.00012283, No Iterations 4 time step continuity errors : sum local = 0.00520864, global = 0.000614958, cumulative = 0.557396 smoothSolver: Solving for epsilon, Initial residual = 0.00014159, Final residual = 8.97856e-06, No Iterations 3 smoothSolver: Solving for k, Initial residual = 0.000227002, Final residual = 1.38911e-05, No Iterations 4 ExecutionTime = 4.63522 s ClockTime = 5 s Time = 285s smoothSolver: Solving for Ux, Initial residual = 0.000122524, Final residual = 1.21333e-05, No Iterations 5 smoothSolver: Solving for Uy, Initial residual = 0.000992942, Final residual = 6.75593e-05, No Iterations 6 GAMG: Solving for p, Initial residual = 0.00123295, Final residual = 0.000112114, No Iterations 4 time step continuity errors : sum local = 0.0047489, global = 0.000564888, cumulative = 0.55796 smoothSolver: Solving for epsilon, Initial residual = 0.000139397, Final residual = 8.97324e-06, No Iterations 3 smoothSolver: Solving for k, Initial residual = 0.000221974, Final residual = 1.35975e-05, No Iterations 4 ExecutionTime = 4.64914 s ClockTime = 5 s SIMPLE solution converged in 285 iterations streamlines streamlines write: Seeded 10 particles Sampled 12040 locations End joo@JOO-DESKTOP:~/OpenFOAM/joo-13/run/pitzDailySteady$ checkMesh | tee log.checkMesh /*---------------------------------------------------------------------------*\ ========= | \\ / F ield | OpenFOAM: The Open Source CFD Toolbox \\ / O peration | Website: https://openfoam.org \\ / A nd | Version: 13 \\/ M anipulation | \*---------------------------------------------------------------------------*/ Build : 13-cde978a97c93 Exec : checkMesh Date : Jan 27 2026 Time : 21:57:45 Host : "JOO-DESKTOP" PID : 10415 I/O : uncollated Case : /home/joo/OpenFOAM/joo-13/run/pitzDailySteady nProcs : 1 sigFpe : Enabling floating point exception trapping (FOAM_SIGFPE). fileModificationChecking : Monitoring run-time modified files using timeStampMaster (fileModificationSkew 10) allowSystemOperations : Allowing user-supplied system call operations // * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * // Create time Create mesh for time = 0 Time = 0s Mesh stats points: 25012 internal points: 0 faces: 49180 internal faces: 24170 cells: 12225 faces per cell: 6 boundary patches: 5 point zones: 0 face zones: 0 cell zones: 0 Overall number of cells of each type: hexahedra: 12225 prisms: 0 wedges: 0 pyramids: 0 tet wedges: 0 tetrahedra: 0 polyhedra: 0 Checking topology... Boundary definition OK. Cell to face addressing OK. Point usage OK. Upper triangular ordering OK. Face vertices OK. Number of regions: 1 (OK). Checking patch topology for multiply connected surfaces... Patch Faces Points Surface topology inlet 30 62 ok (non-closed singly connected) outlet 57 116 ok (non-closed singly connected) upperWall 223 448 ok (non-closed singly connected) lowerWall 250 502 ok (non-closed singly connected) frontAndBack 24450 25012 ok (non-closed singly connected) Checking geometry... Overall domain bounding box (-0.0206 -0.0254 -0.0005) (0.29 0.0254 0.0005) Mesh has 2 geometric (non-empty/wedge) directions (1 1 0) Mesh has 2 solution (non-empty) directions (1 1 0) All edges aligned with or perpendicular to non-empty directions. Max cell openness = 2.21044e-16 OK. Max aspect ratio = 8.1407 OK. Minimum face area = 1.68589e-07. Maximum face area = 5.14451e-06. Face area magnitudes OK. Min volume = 1.6902e-10. Max volume = 3.83579e-09. Total volume = 1.4516e-05. Cell volumes OK. Mesh non-orthogonality Max: 5.95045 average: 1.63034 Non-orthogonality check OK. Face pyramids OK. Max skewness = 0.260575 OK. Coupled point location match (average 0) OK. Mesh OK. End
```
SIMPLE solution converged in 285 iterations + End → 해석이 정상적으로 수렴하고 종료됨. (pitzDailySteady 예제는 이런 형태로 수렴 메시지가 찍히는 게 정상 흐름이다.)

checkMesh에서 Mesh OK. → 메쉬 품질 검사 통과.

즉! 드디어 OpenFOAM 설치와 환경 구축 그리고 튜토리얼 실행까지 모두 완료가 되었다.

이제 기다리고 기다리던 후처리 작업이다.
사실 이 후처리 작업이 제일 다가오기 쉽고 예쁘니 기대를 안할 수가 없었다.

케이스 폴더에서 paraFoam을 출력하면 Linux GUI 앱(WSLg) 실행이 되며 ParaView가 뜰 것이다.

```
joo@JOO-DESKTOP:~/OpenFOAM/joo-13/run/pitzDailySteady$ paraFoam Created temporary 'pitzDailySteady.OpenFOAM' I/O : uncollated
```
이렇게 실행했더니 ParaView 창이 떠서 후처리를 다음과 같이 확인할 수 있었다.

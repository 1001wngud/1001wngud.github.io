# pitzDailySteady BFS(Backward-Facing Step) 2D - 2

## 재부착 길이 추출, 파이프라인 구축, 난류모델 민감도 측정

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

gnuplot 안에서 실행시켜 figure를 얻을 수 있었다.

```bash
set style data linespoints
plot \
"postProcessing/graphUniform_xH1/285/line.xy"  u 2:1 title "x/H=1", \
"postProcessing/graphUniform_xH4/285/line.xy"  u 2:1 title "x/H=4", \
"postProcessing/graphUniform_xH6/285/line.xy"  u 2:1 title "x/H=6", \
"postProcessing/graphUniform_xH10/285/line.xy" u 2:1 title "x/H=10"
```

[이미지3] [이미지5]

Backward-Facing Step(BFS)에서는 스텝 직후에 유동이 분리(separation)되고, 아래쪽에 **재순환(역류) 영역**이 생겼다가, 아래로 갈수록 재부착(reattachment)하고 다시 회복한다.

- 어떤 y 구간에서 Ux가 음수(왼쪽으로 튐) → 그 구간은 역류/재순환이 있다는 뜻
- x/H가 커질수록(1→4→6→10) → 보통 역류 구간이 줄고 프로파일이 “정상적인” 모양으로 회복됨
- 위쪽(upper wall 쪽)과 아래쪽(lower wall 쪽) 프로파일이 다르게 휘는 건 벽 영향 + 분리/전단층 발달이 주된 원인이라 판단함
# pitzDailySteady BFS(Backward-Facing Step) 2D - 4(End)

## 재부착 길이 추출, 파이프라인 구축, 난류모델 민감도 측정

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

kOmegaSST는 RANS에서 쓰는 2-방정식 난류모델로, k(난류 운동에너지)와 ω(특정 소산율)를 풀어 난류점성(난류 혼합의 강도)을 계산한다. 또한 SST는 벽 근처에서는 k-ω 성격을, 자유류에서는 k-ε 성격을 섞는(블렌딩) 구조이다. kEpsilon과 kOmegaSST의 차이는 평균 방정식이 바뀐 것이 아니라, 평균 방정식에 들어간 난류 응력(추가 저항/혼합 효과)을 계산하기 위해 쓰는 가정과 계산식가 바뀐 것이다. 쉽게 말해 난류점성 νt를 만드는 방식이 달라졌다고 할 수 있다.

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

그리고 `0/omega` 값이 튜토리얼 권장 계열인지 확인했다. 나 같은 경우는 internalField≈440.15, inlet value가 $internalField로 내부값과 inlet 값이 일관되게 맞춰있었고 OpenFOAM v13 backwardStep 튜토리얼에서도 `0/omega`에 440.2를 사용한다.

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

[이미지11]

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

**왜 차이가 났을까?**

k‑ε에서 k‑ω SST로 난류모델을 교체한 뒤 재부착 길이 xr/H가 **6.760→7.85** 수준으로 증가한 것은, Backward‑Facing Step(BFS)처럼 **박리–재부착이 지배적인 유동에서 난류모델이 박리 전단층의 난류 혼합(eddy viscosity)을 예측하는 방식이 달라지기 때문**이다. 재부착 위치는 전단층이 얼마나 빠르게 성장하고 혼합되어 벽으로 다시 붙는지에 민감하고 표준 k‑ε 모델은 BFS류 문제에서 재부착 길이를 실험보다 짧게 예측하는 경향이 여러 검증/연구에서 반복적으로 보고되어 왔다고 한다. (NASA TMR 참고)

반면, k-ω SST (Shear Stress Transport) 모델에서는 두 개의 난류 모델 상수/항을 블렌딩 함수 F₁로 연결하여 사용하는 구조를 갖는다.
즉 SST는 서로 다른 난류 모델 상수 $\phi_1$과 $\phi_2$를 다음과 같이 가중 평균 형태로 결합한다:

$\phi = F_{1}\,\phi_{1} + (1 - F_{1})\,\phi_{2}$

여기서 F₁은 상황에 따라 0과 1 사이 값을 갖는 블렌딩 함수이다.
또한 SST에서 난류 점성 계수 μₜ는 다음과 같이 최댓값(max) 형태의 limiter를 포함한 식으로 계산된다:
$\mu_{t} = \frac{\rho\,a_{1}\,k}{\max\bigl(a_{1}\,\omega,\ \Omega\,F_{2}\bigr)}$

이 식에서

- ρ는 유체 밀도,
- a₁은 모델 상수,
- k는 난류 운동 에너지,
- ω는 난류 주파수,
- Ω는 절대 변형률(strain rate),
- F₂는 또 다른 블렌딩 함수

를 나타낸다.

이런 구조는 박리·불리한 압력구배에서 난류점성(혼합)의 크기와 분포를 k‑ε와 다르게 만들고, 그 결과 **재순환 버블 크기 및 재부착 위치가 달라질 수 있다.**

쉽게 다시 말하자면 SST는 단순히 k와 ω만 쓰는 게 아니라, 두 가지 기준 값 중에서 큰 쪽을 선택해서 난류 점성을 계산한다. 즉, $max(a_1\omega, \Omega F_2)$부분 때문에 일반 k-ω 모델과 k‑ε 모델이 서로 다른 난류 점성 분포가 나온다는 뜻이다.

또한 내 SST 케이스에서는 OpenFOAM 가이드에 나온 것처럼 **SST에서 step 근처 secondary vortex가 미세하게 진동할 수 있어** 이를 안정화하기 위해 `grad(U) cellLimited ...` 같은 수치적 안정화(기울기 제한)를 적용했다. 이 스킴 변경은 수치 확산/안정성을 바꿔 **재부착 위치에 “모델 차이 + 계산 방식 차이”가 함께 반영**되도록 만들 수 있다. k‑ε에서 k‑ω SST에서의 재부착 길이 xr/H 차이는 **SST의 박리/혼합 예측 특성**과 **SST에서 추가된 수치 안정화 설정**이 합쳐진 결과라고 볼 수 있다.
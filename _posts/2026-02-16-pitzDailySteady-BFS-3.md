# pitzDailySteady BFS(Backward-Facing Step) 2D - 3

## 재부착 길이 추출, 파이프라인 구축, 난류모델 민감도 측정

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
Reattachment estimate: x ~= 0.172121 m (between 0.17204 and 0.172403)
```

<details markdown="block">
<summary>로그 원문(클릭)</summary>

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

</details>

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
H = 0.0254 m
xr = 0.172121 m
xr/H = 6.7764173
Ux min=-0.0048397, max=0.00189621
```

<details markdown="block">
<summary>로그 원문(클릭)</summary>

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

</details>

![image06]({{ '/assets/posts/2026-02-16-pitzDailySteady-BFS-3/image06.png' | relative_url }})

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

- `(0 -0.0249 0)` → `(0.29 -0.0249 0)`  → y0.5mm
- `(0 -0.0244 0)` → `(0.29 -0.0244 0)`  → y1.0mm

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
==== graphUniform_reattach ====
xr = 0.172121 m (between 0.17204 and 0.172403)
==== graphUniform_reattach_y05mm ====
xr = 0.170107 m (between 0.169862 and 0.170225)
==== graphUniform_reattach_y10mm ====
xr = 0.168657 m (between 0.168411 and 0.168773)
```

<details markdown="block">
<summary>로그 원문(클릭)</summary>

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

</details>

![image07]({{ '/assets/posts/2026-02-16-pitzDailySteady-BFS-3/image07.png' | relative_url }})

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

<div class="img-grid-2">
 <img src="{{ '/assets/posts/2026-02-16-pitzDailySteady-BFS-3/image08.png' | relative_url }}" alt="image08">
 <img src="{{ '/assets/posts/2026-02-16-pitzDailySteady-BFS-3/image09.png' | relative_url }}" alt="image09">
</div>

그 이후에도 y가 0.2 mm, 0.5 mm, 1.0 mm, 2.0 mm 오프셋을 줬을 때의 재부착 길이를 각각 보간법으로 구해 민감도를 확인할 수 있었다.

**의문**

**왜 y를 위로 올릴수록 xr이 작아질까 의문이 생겼다.**

내가 내린 결론은 이 방법이 일단 정의가 아니고, 벽 그 자체에서의 재부착이 아니라 **벽에서 y만큼 떨어진 선에서 유동의 방향이 바뀌는 위치, 즉 근사를 해서 그렇다고 판단되었다.**

재순환 버블은 보통 벽 근처가 더 오래(더 downstream까지) 역류가 남고, 위로 갈수록 역류 두께가 줄어들기 때문에 y를 위로 올리면 순방향으로 바뀌는 xr이 조금씩 감소한다.

## 벽 전단응력으로  재부착점 추출

이번에는 재부착을 정의 기반으로 잡아서 추출 해볼 것이다.
재부착점 : lowerWall에서 벽 전단응력 또는 skin-friction(Cf)이 0으로 바뀌는 지점 
graphUniform_tauLowerWall의 입력값으로는 `wallShearStress` 필드가 필요하다.
따라서 `wallShearStress` 를 먼저 계산하고 같은 foamPostProcess 실행 안에서 이어서 graphUniform을 돌릴 것이다. 
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

![image10]({{ '/assets/posts/2026-02-16-pitzDailySteady-BFS-3/image10.png' | relative_url }})

- 내 케이스에서는 `line.xy`에서 **x/H>1 이후 sign-change가 딱 1번 나타나고**,
    
    그 값인 **xr≈0.17169281** 에서 **xr/H=6.760** 나왔다.
    
- **xr 이후 구간에서 tau_x의 부호가 바뀌지 않고** 한쪽 부호로 유지된 것으로 보아 재부착 점이라고 판단할 수 있었다. `AFTER xr: tau_x min=-0.0305978, max=-0.000395067`

### **결론**

본 연구에서는 k-ε 난류모델을 기준으로 Backward-Facing Step(2D BFS) 유동의 재부착 위치 $x_r/H$를
두 가지 방법으로 산출하였다.

첫째, **유속 $U_x$ 기준**으로는 near-wall에서 유속이 0이 되는 지점을 재부착점으로 정의하였다.
각 y 위치별 결과는 다음과 같다:

- $y = 0\,\text{mm}$ : $x_r/H \approx 6.776$
- $y = 0.5\,\text{mm}$ :  $x_r/H \approx 6.760$
- $y = 1.0\,\text{mm}$ : $x_r/H \approx 6.697$

이를 종합하면 유속 기준에서 재부착 길이 $x_r/H$는 **약 6.64~6.78 범위**로 나타났다.

둘째, **전단응력 $\tau_w$ 기준**에서는 벽면 전단응력이 0이 되는 지점을 재부착점으로 정의하였고,
그 결과는 $x_r/H = 6.760$ 이다.

두 방법 모두 동일한 k-ε 난류모델 조건 하에서 수행되었으며, 재부착 위치 값은 일관되게 나타났다.
유속 기준에서 위치별 약간의 편차는 있었으나, 전단응력 기준 값 $x_r/H = 6.760$은
유속 기준의 중앙값과 매우 유사하게 일치하였다.

따라서 나는 유속 기준과 전단응력 기준 모두 유사한 재부착 위치를 예측했고, 그 중 전단응력 기준으로 산출한 $x_r/H = 6.760$을 대표 재부착 길이로 채택했다.
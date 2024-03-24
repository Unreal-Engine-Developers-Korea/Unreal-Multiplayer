# CPU 연산 최소화
## 기본 코드 팁
### Tick 주의

- [ ] Blueprint Tick는 C++ `Tick()`보다 느리기 때문에 가능하면 C++로 구현하자
    1. `UFUNCTION(BlueprintImplementableEvent)`로 C++ 헤더파일 선언 -> Blueprint에서 구현 -> C++ Tick에서 호출
    2. 또는 C++로만 구현

![image](https://gist.github.com/assets/57009810/5360524b-cea7-4abf-b22e-a3b84e51dc65)
(Fortnite 프로파일링)

- [ ] `Tick`이 필요없는 Actor/ActorComponent들은 `SetActorTickEnabled(false)`/`SetComponentTickEnabled(false)`로 비활성화하자; 필요할때 다시 활성화
    - `Tick`을 너무 많이 쓰면 게임 스레드(Game Thread)에서 오버로드 현상이 발생할 수 있음 -> 프레임 드랍(Frame Drop) 발생

![image](https://gist.github.com/assets/57009810/9c553c02-d5cf-4365-be2a-9b7c81203dfa)


- [ ] UI 구현은 `Tick` Event Dispatcher 또는 Delegate로 처리하자
- [ ] `SetActorTickInterval()`로 Tick 주기를 줄일 수 있다
    - (인자값1이 높을수록 Tick이 느려짐)

![image](https://gist.github.com/assets/57009810/6c361cf1-66be-4454-8eff-0716f05fe9a4)


### 이동하는 Scene Component

- **Profiling**할 때 `stat component` 

- [ ] `Moveable` Scene Component은 가능하면 Overlap Event를 비활성화
    - Overlap Event가 활성화된 모든 Component들은 움직일때마다 Physics Check를 함 -> 프레임드랍 -> 따라서 Overlap Event는 Trigger vs 플레이어 할때 쓰고 나머지는 `Trace`로 대체하자 -> Projectile (총알), 무기에는 *절대로* Overlap Event하면 안됨

![image](https://gist.github.com/assets/57009810/4f12dba2-0a30-47c3-bfd5-d4391dae8b00)

![image](https://gist.github.com/assets/57009810/480b3f05-1c3a-420c-bc73-bfd4fd7c3f05)

- [ ] 언리얼은 모든 움직이는 컴포넌트의 Transform을 연산하기 때문에 소멸식 컴포넌트는 Auto Manage를 켜주자 -> `bAutoManageAttachment = true` (UAudioComponent) 또는 `SetUseAutoManageAttachment(true)` (UNiagaraComponent)
    - 사용안할때는 자동으로 Detach하고 사용할때 알아서 Attach함 -> 따라서 사용안할 때는 해당 컴포넌트의 Transform 연산을 스킵함
    - 사소한거지만 어떻게든 Game Thread를 쥐어짜내자ㅎㅎ

![image](https://gist.github.com/assets/57009810/15ba9164-498c-494f-96bf-e2dd728878bf)

### Cast 주의
- [ ] Cast은 필요할때만 사용하자 -> Casting 할 때마다 Class 전체를 본 Class에 실린다고 생각하면 됨 -> 프레임 드랍
    - 가능하면 BeginPlay에서 Cast해주고 전역변수 선언해서 Caching하자

#### Tick에서 프레임마다 Cast하면 이런 느낌

![image](https://gist.github.com/assets/57009810/3c49ac44-0f85-4ec9-a60a-e7fea1e3688d)

#### 가능하면 이걸로 하자
  1. Delegate 또는 Event Dispatcher 
  2. Interface 
  3. Game Tag
 
### Spawn Actor
- [ ] `SpawnActor()`함수는 런타임중에는 최대한 피하자
    -  필요한 Actor들은 `BeginPlay()`에서 Spawn하고 눈에 안보이게 숨기고 (`SetVisibility(false)`) Level밖에서 대기 -> 필요할 때 원하는 장소로 이동

![HowDoesLadyDimitrescuWorkOffCameraResidentEvil8Village-YouTubeand2morepages-Profile1-MicrosoftEdge2024-03-2015-13-29-Trim-ezgif com-video-to-gif-converter](https://gist.github.com/assets/57009810/9a0a542d-b87e-48a5-b4b5-b5e458b8237c)
(https://www.youtube.com/watch?v=2exHZm_6Ig8)

- [ ] Object Pooling을 자주 사용하자

#### Object Pooling 안쓸때 vs 쓸때

![NonVsPooling](https://gist.github.com/assets/57009810/95ff6767-2742-4ad5-b11c-3785fc2c48bf)
(https://www.kodeco.com/847-object-pooling-in-unity)

#### Object Pooling 예시
![ScreenRec6Gif](https://gist.github.com/assets/57009810/331504a5-006e-458d-8640-8d83a39bf476)
(https://www.kodeco.com/847-object-pooling-in-unity)

## Physics
### Trace 주의
- [ ] Line Trace < Sphere Trace < Capsule Trace < Box Trace
- [ ] Single < Multi
### 충돌처리 (Collision)
- [ ] 충돌처리가 필요없는 Actor들은 Collision 끄기 
    - Collision Culling: 거리가 먼 Actor들은 Collision이 필요없음
- [ ] Collision Shape을 최대한 간단한걸로 선택 (Sphere < Capsule < Box) 
- [ ] 가능하면 Convex Collision을 피하고 Blender로 커스텀 Collision을 생성하자
- [ ] Collision LOD도 가능
### Mobility
- [ ] Physics 상호작용이 없는 Actor들은 SceneComponent를 `SetMobility(EComponentMobility::Static)`로 해준다 -> 상호작용이 있는 Actor는 `SetMoblitiy(EComponentMobility::Stationary)` 또는 `SetMoblitiy(EComponentMobility::Movable)`로 필요할 때 바꿔준다
### Transform 연산 줄이기
- [ ] Scene Component 줄이기
    - Scene Component는 Transform(위치, 회전, 크기)을 프레임마다 연산하기 때문에 Level에 Scene Component가 너무 많으면 안됨
- [ ] Transform을 한번에 연산하기
    - [ ] `SetActorLocation()`, `SetActorRotation()`, `SetActorScale3D()`또는 `SetRelativeLocation()`, ... 을 다 따로 적용하지말고 -> 다 계산하고 한번에 `SetActorTransform` 또는 `SetActorLocationAndRotation()`을 사용하자

![image](https://gist.github.com/assets/57009810/b3bc3f57-d68b-4279-b1c5-82d337f59ce2)

### CPU Offloading
- [ ] 상호작용이 필요없는 Actor가 이동하거나 회전을 해야한다면 (예. 천장 선풍기) 코드로 구현하지 말고 쉐이더 코드(Shader Code)로 구현하자
    - 연산 작업을 CPU에서 GPU 덜어줄 수 있다
- [ ] Vertex Animation

![DaysGone_HowSonyBendBuilttheHorde-YouTubeand2morepages-Profile1-MicrosoftEdge2024-03-2017-40-05-Trim-ezgif com-video-to-gif-converter](https://gist.github.com/assets/57009810/f1bee350-6ec3-4dcb-b079-aa25dfe1a62c)

(https://www.youtube.com/watch?v=NIsRvgwNX4Y)

- [ ] 가능하면 자녀 Component들은 `bUseAttachParentBound = True`로 설정하자
    - 렌더링할때는 bound로 활성화/비활성화하기 때문에 각각 컴포넌트의 bound가 있기보다 부모의 bound를 사용하는 것이 더 효율적이다. `bUseAttachParentBound = true` -> Culling할 때 더 간단해짐
    - 만약에 bound가 카메라 frustum안에 있으면 렌더링을 해줌

![1](https://gist.github.com/assets/57009810/8ac969df-78f8-461a-897c-5302709a5518)

![image](https://gist.github.com/assets/57009810/0fda0c86-f5ca-4dfa-868d-021a08928158)

![2](https://gist.github.com/assets/57009810/6f77a870-e47a-42a5-9132-7954090023cb)

![image](https://gist.github.com/assets/57009810/1d0b0d32-4428-4419-bc09-5a5611a9be4b)

# 렌더링
### Draw Call 최소화
- [ ] 레벨에 많이 쓰여지는 *Static Mesh*가 있다면 최대한 많이 인스턴싱(Instancing)하자 -> 한 Actor에 여러게의 *Instance*를 생성가능

#### Actor 6개
![image](https://gist.github.com/assets/57009810/c3fd8173-636e-43c3-ae7d-d96e0db4ae1e)

#### 1 Actor, Instance 6개
![image](https://gist.github.com/assets/57009810/5e1c7e6e-615f-489c-a8da-9a80df825855)


    - *Instanced Static Mesh Component*가 가장 쉬운 방법임 
        - LOD까지 적용하고 싶으면 *Hierarchical Instanced Static Mesh Component* 사용가능
- [ ] LOD (Level of Detail) 적용하기
![LODZoom](https://gist.github.com/assets/57009810/5e6ebc08-ac86-41ee-b935-60ca7275df2d)

- [ ] Culling 적용하기 (언리얼에서 이미 기본 Culling 작동함)
- [ ] Static Mesh Merge하기
### Material
- [ ] 투명한 Material은 되도록이면 피하자 -> 대신 Masked Material 쓸 수 있음
    - Masked Material을 "Dither" 노드에 연결하면 비교적으로 연산비용의 일부만 사용함
- [ ] Material Instancing하기
- [ ] Material Batching하기
### 조명
- [ ] Max Draw Distance (먼거리에서는 Light를 자동으로 꺼주는 기능)
![t-ezgif com-video-to-gif-converter](https://gist.github.com/assets/57009810/b6769b7f-0cff-4b79-b1c6-c752ed8f5cbc)
![image](https://gist.github.com/assets/57009810/a76e2650-5bfc-4ebb-a72f-e8d7f2eec5df)

- [ ] Max Distance Fade Range - 갑자기 빛이 활성화/비활성화하면 부자연스러우면 Fade도 해주면 된다
![t-ezgif com-video-to-gif-converter (1)](https://gist.github.com/assets/57009810/7de870ed-1f21-45f1-9ff3-b84e7caddba3)



![image](https://gist.github.com/assets/57009810/bf4f35eb-acc5-4958-8505-8a2faaa73008)

  
  
# 애니메이션
- [ ] 
  
# VR
- [ ] Round Robin Occlusion (Alternates occlusion between one eye each frame instead of both at the cost of 1 frame of latency)
- [ ] Instanced Stereo
- [ ] Forward Rendering (Lumen & Nanite x) - renders faster, good for baked static lighting
- [ ] Deferred Rendering (better for movable lighting; automatically on if Forward Rendering is off)
- [ ] Vertex Fogging for Opaque
- [ ] MSAA - 4x MSAA for MSAA Sample count
### Post Processing (heavy tho)
- [ ] SSAO low or off
- [ ] Bloom low or off -> Convolution bloom looks good but very expensive
- [ ] SSR off in PPV (Looks wrong in Deferred in VR, unsupported in forward rendering, planar reflection looks good but very expensive)
    - [ ] Support global clip plane for planar reflections - best, expensive
- [ ] Lens Flare off
- [ ] Motion Blur off
- [ ] Screen Effects (Light rays, lens dirt, film grain) off
- [ ] Target Hardware - Optimize project settings for Mobile, scalable
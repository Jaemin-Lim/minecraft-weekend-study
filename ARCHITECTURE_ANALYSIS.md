# Minecraft Weekend Study - 코드 아키텍처 분석

이 문서는 `minecraft-weekend-study` 프로젝트의 전체 코드 구조, 동작 방식, 블록 시스템, 월드 생성 알고리즘, 그리고 사용된 디자인 패턴에 대한 상세한 분석을 제공합니다.

## 목차
1. [전체 파일과 코드의 구조](#1-전체-파일과-코드의-구조)
2. [Main에서 시작하는 동작 방식](#2-main에서-시작하는-동작-방식)
3. [블록 구조체의 구성과 렌더링 방식](#3-블록-구조체의-구성과-렌더링-방식)
4. [월드 생성 알고리즘](#4-월드-생성-알고리즘)
5. [사용된 게임 디자인 패턴](#5-사용된-게임-디자인-패턴)

---

## 1. 전체 파일과 코드의 구조

### 1.1 프로젝트 개요

이 프로젝트는 C 언어로 작성된 Minecraft 클론으로, 48시간 내에 기본 구조가 만들어졌으며 이후 지속적으로 개선되었습니다.

**주요 기능:**
- 무한한 절차적으로 생성된 월드
- 무한한 높이/깊이
- 주야 사이클
- 바이옴 시스템
- ECS 기반 플레이어 및 엔티티 시스템
- 완전한 RGB 조명 시스템
- 투명도 및 반투명 지원
- 애니메이션 블록 (물, 용암)
- 거리 안개
- 다양한 블록 타입

### 1.2 디렉토리 구조

```
src/
├── main.c              # 진입점 및 게임 루프
├── state.h             # 전역 상태 구조체
├── block/              # 블록 정의 및 동작
│   ├── block.h/c       # 블록 시스템 핵심
│   ├── air.c, grass.c, stone.c, water.c... # 개별 블록 구현
│   └── ...
├── entity/             # 엔티티 컴포넌트 시스템 (ECS)
│   ├── ecs.h/c         # ECS 핵심 시스템
│   ├── player.h/c      # 플레이어 로직
│   ├── c_*.h/c         # 개별 컴포넌트 (camera, control, movement, etc.)
│   └── ...
├── gfx/                # 그래픽 렌더링
│   ├── renderer.h/c    # 렌더러 시스템
│   ├── shader.h/c      # 셰이더 관리
│   ├── window.h/c      # 윈도우 관리 (GLFW)
│   ├── vao.h/c, vbo.h/c # OpenGL 버퍼 객체
│   ├── texture.h/c     # 텍스처 관리
│   └── blockatlas.h/c  # 블록 텍스처 아틀라스
├── world/              # 월드 시스템
│   ├── world.h/c       # 월드 관리
│   ├── chunk.h/c       # 청크 시스템
│   ├── chunkmesh.h/c   # 청크 메쉬 생성
│   ├── blockmesh.h/c   # 블록 메쉬 유틸리티
│   ├── light.h/c       # 조명 시스템
│   ├── sky.h/c         # 하늘 렌더링
│   └── gen/            # 월드 생성
│       ├── worldgen.h/c    # 주요 생성 로직
│       ├── noise.h/c       # 노이즈 함수
│       ├── treegen.c       # 나무 생성
│       ├── grassgen.c      # 풀 생성
│       └── ...
├── ui/                 # 사용자 인터페이스
│   ├── ui.h/c          # UI 시스템
│   ├── hotbar.h/c      # 핫바
│   └── crosshair.h/c   # 십자선
└── util/               # 유틸리티 함수
    └── util.h          # 공통 유틸리티 및 타입

lib/
├── glad/               # OpenGL 로더
├── glfw/               # 윈도우 관리 라이브러리
├── cglm/               # 수학 라이브러리 (벡터, 행렬)
├── stb/                # STB 이미지 로딩
└── noise/              # 노이즈 생성 라이브러리

res/
├── shaders/            # GLSL 셰이더
└── textures/           # 텍스처 파일
```

### 1.3 핵심 데이터 구조

#### State 구조체 (`state.h`)
전역 게임 상태를 관리하는 최상위 구조체:

```c
struct State {
    struct Window *window;      // 윈도우 및 입력 관리
    struct Renderer renderer;   // 렌더링 시스템
    struct World world;         // 월드 데이터
    struct UI ui;               // UI 시스템
    size_t ticks;              // 게임 틱 카운터
};
```

#### World 구조체 (`world.h`)
월드의 모든 데이터를 관리:

```c
struct World {
    struct ECS ecs;                    // 엔티티 컴포넌트 시스템
    struct Entity entity_view;         // 카메라 엔티티
    struct Entity entity_load;         // 청크 로드 중심 엔티티
    struct Sky sky;                    // 하늘 상태
    u64 ticks;                         // 월드 틱
    u64 seed;                          // 랜덤 시드
    size_t chunks_size;                // 청크 배열 크기
    struct Chunk **chunks;             // 청크 배열 (3D)
    ivec3s chunks_origin;              // 청크 원점
    ivec3s center_offset;              // 중심 오프셋
    struct Heightmap **heightmaps;     // 높이맵 배열
    struct {...} unloaded_blocks;      // 아직 로드되지 않은 블록
    struct {...} throttles;            // 성능 제한
};
```

#### Chunk 구조체 (`chunk.h`)
32x32x32 블록 단위:

```c
struct Chunk {
    struct World *world;        // 부모 월드
    ivec3s offset, position;    // 청크 위치
    u64 *data;                  // 블록 데이터 (비트필드)
    size_t count;               // 블록 개수
    struct {
        bool empty: 1;          // 빈 청크
        bool generating: 1;     // 생성 중
    } flags;
    struct ChunkMesh *mesh;     // 렌더링 메쉬
};
```

각 `u64` 데이터는 다음 비트필드를 포함:
- 28비트: 메타데이터
- 4비트: 태양광 강도
- 16비트: 횃불 조명 (R, G, B, 강도 각 4비트)
- 16비트: 블록 ID

#### Block 구조체 (`block.h`)
블록 타입의 속성과 동작 정의:

```c
struct Block {
    enum BlockId id;
    bool transparent;           // 투명 여부
    bool liquid;               // 액체 여부
    bool can_emit_light;       // 발광 여부
    bool animated;             // 애니메이션 여부
    enum BlockMeshType mesh_type;
    bool solid;                // 고체 여부 (충돌)
    f32 gravity_modifier;      // 중력 수정자
    f32 drag;                  // 항력
    f32 sliperiness;           // 미끄러움
    
    // 함수 포인터 (다형성)
    ivec2s (*get_texture_location)(...);
    void (*get_mesh_information)(...);
    void (*get_animation_frames)(...);
    Torchlight (*get_torchlight)(...);
    void (*get_aabb)(...);
};
```

---

## 2. Main에서 시작하는 동작 방식

### 2.1 프로그램 시작 흐름

```c
int main(int argc, char *argv[]) {
    window_create(init, destroy, tick, update, render);
    window_loop();
}
```

`main` 함수는 매우 간결하며, 5개의 콜백 함수를 등록하고 이벤트 루프를 시작합니다.

### 2.2 초기화 단계 (`init()`)

1. **블록 시스템 초기화**
   ```c
   block_init();  // 모든 블록 타입 등록
   ```
   - 각 블록 타입의 `_init()` 함수 호출
   - `BLOCKS` 배열에 블록 속성 저장

2. **시스템 초기화**
   ```c
   state.window = &window;
   renderer_init(&state.renderer);  // OpenGL 컨텍스트, 셰이더 등
   world_init(&state.world);        // 월드 데이터 구조
   ui_init(&state.ui);              // UI 요소
   ```

3. **플레이어 엔티티 생성 (ECS)**
   ```c
   struct Entity player = ecs_new(&state.world.ecs);
   ecs_add(player, C_POSITION);     // 위치 컴포넌트
   ecs_add(player, C_PHYSICS, ...); // 물리 (충돌, 중력)
   ecs_add(player, C_MOVEMENT, ...);// 이동
   ecs_add(player, C_CAMERA, ...);  // 카메라
   ecs_add(player, C_CONTROL);      // 입력 제어
   ecs_add(player, C_BLOCKLOOK,...);// 블록 하이라이트
   ecs_add(player, C_LIGHT);        // 조명 발산
   ```

4. **초기 플레이어 설정**
   ```c
   c_position->position = (vec3s) {{ 0, 80, 0 }};
   state.world.entity_load = player;
   state.world.entity_view = player;
   ```

### 2.3 게임 루프

GLFW 이벤트 루프 내에서 세 가지 주요 단계가 반복됩니다:

#### 2.3.1 Tick 단계 (고정 시간 간격)

```c
void tick() {
    state.ticks++;
    world_tick(&state.world);  // 월드 로직 업데이트
    ui_tick(&state.ui);        // UI 로직
    
    // 디버그 기능
    if (state.window->keyboard.keys[GLFW_KEY_L].down) {
        state.world.ticks += 30;  // 시간 빠르게
    }
}
```

`world_tick()`의 주요 작업:
- ECS 틱 이벤트 (`ecs_event(ecs, ECS_TICK)`)
- 청크 생성 및 로드
- 조명 업데이트
- 물리 시뮬레이션

#### 2.3.2 Update 단계 (프레임마다)

```c
void update() {
    renderer_update(&state.renderer);
    world_update(&state.world);  // 월드 상태 업데이트
    ui_update(&state.ui);
    
    // 입력 처리
    if (state.window->keyboard.keys[GLFW_KEY_T].pressed) {
        state.renderer.flags.wireframe = !state.renderer.flags.wireframe;
    }
}
```

`world_update()`의 주요 작업:
- 플레이어 위치에 따라 청크 로드/언로드
- 청크 메쉬 생성 (지연 처리)
- ECS 업데이트 이벤트
- 하늘 색상 업데이트 (주야 사이클)

#### 2.3.3 Render 단계 (프레임마다)

```c
void render() {
    // 3D 렌더링
    renderer_prepare(&state.renderer, PASS_3D);
    world_render(&state.world);
    
    // 2D UI 렌더링
    renderer_prepare(&state.renderer, PASS_2D);
    renderer_push_camera(&state.renderer);
    {
        renderer_set_camera(&state.renderer, CAMERA_ORTHO);
        ui_render(&state.ui);
    }
    renderer_pop_camera(&state.renderer);
}
```

`world_render()`의 주요 작업:
- 가시 범위 내 청크 렌더링
- 불투명 블록 먼저, 투명 블록 나중에 (깊이 정렬)
- 하늘 박스 렌더링

### 2.4 종료 단계 (`destroy()`)

```c
void destroy() {
    renderer_destroy(&state.renderer);
    world_destroy(&state.world);
    ui_destroy(&state.ui);
}
```

모든 리소스 정리 및 메모리 해제.

### 2.5 전체 실행 흐름 다이어그램

```
main()
 │
 ├─> window_create()
 │    ├─> init()
 │    │    ├─> block_init()
 │    │    ├─> renderer_init()
 │    │    ├─> world_init()
 │    │    ├─> ui_init()
 │    │    └─> ecs_new() + ecs_add() (플레이어)
 │    │
 │    └─> window_loop()
 │         │
 │         └─> [무한 반복]
 │              ├─> 입력 처리 (GLFW)
 │              ├─> tick() (고정 간격)
 │              ├─> update() (프레임마다)
 │              ├─> render() (프레임마다)
 │              └─> 버퍼 스왑
 │
 └─> destroy()
```

---

## 3. 블록 구조체의 구성과 렌더링 방식

### 3.1 블록 시스템 아키텍처

블록 시스템은 **데이터 지향 설계**와 **다형성**을 결합한 구조입니다.

#### 3.1.1 블록 ID 열거형

```c
enum BlockId {
    AIR = 0,
    GRASS = 1,
    DIRT = 2,
    STONE = 3,
    SAND = 4,
    WATER = 5,
    GLASS = 6,
    LOG = 7,
    LEAVES = 8,
    ROSE = 9,
    // ... 총 25개 블록 타입
    PINE_LEAVES = 24
};
```

### 3.2 블록 구조체 상세

```c
struct Block {
    enum BlockId id;
    
    // 렌더링 속성
    bool transparent;        // true면 뒤 블록이 보임
    bool animated;           // true면 애니메이션 텍스처 사용
    enum BlockMeshType mesh_type;  // BLOCKMESH_DEFAULT, SPRITE, LIQUID, CUSTOM
    
    // 물리 속성
    bool liquid;             // 액체 동작 (물, 용암)
    bool solid;              // 충돌 체크
    f32 gravity_modifier;    // 액체의 중력 영향
    f32 drag;                // 액체의 항력
    f32 sliperiness;         // 얼음의 미끄러움
    
    // 조명 속성
    bool can_emit_light;     // 횃불 등 발광 블록
    
    // 동작 함수 포인터 (다형성)
    ivec2s (*get_texture_location)(struct World *world, ivec3s pos, enum Direction d);
    void (*get_mesh_information)(...);
    void (*get_animation_frames)(ivec2s out[BLOCK_ATLAS_FRAMES]);
    Torchlight (*get_torchlight)(struct World *world, ivec3s pos);
    void (*get_aabb)(struct World *world, ivec3s pos, AABB dest);
};
```

### 3.3 블록 초기화 패턴

각 블록은 자체 초기화 함수를 가집니다. 예: `grass_init()`

```c
// grass.c 예시
void grass_init() {
    BLOCKS[GRASS] = BLOCK_DEFAULT;
    BLOCKS[GRASS].id = GRASS;
    BLOCKS[GRASS].get_texture_location = get_texture_location;
}

static ivec2s get_texture_location(struct World *world, ivec3s pos, enum Direction d) {
    switch (d) {
        case UP:    return (ivec2s) {{ 0, 15 }};  // 풀 텍스처
        case DOWN:  return (ivec2s) {{ 2, 15 }};  // 흙 텍스처
        default:    return (ivec2s) {{ 1, 15 }};  // 측면 텍스처
    }
}
```

**장점:**
- 각 블록이 독립적으로 동작 정의
- 기본 동작(`BLOCK_DEFAULT`)을 상속하고 필요한 부분만 오버라이드
- 새 블록 추가가 쉬움

### 3.4 블록 메쉬 타입

#### BLOCKMESH_DEFAULT
일반적인 정육면체 블록 (돌, 흙, 나무 등)

#### BLOCKMESH_SPRITE
교차하는 평면 두 개로 구성 (꽃, 풀)
```
  ╲ ╱
   X
  ╱ ╲
```

#### BLOCKMESH_LIQUID
애니메이션되는 액체 블록 (물, 용암)
- 위쪽 면이 약간 낮음 (0.9 블록 높이)
- 텍스처 애니메이션

#### BLOCKMESH_CUSTOM
특수한 형태 (횃불 등)

### 3.5 청크 메쉬 생성 과정

청크는 32x32x32 블록을 포함하며, 렌더링 효율을 위해 **하나의 메쉬**로 결합됩니다.

#### 3.5.1 메쉬 생성 알고리즘 (`chunkmesh.c`)

1. **가시 면만 생성 (Greedy Meshing)**
   ```c
   for each block in chunk:
       for each face direction (UP, DOWN, NORTH, SOUTH, EAST, WEST):
           neighbor = get_neighbor_block(direction)
           if should_render_face(block, neighbor, direction):
               add_face_to_mesh(block, direction)
   ```

2. **가시성 판단**
   - 인접 블록이 투명하면 면 생성
   - 인접 블록이 불투명하면 면 생략 (최적화)

3. **버텍스 데이터 생성**
   각 면은 4개의 버텍스와 6개의 인덱스 (2개의 삼각형):
   ```c
   struct Vertex {
       vec3 position;        // 3D 위치
       vec2 uv;              // 텍스처 좌표
       vec3 normal;          // 노멀 (조명 계산)
       u32 light;            // 패킹된 조명 데이터
   };
   ```

4. **조명 데이터 베이킹**
   각 버텍스에 주변 조명 값을 미리 계산하여 저장:
   - 태양광 (Sunlight)
   - 횃불 조명 (Torchlight - RGB)

5. **메쉬 분류**
   - **BASE**: 불투명 블록
   - **TRANSPARENT**: 투명 블록 (물, 유리 등)
   
   투명 블록은 별도로 관리되어 깊이 정렬 후 나중에 렌더링됩니다.

#### 3.5.2 텍스처 아틀라스

모든 블록 텍스처는 하나의 큰 이미지 파일에 통합됩니다.

```c
// blockatlas.h
#define BLOCK_ATLAS_SIZE 16  // 16x16 텍스처 그리드
#define BLOCK_ATLAS_FRAMES 3 // 애니메이션 프레임 수
```

각 블록은 아틀라스 내 위치(UV 좌표)를 반환:
```c
ivec2s (*get_texture_location)(struct World *world, ivec3s pos, enum Direction d);
```

예: 풀 블록
- 윗면: (0, 15)
- 옆면: (1, 15)
- 아랫면: (2, 15)

### 3.6 렌더링 파이프라인

#### 3.6.1 청크 렌더링 순서

```c
void world_render(struct World *self) {
    // 1. 하늘 박스
    sky_render(&self->sky);
    
    // 2. 불투명 블록 (앞에서 뒤로, 깊이 테스트)
    for each visible chunk:
        chunkmesh_render(chunk->mesh, BASE);
    
    // 3. 투명 블록 (뒤에서 앞으로 정렬, 알파 블렌딩)
    sort_transparent_faces_by_distance();
    for each visible chunk:
        chunkmesh_render(chunk->mesh, TRANSPARENT);
    
    // 4. 엔티티 렌더링
    ecs_event(&self->ecs, ECS_RENDER);
}
```

#### 3.6.2 셰이더 시스템

**버텍스 셰이더**: 
- 월드 좌표 → 클립 공간 변환
- 조명 데이터 언팩

**프래그먼트 셰이더**:
- 텍스처 샘플링
- 조명 적용 (태양광 + 횃불)
- 안개 효과 (거리 기반)
- 주야 사이클에 따른 색상 조정

### 3.7 조명 시스템

#### 3.7.1 조명 타입

```c
typedef u16 Torchlight;  // 16비트: R(4) G(4) B(4) I(4)
typedef u8 Sunlight;     // 8비트: 강도 (0-15)
```

#### 3.7.2 조명 전파 알고리즘 (BFS)

```c
void light_update(struct World *world, ivec3s pos) {
    // 1. 광원 추가/제거
    // 2. BFS로 인접 블록에 전파
    // 3. 강도는 거리에 따라 감소
    // 4. 불투명 블록에서 차단
}
```

태양광은 위에서 아래로 직선 전파, 횃불은 모든 방향으로 감쇠하며 전파됩니다.

---

## 4. 월드 생성 알고리즘

### 4.1 절차적 생성 개요

월드는 **시드 기반**으로 결정론적으로 생성되며, 같은 시드는 항상 같은 월드를 생성합니다.

### 4.2 노이즈 기반 지형 생성

#### 4.2.1 노이즈 함수 (`noise.h/c`)

여러 노이즈 레이어를 조합하여 자연스러운 지형을 생성:

```c
struct Noise {
    f32 (*compute)(void *params, u64 seed, f32 x, f32 y);
    void *params;
};

// 노이즈 타입
struct Noise basic(int octave);           // 기본 펄린 노이즈
struct Noise octave(int octaves, int offset);  // 여러 옥타브 조합
struct Noise combined(Noise *a, Noise *b);    // 두 노이즈 결합
struct Noise expscale(Noise *n, f32 exp, f32 scale);  // 스케일 조정
```

#### 4.2.2 사용되는 노이즈 맵

```c
void worldgen_generate(struct Chunk *chunk) {
    // 1. 높이 노이즈 (n_h): 지형의 기본 높이
    struct Noise n_h = expscale(&os[0], 1.3f, 1.0f / 128.0f);
    
    // 2. 습도 노이즈 (n_m): 바이옴 결정
    struct Noise n_m = expscale(&cs[0], 1.0f, 1.0f / 512.0f);
    
    // 3. 온도 노이즈 (n_t): 바이옴 결정
    struct Noise n_t = expscale(&cs[1], 1.0f, 1.0f / 512.0f);
    
    // 4. 거칠기 노이즈 (n_r): 지형 디테일
    struct Noise n_r = expscale(&cs[2], 1.0f, 1.0f / 16.0f);
    
    // 5. 산 노이즈 (n_n): 산악 지형
    struct Noise n_n = expscale(&cs[3], 3.0f, 1.0f / 512.0f);
    
    // 6. 봉우리 노이즈 (n_p): 산 정상
    struct Noise n_p = expscale(&cs[4], 3.0f, 1.0f / 512.0f);
}
```

### 4.3 바이옴 시스템

#### 4.3.1 바이옴 타입

```c
enum Biome {
    OCEAN,      // 해양
    RIVER,      // 강
    BEACH,      // 해변
    DESERT,     // 사막
    SAVANNA,    // 사바나
    JUNGLE,     // 정글
    GRASSLAND,  // 초원
    WOODLAND,   // 삼림 지대
    FOREST,     // 숲
    RAINFOREST, // 열대우림
    TAIGA,      // 타이가
    TUNDRA,     // 툰드라
    ICE,        // 빙원
    MOUNTAIN    // 산
};
```

#### 4.3.2 바이옴 선택 알고리즘

```c
static enum Biome get_biome(f32 h, f32 m, f32 t, f32 n, f32 i) {
    // h = 높이 [-1, 1]
    // m = 습도 [0, 1]
    // t = 온도 [0, 1]
    // n = 산 노이즈 [0, 1]
    // i = 수정된 높이맵 노이즈 [0, 1]
    
    if (h <= 0.0f || n <= 0.0f) {
        return OCEAN;  // 해수면 이하
    } else if (h <= 0.005f) {
        return BEACH;  // 해안선
    }
    
    if (n >= 0.1f && i >= 0.2f) {
        return MOUNTAIN;  // 높은 산
    }
    
    // 온도와 습도 기반 바이옴 테이블 조회
    return BIOME_TABLE[moisture_index][temperature_index];
}
```

#### 4.3.3 바이옴 테이블 (온도 x 습도)

```c
const enum Biome BIOME_TABLE[6][6] = {
    //  매우추움  추움      온화      따뜻함    더움      매우더움
    { ICE,     TUNDRA,  GRASSLAND, DESERT,   DESERT,   DESERT   },  // 매우건조
    { ICE,     TUNDRA,  GRASSLAND, GRASSLAND,DESERT,   DESERT   },  // 건조
    { ICE,     TUNDRA,  WOODLAND,  WOODLAND, SAVANNA,  SAVANNA  },  // 보통
    { ICE,     TUNDRA,  TAIGA,     WOODLAND, SAVANNA,  SAVANNA  },  // 습함
    { ICE,     TUNDRA,  TAIGA,     FOREST,   JUNGLE,   JUNGLE   },  // 매우습함
    { ICE,     TUNDRA,  TAIGA,     TAIGA,    JUNGLE,   JUNGLE   }   // 극습함
};
```

#### 4.3.4 바이옴 데이터

각 바이옴은 고유한 속성을 가집니다:

```c
struct BiomeData {
    enum BlockId top_block;      // 표면 블록
    enum BlockId bottom_block;   // 지하 블록
    f32 roughness;               // 거칠기
    f32 scale;                   // 높이 스케일
    f32 exp;                     // 높이 지수
    struct Decoration decorations[MAX_DECORATIONS];  // 장식물
};
```

예시:
```c
[FOREST] = {
    .top_block = GRASS,
    .bottom_block = DIRT,
    .roughness = 1.0f,
    .scale = 1.0f,
    .exp = 1.0f,
    .decorations = {
        { .f = worldgen_tree, .chance = 0.009f },    // 0.9% 나무
        { .f = worldgen_flowers, .chance = 0.003f }, // 0.3% 꽃
        { .f = worldgen_grass, .chance = 0.008f }    // 0.8% 풀
    }
}
```

### 4.4 지형 생성 단계

#### 4.4.1 높이맵 생성

```c
for (s64 x = 0; x < CHUNK_SIZE.x; x++) {
    for (s64 z = 0; z < CHUNK_SIZE.z; z++) {
        s64 wx = chunk->position.x + x;
        s64 wz = chunk->position.z + z;
        
        // 1. 노이즈 값 계산
        f32 h = n_h.compute(&n_h.params, seed, wx, wz);  // 높이
        f32 m = n_m.compute(&n_m.params, seed, wx, wz);  // 습도
        f32 t = n_t.compute(&n_t.params, seed, wx, wz);  // 온도
        f32 r = n_r.compute(&n_r.params, seed, wx, wz);  // 거칠기
        f32 n = n_n.compute(&n_n.params, seed, wx, wz);  // 산
        f32 p = n_p.compute(&n_p.params, seed, wx, wz);  // 봉우리
        
        // 2. 산 노이즈에 봉우리 추가
        n += safe_expf(p, (1.0f - n) * 3.0f);
        
        // 3. 높이에 따라 온도 감소
        t -= 0.4f * n;
        
        // 4. 바이옴 결정
        enum Biome biome = get_biome(h, m, t, n, n + h);
        
        // 5. 바이옴 특성 적용
        h = sign(h) * powf(fabsf(h), biome.exp);
        
        // 6. 최종 높이 계산
        f32 final_height = ((h * 32.0f) + (n * 256.0f)) * biome.scale 
                          + (biome.roughness * r * 2.0f);
        
        // 7. 저장
        heightmap->worldgen_data[x * CHUNK_SIZE.x + z] = {
            .h_b = final_height,
            .b = biome
        };
    }
}
```

#### 4.4.2 높이맵 스무딩

```c
// 4방향 평균으로 부드럽게
for (s64 x = 0; x < CHUNK_SIZE.x; x++) {
    for (s64 z = 0; z < CHUNK_SIZE.z; z++) {
        f32 v = 0.0f;
        v += heightmap[x-1, z-1].h_b;
        v += heightmap[x+1, z-1].h_b;
        v += heightmap[x-1, z+1].h_b;
        v += heightmap[x+1, z+1].h_b;
        v *= 0.25f;
        heightmap[x, z].h = v;
    }
}
```

#### 4.4.3 블록 배치

```c
for (s64 x = 0; x < CHUNK_SIZE.x; x++) {
    for (s64 z = 0; z < CHUNK_SIZE.z; z++) {
        s64 h = heightmap[x, z].h;  // 높이
        enum Biome biome = heightmap[x, z].b;
        
        for (s64 y = 0; y < CHUNK_SIZE.y; y++) {
            s64 y_w = chunk->position.y + y;
            
            if (y_w > h && y_w <= WATER_LEVEL) {
                block = WATER;  // 해수면 이하는 물
            } else if (y_w > h) {
                block = AIR;    // 공중
            } else if (y_w == h) {
                block = biome_data.top_block;  // 표면 (풀, 모래 등)
            } else if (y_w >= (h - 3)) {
                block = biome_data.bottom_block;  // 지하 얕은 층 (흙)
            } else {
                block = STONE;  // 깊은 곳은 돌
            }
            
            chunk_set_block(chunk, {x, y, z}, block);
        }
    }
}
```

#### 4.4.4 장식물 생성

```c
if (y_w == h) {  // 표면에만 장식
    for (size_t i = 0; i < MAX_DECORATIONS; i++) {
        if (biome_data.decorations[i].f == NULL) break;
        
        if (RANDCHANCE(biome_data.decorations[i].chance)) {
            biome_data.decorations[i].f(chunk, _get, _set, x, y, z);
        }
    }
}
```

장식물 생성 함수:
- `worldgen_tree()`: 나무 생성 (줄기 + 잎)
- `worldgen_pine()`: 소나무 생성
- `worldgen_flowers()`: 꽃 생성 (장미, 미나리)
- `worldgen_grass()`: 긴 풀 생성
- `worldgen_shrub()`: 관목 생성

### 4.5 청크 로딩 전략

#### 4.5.1 중심 기반 로딩

플레이어 위치를 중심으로 일정 반경 내의 청크만 로드:

```c
void world_set_center(struct World *self, ivec3s center_pos) {
    ivec3s new_offset = world_pos_to_offset(center_pos);
    
    if (!glms_ivec3_eqv(new_offset, self->center_offset)) {
        // 청크 배열을 이동하고 새 청크 생성
        shift_chunks(self, new_offset);
        self->center_offset = new_offset;
    }
}
```

#### 4.5.2 지연 생성 (Throttling)

프레임 드롭을 방지하기 위해 한 프레임에 생성/메쉬화할 청크 수를 제한:

```c
struct {
    struct {
        u64 count, max;
    } mesh, load;  // 프레임당 최대 생성/메쉬 수
} throttles;
```

---

## 5. 사용된 게임 디자인 패턴

### 5.1 Entity Component System (ECS)

#### 5.1.1 개념

전통적인 객체 지향 대신 **데이터 지향** 설계를 사용합니다.

**전통적 OOP:**
```c
class Player : public Entity {
    Position position;
    Physics physics;
    Camera camera;
    Control control;
    // ...
};
```

**ECS:**
```c
// 엔티티는 단순한 ID
struct Entity {
    EntityId id;
    struct ECS *ecs;
};

// 컴포넌트는 순수 데이터
struct PositionComponent { vec3s position; };
struct PhysicsComponent { AABB size; bool gravity; bool collide; };
struct CameraComponent { vec3s offset; f32 fov; };

// 시스템은 컴포넌트를 처리하는 로직
void movement_system_update(void *data, struct Entity entity) {
    PositionComponent *pos = ecs_get(entity, C_POSITION);
    MovementComponent *mov = ecs_get(entity, C_MOVEMENT);
    // 로직...
}
```

#### 5.1.2 장점

1. **유연성**: 런타임에 컴포넌트 추가/제거 가능
2. **재사용성**: 컴포넌트를 여러 엔티티에서 공유
3. **성능**: 캐시 친화적인 메모리 레이아웃
4. **확장성**: 새 컴포넌트/시스템 추가가 쉬움

#### 5.1.3 구현

**컴포넌트 등록:**
```c
enum ECSComponent {
    C_POSITION,
    C_PHYSICS,
    C_MOVEMENT,
    C_CAMERA,
    C_CONTROL,
    C_BLOCKLOOK,
    C_LIGHT,
    // ...
};

void ecs_init(struct ECS *self, struct World *world) {
    ecs_register(C_POSITION, struct PositionComponent, self, ...);
    ecs_register(C_PHYSICS, struct PhysicsComponent, self, ...);
    // ...
}
```

**엔티티 생성:**
```c
struct Entity player = ecs_new(&world.ecs);
ecs_add(player, C_POSITION);
ecs_add(player, C_PHYSICS, (struct PhysicsComponent) { ... });
```

**컴포넌트 접근:**
```c
struct PositionComponent *pos = ecs_get(player, C_POSITION);
pos->position.x += 1.0f;
```

**시스템 이벤트:**
```c
void ecs_event(struct ECS *self, enum ECSEvent event) {
    for (each component type) {
        for (each entity with this component) {
            call_system_subscriber(event, entity);
        }
    }
}
```

### 5.2 Observer 패턴 (이벤트 시스템)

#### 5.2.1 ECS 이벤트

```c
enum ECSEvent {
    ECS_INIT,     // 컴포넌트 추가시
    ECS_DESTROY,  // 컴포넌트 제거시
    ECS_RENDER,   // 렌더링시
    ECS_UPDATE,   // 업데이트시
    ECS_TICK      // 틱시
};

union ECSSystem {
    struct {
        ECSSubscriber init, destroy, render, update, tick;
    };
};
```

각 컴포넌트 타입은 이벤트별 콜백을 등록할 수 있습니다:

```c
ecs_register(C_CAMERA, struct CameraComponent, ecs, (union ECSSystem) {
    .update = camera_system_update,
    .render = camera_system_render
});
```

#### 5.2.2 청크 수정 이벤트

```c
void chunk_on_modify(struct Chunk *self, ivec3s pos, u64 prev, u64 data) {
    // 블록 변경시 자동 호출
    // - 조명 업데이트
    // - 인접 청크 메쉬 재생성 표시
    // - 높이맵 업데이트
}
```

### 5.3 Object Pool 패턴

#### 5.3.1 청크 재사용

청크는 동적으로 할당/해제하는 대신 배열에 미리 할당:

```c
struct World {
    size_t chunks_size;        // 예: 9 (9x9x9 = 729개)
    struct Chunk **chunks;     // 고정 크기 배열
    ivec3s chunks_origin;      // 배열의 월드 오프셋
};
```

플레이어 이동시 배열을 순환시키고 멀어진 청크는 재초기화:

```c
void shift_chunks(struct World *self, ivec3s new_center) {
    for (each chunk in array) {
        if (chunk_too_far(chunk, new_center)) {
            chunk_unload(chunk);
            chunk_init(chunk, new_position);
        }
    }
}
```

#### 5.3.2 엔티티 ID 재사용

```c
struct ECS {
    EntityId *ids;           // 엔티티 ID 배열
    Bitmap used;             // 사용 중 비트맵
    size_t capacity;
    EntityId next_entity_id;
};
```

삭제된 엔티티의 슬롯은 새 엔티티에 재사용됩니다.

### 5.4 Flyweight 패턴

#### 5.4.1 블록 데이터 공유

블록 타입별 속성은 `BLOCKS` 배열에 한 번만 저장:

```c
extern struct Block BLOCKS[MAX_BLOCK_ID];
```

각 청크는 블록 ID만 저장:

```c
u64 *data;  // 각 u64는 16비트 블록 ID 포함
```

수천만 개의 블록이 25개의 `Block` 구조체를 공유합니다.

#### 5.4.2 텍스처 아틀라스

모든 블록 텍스처를 하나의 이미지로 통합하여 GPU 메모리와 드로우 콜을 최소화합니다.

### 5.5 Strategy 패턴

#### 5.5.1 블록 동작 다형성

각 블록은 함수 포인터를 통해 다른 동작을 가질 수 있습니다:

```c
struct Block {
    ivec2s (*get_texture_location)(...);
    void (*get_mesh_information)(...);
    Torchlight (*get_torchlight)(...);
    void (*get_aabb)(...);
};
```

예시:
- 풀 블록: 방향별로 다른 텍스처 반환
- 물 블록: 애니메이션 프레임 반환
- 횃불: RGB 조명 값 반환

#### 5.5.2 노이즈 전략

여러 노이즈 알고리즘을 동일한 인터페이스로 조합:

```c
struct Noise {
    f32 (*compute)(void *params, u64 seed, f32 x, f32 y);
    void *params;
};
```

### 5.6 Factory 패턴

#### 5.6.1 블록 초기화

```c
void block_init() {
    air_init();
    grass_init();
    dirt_init();
    // ...
}
```

각 `*_init()` 함수는 해당 블록 타입을 `BLOCKS` 배열에 등록합니다.

#### 5.6.2 엔티티 생성

```c
struct Entity ecs_new(struct ECS *self) {
    // 엔티티 ID 할당
    // 초기 상태 설정
    return entity;
}
```

### 5.7 Command 패턴 (콜백)

게임 루프는 콜백 함수를 통해 제어됩니다:

```c
window_create(
    init,     // 초기화 명령
    destroy,  // 종료 명령
    tick,     // 틱 명령
    update,   // 업데이트 명령
    render    // 렌더 명령
);
```

이는 게임 로직과 윈도우 관리를 분리합니다.

### 5.8 Singleton 패턴

전역 상태는 하나의 인스턴스만 존재:

```c
extern struct State state;
```

모든 서브시스템은 이 전역 상태를 참조합니다.

### 5.9 Spatial Hashing (공간 분할)

#### 5.9.1 청크 시스템

월드를 32x32x32 블록 단위로 분할하여 효율적인 쿼리:

```c
// 블록 위치 -> 청크 오프셋
ivec3s offset = world_pos_to_offset(block_pos);

// 청크 오프셋 -> 청크 포인터
struct Chunk *chunk = world_get_chunk(world, offset);
```

#### 5.9.2 높이맵

각 X-Z 컬럼의 최고 블록 높이를 캐시:

```c
struct Heightmap {
    ivec2s offset;
    s64 *data;  // [CHUNK_SIZE.x * CHUNK_SIZE.z]
};
```

이는 빠른 충돌 검사와 조명 계산에 사용됩니다.

### 5.10 Lazy Evaluation (지연 평가)

#### 5.10.1 청크 메쉬 생성

청크는 카메라에 보일 때만 메쉬를 생성합니다:

```c
void chunkmesh_prepare_render(struct ChunkMesh *self) {
    if (self->flags.dirty) {
        rebuild_mesh(self);
        self->flags.dirty = false;
    }
}
```

#### 5.10.2 조명 계산

조명은 블록 수정시에만 재계산:

```c
void chunk_on_modify(...) {
    if (block_changed_affects_light) {
        light_update(world, pos);
    }
}
```

### 5.11 Producer-Consumer 패턴

#### 5.11.1 비동기 청크 생성

메인 스레드는 생성 요청을 큐에 넣고, 워커 스레드가 처리:

```c
// 메인 스레드
queue_chunk_generation(chunk);

// 워커 스레드
while (has_pending_chunks()) {
    chunk = dequeue_chunk();
    worldgen_generate(chunk);
    mark_chunk_ready(chunk);
}
```

(현재 구현은 단일 스레드지만 구조는 멀티스레딩을 지원)

### 5.12 Dirty Flag 패턴

#### 5.12.1 청크 메쉬

```c
struct ChunkMesh {
    struct {
        bool dirty : 1;        // 재생성 필요
        bool finalize : 1;     // 업로드 필요
        bool depth_sort : 1;   // 정렬 필요
    } flags;
};
```

#### 5.12.2 높이맵

```c
struct Heightmap {
    struct {
        bool generated : 1;  // 생성 여부
    } flags;
};
```

변경시에만 재계산하여 성능을 향상시킵니다.

---

## 결론

이 프로젝트는 C 언어로 작성되었지만 현대적인 게임 개발 패턴을 효과적으로 활용합니다:

1. **ECS 아키텍처**로 유연하고 확장 가능한 엔티티 시스템
2. **데이터 지향 설계**로 캐시 효율성 최적화
3. **절차적 생성**으로 무한 월드 구현
4. **공간 분할(청크)**로 효율적인 렌더링과 물리
5. **Greedy Meshing**으로 렌더링 부하 최소화
6. **다형성(함수 포인터)**로 블록 시스템의 유연성
7. **Dirty Flag 및 Lazy Evaluation**으로 불필요한 계산 회피

이러한 패턴들은 복잡한 복셀 기반 게임을 효율적으로 구현하는 데 핵심적인 역할을 합니다.

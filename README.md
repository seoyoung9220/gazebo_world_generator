# Gazebo World Generator (개인 포크 — Jazzy/Harmonic 포팅)
![License](https://img.shields.io/badge/License-MIT-blue.svg)
![ROS Version](https://img.shields.io/badge/ROS-Jazzy-orange.svg)
![Gazebo Version](https://img.shields.io/badge/Gazebo-Harmonic-purple.svg)
![Python Version](https://img.shields.io/badge/Python-3.8+-green.svg)

**한 문장으로 복잡한 Gazebo 월드를 만든다.**

Gazebo World Generator는 아이디어를 바로 동작하는 Gazebo 시뮬레이션으로 바꿔줍니다. LLM(대형 언어 모델)의 힘을 빌려, 이 ROS2 패키지는 평범한 영어 문장을 해석해서 충돌 없는 배치를 자동으로 생성하고, 가구를 배치하고, 2D 내비게이션 맵까지 만들어 줍니다.

콘솔 인터페이스             |  생성된 월드
:-------------------------:|:-------------------------:
![console](https://github.com/robotics-upo/gazebo_world_generator/blob/master/media/gazebo_world_gen_console.gif) | *"8개의 팔레트와 10개의 선반이 있는 12x10m 창고와, 책상과 의자가 있는 작은 사무실"* ![world](https://github.com/robotics-upo/gazebo_world_generator/blob/master/media/gazebo_world_gen_world.gif)

> [!NOTE]
> 아직 개발 중인 프로젝트입니다. 정확도를 위해 노력하고 있지만, 생성된 월드는 용도에 맞게 수동 조정이 필요할 수 있습니다.

## 이 포크에 대해

이 레포는 [robotics-upo/gazebo_world_generator](https://github.com/robotics-upo/gazebo_world_generator)를 개인적으로 포크한 것으로, 제 환경에서 돌리기 위해 수정한 내용을 기록으로 남겨두는 용도입니다. **upstream에 머지할 목적이 아닙니다.**

**왜 포크했는가:** 원본 프로젝트는 ROS2 Humble + Gazebo Classic(gazebo11)을 대상으로 하지만, 제 작업 환경은 ROS2 Jazzy + Gazebo Harmonic(gz-sim8)입니다. Gazebo Classic은 수명이 끝난(EOL) 상태라, 죽은 의존성을 억지로 패치하는 대신 현재의 `gz-sim` 스택에서 동작하는 버전이 필요했습니다.

**무엇을, 왜 바꿨는지:**

- **Classic SDF 문법을 Harmonic용으로 포팅.** Harmonic에는 기본 제공되는 `model://ground_plane`이 없어서, `ground_plane`을 `<include>` 대신 인라인 `<plane>` 지오메트리로 생성하도록 바꿨습니다. 마찬가지로 벽에 쓰이던 `Gazebo/Grey` 스크립트 머티리얼(Classic 전용 Ogre 머티리얼 이름)도 Harmonic 렌더러가 해석하지 못하므로 `ambient`/`diffuse`/`specular` 값으로 명시적으로 대체했습니다.
- **모델/캐시 경로를 전부 `~/.gazebo/...`에서 `~/.gz/...`로 이동.** Harmonic의 모델 캐시와 Fuel 다운로드는 Classic 경로인 `~/.gazebo/models`가 아니라 `~/.gz/models`, `~/.gz/fuel/fuel.gazebosim.org` 아래에 저장되기 때문에, `generator_config.yaml`과 resolver 코드의 모든 경로를 수정했습니다. 이미 캐시해둔 모델과의 하위 호환을 위해 `~/.gazebo/models`도 보조 검색 경로로 남겨두었습니다.
- **기본 LLM 백엔드를 Groq 호스팅 모델로 전환** (`llama-3.3-70b-versatile`, `https://api.groq.com/openai/v1`). 로컬 LM Studio 서버 대신, 실제로 평소에 쓰는 환경에 맞췄습니다.
- **토크나이저 fallback 추가** (`placement/engine.py`, `placement/semantic_grouping.py`). 여기서 쓰는 토크나이저는 대략적인 토큰 수 계산용일 뿐인데, `AutoTokenizer.from_pretrained(DEFAULT_MODEL, ...)`가 `llama-3.3-70b-versatile`처럼 HuggingFace repo로 존재하지 않는 provider 내부 모델명에 대해서는 바로 실패합니다. 이제는 실패 시 `Qwen/Qwen2.5-0.5B-Instruct`로 fallback하며, 이는 순수 카운팅 용도일 뿐 실제로 월드를 생성하는 모델에는 영향을 주지 않습니다.
- **모델 심링크의 대소문자 비교 버그 수정** (`models/resolver.py`). 기존에는 mesh URI와 디렉터리명을 비교할 때 대소문자를 무시했는데(`mesh_model_name.lower() != dir_name.lower()`), 대소문자를 구분하는 Linux 파일시스템에서는 mesh `<uri>`가 디렉터리명과 대소문자만 다른 모델의 호환 심링크가 생성되지 않아서 로딩에 실패했습니다.
- **다운로드 직후 모델이 "찾을 수 없음"으로 뜨는 버그 수정.** 온라인 모델을 새로 다운로드한 뒤 `model://` URI를 반환하긴 했지만, 실제 조회에 쓰이는 `get_directory_for_model()`이 참조하는 `_model_name_to_dir_map`을 갱신하지 않아서, 방금 받은 모델이 바로 다음 조회에서 실패하던 문제를 고쳤습니다.
- **맵 생성 에러 메시지 개선.** `generate_map.sh`는 대부분의 진단 메시지를 stderr가 아니라 `echo`로 stdout에 출력하기 때문에, 실패 시 발생하는 `RuntimeError`가 비어있는 경우가 많았습니다. 이제 stderr가 비어있으면 stdout의 마지막 부분을 대신 사용합니다.

ROS2 Humble + Gazebo Classic을 쓰신다면 원본 upstream 레포를 쓰세요 — 이 포크는 범용 대체재가 아니라 제 환경에서만 쓰는 작업용 사본입니다.

## 목차

- [기능](#기능)
- [사전 요구사항](#사전-요구사항)
- [설치](#설치)
- [빠른 시작](#빠른-시작)
- [사용 예시](#사용-예시-비대화형)
- [설정](#설정)
- [출력 파일](#출력-파일)
- [커맨드라인 옵션](#커맨드라인-옵션)
- [동작 원리](#동작-원리)
- [문제 해결](#문제-해결)

## 기능

- **자연어 입력** – 평범한 영어 문장으로 월드를 설명
- **자동 배치** – 사무실엔 스마트한 가구 그룹핑, 창고엔 체계적인 그리드 배치
- **내비게이션 준비 완료** – 로봇 내비게이션용 2D 맵 생성
- **대화형 수정** – 간단한 명령으로 기존 월드 수정
- **유연한 크기 지정** – 방 크기를 정확히 지정 가능

## 사전 요구사항

- ROS2 Jazzy
- Gazebo Harmonic (gz-sim8)
- Python 3.8 이상
- OpenAI 호환 LLM 서버 (예: Groq, LM Studio, Ollama, OpenAI API)

## 설치

### 1. 레포지토리 클론

```bash
cd ~/ros2_ws/src
git clone https://github.com/<your-fork>/gazebo_world_generator.git
git clone https://github.com/<your-fork>/gazebo_ros2_2Dmap_plugin.git   # harmonic-port 브랜치
```

### 2. 의존성 설치

```bash
# Python 패키지
cd gazebo_world_generator
pip install -r requirements.txt --break-system-packages

# ROS2 / Gazebo 패키지 (Jazzy + Gazebo Harmonic)
sudo apt-get install libgz-sim8-dev ros-jazzy-ros-gz-sim ros-jazzy-ros-gz-bridge

# occupancy map 플러그인 빌드 (harmonic-port 브랜치)
cd ~/ros2_ws
colcon build --packages-select gazebo_ros2_2dmap_plugin gazebo_world_generator
source install/setup.bash
```

### 3. LLM 서버 설정

설정 파일 `config/generator_config.yaml`을 엽니다:

```bash
nano config/generator_config.yaml
```

서버 정보를 입력합니다:

```yaml
llm:
  server_url: "http://localhost:1234/v1"
  model_name: "your-model-name"
```

### 4. 패키지 빌드

```bash
cd ~/ros2_ws
colcon build --packages-select gazebo_world_generator
source install/setup.bash
```

## 빠른 시작

생성기 실행:

```bash
ros2 run gazebo_world_generator generate_world
```

대화형 메뉴가 열립니다:

1. 설명으로 새 월드 만들기
2. 기존 월드 수정하기
3. LLM 서버 연결 테스트
4. 종료

## 사용 예시 (비대화형)

### 기본 월드 생성

```bash
ros2 run gazebo_world_generator generate_world \
    --description "Office with 4 desks and chairs"
```

### 방 크기 지정

다음 형식으로 정확한 크기를 지정할 수 있습니다: `"12m x 10m office"`, `"warehouse (20x15m)"`, `"office of size 8x6"`

```bash
ros2 run gazebo_world_generator generate_world \
    --description "12m x 10m warehouse with 8 storage racks"
```

### 창고 환경

창고에 선반·팔레트가 있을 경우, 시스템이 자동으로 최적화된 그리드 기반 배치를 사용합니다:

```bash
ros2 run gazebo_world_generator generate_world \
    --description "Warehouse with 10 storage racks and 5 pallets"
```

**참고:** 객체 개수는 항상 정확히 지정하세요. 시스템은 명시적으로 요청한 만큼만 생성합니다 (예: "5 pallets"라고 하면 정확히 5개만 생성, 더 생성하지 않음).

### 사용 가능한 전체 옵션

```bash
ros2 run gazebo_world_generator generate_world \
    --description "your world description" \
    --llm-server http://localhost:1234/v1 \
    --model your-model-name \
    --output custom-filename.sdf \
    --debug
```

출력 파일명을 지정하지 않으면 타임스탬프 기반 이름이 자동 생성됩니다.

### 수정 예시

대화형 메뉴의 2번 옵션으로 기존 월드를 수정할 수 있습니다:

- "Add 3 more desks with chairs"
- "Remove all bookshelves"
- "Move desks closer to the entrance"

## 설정

`config/generator_config.yaml`에서 LLM 서버 및 기타 설정을 변경합니다:

```yaml
llm:
  server_url: "http://localhost:1234/v1"  # LLM 서버 엔드포인트
  model_name: "your-model-name"            # 모델 이름

output:
  base_directory: "generated_worlds"       # 출력 디렉터리
  auto_generate_map: true                  # 내비게이션 맵 생성 여부

placement:
  min_object_distance: 0.5                 # 객체 간 최소 간격 (미터)
  corridor_width: 1.2                      # 통행 공간 폭 (미터)
```

생성기는 **OpenAI 호환 LLM 서버 엔드포인트**가 필요합니다. LM Studio, Ollama, OpenAI API, 또는 OpenAI 호환 커스텀 API 모두 사용 가능합니다.

### 설정 파일 커스텀 위치

패키지 업데이트 후에도 유지되는 사용자별 설정을 두고 싶다면:

```bash
mkdir -p ~/.config/gazebo_world_generator
cp config/generator_config.yaml ~/.config/gazebo_world_generator/
nano ~/.config/gazebo_world_generator/generator_config.yaml
```

설정 우선순위: 패키지 디렉터리 → 사용자 홈(`~/.config/`) → 시스템(`/etc/`)

## 출력 파일

생성된 월드는 `generated_worlds/`에 다음 파일들로 저장됩니다:

```plaintext
generated_worlds/
├── worlds/
│   └── world_YYYYMMDD_HHMMSS.sdf    # 월드 파일 (SDF 형식)
├── logs/
│   └── world_YYYYMMDD_HHMMSS.log    # 생성 로그
└── occupancy_maps/
    ├── world_YYYYMMDD_HHMMSS.yaml   # 맵 메타데이터
    └── world_YYYYMMDD_HHMMSS.pgm    # 2D 내비게이션 맵
```

**파일 형식 설명:**

- SDF (Simulation Description Format) – Gazebo의 월드 파일 형식
- PGM (Portable Gray Map) – 로봇 내비게이션용 2D occupancy grid
- YAML – 맵 설정 및 메타데이터

## 커맨드라인 옵션

커맨드라인에서 설정값을 오버라이드할 수 있습니다:

- `--description` – 월드 설명 (비대화형 모드에서 필수)
- `--llm-server` – LLM 서버 URL 오버라이드
- `--model` – 모델 이름 오버라이드
- `--output` – 출력 파일명 지정
- `--debug` – 상세 로그 활성화

## 동작 원리

생성기는 최적화된 멀티 페이즈 파이프라인으로 설명을 완전한 시뮬레이션 월드로 변환합니다:

1. **크기 추출** – 명시적인 치수 표현을 파싱 (예: "12m x 10m")
2. **LLM 파싱** – 자연어를 정확한 객체 개수가 담긴 구조화된 JSON으로 변환
3. **객체 확장** – 압축된 객체 개수를 개별 인스턴스로 확장
4. **모델 해석** – 지능적 캐싱을 통해 객체명을 Gazebo 모델에 매핑
5. **배치 전략** – 사무실(시맨틱 그룹핑) 또는 창고(그리드 배치) 전략 선택
6. **방 레이아웃** – 방 위치와 연결을 최적화
7. **충돌 없는 배치** – 자동 충돌 회피로 객체 배치
8. **월드 생성** – 물리/조명이 포함된 SDF 파일 생성
9. **맵 생성** – 2D 내비게이션 맵 생성

## 문제 해결

### LLM 서버 연결 문제

연결 테스트:

```bash
# 내장 테스트
ros2 run gazebo_world_generator generate_world
# 3번 옵션 선택: "Test LLM connection"

# 수동 테스트
curl http://localhost:1234/v1/models
```

### 호환되는 Gazebo 모델을 찾을 수 없음

요청한 내용에 맞는 모델 파일을 생성기가 찾지 못할 때 발생하는 에러입니다.

먼저 Gazebo 모델 경로가 올바르게 설정되어 있는지 확인하세요. 커스텀 모델이 있다면 GZ_SIM_RESOURCE_PATH 환경 변수에 위치를 추가합니다:

```bash
export GZ_SIM_RESOURCE_PATH=~/my_gazebo_models:$GZ_SIM_RESOURCE_PATH
```

**중요**: 생성기가 모델을 성공적으로 스폰하려면, 해당 파일이 다음 두 곳 중 하나에 존재해야 합니다:

- 로컬 GZ_SIM_RESOURCE_PATH 경로

- 제공되는 온라인 Gazebo 모델 레포지토리

시스템은 동의어도 찾을 만큼 똑똑합니다(예: "sofa"를 요청하면 "couch"도 검색). 하지만 결국 실제 모델 파일을 위 위치 중 하나에서 찾아야 하며, 파일이 없으면 실패합니다.

### LLM 컨텍스트 한도 에러

`'max_tokens' is too large` 같은 에러가 보인다면, 시스템이 다음과 같이 자동으로 처리합니다:

- 설명으로부터 입력 토큰 수 추정
- 모델 제약에 맞게 출력 토큰 한도 조정

매우 큰 월드(20개 이상 객체)의 경우 시스템이 자동으로 응답 크기를 줄입니다. 생성이 실패한다면:

- 더 작은 방으로 나누거나 배치 수정 작업을 사용
- 더 큰 컨텍스트를 지원하는 모델 사용

### 객체가 빠지거나 개수가 안 맞음

생성기는 명시적으로 지정한 객체**만** 생성합니다:

- ❌ "warehouse" → 빈 방 (자동으로 객체가 추가되지 않음)
- ✅ "warehouse with 10 storage racks" → 정확히 10개의 선반, 그 이상은 없음

설명에 항상 정확한 개수를 지정하세요.

### 객체 겹침

시스템에는 자동 충돌 회피가 포함되어 있습니다. 문제가 계속되면:

- 설명에서 방 크기를 늘리기
- 가구 개수를 줄이기
- `generated_worlds/logs/`에서 경고 확인

### 빌드 실패

클린 리빌드:

```bash
cd ~/ros2_ws
rm -rf build/ install/ log/
rosdep install --from-paths src --ignore-src -r -y
colcon build --packages-select gazebo_world_generator
```

### 패키지를 찾을 수 없음

워크스페이스를 source하세요:

```bash
source ~/ros2_ws/install/setup.bash
ros2 pkg list | grep gazebo_world_generator
```

## 라이선스

MIT License - Copyright (c) 2025 Service Robotics Lab

자세한 내용은 [LICENSE](LICENSE) 파일을 참고하세요.

[robotics-upo/gazebo_world_generator](https://github.com/robotics-upo/gazebo_world_generator)를 포크했습니다. 이 포크는 ROS2 Jazzy + Gazebo Harmonic용 개인 작업 사본입니다 — 무엇을, 왜 바꿨는지는 위의 [이 포크에 대해](#이-포크에-대해) 섹션을 참고하세요.

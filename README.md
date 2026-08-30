# Gesture-Controlled Object Tracking Robot

> 🎓 ROKEY 부트캠프 스터디 **5인 팀 프로젝트** · 2026.07.10 ~ 2026.08.05
> 📦 원본(팀 저장소): [Jik-Kim/study-project](https://github.com/Jik-Kim/study-project)

손 제스처로 로봇의 물체 추적을 제어하는 시스템입니다.
**보자기 → 추적 시작, 주먹 → 정지.** 카메라가 HSV 색상으로 빨간 공을 검출하면,
화면 중심 대비 위치·면적 오차를 속도 명령으로 변환해 turtlesim(2D)·Gazebo(3D)에서 로봇이 공을 추종합니다.

<br>

## 🎬 Demo

<!-- README 편집 화면에 GIF/영상 파일을 드래그해서 이 자리에 넣기 -->
_시연 영상 준비 중 (2026.08.04 촬영본)_

<br>

## 🧩 System Architecture

| 노드 | 역할 |
|------|------|
| `camera_node` | 웹캠 프레임 캡처 → `/camera/image_raw` 발행 |
| `gesture_node` | MediaPipe 손 랜드마크 → `START` / `STOP` / `NONE` 명령 |
| `object_tracking_node` | HSV 필터로 빨간 공 검출 → 위치 오차(`error_x/y`)·면적(`area`) 계산 |
| **`controller_node`** ⭐ | **제스처·추적 오차 → `geometry_msgs/Twist` 속도 명령 (담당 파트)** |
| `main_ui` | Tkinter 4분할 시각화 (영상 · 상태 · 토픽 흐름 · 속도 그래프) |

```
        ┌─▶ gesture_node ─────────▶ /gesture/command ──┐
camera ─┤                                              ├─▶ controller_node ──▶ /turtle1/cmd_vel
        └─▶ object_tracking_node ─▶ /tracking/object ──┘
```

<br>

## 🙋 나의 역할 — 상태 및 이동 제어 (Control)

> 제스처와 추적 오차를 입력받아 로봇 속도 명령을 생성하는 파트를 담당했습니다.
> `controller_node`, `tracking_controller.py`, `tracking_state.py`는 전부 직접 설계·구현했습니다.

### 1. 제어 인터페이스 계약 설계

팀 회의에서 제어 노드가 소비할 데이터 규약을 제안·확정하고 `docs/SOT.md`·`interfaces.md`에 반영했습니다.

- **오차 부호 규칙** — `error_x = ball_center_x - screen_center_x` (오른쪽 +, 왼쪽 −)
- **거리 근삿값** — 검출 면적(`area`)을 거리 대용으로 사용, `error_y`는 MVP 미사용(필드만 유지)
- **QoS 정책** — 제어 명령 `RELIABLE`, 추적 결과 `BEST_EFFORT` (KEEP_LAST, depth 1)
- **예외 동작** — STOP·미검출·메시지 타임아웃 시 정지 규칙 정의

### 2. 제어 알고리즘 — `core/tracking_controller.py`

- **회전** — `angular_z = -angular_gain × error_x` · 중앙 deadband · 최대 각속도 제한
- **전진** — `linear_x = linear_gain × (target_area - area)` · 목표 면적 deadband 내 정지 · 과대 검출 시 후진 없이 정지 · 최대 선속도 제한
- **구조 분리** — 계산 결과를 `VelocityCommand` 값 객체로 반환해 제어 로직과 ROS 계층을 분리

### 3. 추적 상태 머신 — `core/tracking_state.py`

- 초기 상태 `STOP`
- 제스처 명령으로 `START` / `STOP` 전이, `NONE`은 이전 상태 유지

### 4. 제어 ROS 2 노드 — `nodes/controller_node.py`

- 이동 제어 파라미터 **8종**을 선언해 런타임 튜닝 가능하도록 구성
- `/gesture/command`(RELIABLE) · `/tracking/object`(BEST_EFFORT) 구독 → `/turtle1/cmd_vel` 발행
- **안전 정지** — STOP 제스처 즉시 정지 / `detected=false` 즉시 정지 / 10 Hz 타이머로 추적 신호 끊김 감지 시 정지
- 재검출 시 상태가 `START`면 이동 자동 재개

### 5. 테스트 · 튜닝

- `TrackingController` 단위 테스트 작성 — 목표 면적 정지, 과대 검출 비후진, STOP·미검출 0속도
- 통합 테스트 후 게인 초깃값을 `0.004`로 보정

### 그 외

- `sim_bringup` 패키지 초기 구축 및 turtlesim / Gazebo bringup launch 최초 작성
  (이후 4분할 UI·Gazebo 확장은 통합 담당자가 발전시킴)

**관련 PR:** [#2](https://github.com/Jik-Kim/study-project/pull/2) · [#14](https://github.com/Jik-Kim/study-project/pull/14) · [#18](https://github.com/Jik-Kim/study-project/pull/18)

<br>

## 🛠️ Tech Stack

`Python 3.12` · `ROS 2 Jazzy` · `rclpy` · `rosidl` · `OpenCV` · `MediaPipe` · `Gazebo` · `turtlesim` · `pytest`

<br>

## ▶️ 실행 방법

```bash
# 1. Python 의존성 설치 (ROS 2 Jazzy 호환 조합 — 버전 고정 필수)
pip install --break-system-packages "numpy==1.26.4" "mediapipe==0.10.14"

# 2. 빌드 및 환경 로드
cd ros2_ws
colcon build --symlink-install
source install/setup.bash

# 3-a. turtlesim 2D 통합 실행
ros2 launch sim_bringup turtlesim_bringup.launch.py

# 3-b. Gazebo 3D (TurtleBot3) 실행
ros2 launch sim_bringup gazebo_bringup.launch.py
```

<br>

## 📝 배운 점 · 트러블슈팅

**토픽별 QoS 분리**
제스처 명령은 유실되면 안 되어 `RELIABLE`, 추적 결과는 최신값이 중요해 `BEST_EFFORT`로 설정.
한 노드 안에서도 토픽 성격에 따라 QoS를 나눠야 한다는 것을 배움.

**"신호 없음" 감지**
구독 콜백만으로는 추적 메시지가 끊긴 상황을 알 수 없어, 마지막 수신 시각을 검사하는
10 Hz 타이머로 타임아웃 정지를 구현.

**비례 제어 게인 튜닝**
게인이 크면 목표 면적 부근에서 진동, 작으면 반응이 느려 deadband와 함께 통합 테스트로 조정(`0.004`).

**의존성 충돌 (팀 공통)**
MediaPipe 설치가 `numpy`를 2.x로 올려 ROS 2 Jazzy의 `cv_bridge`와 충돌(`KeyError: 16`).
`numpy==1.26.4` + `mediapipe==0.10.14` 고정으로 해결하고 `docs/setup.md`에 기록.

<br>

## 📄 License

MIT License — 원본 프로젝트 라이선스 유지. [LICENSE](LICENSE) 참고.

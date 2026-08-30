# Gesture-Controlled Object Tracking Robot

> 🎓 ROKEY 부트캠프 스터디 **5인 팀 프로젝트** (2026.07.10 ~ 2026.08.05)
> 📦 원본(팀 저장소): [Jik-Kim/study-project](https://github.com/Jik-Kim/study-project)

웹캠으로 사용자의 손 제스처를 인식해 로봇의 물체 추적을 제어하는 시스템입니다.
보자기 → `START`(추적 시작), 주먹 → `STOP`(추적 정지).
카메라가 HSV 색상 기반으로 빨간 공을 검출하고, 화면 중심 대비 위치·면적 오차를
속도 명령으로 변환해 turtlesim(2D)과 Gazebo/TurtleBot3(3D)에서 로봇이 공을 추종합니다.

## 🎬 Demo

<!-- README 편집 화면에 GIF/영상 파일을 드래그해서 이 자리에 넣기 -->
_시연 영상 준비 중 (2026.08.04 촬영본)_

## 🧩 System Architecture

| 노드 | 역할 |
|---|---|
| `camera_node` | 웹캠 프레임 캡처 → `/camera/image_raw` 발행 |
| `gesture_node` | MediaPipe 손 랜드마크 → START/STOP/NONE 제스처 + 신뢰도 |
| `object_tracking_node` | HSV 색상 필터로 빨간 공 검출, 화면 중심 대비 `error_x/y`·`area` 계산 |
| **`controller_node`** | **제스처·추적 오차 → `geometry_msgs/Twist` 속도 명령 (담당)** |
| `main_ui` | Tkinter 기반 카메라 영상 · 상태 · 토픽 흐름 · 속도 그래프 4분할 시각화 |

```
camera ─┬─▶ gesture_node ──▶ /gesture/command ─┐
        └─▶ object_tracking_node ──▶ /tracking/object ─┴─▶ controller_node ──▶ /turtle1/cmd_vel
```

## 🙋 나의 역할 (Injae Hwang) — 상태 및 이동 제어 (Control)

**1. 제어 인터페이스 계약 설계**
팀 회의에서 제어 노드가 소비할 데이터 규약을 제안·확정하고 문서(`docs/SOT.md`, `interfaces.md`)에 반영.
- 오차 부호 규칙: `error_x = ball_center_x - screen_center_x` (오른쪽 +, 왼쪽 −)
- `area`(검출 면적)를 거리 근삿값으로 사용, `error_y`는 MVP 미사용·필드만 유지
- QoS 정책: 제어 명령 `RELIABLE`, 추적 결과 `BEST_EFFORT`(KEEP_LAST, depth 1)
- 미검출·STOP·타임아웃 시 동작 정의

**2. 제어 알고리즘 구현** (`core/tracking_controller.py`)
- 회전: `angular_z = -angular_gain * error_x`, `angular_deadband`·`max_angular_speed` 적용
- 전진: `linear_x = linear_gain * (target_area - area)`, `area_deadband` 내 정지,
  목표보다 크게 검출되면 후진 없이 정지, `max_linear_speed` 클램핑
- `VelocityCommand` 값 객체로 계산부와 ROS 계층 분리

**3. 추적 상태 관리** (`core/tracking_state.py`)
- `TrackingStateMachine` — 초기 `STOP`, 제스처로 `START`/`STOP` 전이, `NONE`은 이전 상태 유지

**4. 제어 ROS 2 노드 구현** (`nodes/controller_node.py`)
- 이동 제어 파라미터 8종 선언 후 `TrackingController`에 주입
- `/gesture/command`(RELIABLE)·`/tracking/object`(BEST_EFFORT) 구독 → `/turtle1/cmd_vel` 발행
- 안전 정지: STOP 제스처 즉시 `Twist(0,0)`, `detected=false` 즉시 정지,
  10 Hz 타이머로 추적 메시지 중단(`tracking_timeout_sec`) 감지 시 정지
- 재검출 시 상태가 `START`면 이동 자동 재개

**5. 단위 테스트 작성** — `test_tracking_controller.py` (목표 면적 정지, 과대검출 비후진, STOP/미검출 0속도 등)

**6. 파라미터 튜닝** — 통합 테스트 후 `linear_gain`/`angular_gain` 초깃값을 `0.004`로 보정

**7. 코드 리뷰** — 이시율의 ROS 2 통신·인터페이스 PR 리뷰 담당(주차 로테이션)

**부가 기여** — `sim_bringup` 패키지 초기 구축 및 turtlesim/Gazebo bringup launch 최초 작성
(이후 4종 통합 UI·Gazebo 확장은 통합 담당자가 발전)

> 관련 PR: [#2](https://github.com/Jik-Kim/study-project/pull/2) ·
> [#14](https://github.com/Jik-Kim/study-project/pull/14) ·
> [#18](https://github.com/Jik-Kim/study-project/pull/18)

## 🛠️ Tech Stack

Python 3.12 · ROS 2 Jazzy · rclpy · rosidl · OpenCV · MediaPipe · Gazebo · turtlesim · pytest

## ▶️ 실행 방법

```bash
# 1. Python 의존성 설치 (Jazzy 호환 골든 조합 — 버전 고정 필수)
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

## 📝 배운 점 / 트러블슈팅

- **토픽별 QoS 분리** — 제스처 명령은 유실되면 안 되어 `RELIABLE`, 추적 결과는 최신값이
  중요해 `BEST_EFFORT`로 두었다. 하나의 노드 안에서도 토픽 성격에 따라 QoS를 나눠야 함을 배움.
- **"신호 없음" 감지** — 구독 콜백만으로는 추적 메시지가 끊긴 상황을 알 수 없어,
  마지막 수신 시각을 검사하는 10 Hz 타이머로 타임아웃 정지를 구현.
- **비례 제어 게인 튜닝** — 게인이 크면 목표 면적 부근에서 진동, 작으면 반응이 느려
  deadband와 함께 통합 테스트로 조정(`0.004`).
- **의존성 충돌(팀 공통)** — MediaPipe 설치가 `numpy`를 2.x로 올려 ROS 2 Jazzy의
  `cv_bridge`와 충돌(`KeyError: 16`). `numpy==1.26.4` + `mediapipe==0.10.14` 고정으로 해결,
  `docs/setup.md`에 트러블슈팅으로 기록.

## 📄 License

MIT License — 원본 프로젝트 라이선스 유지. [LICENSE](LICENSE) 참고.

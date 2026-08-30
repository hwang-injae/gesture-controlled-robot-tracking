# Gesture-Controlled Object Tracking Robot

> 🎓 ROKEY 부트캠프 스터디 **5인 팀 프로젝트** (2026.07.10 ~ 2026.08.05)
> 📦 원본(팀 저장소): [Jik-Kim/study-project](https://github.com/Jik-Kim/study-project)

손동작으로 로봇의 물체 추적을 제어하는 시스템입니다.
손 보자기 → `START`(추적 시작), 주먹 → `STOP`(추적 정지).
카메라가 HSV 색상 기반으로 물체를 인식하고, turtlesim(2D) · Gazebo/TurtleBot3(3D)
시뮬레이션에서 로봇이 대상을 추적합니다.

## 🎬 Demo

<!-- 여기에 GIF 또는 영상 파일을 드래그해서 넣기 -->
채우기: 시연 GIF

## 🙋 나의 역할 (Injae Hwang)

- **추적 제어 노드(controller_node) 개발/개선**
  - 물체 수평 위치 오차 → 각속도, 크기(면적) 차이 → 선속도로 변환하는 pursuit 제어
  - 물체 소실 시 속도 0 정지 처리 (추적 상태는 유지 → 재감지 시 자동 재개)
- **제어 파라미터 튜닝** — 이동 제어 게인 초깃값 조정
- **시뮬레이션 통합 환경 구축** — `sim_bringup` 패키지 작성, Gazebo / TurtleBot3 연동 launch 구성
- 제어 노드 토픽 네이밍을 팀 인터페이스 규약(SOT)에 맞게 정리
- 시뮬레이션 UI 기능 개발 참여

> PR: [#11](https://github.com/Jik-Kim/study-project/pull/11) ·
> [#15](https://github.com/Jik-Kim/study-project/pull/15) ·
> [#19](https://github.com/Jik-Kim/study-project/pull/19)

## 🧩 System Architecture

| 노드 | 역할 |
|---|---|
| `camera_node` | 카메라 프레임 캡처 |
| `gesture_node` | MediaPipe 손 랜드마크 → 제스처 인식 |
| `object_tracking_node` | HSV 색상 필터로 물체 검출 |
| `controller_node` | 위치·크기 오차 → 로봇 속도 명령 (담당) |
| `main_ui` | Tkinter 기반 카메라/상태/텔레메트리 시각화 |

## 🛠️ Tech Stack

Python 3.12 · ROS 2 Jazzy · rclpy · OpenCV · MediaPipe · Gazebo · turtlesim

## ▶️ 실행 방법

채우기: 원본 저장소 README의 설치/빌드/실행 명령을 그대로 복사해서 넣기
(대략 아래 형태)

```bash
# 의존성 설치
pip install -r requirements.txt

# 빌드
cd ros2_ws
colcon build
source install/setup.bash

# 2D (turtlesim) 실행
ros2 launch sim_bringup <turtlesim_launch_file>

# 3D (Gazebo) 실행
ros2 launch sim_bringup <gazebo_launch_file>

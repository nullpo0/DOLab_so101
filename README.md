# Orange Pi Zero 3 기반 SO-101 초기 설정 및 텔레오퍼레이션 가이드

## 1. 목적

본 문서는 Orange Pi Zero 3를 SO-101 로봇암의 로컬 제어 노드로 사용하여 다음 작업을 수행하기 위한 환경 구축 과정을 정리한다.

- Orange Pi SSH 접속
- LeRobot 환경 구축
- SO-101 Leader / Follower 모터 등록
- Leader / Follower 캘리브레이션
- Leader → Follower 텔레오퍼레이션
- 주요 오류 해결

최종적으로 다음과 같은 구조를 구성한다.

```text
SO-101 Leader ── USB ─┐
                      │
                      ├── Orange Pi Zero 3
                      │
SO-101 Follower ─ USB ┘
                      │
                      └── Network ── Main Server
```

Orange Pi는 SO-101의 모터와 직접 통신하는 로컬 제어 장치로 사용한다.

추후 데이터셋 저장, 학습, VLA inference 등 상대적으로 연산량이 큰 작업은 별도의 메인 서버에서 수행할 수 있다.

---

# 2. 사용 환경

본 가이드에서 사용한 환경은 다음과 같다.

```text
Board       : Orange Pi Zero 3 2GB
Architecture: ARM64 / aarch64
OS          : Debian/Ubuntu 계열
Python      : 3.12
Environment : uv virtual environment
Framework   : Hugging Face LeRobot
Robot       : SO-101 Leader / Follower
Motor       : Feetech STS3215
```

Leader와 Follower의 Bus Servo Adapter는 USB 허브를 통해 Orange Pi에 연결하였다.

예시:

```text
Orange Pi
   │
   └── USB Hub
          ├── SO-101 Leader Bus Adapter
          └── SO-101 Follower Bus Adapter
```

---

# 3. SSH 설정

Orange Pi에서 SSH 서버 상태를 확인한다.

```bash
sudo systemctl status ssh
```

SSH 서버가 설치되어 있지 않은 경우:

```bash
sudo apt update
sudo apt install openssh-server
```

SSH 활성화:

```bash
sudo systemctl enable --now ssh
```

Orange Pi의 IP 주소 확인:

```bash
hostname -I
```

다른 PC에서 접속:

```bash
ssh <USERNAME>@<ORANGE_PI_IP>
```

예:

```bash
ssh dolab@192.168.0.25
```

---

# 4. 기본 패키지 설치

```bash
sudo apt update

sudo apt install -y \
    git \
    curl \
    build-essential \
    python3-dev \
    pkg-config
```

---

# 5. uv 설치

Python 환경 관리는 `uv`를 사용하였다.

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

현재 셸에 PATH 적용:

```bash
export PATH="$HOME/.local/bin:$PATH"
```

설치 확인:

```bash
uv --version
```

---

# 6. SO-101 Python 환경 생성

프로젝트 디렉터리를 만든다.

```bash
mkdir -p ~/so101
cd ~/so101
```

Python 3.12 설치:

```bash
uv python install 3.12
```

가상환경 생성:

```bash
uv venv --python 3.12
```

가상환경 활성화:

```bash
source .venv/bin/activate
```

정상적으로 활성화되면 프롬프트가 다음과 비슷하게 나타난다.

```text
(so101) dolab@orangepizero3:~/so101$
```

Python 버전 확인:

```bash
python --version
```

---

# 7. LeRobot 설치

Orange Pi에서는 GPU 학습 환경이 아니라 SO-101 하드웨어 제어를 목적으로 사용하므로 CPU 환경으로 설치한다.

```bash
uv pip install --torch-backend cpu "lerobot[feetech]"
```

설치 후 다음 명령이 정상적으로 실행되는지 확인한다.

```bash
lerobot-find-port --help
```

---

# 8. USB Serial 권한 설정

현재 사용자를 `dialout` 그룹에 추가한다.

```bash
sudo usermod -aG dialout $USER
```

이후 SSH를 종료하고 다시 접속한다.

재접속 후:

```bash
cd ~/so101
source .venv/bin/activate
```

현재 그룹 확인:

```bash
groups
```

출력에 다음이 포함되어 있으면 된다.

```text
dialout
```

---

# 9. Leader / Follower 포트 확인

USB Bus Adapter들을 연결한 후:

```bash
ls -l /dev/ttyACM*
```

예시:

```text
/dev/ttyACM0
/dev/ttyACM1
```

본 구축 환경에서는 다음과 같이 사용하였다.

```text
Leader   : /dev/ttyACM0
Follower : /dev/ttyACM1
```

단, USB를 뺐다가 다시 연결하거나 재부팅한 경우 `ttyACM0`, `ttyACM1` 번호가 서로 바뀔 수 있다.

따라서 필요하면 LeRobot의 포트 탐색 기능을 사용한다.

```bash
lerobot-find-port
```

프로그램 안내에 따라 확인하고자 하는 Bus Adapter를 USB에서 제거하면 해당 장치의 포트를 확인할 수 있다.

---

# 10. USB 허브 사용 시 확인 사항

Orange Pi의 USB 포트가 부족하여 USB 허브를 사용할 수 있다.

예:

```text
Orange Pi
   │
   └── USB Hub
          ├── Leader Adapter
          └── Follower Adapter
```

장치 인식 여부 확인:

```bash
lsusb
```

USB 토폴로지 확인:

```bash
lsusb -t
```

Serial 장치 확인:

```bash
ls -l /dev/ttyACM* /dev/ttyUSB* 2>/dev/null
```

연결 문제 발생 시:

```bash
dmesg | tail -n 60
```

또는 실시간으로:

```bash
sudo dmesg -w
```

장치 하나는 정상적으로 동작하지만 두 개를 동시에 연결했을 때 문제가 발생한다면 USB 허브 전원 부족 가능성이 있다.

이 경우 별도의 전원을 사용하는 Powered USB Hub 사용을 고려한다.

---

# 11. Leader 모터 등록

Leader Bus Adapter 이외의 불필요한 연결을 확인한 후 다음 명령을 실행한다.

```bash
lerobot-setup-motors \
    --teleop.type=so101_leader \
    --teleop.port=/dev/ttyACM0
```

프로그램의 안내에 따라 모터를 **한 개씩** 연결한다.

모터 등록 시 모든 모터를 데이지 체인으로 동시에 연결하면 안 된다.

예:

```text
Bus Adapter
    │
    └── 현재 등록할 모터 1개
```

프로그램이 다음 모터를 요구하면 기존 모터를 제거한 뒤 해당 모터를 연결한다.

이 과정을 반복하여 Leader의 모든 모터를 등록한다.

---

# 12. Follower 모터 등록

Follower가 `/dev/ttyACM1`인 경우:

```bash
lerobot-setup-motors \
    --robot.type=so101_follower \
    --robot.port=/dev/ttyACM1
```

처음에는 다음과 같은 메시지가 출력된다.

```text
Connect the controller board to the 'gripper' motor only and press enter.
```

Bus Adapter에는 현재 등록할 모터 **한 개만 연결한다.**

예:

```text
Follower Bus Adapter
        │
        └── Gripper Motor
```

프로그램의 안내에 따라 이후 모터도 한 개씩 등록한다.

---

# 13. 모터가 발견되지 않는 경우

다음 오류가 발생할 수 있다.

```text
RuntimeError: Motor 'gripper' (model 'sts3215') was not found.
Make sure it is connected.
```

이 경우 다음 항목을 확인한다.

1. 사용한 `/dev/ttyACM*`가 실제 해당 Bus Adapter의 포트인지 확인한다.
2. 모터 전원이 정상적으로 공급되는지 확인한다.
3. Bus Adapter와 모터 사이의 3핀 케이블을 다시 연결한다.
4. 모터 한 개만 연결되어 있는지 확인한다.
5. Bus Adapter의 USB 통신 설정 및 점퍼 위치를 확인한다.
6. USB 허브를 사용하는 경우 전원 부족 여부를 확인한다.

포트가 의심되는 경우:

```bash
lerobot-find-port
```

를 다시 실행한다.

---

# 14. Follower 캘리브레이션

Follower가 `/dev/ttyACM1`인 경우:

```bash
lerobot-calibrate \
    --robot.type=so101_follower \
    --robot.port=/dev/ttyACM1 \
    --robot.id=so101_follower
```

먼저 로봇암을 각 관절 가동범위의 중간 정도 자세로 위치시킨다.

안내가 나타나면 Enter를 누른다.

이후 다음 관절들을 각각 전체 가동범위까지 움직인다.

```text
shoulder_pan
shoulder_lift
elbow_flex
wrist_flex
gripper
```

최소 위치와 최대 위치가 모두 기록되도록 각 관절을 충분히 움직인다.

완료되면 Enter를 눌러 캘리브레이션을 저장한다.

---

# 15. Leader 캘리브레이션

Leader가 `/dev/ttyACM0`인 경우:

```bash
lerobot-calibrate \
    --teleop.type=so101_leader \
    --teleop.port=/dev/ttyACM0 \
    --teleop.id=so101_leader
```

다음 메시지가 나타날 수 있다.

```text
Move all joints except 'wrist_roll' sequentially through their entire ranges of motion.
```

SO-101에는 6개의 모터가 있지만, 이 단계에서 화면에는 다음 5개만 표시되는 것이 정상이다.

```text
shoulder_pan
shoulder_lift
elbow_flex
wrist_flex
gripper
```

`wrist_roll`은 별도로 처리되기 때문에 MIN/MAX 기록 목록에서 제외된다.

캘리브레이션 화면 예:

```text
NAME            |    MIN |    POS |    MAX
shoulder_pan    |   2047 |   2047 |   2047
shoulder_lift   |   2047 |   2047 |   2047
elbow_flex      |   2047 |   2047 |   2047
wrist_flex      |   2047 |   2047 |   2047
gripper         |   2047 |   2047 |   2047
```

처음에는 MIN과 MAX가 거의 동일하게 보인다.

각 관절을 움직이면 예를 들어 다음과 같이 범위가 증가한다.

```text
NAME            |    MIN |    POS |    MAX
shoulder_pan    |    900 |   1850 |   3200
shoulder_lift   |   1100 |   2100 |   3300
...
```

모든 관절을 전체 가동범위로 움직인 뒤 Enter를 눌러 저장한다.

---

# 16. Teleoperation

Leader와 Follower의 캘리브레이션이 완료되면 텔레오퍼레이션을 수행한다.

본 환경에서는:

```text
Leader   : /dev/ttyACM0
Follower : /dev/ttyACM1

Leader ID   : so101_leader
Follower ID : so101_follower
```

다음 명령을 실행한다.

```bash
lerobot-teleoperate \
    --robot.type=so101_follower \
    --robot.port=/dev/ttyACM1 \
    --robot.id=so101_follower \
    --teleop.type=so101_leader \
    --teleop.port=/dev/ttyACM0 \
    --teleop.id=so101_leader
```

정상적으로 연결되면 다음과 같은 흐름으로 작동한다.

```text
사용자가 Leader를 움직임
        ↓
Orange Pi가 Leader joint position 읽음
        ↓
Follower의 target position으로 전달
        ↓
Follower가 Leader의 움직임을 따라감
```

처음 테스트할 때는 한 관절씩 천천히 움직이는 것을 권장한다.

확인할 관절:

```text
shoulder_pan
shoulder_lift
elbow_flex
wrist_flex
wrist_roll
gripper
```

Follower가 비정상적인 방향으로 급격히 움직이거나 관절 제한 위치에 충돌하려는 경우 즉시:

```text
Ctrl + C
```

를 눌러 프로그램을 종료한다.

---

# 17. 재부팅 또는 SSH 재접속 후

SSH에 다시 접속한 경우 먼저 가상환경을 활성화한다.

```bash
cd ~/so101
source .venv/bin/activate
```

그 다음 USB 포트를 확인한다.

```bash
ls -l /dev/ttyACM*
```

USB 장치 번호는 재부팅 또는 재연결 후 변경될 수 있으므로 주의한다.

필요하면:

```bash
lerobot-find-port
```

로 다시 확인한다.

---

# 18. 현재까지 구축 완료 상태

현재 구성에서는 다음 작업까지 정상 동작함을 확인하였다.

```text
[완료] Orange Pi OS 구축
[완료] SSH 접속
[완료] Python 3.12 + uv 환경 구축
[완료] LeRobot 설치
[완료] Leader Bus Adapter 연결
[완료] Follower Bus Adapter 연결
[완료] USB Hub를 통한 두 Adapter 동시 사용
[완료] Leader Motor Setup
[완료] Follower Motor Setup
[완료] Leader Calibration
[완료] Follower Calibration
[완료] Leader → Follower Teleoperation
```

---

# 19. 이후 진행 예정

다음 단계는 카메라와 데이터 수집 환경을 추가하는 것이다.

전체 목표 흐름은 다음과 같다.

```text
Motor Setup
    ↓
Calibration
    ↓
Teleoperation
    ↓
Camera Setup
    ↓
Dataset Recording
    ↓
Dataset → Main Server
    ↓
VLA Fine-tuning
    ↓
Inference
```

Orange Pi는 가능한 한 로봇 하드웨어 제어와 센서 수집을 담당하도록 하고, 모델 학습 및 대규모 inference는 메인 서버에서 수행하는 구조를 권장한다.

---

# 20. 자주 사용하는 명령 모음

## 가상환경 활성화

```bash
cd ~/so101
source .venv/bin/activate
```

## USB Serial 확인

```bash
ls -l /dev/ttyACM*
```

## SO-101 포트 검색

```bash
lerobot-find-port
```

## Leader Motor Setup

```bash
lerobot-setup-motors \
    --teleop.type=so101_leader \
    --teleop.port=/dev/ttyACM0
```

## Follower Motor Setup

```bash
lerobot-setup-motors \
    --robot.type=so101_follower \
    --robot.port=/dev/ttyACM1
```

## Leader Calibration

```bash
lerobot-calibrate \
    --teleop.type=so101_leader \
    --teleop.port=/dev/ttyACM0 \
    --teleop.id=so101_leader
```

## Follower Calibration

```bash
lerobot-calibrate \
    --robot.type=so101_follower \
    --robot.port=/dev/ttyACM1 \
    --robot.id=so101_follower
```

## Teleoperation

```bash
lerobot-teleoperate \
    --robot.type=so101_follower \
    --robot.port=/dev/ttyACM1 \
    --robot.id=so101_follower \
    --teleop.type=so101_leader \
    --teleop.port=/dev/ttyACM0 \
    --teleop.id=so101_leader
```

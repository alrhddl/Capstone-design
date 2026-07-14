# EMG 손목 제스처 컨트롤러 제작 가이드
### 손 움직임으로 안경 외 기기(PC·스마트폰·스마트홈 등)를 조작하는 웨어러블 만들기

---

## 0. 전체 그림 먼저

Meta Neural Band, Mudra, 예전 Myo가 하는 일은 결국 아래 4단계의 파이프라인입니다. 이 구조만 이해하면 규모를 자유롭게 줄이거나 키울 수 있습니다.

```
[손목 근육 전기신호]
      │  ① 센서 + 아날로그 프론트엔드(AFE)로 수집
      ▼
[디지털 EMG 신호 (+ IMU 움직임)]
      │  ② 필터링·윈도잉·특징 추출 (신호 처리)
      ▼
[특징 벡터]
      │  ③ 머신러닝 분류기로 "어떤 제스처인가" 판별
      ▼
[제스처 라벨: 핀치 / 주먹 / 스와이프 …]
      │  ④ 제스처 → 명령 매핑 후 기기로 전송
      ▼
[BLE HID / MQTT / IR 등으로 기기 조작]
```

핵심은 ④번입니다. 안경이 아닌 기기를 조작하려면 **출력 인터페이스를 범용 표준(BLE HID, MQTT/Matter 등)으로 만들면** 됩니다. 이 부분만 바꾸면 같은 밴드로 PC, 스마트폰, 조명, TV를 모두 제어할 수 있습니다.

---

## 1. 만들기 전에: 3가지 경로 선택

난이도와 목표에 따라 셋 중 하나를 고르세요. 처음이라면 **경로 A로 개념 검증(MVP) → 경로 B로 확장**을 강력히 권합니다.

| 경로 | 방식 | 난이도 | 비용(대략) | 추천 대상 |
|------|------|--------|-----------|-----------|
| **A. IMU 우선** | 가속도계/자이로만으로 손목 동작·간단 제스처 인식 | ★☆☆ | 3~7만원 | 빠른 프로토타입, 초심자 |
| **B. EMG + IMU 융합** | 근전도로 손가락 제스처 + IMU로 방향/포인팅 | ★★☆ | 8~25만원 | 진짜 "Neural Band 스타일" |
| **C. 고밀도 EMG** | 다채널(8ch+) 연구급 수집 | ★★★ | 30만원~ | 정밀도·연구·논문 재현 |

> **왜 A부터?** EMG는 전극 접촉·노이즈·개인차 때문에 초반 좌절이 큽니다. IMU만으로도 "손목 튕기기, 회전, 상하 스와이프" 같은 제스처는 충분히 잡히고, ④ 출력 파이프라인(BLE HID 등)을 먼저 완성해두면 나중에 센서만 EMG로 갈아끼우기 쉽습니다. Apple Watch 더블탭도 사실 광학+IMU 기반입니다.

---

## 2. 하드웨어 파트

### 2-1. 센서 방식의 이해

- **EMG(표면 근전도, sEMG):** 손가락을 움직일 때 손목 근육에서 나오는 미세 전기신호를 읽습니다. 손이 안 보여도 되고, 미세한 핀치·개별 손가락 구분까지 가능. 단점은 노이즈·전극 접촉·개인/세션별 편차.
- **IMU(관성):** 손목 자체의 기울기·회전·가속을 읽습니다. 스와이프·회전·포인팅에 강하지만, "가만히 있는 손가락 핀치"는 못 잡음.
- **융합(EMG+IMU):** 실제 상용 밴드가 쓰는 방식. EMG=손가락 제스처, IMU=방향/포인터. 둘을 합치면 인식률이 크게 오릅니다(연구에서 단일 대비 10%p 이상 향상 사례 다수).

### 2-2. 부품 목록 (BOM)

**공통 (모든 경로)**
- **MCU/무선 보드**: 아래 중 택1
  - *Seeed XIAO nRF52840 (Sense)* — 초소형, BLE 내장, 온보드 IMU, 손목형에 최적 ✅추천
  - *Arduino Nano 33 BLE Sense Rev2* — IMU·마이크 등 센서 풍부, TinyML 예제 많음
  - *ESP32 (예: XIAO ESP32-S3)* — Wi-Fi까지 필요(스마트홈 MQTT 직접 연결)하면 유리
- **배터리**: LiPo 3.7V 150~400mAh (손목 크기 고려) + 충전관리(TP4056 또는 보드 내장 충전회로)
- **기타**: 얇은 배선, 커넥터, 스위치, 3D 프린팅 케이스/밴드 스트랩

**경로 A 추가 부품**: 없음 (XIAO/Nano의 온보드 IMU 사용). 필요시 외장 *MPU-6050/9250* 또는 *BMI270*.

**경로 B 추가 부품 (EMG 1~4채널)**
- **MyoWare 2.0 Muscle Sensor** ×1~4 — 메이커 표준. 정류·적분된 매끈한 신호와 **RAW 출력** 둘 다 제공하고, 온보드 게인 조절 가능. 초보 접근성 최고 ✅
  - 전극: 일회용 Ag/AgCl 젤 전극 또는 MyoWare용 스냅 전극. 손목 밴드화하려면 재사용 **건식 전극(스테인리스/전도성 패브릭)** 을 밴드 안쪽에 배치.
- (대안) **Gravity/OYMotion 아날로그 EMG 센서** — 저렴하지만 SNR은 MyoWare보다 낮음.

**경로 C 추가 부품 (고밀도/연구급)**
- **ADS1299** 기반 보드 (24-bit, 채널당 ~1kHz, Right-Leg-Drive 내장) — 상용 밴드/논문들이 쓰는 칩.
- 또는 **OpenBCI Cyton (8ch)** — 바로 쓸 수 있는 완제품 보드, EMG/EEG/ECG 지원, SDK(BrainFlow) 제공. 가장 빠른 연구급 시작점.
- 또는 완제품 **8채널 BLE EMG 암밴드(gForce 등)** — 하드웨어 제작을 건너뛰고 SDK로 소프트웨어만 개발하고 싶을 때.

> **팁:** 하드웨어 제작이 목표가 아니라 "소프트웨어/제어 로직"이 목표라면, 완제품 8채널 암밴드나 OpenBCI를 사고 소프트웨어(②③④)에 집중하는 게 훨씬 빠릅니다.

### 2-3. 전극/센서 배치 (경로 B·C 핵심)

- 손목보다 **전완(forearm) 근위부**가 신호가 크지만, "손목 밴드" 폼팩터를 원하면 손목 둘레에 채널을 **등간격 링(ring)** 형태로 배치합니다(상용 밴드 방식).
- 채널 수: 손가락 구분까지 원하면 최소 **4채널**, 정밀하면 8채널.
- 각 EMG 채널: **+전극 / −전극(차동)** 이 근육 결을 따라 2~2cm 간격, **기준(REF)/바이어스 전극**을 뼈 위(전기적으로 조용한 곳)에.
- 건식 전극은 접촉 압력이 중요 → 밴드를 **약간 타이트하게**. (실사용 후기에서도 "밴드를 꽉 조여야 제스처가 잡힌다"는 얘기가 많습니다.)

### 2-4. 배선 예 (경로 B, MyoWare + XIAO nRF52840)

```
MyoWare 2.0 (각 채널)      XIAO nRF52840
   VIN  ───────────────►   3V3 (또는 배터리 승압)
   GND  ───────────────►   GND
   ENV/RAW ────────────►   A0, A1, A2, A3 (채널별 ADC 핀)
   (게인은 온보드 포텐셔미터로 조절)

LiPo 3.7V ─► 보드 배터리 패드(BAT+/BAT−)  ※ XIAO는 충전회로 내장
```

- **RAW 핀**을 쓰면 원신호가 나오고(직접 필터링해야 함), **ENV(envelope) 핀**을 쓰면 이미 정류·평활된 신호라 처리가 쉽습니다. 학습 초반엔 ENV, 정밀도 원하면 RAW를 권장.
- ADC 해상도: Arduino계 10-bit는 한계가 있으니, 가능하면 12-bit 이상 ADC를 가진 보드(nRF52/ESP32)를 쓰세요. 연구급은 ADS1299의 24-bit.

### 2-5. 전원·조립

- 목표 소비전류를 낮추려면 EMG 채널 수를 줄이고, MCU의 저전력 모드를 적극 활용. Neural Band가 18시간 가는 것도 저전력 EMG 설계 덕분.
- 케이스는 3D 프린팅(TPU 밴드 + PLA/레진 하우징). 전극은 밴드 안쪽에 노출, 전자부는 방수(IPX 등급 원하면 실리콘 포팅).

---

## 3. 신호 처리 파트 (② 단계)

여기가 인식률의 절반을 좌우합니다. 펌웨어(MCU) 또는 PC 어느 쪽에서 해도 됩니다. 초반엔 **PC에서 처리·학습 → 완성 후 MCU로 이식(TinyML)** 순서가 편합니다.

### 3-1. 샘플링

- EMG 유효 대역은 대략 **20~450Hz** → 나이퀴스트상 최소 1kHz 샘플링 권장(연구는 1~2kHz).
- IMU는 50~200Hz면 충분.

### 3-2. 필터링 (RAW 신호 사용 시)

1. **밴드패스** 20~450Hz (근육 신호 대역만 통과)
2. **노치필터** 50/60Hz (전원 노이즈 제거 — 한국은 60Hz)
3. **정류(rectify)** 후 **이동평균/RMS**로 포락선(envelope) 생성

### 3-3. 윈도잉(세그멘테이션)

- 신호를 **150~250ms 창(window)** 으로 자르고, **50~100ms overlap** 으로 슬라이딩. (반응성과 정확도의 트레이드오프. 실시간 제어는 250ms 이하 권장.)

### 3-4. 특징 추출 (전통 ML 경로)

윈도마다 채널별로 아래 시간영역 특징을 뽑습니다(고전적이지만 여전히 강력):

- **MAV** (Mean Absolute Value)
- **RMS** (Root Mean Square)
- **WL** (Waveform Length)
- **ZC** (Zero Crossings)
- **SSC** (Slope Sign Changes)
- (선택) 주파수영역: 중앙주파수, 평균주파수

4채널 × 5특징 = 20차원 벡터가 분류기 입력이 됩니다. 딥러닝(CNN)을 쓸 거면 이 단계를 생략하고 원신호 창을 그대로 넣어도 됩니다.

---

## 4. 소프트웨어 / 머신러닝 파트 (③ 단계)

### 4-1. 데이터 수집 (가장 중요, 절대 건너뛰지 말 것)

- 제스처를 정하세요. 초반 추천 세트(4~6개): **핀치(엄지+검지) / 주먹 쥐기 / 손가락 펴기 / 손목 왼→오른 스와이프 / 손목 오른→왼 / 휴지(rest)**.
- 각 제스처를 **여러 세션·여러 날·약간씩 다른 밴드 착용 위치**에서 수집(개인 내 편차가 커서 이게 일반화의 핵심).
- 라벨링: 화면 프롬프트에 맞춰 제스처를 취하게 하고 자동 라벨(Screen-Guided Training). 각 제스처 최소 수백 개 윈도.
- **rest/무동작 클래스**를 반드시 포함해야 오작동(false trigger)이 줄어듭니다.

### 4-2. 모델 선택

| 접근 | 모델 | 특징 |
|------|------|------|
| 전통 ML (추천 시작) | LDA, SVM, **Random Forest** | 특징벡터 기반, 가볍고 MCU 이식 쉬움, 데이터 적어도 잘됨 |
| 딥러닝 | 1D-CNN, CNN-LSTM/GRU | 원신호 직접 입력, 정확도↑, 데이터·연산 더 필요 |
| 융합 | EMG-CNN + IMU 특징 결합 | 상용 밴드 스타일, 최고 정확도 |

**Python 학습 스택**: `numpy/scipy`(필터), `scikit-learn`(전통 ML) 또는 `pytorch/tensorflow`(딥러닝). OpenBCI를 쓰면 `BrainFlow`로 데이터 스트리밍이 간편.

### 4-3. 학습 코드 골격 (전통 ML 예)

```python
import numpy as np
from scipy.signal import butter, filtfilt, iirnotch
from sklearn.ensemble import RandomForestClassifier
from sklearn.model_selection import train_test_split

FS = 1000  # 샘플링레이트

def preprocess(raw):  # raw: (samples, channels)
    b, a = butter(4, [20, 450], btype='band', fs=FS)
    x = filtfilt(b, a, raw, axis=0)
    bn, an = iirnotch(60, 30, FS)     # 한국 60Hz
    x = filtfilt(bn, an, x, axis=0)
    return x

def features(win):  # win: (win_samples, channels)
    mav = np.mean(np.abs(win), axis=0)
    rms = np.sqrt(np.mean(win**2, axis=0))
    wl  = np.sum(np.abs(np.diff(win, axis=0)), axis=0)
    zc  = np.sum(np.diff(np.sign(win), axis=0) != 0, axis=0)
    return np.concatenate([mav, rms, wl, zc])

# 1) 수집한 (windows, labels) 로부터 특징행렬 X, y 구성
X = np.array([features(preprocess(w)) for w in windows])
y = np.array(labels)

# 2) 학습
Xtr, Xte, ytr, yte = train_test_split(X, y, test_size=0.2, stratify=y)
clf = RandomForestClassifier(n_estimators=200)
clf.fit(Xtr, ytr)
print("acc:", clf.score(Xte, yte))
```

### 4-4. 온디바이스 추론(TinyML)

- PC에서 만족스러우면 모델을 MCU로 옮겨 밴드 단독 동작을 만듭니다.
- **Edge Impulse**(EMG/IMU용 파이프라인·데이터수집·모델변환을 웹에서 제공, 초보에게 매우 추천) 또는 **TensorFlow Lite for Microcontrollers**, 전통 ML은 **micromlgen**으로 C 코드 변환.
- nRF52840/ESP32-S3 정도면 소형 CNN·RF 추론은 실시간(수 ms)으로 충분히 돌아갑니다.

---

## 5. 기기 조작 인터페이스 (④ 단계) — "안경 외 조작"의 핵심

제스처를 분류했으면, 그 라벨을 **표준 프로토콜**로 내보냅니다. 조작 대상별로 방법이 다릅니다.

### 5-1. PC / 스마트폰 조작 → BLE HID (가장 범용)

MCU를 **BLE 마우스/키보드/미디어 컨트롤러**로 위장시키면, 별도 드라이버 없이 Windows·macOS·iOS·Android가 그냥 인식합니다. Apple/Samsung 워치 제스처나 Doublepoint WowMouse가 하는 방식과 동일한 계열.

- 라이브러리: nRF52 계열은 `Adafruit Bluefruit`의 BLE HID, ESP32는 `ESP32-BLE-Keyboard`/`ESP32-BLE-Mouse`.
- 매핑 예:
  - 핀치 → 마우스 좌클릭
  - 주먹 → 미디어 재생/정지
  - 스와이프 좌/우 → 방향키 또는 트랙 이전/다음
  - IMU 기울기 → 마우스 커서 이동(포인터 모드)

```cpp
// ESP32 예시 (개념)
#include <BleMouse.h>
BleMouse bleMouse("MyGestureBand", "DIY", 100);

void loop() {
  int g = classifyGesture();      // ③에서 얻은 제스처 라벨
  if (g == PINCH)      bleMouse.click(MOUSE_LEFT);
  if (g == SWIPE_UP)   bleMouse.move(0, -20);
  if (g == FIST)       { /* 미디어키는 BLE Keyboard 조합 사용 */ }
}
```

### 5-2. 스마트홈(조명·플러그·온도·커튼) → MQTT / Home Assistant / Matter

- **가장 쉬운 길**: ESP32(Wi-Fi)로 **MQTT** 메시지 발행 → **Home Assistant**가 구독해 기기 동작. 제스처 하나를 자동화 트리거로 연결.
  - 예: 핀치 2회 → `home/livingroom/light` 토픽에 `toggle` 발행 → 조명 on/off.
- **표준으로 가려면 Matter**: ESP-Matter SDK로 밴드를 컨트롤러/브리지로 만들면 Matter 호환 기기 전반을 제어(생태계 종속 줄어듦). 난이도는 높음.
- 즉시 실험용으로는 밴드가 BLE로 스마트폰 앱에 제스처를 보내고, **앱이 스마트홈 API를 호출**하는 구조도 간단합니다.

### 5-3. TV·에어컨 등 레거시 가전 → IR

- 밴드(또는 옆의 허브)에 **IR LED**를 달아 적외선 리모컨 신호 송신. 라이브러리 `IRremoteESP8266`.
- 제스처 → 채널/볼륨/전원 IR 코드 매핑.

### 5-4. 커스텀 PC 앱 / 게임 / 로봇

- 밴드는 BLE로 제스처 라벨만 던지고, **PC 쪽 파이썬 브리지**(`bleak`로 BLE 수신 → `pyautogui`로 키/마우스, 또는 게임·로봇 SDK 호출)가 실제 동작 수행. 프로토타이핑이 가장 유연.

### 5-5. 매핑 레이어 설계 팁

- 제스처↔명령을 **코드에 하드코딩하지 말고 설정(config/JSON)으로 분리**하세요. Mudra의 "제스처 매퍼"처럼, 같은 밴드로 여러 기기 프로파일을 전환할 수 있게 됩니다.
- **모드 전환 제스처**(예: 손목 2회 회전 = 프로파일 순환)를 하나 두면 PC모드 ↔ 스마트홈모드 ↔ TV모드 전환이 편합니다.

---

## 6. 단계별 개발 로드맵 (추천 순서)

1. **주차 1 — 출력부터**: XIAO/ESP32로 BLE 마우스 예제를 띄워 버튼 하나로 클릭이 되게. (④가 되는지 먼저 확인)
2. **주차 2 — IMU 제스처(경로 A)**: 온보드 IMU로 스와이프/기울기 감지 → BLE로 커서·미디어 제어. 여기까지면 이미 "안경 없이 기기 조작하는 밴드" 완성.
3. **주차 3~4 — EMG 도입(경로 B)**: MyoWare 1~2채널 추가, RAW→필터→envelope 파이프라인 구축, 오실로스코프/시리얼 플로터로 신호 눈으로 확인.
4. **주차 5~6 — 데이터 수집·학습**: Screen-Guided로 6제스처 수집 → scikit-learn 또는 Edge Impulse로 분류기 학습 → 실시간 추론 붙이기.
5. **주차 7+ — 온디바이스화·매핑·케이스**: TinyML 이식, 프로파일 전환, 3D 프린팅 하우징, 전력 최적화.

---

## 7. 자주 겪는 문제 & 해결

| 증상 | 원인 | 해결 |
|------|------|------|
| 신호가 60Hz로 출렁 | 전원 노이즈 | 노치필터, 배터리 구동, 케이블 짧게/트위스트 |
| 제스처가 세션마다 안 맞음 | 밴드 착용 위치·압력 변동 | 여러 위치서 학습 데이터 수집, 재보정(calibration) 루틴, 착용 마커 |
| rest에서 오작동 | rest 클래스 부족 / 임계값 없음 | rest 데이터 대량 수집 + 확신도(threshold) 미달 시 무동작 |
| 전극 접촉 불량 | 건식전극 압력 부족 | 밴드 타이트하게, 전도성 패브릭/젤, 피부 청결 |
| 지연이 큼 | 창 길이·연산 과다 | 윈도 축소(≤250ms), 경량 모델(RF/작은 CNN) |
| 개별 손가락 구분 안 됨 | 채널 수 부족 | 채널 늘리기(4→8), 전극 배치 최적화, IMU 융합 |

---

## 8. 참고할 만한 오픈소스·자료

- **OpenBCI + BrainFlow** — 연구급 다채널 수집을 소프트웨어로 바로 시작.
- **Edge Impulse** — EMG/IMU 데이터 수집→모델→MCU 배포까지 웹에서 (초보 추천).
- **MyoWare 2.0 공식 가이드/예제** — 아날로그 EMG 배선·코드 튜토리얼.
- **mMyoHMI 등 학술 프로젝트** — MyoWare 4채널 + BLE로 Myo 유사 밴드를 재현한 사례.
- **IRremoteESP8266 / ESP32-BLE-Mouse·Keyboard / ESP-Matter** — ④ 출력부 라이브러리.
- **논문 키워드**: "sEMG hand gesture recognition wrist", "EMG-IMU fusion gesture", "electrode configuration wrist sEMG" (2024~2025 리뷰 논문에 최신 기법·정확도 정리 다수).

---

## 9. 요약

- **안경 종속을 벗어나는 핵심은 ④ 출력부를 BLE HID / MQTT·Matter 같은 표준으로 만드는 것.** 이 한 부분만 범용으로 짜두면 같은 밴드로 PC·폰·조명·TV·로봇 다 조작됩니다.
- **IMU로 MVP를 먼저 완성**하고, EMG는 그 위에 얹으세요. 좌절을 크게 줄여줍니다.
- **데이터 수집과 rest 클래스**가 정확도의 8할입니다. 모델보다 데이터에 시간을 쓰세요.
- 하드웨어 자체가 목적이 아니라면 **OpenBCI나 완제품 8채널 암밴드 + Edge Impulse** 조합으로 소프트웨어에 집중하는 게 가장 빠른 길입니다.

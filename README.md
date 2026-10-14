# yolov8n-fine-tuning

물류 AMR 시뮬레이션 프로젝트(amr-fleet-system)에서 쓰는 학습 모델 가중치를 배포하는 저장소다. 가중치 파일은 저장소가 아니라 **Release 첨부 파일**로 둔다.

YOLOv8n detector fine-tuned for a simulated logistics warehouse (classes: person, cargo, sign). Weights are published as release assets.

## 모델 목록

| 모델 | Release | 파일 | 클래스 | 크기 |
| --- | --- | --- | --- | --- |
| YOLOv8n 3종 v1 | [`yolov8n-amr-v1`](../../releases/tag/yolov8n-amr-v1) | `yolov8n_amr_v1.pt` | `0 person`, `1 cargo`, `2 sign` | 6.2 MB |

## yolov8n_amr_v1

### 용도

- Gazebo(Fortress) 창고 시뮬레이터의 로봇 RGB 카메라(640×480) 영상에서 작업자·화물 상자·표지판을 찾는다.
- ROS 2 검출 노드가 쓰는 2D 검출기다. 3D 위치는 노드가 Depth 영상과 카메라 모델로 따로 계산한다.
- **시뮬레이터 영상으로만 학습·검증했다.** 실제 카메라 영상에서의 성능은 확인하지 않았다.

### 학습

| 항목 | 값 |
| --- | --- |
| 시작 가중치 | Ultralytics `yolov8n.pt` (COCO 사전학습, assets v8.4.0) |
| 데이터 | Gazebo 라벨 카메라로 자동 라벨링한 2,500장 (학습 1,975 / 검증 525). 학습·검증은 찍은 장소(5 m 칸)로 나눴다 |
| 상자 수 | person 563, cargo 2,929, sign 1,169 |
| 입력 크기 | 640 |
| epoch | 100 (patience 20, 마지막 10 epoch는 mosaic 끔) |
| batch | 16 |
| optimizer | AdamW (ultralytics 자동 선택, lr 0.001429) |
| 증강 | ultralytics 기본값 (mosaic 1.0, 좌우 반전 0.5, HSV 0.015/0.7/0.4, 이동 0.1, 크기 0.5) |
| seed | 0 |
| 장비 | Apple M5 (MPS), 1.8시간 |
| 버전 | ultralytics 8.4.164, torch 2.14.1, numpy 1.26.4, opencv-python 4.11 |

### 검증 결과 (검증 525장, 상자 910개)

| 클래스 | 정밀도 | 재현율 | mAP50 | mAP50-95 |
| --- | --- | --- | --- | --- |
| 전체 | 0.981 | 0.907 | 0.931 | 0.870 |
| person | 0.960 | 0.943 | 0.972 | 0.914 |
| cargo | 0.984 | 0.892 | 0.912 | 0.827 |
| sign | 1.000 | 0.887 | 0.908 | 0.870 |

- 클래스끼리 헷갈리는 경우는 없었다. 틀린 것은 대부분 놓침(배경으로 판단)이다: person 5%, cargo 11%, sign 11%.

### 속도 (참고)

| 환경 | 추론 시간 (640 입력) |
| --- | --- |
| Docker 컨테이너 CPU (Apple M5, 스레드 4, 다른 부하 없음) | 약 35 ms |
| 같은 컨테이너에서 Gazebo가 카메라를 CPU로 렌더링하는 중 | 약 120~150 ms |

### 사용

```python
from ultralytics import YOLO

model = YOLO("yolov8n_amr_v1.pt")
results = model.predict("image.jpg", imgsz=640, conf=0.5, iou=0.45)
```

내려받기 (로그인 필요 없음):

```bash
curl -L -o yolov8n_amr_v1.pt \
  https://github.com/ye0n-w0/yolov8n-fine-tuning/releases/download/yolov8n-amr-v1/yolov8n_amr_v1.pt
echo "b623606a497cb0d00b78ff28e938c06d8b7b3d20f13cc3e82237b94a513d53a0  yolov8n_amr_v1.pt" | shasum -a 256 -c
```

### 알려진 한계

- 시뮬레이터 영상 전용이다. 조명·질감이 다른 실제 영상에는 그대로 쓰기 어렵다.
- 한 변이 아주 작은(먼) 물체를 놓치는 경우가 많다. 학습 데이터에 4 px 이상의 상자를 모두 넣었다.
- 표지판은 이 시뮬레이터 창고의 표지판 모델(13개)만 학습했다.

## 라이선스

AGPL-3.0. Ultralytics YOLOv8과 그 사전학습 가중치가 AGPL-3.0이라, 이를 파인튜닝한 이 가중치도 같은 라이선스를 따른다.

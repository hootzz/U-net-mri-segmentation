# 🧠 MRI 기반 뇌종양 Segmentation (문제 해결 과정)

## 프로젝트 개요
- **목표**: 딥러닝으로 MRI 영상에서 뇌종양 영역 자동 분할(Segmentation)
- **데이터**: [Kaggle - LGG Brain MRI Segmentation](https://www.kaggle.com/datasets/mateuszbuda/lgg-mri-segmentation)
- **모델**: U-Net (PyTorch 기반)
- **학습 환경**: Colab 무료 버전
- **손실 함수**: BCEWithLogitsLoss

  ![image](https://github.com/user-attachments/assets/55925c84-29a2-4e50-8de9-4a42c656bdf3)


프로젝트 진행 중 손실 값이 -500까지 떨어지는 현상을 경험했고, 이를 해결하는 과정에서 많은 것을 배우며 문제점과 해결 방법, 그리고 개념을 학습하며 떠오른 질문들을 정리했습니다.

---

## 해결한 문제들

### 1. 손실 함수 값 이상 (-500)

**문제 코드**
```python
criterion = nn.BCEWithLogitsLoss()
# ... 중략 ...
loss = criterion(outputs, masks) 
```

**원인**
BCEWithLogitsLoss는 내부적으로 sigmoid를 적용하기 때문에 logits 값을 입력해야 하는데, 이미 sigmoid를 거친 값이 들어가게 되면서 손실 값이 왜곡된 것이었습니다.

**해결 방법**
```python
# 방법 1: 명시적으로 확인
outputs = model(images)  # raw logits인지 확인
loss = criterion(outputs, masks)

# 방법 2: 다른 손실 함수 사용
criterion = nn.BCELoss()
outputs = torch.sigmoid(model(images))
loss = criterion(outputs, masks)
```

### 2. TPU/GPU 사용량 제한 문제

**문제 상황**
Colab 무료 버전에서 TPU 사용을 시도했으나, 사용량 한계로 인해 학습 진행이 어려웠습니다. GPU도 마찬가지로 사용량 제한에 도달했습니다.

**원인**
```python
import torch_xla  # TPU 사용 시도
import torch_xla.core.xla_model as xm
```

**해결 방법**
- 일부 데이터셋만 사용하여 모델 구조 검증
- 배치 크기를 줄여 메모리 사용량 최적화
- 추후 더 많은 리소스 확보 시 전체 학습 진행 계획

### 3. Segmentation 결과 처리 문제

**문제 코드**
```
pred_mask = model(sample_img).cpu().numpy()  # 그냥 logits 값 사용
```

**원인**
모델 출력값(logits)을 바로 마스크로 사용해, 0과 1 사이의 확률이 아니라 임의의 float 값이 되어 시각화가 제대로 안 된 것이었스빈다.

**해결 방법**
```
pred_mask = (torch.sigmoid(model(sample_img)) > 0.5).float().cpu().numpy()
```

---

# 배운 점 & 떠올린 질문들

이 프로젝트를 진행하면서 여러 개념들을 공부하게 되었고 다양한 의문 또한 생겼습니다.
아래는 프로젝트를 수행하며 배운 점과 떠올린 질문들을 정리한 내용입니다.  

## 1. U-Net

U-Net: 의료 영상 분할에 자주 사용되는 모델

            인코더-디코더 구조에 스킵 커넥션을 추가한 것이 특징
             
### 떠오른 질문들
- **스킵 커넥션이 없으면 모델의 세그멘테이션 경계가 흐려지는 이유는 뭘까?**  
- **스킵 커넥션을 변형해서 특정 패턴(예: 종양 경계)만 더 강조할 수 있나?**  
- **U-Net이 작은 종양을 탐지하기 어려운 경우, 특정 부분을 증폭해서 학습하면 효과가 있을까?**  
- **MRI 데이터는 기본적으로 3D인데, U-Net이 2D 기반이라면 정보 손실이 클까?**  
- **3D U-Net을 사용하면 해결될까? 아니면 2D에서도 3D 정보를 반영할 방법이 있을까?**  

## 2. MRI 데이터

MRI 영상: 픽셀 값이 실제 조직의 물리적 특성을 반영하는 데이터

                픽셀 강도 자체가 조직의 신호 강도를 나타내는 것이 특징

### 떠오른 질문들
- **같은 뇌라도 MRI 촬영 조건이 다르면 픽셀 값이 다르게 나올 텐데, 모델이 그 차이를 어떻게 일반화할 수 있을까?**  
- **픽셀 강도 차이를 단순 정규화로 해결할 수 있을까, 아니면 도메인 적응(Domain Adaptation) 같은 기법이 필요할까?**  
- **MRI에는 T1 강조, T2 강조, FLAIR 같은 촬영 방식이 있는데, 이를 동시에 학습하면 성능이 더 좋아질까?**  
- **각 촬영 방식별로 따로 네트워크를 학습하고, 마지막에 합치는 방식도 가능할까?**  

## 3. 노이즈, 저주파-고주파 분리

의료 영상은 항상 깔끔한 데이터만 있는 것이 아니라, 

노이즈가 많거나 촬영 기계에 따라 품질이 달라질 수 있음

이런 점에서 "AI 모델이 노이즈를 먼저 학습하고, 학습한 걸 바탕으로 데이터의 노이즈를 제거한 뒤 학습하면 더 정확한 결과를 얻을 수 있지 않을까?"라는 의문이 들었습니다.

### 떠오른 질문들
- **일부러 노이즈를 강하게 만들어서, AI가 노이즈 제거를 먼저 학습하게 하면 효과가 있을까?**  
- **Denoising Autoencoder 같은 방식을 적용하면 세그멘테이션 성능이 더 좋아질까?**  
- **고주파 성분(경계, 세부 패턴)과 저주파 성분(큰 구조)을 따로 학습하면 성능이 개선될까?**  
- **저주파-고주파를 따로 학습하는 네트워크를 만들고, 마지막에 합치는 방식이 가능할까?**  
- **이걸 U-Net에 적용하면 경계를 더 뚜렷하게 만들 수 있을까?**  

## 4. 경계선을 따로 학습하면 모델이 더 좋아질까?

U-Net의 한계 중 하나는 **경계선이 불명확할 때 성능이 저하되는 것**

이런 점에서 "경계선만 따로 학습하는 네트워크를 추가하면 성능을 더 개선할 수 있지 않을까? 라는 생각이 떠올랐습니다.

### 떠오른 질문들
- **경계선을 탐지하는 CNN을 병렬로 학습하고, 원본 U-Net의 예측 결과와 합치면 효과가 있을까?**  
- **이걸 실제로 구현해 보면 어떤 차이가 날까?**  
- **이미 존재하는 Boundary-aware U-Net 같은 모델을 적용하면 개선될까?**  

조사를 진행하면서 Dual-Branch U-Net이라는 개념을 접했고,  

이런 방식이 이미 연구되고 있다는 것을 확인했습니다.

여러 개의 U-Net을 합친 모델을 만들 수도 있지 않을까? 라는 생각을 하게 되었습니다.

## 5. 픽셀 단위 vs 객체 단위 분류, 둘울 합칠 수도 있을까?

지금까지는 픽셀 단위로만 종양을 분류하는 방식이었기에 픽셀이 아닌 다른 분류 방식이 있는지 궁금해졌습니다.

### 떠오른 질문
- **픽셀 단위(U-Net)와 객체 단위(Mask R-CNN)를 결합하는 방식이 가능한가?**  


조사를 진행하면서, **Instance Segmentation**이라는 개념이 이미 연구되고 있었다는 것을 알게 되었습니다.

## 6. 이걸 다 합치면 하나의 거대한 모델이 나올 수도 있을까?

이 프로젝트를 진행하면서, 의료 영상 분석에서 사용될 수 있는 다양한 접근법을 접했고,
위의 개선점들을 전부 합친 모델을 실제로 구현 가능한지 궁금했습니다.


---

# 앞으로의 개선 방향

MRI 영상 데이터의 특성과 U-Net 기반 세그멘테이션 모델의 한계를 파악할 수 있었고, 

위 내용을 바탕으로 연구 및 실험을 진행할 때 고려해야 할 개선 방향을 정리했습니다.



### **1️. 컴퓨팅 리소스 확보 및 최적화**  
- **GPU/TPU 환경 확보**를 위한 클라우드 또는 로컬 서버 환경 구축 검토  
- 계산량이 증가하는 **3D U-Net 및 멀티스트림 네트워크 실험을 위한 효율적인 하드웨어 사용 전략 필요**  
- 모델 학습 속도 및 실험 반복성을 높이기 위한 **TPU 최적화 및 mixed-precision training 적용 가능성 탐색**  

### **2️. 모델 아키텍처 개선 및 실험**  
- U-Net의 **스킵 커넥션을 변형하여 특정 패턴 강조 가능성 탐색**  
- **Attention U-Net**과 같은 메커니즘을 추가하여 종양이 있는 영역을 더 정확하게 집중할 수 있는지 실험  
- **경량화된 U-Net 구조 실험**을 통해, 계산 복잡도를 줄이면서도 성능 저하를 최소화할 방법 검토  
- **다중 입력 모델(Multi-Input U-Net) 실험**을 통해, 저주파-고주파 분리, 경계선 정보 등을 동시에 학습하는 효과 분석  

### **3️. 데이터 처리 및 일반화 개선**  
- MRI 영상의 특성을 고려한 **정규화 기법과 도메인 적응(Domain Adaptation) 적용 실험**  
- **다양한 MRI 시퀀스(T1, T2, FLAIR) 조합 실험**을 통해, 최적의 데이터 구성 탐색  
- **데이터 증강(Augmentation) 기법 적용**을 통해, 작은 종양 및 경계선 탐지 성능 향상 가능성 검토  
- **노이즈 제거 기법(Denoising Autoencoder, GAN 기반 Super-Resolution 등) 적용 실험**을 통해 영상 품질 개선 가능성 분석  

---

# 추가적으로 고민해 볼 영역
1. 데이터 증강(Augmentation)
- 데이터 증강으로 일반화 성능 향상
2. Attention U-Net
- 특징을 강조하는 Attention 구조로 모델의 정확성을 높이기
3. Metric 기반 평가
- 추가 메트릭으로 성능 명확히 평가

---
# 다음으로 공부해 볼 영역

MRI Segmentation (뇌 MRI 분할)

### 모델 구조  
- UNet variants (DeepUNet, Attention UNet)  
- 깊이와 skip connection이 어떻게 성능에 영향을 미치는지  

### Loss functions  
- Dice Loss (세그멘테이션 특화)  
- IoU Loss (교집합 기반)  
- Tversky Loss (클래스 불균형 고려 특화)  
- 클래스 불균형 문제  
  - pos_weight 조정 실험  
- Focal Loss (클래스 불균형 다룰 때 사용)  

### 학습 최적화 방법  
- Learning Rate Scheduling (학습률 감소 전략)  
- AdamW Optimizer의 튜닝  
- TPU와 PyTorch/XLA 사용법  
  - xm.optimizer_step의 역할과 barrier의 의미  
  - XLA 텐서 vs CPU 텐서의 호환성 및 변환 과정
  - xm.mark_step의 중요성 (동기화 관리)
    
---

이번 프로젝트를 통해 MRI 영상 데이터의 특성과 모델이 이를 어떻게 해석하는지에 대한 기초를 공부할 수 있었습니다.
앞으로 보다 정밀한 의료 영상 분석 모델을 개발하기 위한 실험과 개선 방향을 추가적으로 검토해 보고 싶습니다.

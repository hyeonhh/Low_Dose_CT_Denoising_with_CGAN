# LDCT-Denoising-CGAN : Deep Learning based Low-dose CT Image Reconstruction
저선량 CT 영상의 노이즈를 제거하여 방사선 노출을 줄이면서도 진단 정확도를 높이기 위한 프로젝트 
## 데이터셋
- 출처 : https://www.kaggle.com/datasets/andrewmvd/ct-low-dose-reconstruction
- 데이터 구성 : Quater Dose(Input) , Full Dose(Target)
- 슬라이스 두께(1mm, 3mm) 및 커널 종류(Sharp, Soft)정보
  - 슬라이스 두께 : 이미지 한 장의 단면이 얼마나 두꺼운지를 나타낸다.
      - 1mm : 해상도가 매우 높고, 세밀한 구조를 보기 좋다. 그러나 노이즈가 매우 심하다.
      - 3mm : 여러 데이터를 평균 내서 합치기 떄문에 영상이 부드럽고, 노이즈가 적다. 그러나 두껍게 겹쳐서 보기 대문에 세밀한 디테일이 뭉개질 수 있다.
   
  - 커널 종류 : 촬영한 원본 데이터를 영상으로 만들 때 사용한 필터 알고리즘
    - sharp kernel : 뼈, 폐 조직 등 경계를 뚜렷하게 보기 위해 사용한다. 경계선을 강조하기 때문에 노이즈도 같이 증폭되어서 영상이 매우 거칠어 보인다.
    - soft kernel : 복부 장기나 근육처럼 부드러운 조직 사이의 미세한 밀도 차이를 보기 위해 사용한다. 노이즈를 억제해서 영상이 부드럽지만 경계선이 약간 흐릿하다.  

## 모델 구조 
- Generator : Unet 기반 
- Discriminator : PatchGAN
- Condition : 슬라이스 두께와 커널 종류 조합 4가지를 cGAN 모델에 조건으로 함께 넣어줌.
- Condition 사용 이유 : 4가지 조합의 이미지 특징이 모두 다르기 때문에, 모든 환경에서 잘 작동하는 모델을 만들기 위함
- Loss function :
- Total Loss:  $L_G = L_{GAN} + \lambda_{L1} L_{L1} + \lambda_{edge} L_{Sobel}$ 
  - L_gan : Discriminator에게 가짜 이미지를 진짜라고 믿게 하기 위해서 실제 CT 영상의 질감과 선명도를 복원하기 위함.
  - L_L1 :생성된 영상과 정답 영상 사이의 픽셀 값 차이를 최소화하여 전체적인 밝기를 보존하기 위함.
  - L_sobel : 
    - 도입 배경 :  L1 Loss의 경우 픽셀 오차를 평균화하고 있어 경계선이 흐릿해지는 현상이 발생함.
    - 역할 : Kornia 라이브러리의 Sobel Filter를 사용하여 이미지의 gradient를 비교함으로써 장기 윤곽선을 유지하도록 함.

## 실험 과정 및 결과
- V1 : GAN Loss + L1 Loss 중심 , 수치적으로는 정답과 유사했지만 경계부분이 많이 흐릿해지는 문제 발생
- V3 : Sobel Edge Loss 도입(가중치 0.01), 윤곽선은 잘 복원했으나 높은 가중치로 인해서 영상 내 노이즈 성분이 과도하게 강조되고, 특정 부위에 아티팩트 발생
- V5 : Sobel Edge Loss 가중치 조정(0.008)을 통해 윤곽선 복원 및 아티팩트 억제
- V6 : V5와 동일한 조건에서 에포크 수 늘려서 학습(10 추가), 다시 노이즈 성분 및 특정 부위에 아티팩트 발생  
- V7 : Sobel Edge Loss 가중치 0.009로 조정 
- V8 : 오차 곡선의 진동 및 인위적인 아티팩트 발생을 해결하기 위해 PatchGAN 도입 
- V9(Final) : V8에서 5 epoch 학습을 추가 진행

**최종 모델은 **PSNR 43.28dB, SSIM 0.9849**의 우수한 수치를 기록했습니다.
**
## 정량적 평가 : 복원된 영상의 품질을 객관적으로 측정하기 위해 두 가지 지표 사용
1. PSNR (Peak Signal-to-Noise Ratio) : Generator가 이미지를 만들면서 발생한 Full dose CT 이미지와의 오차를 계산하기 위해 사용, 수치가 높을수록 Low dose noise가 성공적으로 제거되었음을 의미함
2. SSIM (Structural Similarity Index Measure) : Generator가 생성한 이미지가 Full Dose CT 이미지와 구조적으로 얼마나 닮았는지 측정하기 위해 사용

### 1mm & Sharp
| Metric | V1 |(Final)V9|
|---|---|---|
| **PSNR(Original)** | 35.9679 | 35.9679|
| **PSNR** | 37.9634 |42.5270|
| **SSIM(Original)** | 0.9507  |0.9507 |
| **SSIM** | 0.9616 | 0.9797|


<table border="0">
   <tr>
    <td align="center"><b>Model Version</b></td>
    <td align="center" colspan="3"><b>Comparison (Low Dose | Generated | Full Dose | Difference)</b></td>
  </tr>
  <tr>
    <td align="center"><b>Model V1</b></td>
    <td colspan="3">
     <img width="1000"src="https://github.com/user-attachments/assets/7f49d0b2-4852-440a-b841-073c19c171f0" />
  </tr>
  <tr>
    <td align="center"><b>Model V9</b></td>
    <td colspan="3">
    <img width="1000" src="https://github.com/user-attachments/assets/f29626e8-50c2-442a-848b-4f1234bb0eae" />
  </tr>
</table>



### 1mm & Soft
| Metric | V1 |V9(Final Model|
|---|---|---|
| **PSNR(Original)** |43.3312 | 43.3312 |
| **PSNR** |38.0997 |42.1794|
| **SSIM(Original)** | 0.9738 | 0.9738|
| **SSIM** |  0.9642|0.9834|


<table border="0">
  <tr>
    <td align="center"><b>Model Version</b></td>
    <td align="center" colspan="3"><b>Comparison (Low Dose | Generated | Full Dose | Difference)</b></td>
  </tr>
 <tr>
    <td align="center"><b>Model V1</b></td>
    <td colspan="3">
   <img width="1000" src="https://github.com/user-attachments/assets/f0117b1a-5bbd-4d3a-8504-e40a70c29db1" />
    </td>
 </tr>
  <tr>
    <td align="center"><b>Model V9</b></td>
    <td colspan="3">
   <img width="1000" src="https://github.com/user-attachments/assets/fa150526-2bee-4204-8600-0e95f0764bc0" />
    </td>
 </tr>
</table>




### 3mm & Sharp
| Metric | V1 |  Final Model(V9)|
|---|---|---|
| **PSNR(Original)** | 44.1647 |  44.1647
| **PSNR** |38.2807 |44.5578|
| **SSIM(Original)** |  0.9790| 0.9790|
| **SSIM** | 0.9690 |0.9883  |

<table border="0">
  <tr>
    <td align="center"><b>Model Version</b></td>
    <td align="center" colspan="3"><b>Comparison (Low Dose | Generated | Full Dose | Difference)</b></td>
  </tr>
  
 <tr>
   <td align="center"><b>Model V1</b></td>
    <td colspan="3">
   <img width="1000" src="https://github.com/user-attachments/assets/7318f475-5462-4ecb-a77c-42d31d6e82d9"" />
    </td>
 </tr>
 <tr>
   <td align="center"><b>Model V9</b></td>
    <td colspan="3">
   <img width="1000" src="https://github.com/user-attachments/assets/f660cb99-6aff-4792-9be0-dc67820aea10" />
    </td>
 </tr>
</table>



### 3mm & Soft
| Metric | V1 |Final Model(V9)|
|---|---|---|
| **PSNR(Original)** | 47.5858 |  47.5858 |
| **PSNR** |37.8747 |  43.8558|
| **SSIM(Original)** | 0.9912|  0.9912 |
| **SSIM** |0.9702|0.9885|

<table border="0">
  <tr>
    <td align="center"><b>Model Version</b></td>
    <td align="center" colspan="3"><b>Comparison (Low Dose | Generated | Full Dose | Difference)</b></td>
  </tr>
  
  <tr>
    <td align="center"><b>Model V1</b></td>
    <td colspan="3">
      <img src="https://github.com/user-attachments/assets/d1429da0-e77c-4bcb-ac41-a841accc8b2f" width="1000">
    </td>
  </tr>
  
  
  <tr>
    <td align="center"><b>Model V9</b></td>
    <td colspan="3">
      <img src="https://github.com/user-attachments/assets/beec87e8-7f71-4b3d-b7ba-c51e10fb1043" width="1000">
    </td>
  </tr>
</table>

  
</table>

## 정량적 평가 결과 분석
프로젝트 실험 결과 대부분의 경우에서 BaseLine 대비 모델의 출력물의 PSNR, SSIM 수치가 하락하는 경향을 보였음.
1. **PSNR/SSIM의 수치적 특성**:
   - 저선량 노이즈 CT의 경우 노이즈를 포함하고 있으나, 정답 영상과 픽셀 위치가 일치되어 있어 수치상으로는 높게 측정됨.
   - 모델 생성 이미지의 경우, 노이즈를 제거하고 에지를 강화하는 과정에서 픽셀 재구성이 나타나고, 이 과정에서 수치적인 오차가 증가함.

## 주파수 스펙트럼 결과
- 각 모델버전 별로 주파수 스펙트럼 이미지를 분석한 결과로 Full Dose CT의 주파수 스펙트럼에서 모델이 생성한 이미지의 주파수 스펙트럼의 오차를 시각화한 것으로, V1에 비해 V8, V9에서 노이즈가 옅어지는 경향을 보임, 여전히 고주파 성분에서 오차가 있어 개선이 필요함.

<table border="0">
  <tr>
    <th align="center">Model V1</th>
    <th align="center">Model V8 (Final)</th>
    <th align="center">Model V9</th>
  </tr>

  <tr>
    <td align="center">
      <img src="https://github.com/user-attachments/assets/b8e16fe9-831e-4504-b5d0-f4b55df35169" width="300">
    </td>
    <td align="center">
      <img src="https://github.com/user-attachments/assets/27ab07f7-c939-408b-9000-3c70a500b6bf" width="300">
    </td>
    <td align="center">
      <img src="https://github.com/user-attachments/assets/6924816b-a49f-443b-b1c9-72e0383b4d2b" width="300">
    </td>
  </tr>
  
  <tr>
    <td align="center" colspan="3"><b>Magnitude Spectrum: Target Dose - Generated</b></td>
  </tr>
</table>

## 추가로 개선할 점
1. **고주파 신호 오차 개선** :
   - 고주파 노이즈 제거 및 엣지 보존을 위한 Focal Frequency Loss 검토 후 성능 개선이 필요함.
   - 장기 내부 조직 질감 유지를 위해 Perceptual Loss 도입 검토 필요함.
2. **데이터 로딩 병목 해결을 위한 전처리 로직 최적화** (해결 완료)
   - 데이터셋, 데이터 로더 로딩 시  저장된 path를 통해 image load, preprocess, resize 가 이루어지는 것이 원인
     - 해결: 코드 실행 시간을 통해 train_test_split 과정에 시간이 약 5분 소요됨을 확인했음 -> dataset 자체를 split하지 않고, index로 분할하는 방식 사용 후 300초에서 0.03초로 단축됨.
     - 관련 링크 : https://stackoverflow.com/questions/62968187/why-does-train-test-split-take-a-long-time-to-run





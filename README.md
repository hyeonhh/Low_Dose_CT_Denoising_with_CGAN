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
- Condition : 슬라이스 두께와 커널 종류 조합 4가지를 CGAN모델에 조건으로 함께 넣어줌.
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
- V3 : Sobel Edge Loss 도입(가중치 0.01), 윤곽선은 잘 복원했으나 높은 에지 가중치로 인해서 영상 내 노이즈 성분이 과도하게 강조되고, 특정 부위에 아티팩트 발생
- V5 : Sobel Edge Loss 가중치 조정(0.008)을 통해 윤곽선 복원 및 아티팩트 억제
- V6 : V5와 동일한 조건에서 에포크 수 늘려서 학습(10 추가)


## 정량적 평가 : 복원된 영상의 품질을 객관적으로 측정하기 위해 두 가지 지표 사용
1. PSNR (Peak Signal-to-Noise Ratio) : Generator가 이미지를 만들면서 발생한 Full dose CT 이미지와의 오차를 계산하기 위해 사용, 수치가 높을수록 Low dose noise가 성공적으로 제거되었음을 의미함
2. SSIM (Structural Similarity Index Measure) : Generator가 생성한 이미지가 Full Dose CT 이미지와 구조적으로 얼마나 닮았는지 측정하기 위해 사용

### 1mm & Sharp
| Metric | Model V1 | Model V3 | Model V5 (Final) | Model V6 |
|---|---|---|---|---|
| **PSNR(Quater Dose ,Full Dose)** | 35.9 | 35.9|35.9|35.9
| **PSNR** | 37.3 | 33.12 |**38.97**|  30.34|
| **SSIM(Quater Dose ,Full Dose)** | 0.9507  |  0.9507 | 0.9507 |   0.9507 |
| **SSIM** | 0.953 | 0.94 |**0.96**|  0.88|

<table border="0">
   <tr>
    <td align="center"><b>Model Version</b></td>
    <td align="center" colspan="3"><b>Comparison (Low Dose | Generated | Full Dose | Difference)</b></td>
  </tr>
  <tr>
    <td align="center"><b>Model V1</b></td>
    <td colspan="3">
     <img width="1000"src="https://github.com/user-attachments/assets/9e020ee9-0332-4521-a706-103007d897f0" />
  </tr>
  <tr>
    <td align="center"><b>Model V3</b></td>
    <td colspan="3">
  <img  width="1000" src="https://github.com/user-attachments/assets/238e8161-44ae-442a-b4cd-7807ce251810" />
  </tr>
  <tr>
    <td align="center"><b>Model V5</b></td>
    <td colspan="3">
    <img width="1000" src="https://github.com/user-attachments/assets/a6727b51-754b-4a0c-9f7e-e881158ed20d" />
  </tr>
  <tr>
    <td align="center"><b>Model V6</b></td>
    <td colspan="3">
    <img width="1000" src="https://github.com/user-attachments/assets/5a288510-7624-44ba-be06-751af13110d0" />
  </tr>

  
</table>


1mm & Soft
| Metric | Model V1 | Model V3 | Model V5 (Final) | Model V6 |
|---|---|---|---|---|
| **PSNR(Quater Dose ,Full Dose)** |43.3 | 43.33| 44.16|
| **PSNR** | 38.4 |  29.65 |**39.53**|33.7|
| **SSIM(Quater Dose ,Full Dose)** | 0.97 | 0.97 | 0.97|  0.97|
| **SSIM** | 0.95 | 0.92 |**0.96**|0.901|

<table border="0">
  <tr>
    <td align="center"><b>Model Version</b></td>
    <td align="center" colspan="3"><b>Comparison (Low Dose | Generated | Full Dose | Difference)</b></td>
  </tr>
 <tr>
    <td align="center"><b>Model V1</b></td>
    <td colspan="3">
   <img width="1000" src="https://github.com/user-attachments/assets/1c5c483c-715e-4ca9-86bc-62ef1eff1770" />
    </td>
 </tr>
 <tr>
    <td align="center"><b>Model V3</b></td>
    <td colspan="3">
   <img width="1000" src="https://github.com/user-attachments/assets/e051e76c-74b2-45e8-bea8-f0d8fbc6b698" />
    </td>
 </tr>
 <tr>
    <td align="center"><b>Model V5</b></td>
    <td colspan="3">
   <img width="1000"  src="https://github.com/user-attachments/assets/f69e6216-ef82-4ad3-b6a0-77d5b9e7ca42" />
    </td>
 </tr>
  <tr>
     <td align="center"><b>Model V6</b></td>
    <td colspan="3">
    <img width="1000"  src="https://github.com/user-attachments/assets/cb11e67d-525c-433b-aa11-b08d65ddfe0b" />
    </td>
  </tr>
</table>



3mm & Sharp
| Metric | Model V1 | Model V3 | Model V5 (Final) | Model V6 |
|---|---|---|---|---|
| **PSNR(Quater Dose ,Full Dose)** | 44.16 | 44.16 | 44.16| 44.16
| **PSNR** |32.7 | 32.00 | **33.57** |31.9|
| **SSIM(Quater Dose ,Full Dose)** | 0.97 |0.97|0.97|0.97|
| **SSIM** |0.93| 0.92| **0.94** |0.88|

<table border="0">
  <tr>
    <td align="center"><b>Model Version</b></td>
    <td align="center" colspan="3"><b>Comparison (Low Dose | Generated | Full Dose | Difference)</b></td>
  </tr>
  
 <tr>
   <td align="center"><b>Model V1</b></td>
    <td colspan="3">
   <img width="1000" src="https://github.com/user-attachments/assets/8ee9d468-f7a1-48d9-9de8-abcfe5ebcf92" />
    </td>
 </tr>
 <tr>
    <td align="center"><b>Model V3</b></td>
    <td colspan="3">
   <img width="1000" src="https://github.com/user-attachments/assets/60c0d339-4938-46ca-884d-574982def237" />
    </td>
 </tr>
 <tr>
   <td align="center"><b>Model V5</b></td>
    <td colspan="3">
   <img width="1000" src="https://github.com/user-attachments/assets/9f31bb83-29b1-4a41-9293-8b024a524d64" />
    </td>
 </tr>
 <tr>
      <td align="center"><b>Model V6</b></td>
    <td colspan="3">
   <img width="1000"  src="https://github.com/user-attachments/assets/8f28e519-dd6a-4aa3-bfe7-d44a8dab91d7" />
    </td>
 </tr>
  
</table>



3mm & Soft
| Metric | Model V1 | Model V3 | Model V5 (Final) | Model V6 |
|---|---|---|---|---|
| **PSNR(Quater Dose ,Full Dose)** |47.58 | 47.58|47.58|47.58|
| **PSNR** |33.9 | 32.79| **35.80**|32.70|
| **SSIM(Quater Dose ,Full Dose)** | 0.99| 0.99 |0.99|  0.99 
| **SSIM** |0.93| 0.91 |**0.95**|0.88|

<table border="0">
  <tr>
    <td align="center"><b>Model Version</b></td>
    <td align="center" colspan="3"><b>Comparison (Low Dose | Generated | Full Dose | Difference)</b></td>
  </tr>
  
  <tr>
    <td align="center"><b>Model V1</b></td>
    <td colspan="3">
      <img src="https://github.com/user-attachments/assets/d5b421f9-489b-40bb-ba2f-377bbbf7dae9" width="1000">
    </td>
  </tr>
  
  <tr>
    <td align="center"><b>Model V3</b></td>
    <td colspan="3">
      <img src="https://github.com/user-attachments/assets/9f9c4393-d482-4301-848e-089d85ebb848" width="1000">
    </td>
  </tr>
  
  <tr>
    <td align="center"><b>Model V5</b></td>
    <td colspan="3">
      <img src="https://github.com/user-attachments/assets/c9d25a95-81d1-4025-a35f-cddd8e276f3d" width="1000">
    </td>
  </tr>
  
  <tr>
    <td align="center"><b>Model V6</b></td>
    <td colspan="3">
      <img src="https://github.com/user-attachments/assets/87957aa8-d877-4560-b8b3-7f18efa42667" width="1000">
    </td>
  </tr>
</table>
  
</table>

## 추가로 개선할 점
1. **환경별 성능 편차 (3mm & Sharp)**:
   - 물리적으로 정보가 소실된 3mm 슬라이스와 인위적으로 에지를 강조한 Sharp 커널의 조합에서 복원 난이도가 가장 높게 나타남. 이는 입력 데이터 자체의 낮은 신호 대 잡음비(SNR)와 구조적 정보 누락에 기인함.
2. **데이터 로딩 병목 해결을 위한 전처리 로직 최적화**
   - 데이터셋, 데이터 로더 로딩 시  저장된 path를 통해 image load, preprocess, resize 가 이루어지는 것이 원인
3. **병변과 노이즈의 식별 한계 (Clinical Validity)**:
   - 

구글 슬라이드 : https://docs.google.com/presentation/d/1iZF5HvIKIUDfgN91kqR2zuEef4898Ti90Ad7n9allpg/edit?usp=sharing





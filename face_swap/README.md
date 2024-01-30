# Faceswap GAN
face swapping은 source이미지와 Target이미지 두개를 입력으로 받아, 마치 target이미지 속 인물의 얼굴이 source이미지 속 인물의 얼굴로 대체된 것 같아 보이는 얼굴 변환 이미지를 생성하는 것을 목표로 합니다.<br>
<br>
다시말해, source이미지로부터 identity(눈, 코, 입 등의 고유한 생김새) 정보를 추출하고 target이미지로부터 pose(고개 각도, 표정 등의 포즈)와 attribute(조명 컨디션, 메이크업 등의 색감과 관련된 장면 특징) 정보를 추출 후 조합해 원하는 정보(source identity, target pose & attributes)만을 가지는 한장의 이미지를 생성합니다.<br>
<br>
Faceswap의 파이프 라인은 크게 Extraction, Training, Conversion 3개의 파이프라인으로 구성됩니다.
이미지 데이터는 source와 destination이 있습니다.<br>
<br>
먼저 가장 처음인 Extraction을 살펴보도록 하겠습니다.<br>
## 1.Extraction
![image](https://github.com/wonicom/Generative_model/assets/123945441/6557c66b-9319-4292-ae49-ce0d20673011)
Extraction과정에서는 src와 dst 데이터에서 얼굴을 추출하는 것을 목표로 합니다. 

### 1.1 Face Detection
입력 데이터로부터 target face를 찾는 과정, 여러 크기의 얼굴을 감지할 수 있는 S3FD의 default face detector를 사용합니다.<br>
<br>
### 1.2 Face Alignment
두 번째 단계는 얼굴 정렬입니다. 연속된 영상 촬영과 영화 제작에 필수적이며 효과적인 얼굴 랜드마크 아록리즘을 찾아야 합니다.<br>
이를 해결하기 위해 (a)heatmap-baseed facial landmark algorithm 2DFAN (b)PRNet을 사용합니다. 얼굴의 랜드마크를 검색한 후, 단일 촬영의 연속된 프레임에서 얼굴 랜드마크를 부드럽게 처리하는 설정 가능한 시간 간격으로 선택적 기능도 제공합니다. 이를 통해 Front/Side View에 대한 facial landmark template를 만듭니다.<br>
<br>
### 1.3 Face Segmentation
Face Alignment이후 머리카락, 손가락, 안경 등 정확한 segmentation이 수행되도록 fine-grained Face Segmentation network를 사용합니다. 이를 통해 얼굴을 덮은 장애물을 제거합니다.<br>
<br>
## 2. Training
Training단계는 Face swapping에 가장 중요한 역할을 가지고 있습니다. Source와 destination의 pair를 일치시키기 위한 2가지 structure(DF, LIAE)가 사용됩니다.

### 2.1 DF Structure
![image](https://github.com/wonicom/Generative_model/assets/123945441/3e4c51ed-7cb6-49e8-94c1-9ab6c593df1a)
DF Structure는 encoder와 src와 dst사이에 공유된 가중치를 가진 Inter, Src와 dst에 별도로 속하는 두 개의 decoder로 구성됩니다. 공유된 encoder와 inter를 통해 src와 dst의 일반화가 진행되며, 이는 페어링되지 않은 문제를 쉽게 해결합니다. 하지만, DF Structure는 조명과 같은 충분한 정보를 dst로부터 상속할 수 없습니다. 

### LIAE Structure
![image](https://github.com/wonicom/Generative_model/assets/123945441/14231427-8466-445a-aa30-53dc99eb1155)
LIAE structure은 이러한 빛에 대한 일관성 문제를 강화합니다. Inter AB는 src와 dst의 latent code를 만들고 InterB는 dst의 latnet code만을 만듭니다. Inter AB에서 나온 src에 대한 latent code 두개를 concatenate하고 InterAB에서 나온 dst에 대한 latent code와 interB에서 나온 dst에 대한 latent code를 concatenate합니다. InterAB를 통해 well-aligned된 결과의 class를 dst로 방향을 바꾸기 위해 두 layer에서 추출된 dst에 대한 latent code를 concatenate합니다. 이후, decoder에 들어가고 mask와 함께 예측된 src(dst)를 얻습니다. Generalization과 clarity를 위해 DSSIM와 MSE loss를 혼합한 mixed loss를 사용합니다.

## 3.Conversion
![image](https://github.com/wonicom/Generative_model/assets/123945441/f599531a-4217-4ed1-92de-b15d9a9b2a22)
Conversion과정에서는 src를 dst로 변환하며 출력을 더 매끄럽게 만들어줍니다. 첫 번째, dst decoder를 통해 src의 원래 위치로 마스크와 함께 생성된 얼굴로 변환합니다. 다음으로는 블렌딩에 관한 부분인데, 다시 정렬된 재연 얼굴이 대상 이미지의 외곽 윤곽과 완벽하게 어울리도록 하는 것이 목표입니다. 일관된 피부톤을 유지하기 위해, dfl은 재연된 얼굴의 색상을 대상에 맞추기 위해 다섯 가지의 색상 전송 알고리즘을 제공합니다. 마지막으로 날카로움이 필수적입니다. 거의 대부분의 최신 얼굴 교체 작업에서 생성된 얽굴은 어느 정도 부드럽고 작은 세부 사항이 부족하다는 것을 고려하여, 블렌딩된 얼굴을 날카롭게 만들기 위해 사전 훈련된 얼굴 초고해상도 신경망이 추가되었습니다.

## 실험 과정
아래는 직접 훈련을 진행한 과정을 소개해드리도록 하겠습니다.<br>
dst - https://github.com/wonicom/project1128/assets/123945441/664a64de-3982-4919-a5ee-d9c53dff3a7b<br>
src - https://github.com/wonicom/project1128/assets/123945441/1a51c9c2-2ecf-4bd6-a589-6d21778a71a7<br>
해당 2개의 영상들을 input으로 사용합니다.<br>

![KakaoTalk_20240124_023310812](https://github.com/wonicom/project1128/assets/123945441/b32189c2-7d95-4883-8326-850cbf14b98e)

### output

https://github.com/wonicom/project1128/assets/123945441/4888a2ac-5df2-48f5-a53a-eeabcde838aa

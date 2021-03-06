---
description: YOLO (You Only Look Once)
---

# YOLO

 2016년 CVPR에 발표 된 You Only Look Once : Unified, Real-Time Object Detection.

![](../.gitbook/assets/image%20%28268%29.png)

 이미지 내의 bounding box와 classification을 single regression problem으로 간주하여, 이미지를 한번 보는 것으로 object의 종류와 위치를 추측한다. single convolutional network를 통해 multiple bounding box에 대한 class probability를 계산하는 방식이다.  
 \(1\) Resize input image \(448 x 488\)  
 \(2\) Run single convolutional network on the image  
 \(3\) Thresholds the resulting detections by the model's confidence  
  
기존의 R-CNN은 모델들은 Bounding box를 먼저 찾고 각 Bbox에 대하여 classification을 하는 방식이였다. Region Proposal 수도 많고 각각의 proposal 에대해 또 NN을 돌려줘야 했기 때문에 오버헤드가 컸다. 하지만 Yolo는 모든 class에 대한 모든 BBox를 동시에 예측하기 때문에 매우 빠르고 global reasoning이 가능하다. 단 하나의 네트워크가 한번에 특징도 추출하고, bounding box도 만들고, class도 분류하니 당연히 빠를 수 밖에 없다.

![](../.gitbook/assets/image%20%28153%29.png)

 이미지를 네트워크의 input으로 넣어주면 해당 이미지를 S x S 로 grid 하여 연산하고 Bounding boxes + confidence와 Class probability map 이 ouput으로 나오게 된다.

* Bounding boxes + confidence : 어떤 Object가 있을 것 같다고 확신이 들 수록 \(confidence score가 높을수록\) 박스를 굵게 그려줌. 나중에 NMS를 이용해 다수 지워지게 된다.
* Class probability map : 각 grid cell은 해당 영역에서 제안한 Bounding box 안의 object가 어떤 class 인지 컬러로 표현.

input : 448 x 448 / ouput : S x S x \(B \* 5 + C\)   
  - S : grid size / B : cell 마다 테스트 되는 bounding box 갯수 / C : dataset의 class 갯수  
  
\(1\) 이미지는 S x S \(default S = 7\) grid 로 나뉜다.  
\(2\) 각 cell 마다 Bounding box \(default B = 2\)를 테스트한다.  
\(3\) 각 grid cell은 예측한 B개의 Bounding box \(x, y, width, height\)와 각 bounding box 에 대한 confidence score를 갖는다.  
   - confidence score : grid cell에 물체가 없으면 0, 있으면 예측된 BBox와 Ground-truth BBox 사이의 IoU 값  
                                  : Pr\(Object\) \* IoU\(truthpred\)  
\(4\) 각각의 grid cell은 C개의 conditional class probability를 갖는다. \( Pr\(Classi \| Object\) \)

![](../.gitbook/assets/image%20%28207%29.png)

 모델은 다음과 같다. GoogLeNet을 변형한 구조로, 24개의 conv layer와  2개의 fully connected layer로 구성되어 있다. 1x1 reduction layer를 여러번 적용한 것이 특징이다. Fast Yolo는 9개의 conv layer와 더 적은 수의 filter를 사용하여 속도를 더 높였다. \(training/testing을 위한 파라미터는 동일\)

![](../.gitbook/assets/image%20%28289%29.png)

 \* Training  
Pre-training : 먼저 ImageNet으로 pre-training하고 다음에 Pascal VOC에서 fine-tuning 하는 2단계 방식으로 진행  
Training할 때에는 앞의 20개의 conv layer만 사용하고 \(feature extractor\), 실제 object detection 할 때는 마지막 4개의 conv layer와 2개의 fc layer를 추가하여 \(object classifier\) 시스템을 구성했다.  
마지막의 layer는 class probabilities와 bounding box coordinates 두가지를 예측한다.   
 - Bounding box의 w, h는 이미지의 width, height로 normalize \(0~1\)  
 - Bounding box의 x, y는 특정 grid cell 위치의 offset 값 사용 \(0~1\)  
Activation 함수로는 leak Relu 사용.  
불필요한 BBox들을 제거하기 위해 Non-maximal suppression이 사용 된다.   
  
마지막 output으로 나오는 7x7x30이 바로 예측된 결과로, 이 안에 bbox와 class 정보 등 모든 것이 들어있다.  
자세히 보면 아래와 같다.

![](../.gitbook/assets/image%20%28383%29.png)

 S = 7 이라고 했을 때, 49개의 Grid cell이 만들어지고, 각각의 grid cell은 B개\(B=2로 가정\)의 bounding box를 가지고 있다.  
앞 5개의 값은 해당 Grid cell의 첫번째 Bounding box 에 대한 값이 채워지고, 뒤의 5개는 두밴째 Bounding box에 대한 값이 채워진다.  
 - x, y : coordinate of bbox center inside cell \(grid cell로 부터 어디에 위치해 있는지 그 offset\)  
   ex\) 위 그림의 경우, x,y = 0.5 정도  
 - w, h : bbox width, height \(Image size에 맞춰 normalize 된 값\)  
  ex\) 위 그림의 경우, w = 3/7, h = 2/7 정도  
 - c : bbox confidence   
예측되는 Bounding box는 grid cell을 중심으로 하는 RoI다. grid cell 보다 작을 수도 있고, 클 수도 있다. 

![](../.gitbook/assets/image%20%28237%29.png)

 뒤의 20개에는 해당 grid cell에 object가 있을 경우, 그것이 어떤 class인지에 대한 확률이 저장되어 있다.  
즉, 20개의 class에 대한 conditional class probability에 해당된다.

![](../.gitbook/assets/image%20%28379%29.png)

 첫번째 Bounding box의 confidence score와 각 conditional class probability를 곱하면 해당 Bbox의 class specific confidence score가 나온다. 두번째 Bounding box도 마찬가지로 계산하여 class specific confidence score가 나온다. Bbox의 confidence가 0에 가까울 수록, 그 위치에 어떤 class가 있는 지에 대한 정보도 매우 낮아지게 된다.

![](../.gitbook/assets/image%20%286%29.png)

 이 계산을 각 bounding box에 대해 수행하게 되면 총 7\*7\*2 = 98 개의 class specific confidence score를 얻을 수 있다.  
이 98개의 class specific confidence score에 대해 각 20개의 class를 기준으로 non-maximum suppression을 하여, Object에 대한 Class 및 bounding box location을 결정한다.

![](../.gitbook/assets/image%20%28372%29.png)

 confidence score가 threshold 보다 작은 경우\(&lt; 0.2\), 해당 클래스는 0으로 세팅함으로써 절대 이 class는 아닐 거라고 필터링을 하고, 이 이후에 NMS 과정을 거치며 중복되는 Bounding box를 제거한다.  
 - NMS\(Non-maximal suppression\) : 여러 Bounding box가 겹쳐 있을 때 그 중에서 최대값을 갖는 하나의 Object만 빼우고 나머지를 지운다. 반대로 여러 Bounding box가 겹쳐 있지 않을 때는 서로 다른 Object의 Bbox라고 생각하고 그대로 둔다.  
  
20개의 score 중 최대값을 갖는 class가 해당 Bounding box의 예측된 class 결과이다. 가장 큰 값이 0인 경우는 Object가 없는 경우. 하나의 grid cell에서 class가 같은 2개의 object가 나올 수 없는 구조이다.

![](../.gitbook/assets/image%20%2851%29.png)

 Loss Function으로는 기본적으로 sum-of-squared-error 개념이지만, object가 존재하는 grid cell과 object가 존재하지 않는 grid cell 각각에 대하여 coordinates, confidence score, conditional class probability의 loss를 계산한다.  
  
위에서부터 한줄한줄 보면,  
\(1\) Object가 존재하는 grid cell i의 predictor bounding box j 에 대해 x, y의 loss 계산  
     - 실제로 Object가 grid cell i 에 있을 때, 예측한 bounding box j 가 정답과 같아지도록 학습  
\(2\) Object가 존재하는 grid cell i의 predictor bounding box j 에 대해 w, h의 loss 계산  
     큰 box에 대해서는 small deviation을 반영하기 위해, 제곱근을 취한 후 sum-squared error 한다.  
     \(같은 error라도 larger box의 경우 상대적으로 IoU에 영향을 적게 준다.\)  
     - 큰 Bounding box에서는 w, h를 조금만 키워도 넓이가 확 증가해서 미분값이 크지만, 작은 bounding box에서는 w, h를 많이 키워도 넓이의 변화가 크지 않기 때문에, sum-squared error를 사용해서 box의 크기에 따른 미분 차이를 줄여준다.  
\(3\) Object가 존재하는 grid cell i의 predictor bounding box j 에 대해 confidence score의 loss 계산 \(Ci= 1\)  
     - 해당 Bounding box에 대하여 이 Object가 어떤 class 인지 예측  
\(4\) Object가 존재하지 않는 grid cell i의 bounding box j 에 대해 confidence score의 loss 계산 \(Ci = 0\)  
     - 이 loss에 곱해지는 lambda값이 \(3\) loss 보다 작다. Object가 없는 cell에 대한 class 분류 정확도는 좀 낮아도 괜찮다는 뜻. 어차피 background니까.  
\(5\) Object가 존재하는 grid cell i에 대해 conditional class probability의 loss 계산 \(맞는 class일 경우 pi\(c\) = 1, 아니면 pi\(c\) = 0\)  
     - 각 Grid cell i 에서 Bounding box가 B개 나오지만, class 확률 값\(c\)은 공유하기 때문에 B에 대한 Sum은 없다.  
  
lambda\(coord\) : coordinates\(x, y, w, h\)에 대한 loss와 다른 loss들 과의 균형을 위한 balancing parameter  
lambda\(noobj\) : Object가 있는 box와 없는 box간의 균형을 위한 balancing parameter   
 - grid cell의 대부분은 object가 없다. 각 grid cell은 object가 있을 것 같다고 예측하는 confidence score를 만들어야 하는데, object가 거의 없기 때문에 score가 0에 가까워진다. 그래서 object가 있는 경우의 loss는 높이고, 없는 경우의 loss는 줄인다.  
lambda\(coord\) = 5, lambda\(noobj\) = 0.5로 설정하면, object가 있는 경우에 학습을 더 제대로 하게 된다.

![](../.gitbook/assets/image%20%28346%29.png)

 Fast R-CNN의 error는 background  영역에서 대부분 발생하지만, Yolo는 localization 에서 대부분의 error가 발생한다. 이 두 모델의 단점을 보완하기 위해 함께 조합하여 사용함으로써 정확도를 높이고 있다.

![](../.gitbook/assets/image%20%28117%29.png)

 훈련시킨 dataset이 실제 테스트 할 환경의 데이터와 다를 수 있다. 다른 distribution을 가지는 환경에서 모델이 얼머나 잘 일반화하여 학습을 했는지 확인하기 위해, 논문에서는 artwork dataset\(Picasso, People-Art\)에 test를 했다.  
다른 detection 시스템에 비해 높은 AP값과 다양한 Bbox를 검출한 것을 확인할 수 있었다. Yolo는 Object에 대한 좀 더 일반적인 특징을 학습하기 때문.  
  
\* 장점  
 - 간단한 처리과정으로 속도가 매우 빠르다. 기존의 다른 detection 시스템과 비교했을 때 2배 정도 높은 mAP를 보인다.  
 - Image 전체를 한번에 바라보는 방식으로, class에 대한 맥락적 이해도가 높기 때문에 Background Error\(False-Positive\)가 낮다.  
 - Object에 대한 좀 더 일반적인 특징을 학습한다. natural image로 학습하고 artwork에 테스트 했을 때, 다른 detection 시스템에 비해 훨씬 높은 성능을 보인다.  
  
\* 단점  
 - grid cell은 2개 까지의 Bbox와 1의 class만을 예측하기 때문에, 새 떼와 같이 작은 object들의 그룹이나 특이한 종횡비의 Bbox는 잘 검출하지 못한다.   
 - bounding box 의 형태가 training data를 통해서만 학습되기 때문에, 새로운 형태의 bounding box의 경우 정확히 예측하지 못한다.  
 - 몇 단계의 layer를 거쳐서 나온 feature map을 대상으로 bounding box를 예측하므로 localization이 다소 부정확해지는 경우가 있다  
  
이 이후에 Yolo v2가 2016년 12월에 나와서 속도와 정확도가 대폭 향상되어서, SSD 보다 뛰어나다고 한다.


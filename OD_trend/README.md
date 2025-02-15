# Object Detection Overview

## Object Detection
### Definition
> Object detection is a computer vision technique that allows us to identify and locate objects in an image or video. 
> With this kind of identification and localization, object detection can be used to count objects in a scene
> and determine and track their precise locations, all while accurately labeling them.

### Difference from Classification
> Classification task looks for the 'class' of the target scene. The model can only output a **single** class name from a single image.
> However, object detection can find the location of the objects along with the class name of them. Besides, object detector can find **multiple** object's locations and class
> from a single image (multi object detection)

## Overall process of Object Detection
- single stage detector vs two stage detector
<img width="690" alt="image" src="https://user-images.githubusercontent.com/32179857/176093460-0edca585-a9bb-4d08-8b86-210878fa8019.png">

(Image from https://stackoverflow.com/questions/65942471/one-stage-vs-two-stage-object-detection)

## Terms
#### RoI
- Region of Interest
<img width="600" alt="image" src="https://user-images.githubusercontent.com/32179857/176093586-32bc6bb9-44df-46bf-baf0-5916a254e0b3.png">

(Image from https://kr.mathworks.com/help/visionhdl/ref/visionhdl.roiselector-system-object.html)

#### IoU
- Intersection over Union
<img width="541" alt="image" src="https://user-images.githubusercontent.com/32179857/176093902-b7915d25-59a4-4695-bb96-d92f7602dc8b.png">

(Image from https://www.researchgate.net/figure/Intersection-over-Union-IOU-calculation-diagram_fig2_335876570) 

#### NMS
- Non maximum suppression
```python
   import torch
   from IoU import intersection_over_union

   def nms(bboxes, iou_threshold, threshold, box_format='corners'):
       # bboxes가 list인지 확인합니다.
       assert type(bboxes) == list

       # box 점수가 threshold보다 높은 것을 선별합니다.
       # box shape는 [class, score, x1, y1, x2, y2] 입니다.
       bboxes = [box for box in bboxes if box[1] > threshold]
       # 점수 오름차순으로 정렬합니다.
       bboxes = sorted(bboxes, key=lambda x: x[1], reverse=True)
       bboxes_after_nmn = []

       # bboxes가 모두 제거될때 까지 반복합니다.
       while bboxes:
           # 0번째 index가 가장 높은 점수를 갖고있는 box입니다. 이것을 선택하고 bboxes에서 제거합니다.
           chosen_box = bboxes.pop(0)

           # box가 선택된 box와의 iou가 임계치보다 낮거나
           # class가 다르다면 bboxes에 남기고, 그 이외는 다 없앱니다.
           bboxes = [box for box in bboxes if box[0] != chosen_box[0] \
                  or intersection_over_union(torch.tensor(chosen_box[2:]),
                                             torch.tensor(box[2:]),
                                             box_format=box_format)
                       < iou_threshold]

           # 선택된 박스를 추가합니다.
           bboxes_after_nmn.append(chosen_box)

       return bboxes_after_nmn
```


#### mAP
- Mean Average Precision
- TP, FP, TN, FN

    <img width="680" alt="image" src="https://user-images.githubusercontent.com/32179857/176380654-d8c7b29c-539b-4c64-895b-c26dd7b9005f.png">

- Suppose we detect 10 apples for the image that has 5 apples.
- We write the result in the order of confidence scoer (highly confident)
<img width="632" alt="image" src="https://user-images.githubusercontent.com/32179857/176098792-085bb2f3-56a9-4e46-88d8-e10a6a9f8ae0.png">

- AP (Average Precision)
<img width="629" alt="image" src="https://user-images.githubusercontent.com/32179857/176098840-bbd6cad3-eab8-40c4-80d8-05011de64169.png">
<img width="629" alt="image" src="https://user-images.githubusercontent.com/32179857/176098994-40ef704f-5557-4ac6-b192-de20f652e091.png">

- mAP: AP over all classes
- Precision-Recall curve
    https://ardentdays.tistory.com/20

#### Anchors
- anchors are required to get candidate bounding box for the model to inference.
- Anchor boxes
    - fixed anchor for classes
    <img width="317" alt="image" src="https://user-images.githubusercontent.com/32179857/177083179-3ef0b6e0-a0df-48a3-959c-5a7b87481165.png">

    - anchors by clustering (unsupervised)
    <img width="467" alt="image" src="https://user-images.githubusercontent.com/32179857/177083224-c2870d96-295b-4663-a4e1-0c8886159d8f.png">

    - anchor free
    <img width="737" alt="image" src="https://user-images.githubusercontent.com/32179857/177099849-43308ea8-d837-4ff9-a4fa-1fc9cf417f2e.png">


## 1-Stage Detectors
- backbone + detector head

 > We'll refer to this part of the architecture as the "backbone" network, which is usually pre-trained as an image classifier to more cheaply learn how to extract features from an image. This is a result of the fact that data for image classification is easier (and thus cheaper) to label as it only requires a single label as opposed to defining bounding box annotations for each image. Thus, we can train on a very large labeled dataset (such as ImageNet) in order to learn good feature representations.
 
### Process
1. pretrain classification backbone with large dataset

<img width="1018" alt="image" src="https://user-images.githubusercontent.com/32179857/177067649-b29f502b-d95b-4247-b730-8d2a9957d39e.png">

2. remove last few layer so that our backbone network outputs a collection of stacked feature maps (low spatial resolution albeit a high feature (channel) resolution.)

<img width="1029" alt="image" src="https://user-images.githubusercontent.com/32179857/177067743-76fd3250-2710-4134-b94d-69226fdc7a99.png">

3. relate this 7x7 grid back to the original input in order to understand what each grid cell represents relative to the original image.

<img width="1029" alt="image" src="https://user-images.githubusercontent.com/32179857/177067673-1d03ec4e-4971-432f-a996-11e6fd618918.png">

4. determine roughly where objects are located in the coarse (7x7) feature maps by observing which grid cell contains the center of our bounding box annotation.

<img width="825" alt="image" src="https://user-images.githubusercontent.com/32179857/177067985-54bf2723-53df-44a9-8205-50bbb70f5b86.png">

5. In order to detect this object, we will add another convolutional layer and learn the kernel parameters which combine the context of all 512 feature maps in order to produce an activation corresponding with the grid cell which contains our object.

<img width="995" alt="image" src="https://user-images.githubusercontent.com/32179857/177068064-ce8d064c-c878-46ee-bc26-283835bf5b59.png">


### Yolo (You Only Look at Once)
- set initial anchor boxes by k-means clustering
- Rather than directly predicting the bounding box dimensions, we'll reformulate our task in order to simply predict the offset from our bounding box prior dimensions such that we can fine-tune our predicted bounding box dimensions.
- Only detects a single class from a single grid
- No multi scale, so weak at detecting small objects.

  <img width="256" alt="image" src="https://user-images.githubusercontent.com/32179857/177100671-9aae3f63-b366-4872-9056-032d5a0634eb.png">
  
- using 'objectnesss score' ==> set 1.0 to highest IoU scored box, 0 for the rests.
- output:   objectness + class + bounding box coordinates.
- process
```
1. divide into grids
2. predict bounding boxes for each grid
3. do regression
```

<img width="852" alt="image" src="https://user-images.githubusercontent.com/32179857/177107563-48af1876-10d6-4d7f-ba3e-a06cb3d7fc37.png">


### SSD (Single Shot Detector)
- VGG16 model is used for pretrained classifier backbone.
- bounding box
- by removing FC layer from YOLO,
    - no need to fix the input size
    - much smaller # of parameters

> Rather than using k-means clustering to discover aspect ratios, the SSD model manually defines a collection of aspect ratios (eg. {1, 2, 3, 1/2, 1/3}) to use for the B bounding boxes at each grid cell location.
- output:   class + bounding box coordinates.
- process
```
1. multiple conv layers for multiple scaled images.
2. get features from multiple layers
3. do nms for layers to get final bounding boxes
```

<img width="852" alt="image" src="https://user-images.githubusercontent.com/32179857/177107439-b4a39aee-595c-493f-9219-63e3d0992d64.png">

- multi scale sample

<img width="487" alt="image" src="https://user-images.githubusercontent.com/32179857/177226104-f07e1147-0b27-4c32-a82c-958dadb6d7f0.png">



## 2-Stage Detectors
- backbone + rpn neck + detector headd
#### Faster RCNN 
<img width="342" alt="image" src="https://user-images.githubusercontent.com/32179857/177226189-acd5ec86-e043-4d53-a59b-80e4578e24ee.png">

##### Process
1) extract feature map by inputting image to pretrained CNN model.
2) get region proposal by passing feature map to RPN model.
3) region proposals + feature map --> RoI pooling --> get fixed size feature map. 
4) input feature map to CNN model to perform bounding box regression and classification

##### Anchor boxes
- Faster RCNN uses 9 anchor boxes
 <img width="694" alt="image" src="https://user-images.githubusercontent.com/32179857/177227316-a98ec226-e2af-40e5-8e2c-b3614498e5c8.png">

 <img width="585" alt="image" src="https://user-images.githubusercontent.com/32179857/177227424-a5ba37a5-001a-4e97-860c-552af1b76ed6.png">

#### RPN
  <img width="737" alt="image" src="https://user-images.githubusercontent.com/32179857/177227458-7cd922e5-4025-4d09-9531-c42aa20827a5.png">

#### Loss
  <img width="642" alt="image" src="https://user-images.githubusercontent.com/32179857/177227798-b5955760-e45c-4b4b-81e8-9b678487dd1e.png">


## Backbone, Neck, Head
1. Backbone
  - feature extractor
2. Neck
  - FPN
3. Head
  - Detector Head
  
<img width="764" alt="image" src="https://user-images.githubusercontent.com/32179857/177227895-e8890d1b-5467-43b8-811e-ebc0e5fe6b90.png">


### References
- https://deepsense.io/region-of-interest-pooling-explained/
- https://www.jeremyjordan.me/object-detection-one-stage/ 
- https://nuguziii.github.io/survey/S-001/
- https://herbwood.tistory.com/10
- https://velog.io/@hewas1230/ObjectDetection-Architecture

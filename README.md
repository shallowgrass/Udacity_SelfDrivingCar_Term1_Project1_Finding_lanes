# **Udacity SDCND term1 project1: Finding Lane Lines on the Road** 

## 1. Overall strategy

---

The lane detection pipeline contains the following steps:

(1) Edge detection:
    - Smooth source color image
    - Use Otsu method to suggest high threshold for Canny edge detection
    - Use Canny edge detection to extract edges
    - Use RoI based on image shape to mask out non-road edges
(2) Line detection:
    - Use Hough Transform to detect line segments
(3) Lane proposals:
    - Propose left or right lane candidates using sign of line segments slope
    - Clean candidates by slope value and central line of image
    - Use Normal Equation solution to make fusion of line slope
    - Use Exponential Moving Average to propose robust slope value and lane key points
(4) Line overlay:
     - Draw complete lane passing through the key point with proposed slope value


[//]: # (Image References)
[src_img]:./report_src/challenge.png "One sample image"
[img_edges]:./report_src/img_edges.png "All edges detected"
[masked_edges]:./report_src/masked_edges.png "Edges inside RoI"
[line_overlay]:./report_src/line_overlay.png "Chosen lines for lane detection (yellow)"
[lane_detection]:./report_src/lane_detection.png "Source image overlayed with detected lanes"

---

## 2. Pipeline steps explanation

### 2.1 Edge detection

The edges of image are detected using Canny method. Otsu thresholding is applied to propose the high threshold for Canny so that the detection becomes image adaptive. Smoothing approach can use Gaussian blurring or bilateral filter (if edge protection is required). 

![alt text][src_img]
![alt text][img_edges]


### 2.2 Line detection
Apply Hough Transform for line detectioin, and extract edges within road-related region. The RoI is defined in proption to source image shape.
![alt text][masked_edges]


### 2.3 Lane proposals

#### Separate left and right left or right lane candidates using sign of line slope
Compute line slope for each Hough line proposal, and separate them into two groups: negative slope suggesting line segements probably belong to left lane, and similarly, positive slope suggesting right lane.

####  Clean candidates by slope angle and central line of image
Due to noise of line segment proposal, it could happen that some line segments on the far right of image happen to have negative slope, which enables them to be associated to left lane propsal. This will introduce bias of left lane slope estimation, as well as to lane key point computation. Similar is the situation for the right lane propsal. Using central line of image as reference helps to remove false propsal so that the line segments whose centroid is wrongly located can be removed from valid candidates.

A minimum slope is applied to remove nearly horizental line proposals.

#### Use Normal Equation solution to make fusion of line slope
Once obtained the valid proposals of left and right lanes, we can make fusion of the slope to make global slope estimation. Simple statistics such as average or median would not be optimal since the length of the line segments, which is of crutial importance for identifying the good candidates, will NOT be taken into account. 

Therefore, following a classic pseudo-inverse method to solve Normal Equation problem, we can compute the slope by solving the equation below:

(Y2 - Y1) = M (X2 - X1), where X and Y are vectors of line segment coodinates. 
And the slope M can be obtained by:

M = (Y2 - Y1)(X2 - X1)' / (X2 - X1)(X2 - X1)^t, where " ' " stands for matrix transpose. 
In our case, X, Y are both vectors, so matrix multiplication is reduced to dot product.

Now this slope value is computed by associating more importance to longer line segments, and the overall slope could reveal better the real lane information.

#### Use Exponential Moving Average to propose robust slope value and landmarker
The lanes on road do not change abruptly. So instead of using frame-independant slope analysis, which leads to lane detection jittering around some region, Exponential Moving Average of slope proposal is applied. A moving mean version of slope is computated for video clip with a momentum predefined. The gradual changes of lane slope reduce the jittering effect. In addition, it gradually converges to the right slope value as video goes on.

Similar strategy is applied on lane key point estimation, through which the final lane will be drawn. The average localion of all valid candidates could serve as an initial key points estimation. The image below shows final valid lane candiates (high-lighted in yellow) based on which the EMA version of slope and key points are to be computed.

![alt text][line_overlay]


### 2.4 Line overlay

The proposed lanes are now overlayed on source image, and the lane length will be limited inside our defined RoI.

![alt text][lane_detection]


## 3. Discussion and future work

### 3.1 Adaptive edge detection

Canny edge detection needs low threshold and high threshold to work. Using Otsu method can propose automatically the high threshold to use through a bimondal separation. The low threshold can use half value of the high threshold.

The smoothing operation before Canny edge detection can apply bilateral filtering in order to smooth image while protecting image edges. However, in this work, it seems a simple isotropic Gaussian filtering works better. To be further investigated.


### 3.2 Robust lane proposal

Image-independent slope proposal suffers from jittering effect since all frames are isolated in treatment. However, unlike films, abrupt changes of scene for driving video is rare. So for adjacent frame, the slopes and lane key points are supposed to be highly correlated. 

Using EMA approach, cross-frame dependancy could be expoited, and the proposal of slope could be more consistent across video clips. However, finding the trade-off between consistency (larger momentum) and precision (smaller momentum) is still subject to test. Similar situation applies to lane key point estimation.


### 3.3 Remained Hyper-parameters

There are still some hyper-parameters hard to define automatically. For instance, in Hough Transform, the minimal vote number, minum line length, and maximum line length are still emperical in this work. Therefore, current solution will be hard to generalize to different scenes under various lighting conditions or landscapes. Finding an adaptive approach would be mandatory.
## Advanced Lane Finding Project

The goals / steps of this project are the following:

* Compute the camera calibration matrix and distortion coefficients given a set of chessboard images.
* Apply a distortion correction to raw images.
* Use color transforms, gradients, etc., to create a thresholded binary image.
* Apply a perspective transform to rectify binary image ("birds-eye view").
* Detect lane pixels and fit to find the lane boundary.
* Determine the curvature of the lane and vehicle position with respect to center.
* Warp the detected lane boundaries back onto the original image.
* Output visual display of the lane boundaries and numerical estimation of lane curvature and vehicle position.

[//]: # (Image References)

[image1]: ./examples/undistort_output.png "Undistorted"
[image2]: ./examples/-00446_s_undist.png "Road Transformed"
[image3]: ./examples/binary_combo_example.png "Binary Example"
[image4]: ./examples/warped_straight_lines.png "Warp Example"
[image5]: ./examples/color_fit_lines.png "Fit Visual"
[image6]: ./examples/example_output.png "Output"
[video1]: ./test_videos_output/project_video.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is located in the first code cell of the IPython notebook under "./examples/example.ipynb". First I prepare the object points as reference points for the calibration. The object points define the position
of the calibration pattern, which in our case the chessboard, in a reference coordinate system. Then I use cv2.findChessboard to find the chessboard points (image points) in the image. The object and image points are passed to
the cv2.calibrateCamera function to calculate the camera model (calibration). The calibration results are intrinsic and extrinsic parameters of the camera. The extrinsic parameters describe the position of the camera in the
reference coordinate system, which is why the results are different for each chessboard image. However, only the intrinsic parameters are used in this project. The intrinsic parameters (cx,cy,fx,fy,dist) are used to undistort
the images with the cv2.undistort function. An example of the undistorted image is as follows:

![alt text][image1]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

Using the intrinsic parameters and cv2.undistort function I can also undistort other images that uses the same camera:

![alt text][image2]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

The ideal way to do this is basically by combining the image color channels using an appropriate weight and activation function (much more like convolution neural network), but it is very time consuming to pick the weights by hand
and so I did not do it for this project. Instead, I am using more than one thresholded binary images and try to use it in a if else fashion. So for example, if the right line is not found in the S channel, then I will look for it
in the H channel. The idea is to use the strongest filter first and then gradually reduce the tolerance (by using a softer filter) until the lines are found. In this project I use 3 types of channels: yellow channel, S from HLS
channel and gradient magnitude. For the yellow channel I combine the L and b of Lab channel. It is in my opinion the most robust line finding channel in comparison to the other channel, with the weakness that it only detects yellow
line. The S channel is also robust to segment the lines from the scene, however under a weak contrast, the lines can disappear from the scene. The gradient magnitude did a very good job in separating lines from the scene, however
it also detects other irrelevant lines in the scene.

![alt text][image3]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform is included in the constructor of LineDetector, which appears in in the file `example.ipynb`, 3rd code cell.
The `warper()` function takes as inputs an image (`img`), as well as source (`src`) and destination (`dst`) points. This resulted in the following source and destination points:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 557, 475      | 290, 431      | 
| 729, 475      | 990, 431      |
| 253, 697      | 290,719       |
| 1069, 697     | 990, 719      |

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![alt text][image4]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

I chose sliding windows method to identify the lanes on the image. However, instead of having a fix step size on the vertical direction, I modify it so that the windows can move in any direction and
use a fix size for the displacement step. With this approach, it is possible to identify a lane with an extreme curve. Here is an example of a lane identification with an extreme curve:

![alt text][image5]

After that I use polyfit function from numpy to fit their positions with a polynomial.

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

I did this in my code in `./examples/example.ipynb` in the methode `measure_curvature()` from LineDetector. I calculate the radius of the curvature using the tutorial at
https://www.intmath.com/applications-differentiation/8-radius-curvature.php. First of all, we need the polynomial equation of the lanes to calculate the radius using the method described in the tutorial. Actually, we already calculated
the polynomial equation of the lanes on the previous step, however the calculated polynomial was fitted to coordinate of points in pixel instead of meter as we need. To get the result in meter, we have to estimate the pixel displacement
in meter unit. In this project I used ym_per_pix = 30/720 and xm_per_pix = 3.7/700 as a meter per pixel estimation.

The calculation of vehicle distance to the center is pretty straight forward. I only need to calculate the distance of the image center with the position of the first sliding window of the left lane in x axis, afterwards with the
assumption that the vehicle lies between left and right lane and the distance between them is 3.7 m, I can calculate the distance of the vehicle to the center of the road.

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I implemented this step in `./examples/example.ipynb` in the function `process_image()` in the 4th code cell.  Here is an example of my result on a test image:

![alt text][image6]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./test_videos_output/project_video.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

With this implementation, the images are processed independently of each other, so no information from the previous frame is retained in the calculation of the current frame. This sliding windows from the previous frame can actually give us
an initial approximation of the current possible lane position and is therefore very helpful, especially if the lane is only minimally visible in the current frame.

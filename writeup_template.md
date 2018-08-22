## Writeup Template

### You can use this file as a template for your writeup if you want to submit it as a markdown file, but feel free to use some other method and submit a pdf if you prefer.

---

**Advanced Lane Finding Project**

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
[image2]: ./test_images/test1.jpg "Road Transformed"
[image3]: ./examples/binary_combo_example.jpg "Binary Example"
[image4]: ./examples/warped_straight_lines.jpg "Warp Example"
[image5]: ./examples/color_fit_lines.jpg "Fit Visual"
[image6]: ./examples/example_output.jpg "Output"
[video1]: ./project_video.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

You're reading it!

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the first code cell of the IPython notebook located in "./examples/example.ipynb" (or in lines # through # of the file called `some_file.py`).
I start by preparing the object points as a reference points for the calibration. The object points defines the "real" position (as well as the form) of the object shown
in an image, which is chessboard in our case. I then use cv2.findChessboard to get the chessboard points (image points) on each image. Using these image points, object points
and cv2.calibrateCamera function the camera is calibrated. The calibration results are an intrinsic and extrinsic parameters of the camera, whereas the extrinsic parameters are
image specific because they define the position of the camera with respect to the camera coordinate which I defined in the object points. Only the intrinsic parameters (cx,cy,fx,fy,dist)
are used to undistort the images using cv2.undistort function. The undistort result is as follows:

![alt text][image1]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

Using the intrinsic parameters and cv2.undistort function I can also undistort other images that uses the same camera:

![alt text][image2]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

The ideal way to this is by basically combining the image color channels using an appropriate weight and activation function (much more like convolution neural network), but it is very time
consuming to pick the weights by hand and so I did not do it for this project. Instead, I am using more than one thresholded binary images and try to use it in a if else fashion. So for example,
if the right line is not found in the S channel, then I will look for it in the H channel. The idea is to use a strong filter first and then going to the other filter if the lines not found.
In this project I use 3 types of channels: yellow channel, S from HLS channel and gradient magnitude. For the yellow channel I combine the L and b of Lab channel. It is in my opinion the most
robust line finding channel in comparison to the other channel, with the weakness that it only detects yellow line. The S channel is also robust to segment the lines from the scene, but because
it is a very hard filter, sometimes the lines are just disappeared from the scene. The gradient magnitude did a very good job in separating lines from the scene. But it also includes other lines
in the scene.

![alt text][image3]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform is included in the constructor of LineDetector, which appears in in the file `example.ipynb`, 3rd code cell.
The `warper()` function takes as inputs an image (`img`), as well as source (`src`) and destination (`dst`) points. This resulted in the following source and destination points:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 585, 460      | 320, 0        | 
| 203, 720      | 320, 720      |
| 1127, 720     | 960, 720      |
| 695, 460      | 960, 0        |

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![alt text][image4]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

I chose sliding windows method to identify the lanes on the image. However, instead of having a fix step size on the vertical direction, I modify it so that the windows can move in any direction and
use a fix size for the displacement step. With this approach, it is possible to identify a lane with an extreme curve. Here is an example of a lane identification with an extreme curve:

![alt text][image5]

After that I use polyfit function from numpy to fit their positions with a polynomial.

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

I did this in lines # through # in my code in `yet_another_file.py` in the function `map_lane()`. I calculate the radius of the curvature using the tutorial at
https://www.intmath.com/applications-differentiation/8-radius-curvature.php. First of all, we need the polynomial equation of the lanes to calculate the radius using the method described in the tutorial. Actually, we already calculated
the polynomial equation of the lanes on the previous step, however the calculated polynomial was fitted to coordinate of points in pixel instead of meter as we need. To get the result in meter, we have to estimate the pixel displacement
in meter unit. In this project I used ym_per_pix = 30/720 and xm_per_pix = 3.7/700 as a meter per pixel estimation.

The calculation of vehicle distance to the center is pretty straight forward. I only need to calculate the distance of the image center with the position of the first sliding window of the left lane in x axis, afterwards with the
assumption that the vehicle lies between left and right lane and the distance between them is 3.7 m, I can calculate the distance of the vehicle to the center of the road.

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I implemented this step in lines # through # in my code in `yet_another_file.py` in the function `map_lane()`.  Here is an example of my result on a test image:

![alt text][image6]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./project_video.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

Here I'll talk about the approach I took, what techniques I used, what worked and why, where the pipeline might fail and how I might improve it if I were going to pursue this project further.  

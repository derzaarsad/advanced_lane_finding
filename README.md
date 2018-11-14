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
[image3]: ./examples/Figure_1-1.png "Binary Example"
[image4]: ./examples/warped_straight_lines.png "Warp Example"
[image5]: ./examples/Figure_2.png "Fit Visual"
[image6]: ./examples/example_output.jpg "Output"
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
and so I did not do it for this project. Instead of implementing a filter that could work decently for all possible lane color (yellow and white), I created separate filters for each color that is normally used on the street lane, that way I could
experiment with the parameter of each color filter independently. After experimenting for a while, I noticed that it is very hard to create a filter that can capture the lanes clearly without introducing any noise, therefore I
decided to just let the noise or outlier exist and then later filter the resulting false detected windows with RANSAC algorithm. In this project I use 3 types of masks: white mask, yellow mask and gradient magnitude mask. For the white mask I combined the HSV and
L from Luv channels, where for the yellow mask I combined the HSV and b from Lab channels. Both filters are in my opinion very robust, however at the end I saw that the white mask works better under a different light condition in
comparison to the yellow mask. I added a gradient magnitude mask because it does a very good job in separating lines from the scene although it also detects other irrelevant lines in the scene. The binary threshold can be seen in the 4th code cell.
![alt text][image3]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform is included in the constructor of LineDetector, which appears in in the file `example.ipynb` in the `process_image()` method.
The `get_binary_warped()` function takes as inputs an image (`img`). The source (`src`) and destination (`dst`) points are already calculated in the constructor of LineDetector. This resulted in the following source and destination points:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 557, 475      | 290, 431      | 
| 729, 475      | 990, 431      |
| 253, 697      | 290,719       |
| 1069, 697     | 990, 719      |

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![alt text][image4]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

I tried to use a modified sliding windows method to identify the lanes on the image. Instead of having a fix step size on the vertical direction, I modify it so that the windows can move in any direction and use a fix size for the displacement step.
With this approach, it is possible to identify a lane with an extreme curve. However, this kind of approach is not quite robust especially if we don't have a continuous lane like the right white lane on the project_video.mp4. For this reason, I decided
to use a region base window searching. I divided the image into 9 regions along the vertical axis and then calculate the right window placement on each region. I used histogram to calculate the peak of points concentration and then verify the position
of the peak relative to a defined midpoint. At first, half of the image width is taken as the midpoint, and then after a valid window is found, the midpoint is updated relatively to the current window for the next window finding, this is possible
with the assumption that the distance between left and right windows on the same region is approximately constant (in my case 700 pixels). To check the window validity I used 100 as a minimum points concentration. This approach works very good regardless
of the continuity of the lane, however it is very susceptible with noise with a high point concentration, for this reason I implemented a filter based on RANSAC to remove the false windows and then feed the np.polyfit only with good windows.

The code of the RANSAC filter can be seen in the 3rd code cell. RANSAC is an iterative fitting algorithm where in each iteration a set of random points is taken and then a model based on these points is calculated. The inliers or the points that fit with
the model are identified by calculating the distance between the model and all of the points as well as applying a threshold of maximum allowed distance. By each iteration the number of inliers are calculated and then at the end the model with the most inliers is
considered to be the right model. In our case, the model is a polynomial, therefore the model that is calculated on each iteration is made only by 3 points. To achieve a more accurate model generalization, I take all the inliers inclusive the points that are used for the model
themselves and then feed np.polyfit with these inliers. That way, we get a highly accurate line estimation without considering the outliers.
![alt text][image5]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

I did this in my code in './examples/example.ipynb' in the methode `measure_curvature()`. I calculate the radius of the curvature using the tutorial at
https://www.intmath.com/applications-differentiation/8-radius-curvature.php. First of all, we need the polynomial equation of the lanes to calculate the radius using the method described in the tutorial. Actually, we already calculated
the polynomial equation of the lanes on the previous step, however the calculated polynomial was fitted to coordinate of points in pixel instead of meter as we need. To get the result in meter, we have to estimate the pixel displacement
in meter unit. In this project I used ym_per_pix = 30/720 and xm_per_pix = 3.7/700 as a meter per pixel estimation.

To calculate the distance of the vehicle to the center we need to calculate the difference in x axis between the center of the road, which is the midpoint between the bottom right and left windows, with the center of the car, which is
the center of the image. The difference in pixel is multiplied with xm_per_pix resulting a difference between car and the center of the road in meter.

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I implemented this step in `./examples/example.ipynb` in the function `process_image()` in the 9th code cell.  Here is an example of my result on a test image:

![alt text][image6]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./test_videos_output/project_video.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

With this implementation, the images are processed independently of each other, so no information from the previous frame is retained in the calculation of the current frame. The sliding windows from the previous frame can actually be very helpful
to calculate a first approximation to the current possible lane position, especially if the lane is only minimally visible in the current frame. An example of this problem can be seen in [harder_challenge_video](./test_videos_output/harder_challenge_video.mp4)
where the detection of the lane fails when a very bright light strikes the scene. The other improvement is by creating a better filter by combining color channels, weights and activation function, as I mentioned earlier in the pipeline explanation.
Unfortunately this can only be achieved by manually selecting the parameters and trying them out or by training as in a Convolutional Neural Network. Both methods are time consuming and require a large amount of data.

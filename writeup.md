# Advanced Lane Finding Project

The goals / steps of this project are the following:

* Compute the camera calibration matrix and distortion coefficients given a set of chessboard images.
* Apply a distortion correction to raw images.
* Use color transforms, gradients, etc., to create a thresholded binary image.
* Apply a perspective transform to rectify binary image ("birds-eye view").
* Detect lane pixels and fit to find the lane boundary.
* Determine the curvature of the lane and vehicle position with respect to center.
* Warp the detected lane boundaries back onto the original image.
* Output visual display of the lane boundaries and numerical estimation of lane curvature and vehicle position.

The structure of this writeup follows the steps and goals in some grouped form and in the order how I addressed them. Additionally, let me point out that I also documented and commented directly in the notebook, because I usually prefer this way of presenting:
* [Calibration and perspective transformation](./Solution-Part1-Calibration_Perspective.ipynb)
* [Thresholding, lane pixels and output generation](./Revision-Solution-Part2-Pixel_Lane_Output.ipynb)

---

[calibration]: ./docs/undistort_calibration1.jpg "Calibration chessboard"
[undistortion]: ./docs/undistort_straight_lines1.jpg "Undistorted road image"
[perspective1]: ./docs/perspective_straight_lines1.jpg "Perspective transformation straight lines 1"
[perspective2]: ./docs/perspective_straight_lines2.jpg "Perspective transformation straight lines 2"
[perspective_curve]: ./docs/perspective_test2.jpg "Perspective transformation curve"
[parameter_tuning]: ./docs/parameter_tuning.PNG "Parameter tuning"
[threshold1]: ./docs/threshold_test2.jpg "Thresholding 1"
[threshold2]: ./docs/threshold_test5.jpg "Thresholding 2"
[lanefit1]: ./docs/lane_fit_test1.jpg "Lane fit 1"
[lanefit2]: ./docs/lane_fit_test6.jpg "Lane fit 2"
[pipeline1]: ./output_images/pipeline__straight_lines1.jpg "Pipeline 1"
[pipeline2]: ./output_images/pipeline__test3.jpg "Pipeline 2"
[video_out]: ./project_video_out.mp4 "Project video"


# Description

## 1. Calibration

This section includes the following tasks:

* Compute the camera calibration matrix and distortion coefficients given a set of chessboard images.
* Apply a distortion correction to raw images.

At first, we need to calibrate the camera in order to correct for image distortion, namely radial and tangential image distortion. 

This is done by fitting a camera matrix and a collection of distortion coefficients to pairs of 3D world and 2D image coordinate pairs, via the OpenCV function ``cv2.calibrateCamera``. For this one picks the regular, high contrast picture of a chessboard images, where we can pick a fixed grid like coordinate system in 3D, called ``object points``, and where we can automatically detect the corners in the 2D image, named ``image_points``, via appropriate utility functions in 2D.

Note that in this fitting process there are additional rotations and translation parameters, which are initially responsible to transform the 3D world coordinate system to those of the camera. This is the reason why we can pick the same regular grid like object points of the chessboard. An example of such a calibration image is: ![][calibration]

Note that the corner detection, required for automatic image point detection only succeeded in 17 out of 20 cases; in 3 cases the inners corners are too close to the border. Additionally, it is important to use the same ordering in object and image point, which is something that I messed up initially.

After one has performed this correction all remaining images can be undistored and an example of a road image is given in the figure below; biggest visual difference can be seen at the front lid.

![][undistortion]


## 2. Perspective Transformation

* Apply a perspective transform to rectify binary image ("birds-eye view").

Upon detecting the lane line pixels we want to fit a quadratic polynomial, which further enables us to measure the lane curvature. This is most easily done by looking at the situation from above, which requires a perspective transformation. Perspective transformation can be fitted by supplying appropriate source and destination points.

Note that we want to have a plain view on the road from above. Hence, I was mainly focusing on the two ``straight_line*`` images; however also for the other images I was trying to visually check that the lanes are parallel. The source points were taken to be the corner points of the trapezoid forming a straight lane part; its destination points are the top/bottom points with equal x-coordinate, still leaving some space to the left and right side. Those points were picked manually via an interative plotting window. Note the trapezoid is not 100% parallel, most likely the camera is not mounted perfectly at center. Finally, I picked the following points:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| (205, 720)    | (290, 720)    | 
| (581, 460)    | (290, 0)      |
| (701, 460)    | (990, 0)      |
| (1107, 720)   | (990, 720)    |

Examples are given in the following pictures:

![][perspective1]
![][perspective_curve]

Note, this perspective also determines the pixel-to-real-distance mapping, which I read off in the end. Because of this I also tried to pick as the source height a clear ending of one of the dashed lines. This is particular hard in the far view.

According to the course material the lane width is ``3.7m``, while a single dashed lane marker is ``3.048m``. In particular, from the ``straight_line*`` images one sees that we have about 6 times a dashed line. This gives the units:
```
XM_PER_PIX = 3.7 / 700
YM_PER_PIX = (6 * 3.048) / 720
```


## 3. Gradient & Color Thresholds

* Use color transforms, gradients, etc., to create a thresholded binary image.

Naturally one of the main components is to detect the lane pixels in the image. For this we were taught several gradient and color thresholding techniques in the course. 

For clear visible lane markers one can readily detect them by approriate edge detction techniques, all relying on the gradient or nearby pixel variations:

- Derivative in x or y direction: Here one computes the pixel variation along x- or y- direction to detect vertical or horizontal edges. The gradient in a given direction can be computed with ``cv2.Sobel(gray, order)`` with ``order`` being a tuple specifiying the direction, e.g., ``order=(1, 0)`` for the x direction.
- Gradient magnitude: The length of overall gradient can be computed directly from the derivatives in x- and y- direction, and is used to detect clear edges in an arbitrary direction.
- Gradient direction: Usually lane lines and markers are somewhat pointing into a vertical fanned direction, hence its edges are also somewhat directionally constrained. Such edges can be filtered out knowing the gradient direction or angle that can be readily computed from its x and y component via ``np.arctan2``.

Via the gradient thresholds one gets problems if the lane lines or markers are in the shadow or on a not high contrast road surface. Additionally, those are methods are usually applied on a grayscaled version of the image. Moreover for color images one can also select a different color space. The HLS, which stands for hue, lightness and saturation was explained to be favorable option here.

- Color threshold transform: Convert the color image into the HLS space, picks its saturation components and threshold lines there.

Of course the above methods can be combined in an arbitrary fashion and each component requires parameter tuning again. For this I was following closely the examples given in course and was combining:
1. Strong derivate in x and derivate in y
2. Direction and length
3. Color threshold and strong x derivative on saturation image

For parameter tuning I made myself again a small interactive widget that allowed me to scan for good candidates. 

In comparison to the initial submission I added an additional x derivative check on the saturation image, mainly because in some bad frames images from the video, the shadow from the trees was very strongly present and this gave some way to filter it partially.

For parameter tuning I made myself a small interactive widget that allowed me to scan for good candidates:

![][parameter_tuning]

A couple of results are shown in the following pictures:

![][threshold1]
![][threshold2]


## 4. Lane Pixels & Fit

* Detect lane pixels and fit to find the lane boundary.

After warping the thresholded image we need to detect lane pixels and fit a quadratic polynomial to determine the lane curvature and relative lane position in the next step.

Detecting lane pixels from scratch is done along the following line, and a direct implementation has been given in the course material. I only did some cosmetic code modification and annoted the imporant steps in the core function ``find_lane_pixels``. Please see the notebook for this at[Lane Pixel & Fit section](./Revision-Solution-Part2-Pixel_Lane_Output.ipynb#Lane-Pixels-&-Fit):

- First we determine the left and right bottom lane x positions using the histogram peaks in the lower part of the image, ``step a``. 
- Using these positions as starting points we repeatedly look for pixels in a small window centered at these current positions, ``step b``. If we find sufficiently many new pixels in a given window, we take its mean x-position as the new center for the next sligind window, ``step c``. 
- All pixels detected in these windows are lane pixels.


Note if we have prior information about the lane positions, for instance via the fit of a previous frame, we can utilize this information rather than trying to detect lane pixels from scratch. This is the second major change I did in this revision. 

This utility function ``search_around_poly`` was similarly already presented in the course and my changes here are minimal (please refer to the notebook [Lane Pixel & Fit section](./Revision-Solution-Part2-Pixel_Lane_Output.ipynb#Lane-Pixels-&-Fit)); its central point is that the lane pixels are now those being in the vicinity around the prior or the previous fit, ``step A``, and not those searched via a sliding window approach. 

Note that in the image plots below I employ the ``find_lane_pixel`` output as prior to show its effect; in the video this is taken to be the previous fit parameters.

Once having determined the lane pixels we can directly fit a quadratic polynomial ``x = x(y)``, which can be carried out using ``np.polyfit`` function.

![][lanefit1] 
![][lanefit2]


## 5. Compute Measures & Finalizing Output

This section covers the following points:

* Determine the curvature of the lane and vehicle position with respect to center.
* Warp the detected lane boundaries back onto the original image.
* Output visual display of the lane boundaries and numerical estimation of lane curvature and vehicle position.


Finally, we compute curvature and vehicle positions, both of which can be computed from the fits. Curvature of each lane is given by an analytic formula involving the fit coefficients and the according y-position of interest; in our case the bottom of the image because this closest to the vehicle. For a fit of ``x(y) = A*y**2 + B*y + C`` this is given by ``C(y) = (1 + (2 * A * y_eval + B)**2 )**1.5 / abs(2 * A)``.

The relative vehicle position can be computed from the difference between the midpoint of left and right lane spots at the bottom of the warped image and the assumed car center, in our case the horizontal center.

Both pixel values need to be converted to real units by the pixel-to-meter conversions, that we determined during the perspective transformation. 

I had several difficulties in computing the curvature and I was constantly getting too high values. I used the fit paramter scaling trick in the course, but always forgot to scale the y-position of interest. In the end, the following post on the Udacity Knowledge Forum largely resolved these difficulties: [Need Help with reviewers comment on calculating Radius of Curvature](https://knowledge.udacity.com/questions/18801)

Of course, we want to warp the detected lane back to the original, distortion corrected image, and add the computed measures. For this a utility function has been given in the tips and tricks part of the project description.

Examples of the full pipeline including these last steps are:

![][pipeline1]
![][pipeline2]


# Output

I have processed all images provided in the ``test_images`` folder and saved the above seen output showing the essential pipeline steps in the folder ``output_images``. 

A final output of the project video is [project video output](./revision_project_video_out.mp4). Note, that in this video the fit from the previous frame is used as prior in the lane pixel detection. 


# Reflection

As discussed in more detail in the main part I had some issues regarding the camera calibration and curvature computation steps. In particular, the radius computation was giving me difficulties, because this was the last part in the pipeline and I was unsure about the influence of the previous, main manual tuning steps, like perspective transformation and image thresholding. This also took me quite some time so that I also did not pursue any of the more challenging videos.

Regarding the revision, I followed the suggestions by the reviewer to work on the bad frames in the video, which I additionally added to the test images and its previous bad examples to the example folder. Mainly I added an x-derivate threshold to the saturation image, further tuned the parameters of the thresholing part, and enhanced the lane pixel search via a prior to the project.


## Shortcomings

I see the following main deficits in the pipeline:

- Lane fitting process is susceptible to outlier.
- More extreme lightning, shadowing and weather conditions
- Vertial aligned road surface changes, marks, etc.
- Tight curves
- Specific situations: lane change, car in front, ...
- Parameter tuning
- No reset mechanism; if there were some problems in the previous frame this carries on.
  
  
## Possible Improvements

Some tricks were presented in the project description and rubric:

- Use smarter prior information to detect lane, e.g., sometimes on.
- Lane smoothing over multiple frames. 
- Image processing techniques to remove small pixels.
- Other color space information.
- Exponential smoothing of curvature and position values

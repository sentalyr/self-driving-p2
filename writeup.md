## Project 2 - Advance Lane Lines

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

[chessboard_corners]: ./output_images/chessboard_corners.png "Detected Corners"
[undistorted_chessboard]: ./output_images/undistorted_chessboard.png "Undistorted Chessboard"
[undistorted_test_image]: ./output_images/undistorted_test_image.png "Undistorted Test Image"
[test_image_threshold]: ./output_images/test_image_threshold.png "Thresholded test image"
[challenge_threshold]: ./output_images/challenge_threshold.png "Thresholded challenge video frame"
[warped_image]: ./output_images/warped_image.png "Perspective Warp"
[sliding_window_fit]: ./output_images/sliding_window_fit.png "Perspective Warp"
[search_from_previous_fit]: ./output_images/search_from_previous_fit.png "Perspective Warp"
[reverse_warp]: ./output_images/reverse_warp.png "Perspective Warp"
[project_video]: ./project_video_output.mp4 "Video"
[challenge_video]: ./challenge_video_output.mp4 "Video"


## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.

You're reading it!

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the "Compute the camera calibration using chessboard images" section of the IPython notebook located in "./project2.ipynb".  

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.

Here is an one of the chessboard calibration images with the detected corners drawn.

![alt text][chessboard_corners]

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result:

![alt text][undistorted_chessboard]


### Pipeline (single images)
The image processing pipeline consisted of the following steps.

#### 1. Correct for distortion.
Once the distortion coefficients were calculated using the chessboard calibration images, I applied these values to the test images using `cv2.undistort()`. The following image is one of the test images with distortion correction applied.

![alt text][undistorted_test_image]

#### 2. Apply thresholds to undistorted images
I created the following thresholding methods:
* `abs_sobel_thresh` - filter on x and y Sobel gradients.
* `mag_thresh` - filter on the magnitude of the combined x and y gradients.
* `dir_threshold` - filter on the direction of the combined x and y gradients.
* `hls_color_threshold` - filter on one of the HLS color channels.

The implementation of these methods can be found in the "Use color transforms, gradients, etc., to create a thresholded binary image" section of the project2 notebook.

Using the above methods, I created filters for both the X and Y Sobel gradients.  Filters for the Sobel magnitude and direction. And filters for the lightness and saturation channels of the HLS converted image.

I decided to apply these filters to output the intersection of the color (lightness and saturation) detections with one of the Sobel filters.  This was to capture the notion that as a human, I likely identify lane lines based on something that is the color I would expect and has the directionality of a lane line.

Because the Sobel based filters result in thin detected edges, I applied a very large kernel size of 31 to the filters. This was to intentionally thicken the returned detections to increase the likelihood these detections would intersect with one of the color filters.

In the order of the methods above, here are the filters applied to one of the test images:
![alt text][test_image_threshold]

I used a combination of the lightness saturation channels in my color filter because the lightness channel does a nice job capturing the white lane lines, particularly in saturated video frames (characteristic of the frames in the challenge video).  The saturation channel detects the yellow lane lines quite nicely. So a combination of these two color filters does an adequate job of capturing the whites and yellows in the frame.

I tuned the X and Y gradient filters to attempt to select line detections that have a slope similar to a lane line.  I applied a very liberal Y filter as the lane lines would have a steeper gradient in the X direction.  The X gradient filter on the other hand was more restrictive so select gradients that change more sharply.

I tuned the magnitude and direction filters with the same goal of filtering strong gradients (ie high magnitude) with the directionality expected of a lane line.

The following set of images demonstrate the application of these filters to a captured frame from the challenge video.  As shown, the application of these filters appears to adequeately distinguish between the lane lines and the tar strip and shadows that initially led to erroneous lane detections.
![alt text][challenge_threshold]

#### 3. Performing the Perspective Transform
After applying the above filters to the test images, I then performed a perspective warp on the images to create a top-down view of the image. This provides the perspective such that lane curvature can be estimated once the line detections are complete.

The code for my perspective transform algorithm is in the "Perspective Warp" section of the project2 notebook.

The first step of the perspective transform is to determine the source and destinations vertices to be used when computing the transformation matrix.  To determine the source vertices, I picked anchor points at the bottom corners of the image and a cutoff height at 65% of the image (from the top).  I then determined slope values through trial and error and computed the x coordinates at the top of the trapezoid using the slope and anchor points.  Truthfully, this was more convoluted that it needed to be. I should have just picked 4 vertices symmetrical about midpoint of the x-axis.  This computation is performed in the `get_warp_params()` method.  Here are the source vertices this algorithm results in:

src = [[ 547.8261  468.    ]
       [ 732.1739  468.    ]
       [1280.      720.    ]
       [   0.      720.    ]]

I chose destination points 200 pixels in from the sides of the image with the full height.  With the source and destination points defined, I then calculated the transformation matrix using `cv2.getPerspectiveTransform(src, dst)`.  The destination vertices are as follows:

dst = [[ 200.    0.]
       [1080.    0.]
       [1080.  720.]
       [ 200.  720.]]

Armed with the transformation matrix, I then performed a perspective warp on the test images using `cv2.warpPerspective()` (with linear interpolation).  I also defined a method `apply_mask` which uses the source vertices defined in `get_warp_parms` to overlay a semi-transparent mask onto the original binary image.  This was simply so I could visualize my source vertices before the transformation.  The following example is the masked original image and the warped image:

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![alt text][warped_image]

#### 4. Detecting lane pixels in the warped images
I used two techniques to detect lane pixels in the warped images -- histogram and window, and search from previous fit.

##### 4a. Histogram and window
To perform the histogram and windowing lane detection method, I first generated a histogram in which all nonzero pixels were counted in the lower half of the binary input image.  The maximum on the left half and right half of the image was assumed to correspond to point at which the left and right lanes protrude from the bottom of the image. The following figure shows the histogram for one of the test images:
![alt text][histogram]

After obtaining the starting x positions for both the lane lines, a 200px wide window was centered on each base position.  All nonzero pixels in each window were then added to a running tally of left and right pixels that belong to the lane line.  The x positions of these pixels were then averaged to determine the center of a next window positioned vertically above the current window.  The image was divided into 9 windows of equal height and the process of detecting pixels and recentering the subsequent window was performed at each level.  The implementation of this algorithm can be found in `find_lane_pixels_hist()` of the project2 notebook.

The sliding window narrows the search area to a region in which pixels contributing to the lane lines are likely to be found. By separating the frame into various height levels we enable the algorithm to track curves in the lane line.  In cases where the radius of a lane curve is small the lane line can exit either the left or right side of the frame.  I did not design my algorithm to handle the case in which the lane line exits the left or right of the image frame.  Had I attempted to implement this feature, I would have perhaps simply aborted the iteration cycle if the previous window's left or right side exceeded the image frame and the current window did not detect any pixels.

Once a collection of image coordinates were identified as lane line pixels, the coordinates were fit to a second order polynomial function using `np.polyfit()`.  The following image shows the search window during pixel detect.  The detected pixels are show in red and blue for the left and right lane line detections respectively. The yellow line in each half of the image is a plot of the second order polynomial function.

![alt text][sliding_window_fit]

#### 4b. Search from previous fit
The second method to detect pixels relies on the line of best fit calculated from a previous frame.  This algorithm operates on the assumption that the lane lines in the current frame are likely close to those of the previous frame.  So we use this assumption to narrow the region in which we search for lane line pixels.

In this algorithm I created a 200 pixel wide window around the previous frame's line of best fit.  I then iterate up each y coordinate in the image searching for pixels that fall in this window.  Each list is then fed into `np.polyfit()` to generate a second order line of best fit.  The implementation of this algorithm can be found in the method `find_lane_pixels_last_fit` in the project2 notebook.

The following image highlights the search region around the line of best fit.  This image was generated by first feeding the image through the sliding window algorithm to generate the line of best fit.  The same image and computed line of best fit were then fed into the search for previous method -- hence the same set of detected pixels.

![alt text][search_from_previous_fit]

#### 5. Calculate radius and vehicle offset
##### 5a. Calculate radius
After generating the lists of detected pixels and the coefficients for the lines of best fit, I attempted to calculate the radius of the curve and the offset of the vehicle from the center of the lane.  I perform this calculation in the `curvature_meters` function in the "Define methods to calculate curvature and offset" section of the project2 notebook.

To compute the raidus I used the constants from the lesson to approximate the conversion between pixels to meters.
```
    ym_per_pix = 30/720 # meters per pixel in y dimension
    xm_per_pix = 3.7/700 # meters per pixel in x dimension
```
Using these values, I generate a scaling factor for the A and B coefficients of the quadratic lihe fitted to the detected pixels.  I then calculate curvative using the following formula to calculate the radius given the quadratic coefficients:
$$ R_m = ((1 + (2 * A * y + B) ** 2) ** (3 / 2)) / (2 * A) $$

##### 5b. Calculate distance from center
To calculate the distance of the vehicle from the center of the lane, I wrote the `offset_from_center_meters` function.  This function first computes the *x* value at the bottom of the image in pixels.  I then subtrace the x value from the midpoint of the image and then use the same `xm_per_pix` scaling factor to convert this distance to meters.

I perform this calculation on both lane lines.  Then in the `annotate_image` function I calculate the offset as $ (x_left - x_right) / 2.0 $. If the result is positive, the car center is left of the lane center. A negative result indicates the car center is right of the lane center.


#### 6. Reverse warp
In the section "Reverse warp and annotate test imges" of the project2 notebook, I reverse the warped lane detection back onto the undistorted original image.  I also annotate the undistorted image with the calculated curvature and lane offset.

![alt text][reverse_warp]

---

### Pipeline (video)
#### 0. The Line class
The Line class is meant to carry state about each detected lane line over a window of video frames.  I modified the class to perform a moving average of detections over the last 10 video frames.  As the above algorithms are applied to each video frame, I add the captured detections (or lack thereof) to the moving average.  This technique allowed me to smooth the predicted lane lines to prevent jittery detections.

Using the moving average, I also tracked the number of consecutive missed detections.  If the `search_from_previous_fit()`  algorithm failed to detect lane pixels for 4 video frames, the pipeline would fall back to the histogram and sliding window technique to attempt to identify lane lines.

#### 1. Sanity checking detections
The `detections_are_sane` method performs a very rough check to determine if the lane lines are approximately parallel, the lane width is within expected tolerance, and the computed radii are within an order of magnitude of each other.

To determine if the lanes are approximately parallel, I determine the lane width at each y position of of the fitted lines.  If the difference between the maximum and minimum detected distance exceeds a threshold I throw out the detection.

The lane width calculation uses the `xm_per_pix` conversion above and will throw out a detection if the computed with is less than ~8.5ft or greater than ~14ft.

The radii comparision is perhaps a less fitting test than the parallel detection.  As the curve of the line approaches an infinite radius small variations in detected pixels result in significant changes in the calculated radius.  For this sanity check, I took the logarithm of the inverse of the curvature in an attempt to allow for a greater difference between the two lane lines as the radius approaches infinity.

#### 2. Provide a link to your final video output.

Here's a [link to my video result](./project_video_output.mp4)

As seen in the video, the detected lane lines track the actual lanes reasonably well.  When the vehicle bounces the impact of the smoothing algorithm becomes appararent as the detected lane boundaries bounce with the vehicle.  The algorithm struggles the most when the frame contains significant saturation variations such as bright tarmac or shadows from trees.

#### 2. Challenge video

Here's a [link to my challenge video result](./challenge_video_output.mp4)

To improve my detection pipeline for this video, I added the lightness filter to my color thresholding pipeline in an attempt to distinguish between the tar line and the white line on the right side of the vehicle.  This change also improves the algorithms ability to distinguish between the shadows on the left side of the frame.

The high saturation of the images in this video degrade the contrast between the lane lines and the road.  This has the effect of tricking the histograming and serach from previous algorithms into thinking the shadows and tar strip are stronger detections than the actual lane lines.

As seen in the video, the algorithm struggles to accurately detect the lane lines. Looking at the output of the Line class, there are several consecutive frames with missed detections.

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

In this project I struggled to find a set of thresholding parameters that would work well in both the project video and challenge video.  I think a strategy to white balance the images or somehow normalize the colors could help in making the algorithm more performant across a wider variety of operational scenarios.

I also think creating a metric for analyzing the strength of a fit when fitting a curve to a set of detected pixels would be a valuable addition.  In the current design, the algorithm will throw out both the left and right lane detections if either fit is poor and leads to a failed sanity check. In many cases the left line might be detected strong and the right line weak and it is wasteful to throw out the strong deteciton in this situation.

I also did not modify the windowing algorithm to detect if the detected lines exit the left or right of the frame.  This means my windowing algorithm will fail when there is a sharp curvature (such as those in the harder challenge video).

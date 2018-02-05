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

[image1]: ./output_images/undistort_example.png "Undistorted"
[image2]: ./output_images/undistorted_lane_with_poly.png "Undistored Road"
[image3]: ./output_images/combined_threshold.png "Combined Binary Example"
[image4]: ./output_images/warped_image.png "Warp Example"
[image5]: ./output_images/processed_image.png "Processed Image"
[image6]: ./output_images/lower_hist.png "Histogram Peaks"
[image7]: ./output_images/lane_detection.png "Lane Detection"
[image8]: ./output_images/reverted_lane.png "Lane Projection"
[video1]: ./project_video.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

You're reading it!

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the first two code cells of the IPython notebook located in "./advanced-lane-detection.ipynb".  

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image. Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image. `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function. I then wrapped these coefficients in a method for use by my pipeline. In cell three, I undistort all the chessboard images. Here is an example result:

![alt text][image1]

I now have the ability to undistort images taken by that camera.

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

The first step of image preprocessing corrects distortion in video images. Here is an example:

![alt text][image2]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

Next I use gradients and color transforms to identify lane pixels in my images. In testing, I experimented with sobel filters in x and y, magnitude gradients, direction gradients and thresholding the S channel of HLS color space. The code for these calculations is available in Cell 8. When tuning parameters, I adjusted kernel size and binary thresholds. I inspected individual thresholds in cell 9.

The last step of `threshold_image` combines these thresholded gradients into a single thresholded image in locations where (the sobelx binary is one or the s channel binary is one) and the direction binary is one. I found this combination to be effective and produce the least amount of noise across video images.

Here's an example of the combined thresholded image:

![alt text][image3]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

Next, I defined a function to perform perspective transforms on my road images. This step relies on source and destination points. Source points are the pixel coordinates in the undistorted image. Destination points are where the transform should located those coordinates in the warped image.

To derive source points, I took advantage of the fact that lane lines on a straight road are parallel and can be used to define a rectangle. That makes choosing destination points easy. Simply provide points that define a rectangle in given you input image.

Code cell 5 defines source and destination points, then calculates a transform matrix. In cell 6, I wrapped application of this matrix to my image in `warp_image`.

In this image, source points are drawn on the right hand image:

![alt text][image2]

In cell 7, I unwarp the test image and plot destination points. By running on my test image, I verify the function is working properly:

![alt text][image4]

#### 4. Image Preprocessing

Before moving on to lane identification, I tied all of my preprocessing steps into a single method `process_image` in cell 11. After warping the thresholded image, I apply a final threshold that converts blurred portions of the lane lines, that would be excluded by future steps, to color space that will be considered by lane detection. I found this improved lane detection for features further away from the camera, particularly on dotted lines.

In cell 8, I verify the entire process on the test image.

![alt text][image5]

#### 5. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

To detect lane lines in my processed image with no existing knowledge of the line, I wrote `window_search` in cell 15. This method takes as input the processed image (binary_warped) and arrays of nonzero y and x indices in the image. I then calculate the base of the right and left lane using histogram peaks in the lower half of the image. Here's an example of that histogram:

![alt text][image6]

I now have the base of my lane lines. In the next block, I step through image windows centered on the lane position with a margin of 100 and a height defined by the number of windows I wish to employ. After each window, I recalculate the lane center for the next iteration. At the end of the process, I have two arrays, one for each line, that point to the pixels I believe to be in that line.

I map lane pixels back to coordinate space (see cell 16), then pass the x and y coordinates to `fit_polynomial` in cell 15. This method fits a curve to lane line pixels and returns coefficients that define the line. Back in cell 16, I pass the fit for each lane line to `caluclate_radius` (defined in cell 15). This method returns the radius of the line in meters.

Here's an example of the process end to end... The image on the left is my input. The image on the right displays pixels identified as lane lines, the windows used to detect pixels, the polynomial fit (in yellow) and the radius of both curves.

![alt text][image7]

In cases where I have knowledge of the lane line from existing frames, I implemented `margin_search` in cell 15. This function searches for lane pixels within a margin of an existing polynomial fit.

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I implemented `revert_frame` in cell 19. This method takes as input all y coordinates of my image, the polynomial fit for each lane line, my warped image and the original (undist) image. The method creates an empty image (color_warp) in warped coordinate space. It then plots lane shape on that image.

Before applying color_warp to the original image, I unwarp it using `unwarp_image`, which uses the inverse of the transform map calculated using source and destination points. Finally, I overlay the unwarped lane color onto the original image.

![alt text][image8]

#### 7. Lane Class

For processing the video, I implemented the `Line` class in cell 18. As I iterate through each frame of the video, this class stores information about the lane line and preceding frames.

I found this class to be instrumental for two reasons. First, it provides the context necessary to reject bad line detections within a single frame. Second, it allows for smoothing of the lane line across multiple frames of the video.

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./project_video_result-01.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

Once I had a working pipeline, I spent a lot of time tuning my line class and the logic involved in identifying valid lane detections. That is essentially the last component of the pipeline. To improve the pipeline further, I need to identify difficult sections of road/failure cases and build a test suite. With a test suite, I could iterate quickly while tuning upstream components of the pipeline.

## Writeup Template




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

[image1]: ./examples/undistort_output.png "undistort_output"
[image2]: ./examples/distortion-correction.png "distortion-correction"
[image3]: ./examples/combined_binary.png "combined_binary"
[image4]: ./examples/warped.png "warped"
[image5]: ./examples/line_fit.png "line_fit"
[image6]: ./examples/result.png "result"
[video1]: ./project_video.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) 

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

You're reading it!

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the 2nd code cell of the IPython notebook file. 

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

![alt text][image1]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

I created a functon `undistroted()`, which takes in an image, using `cv2.undistort()` with output from `cv2.calibrateCamera()`. 
One test image after applying the distortion correction is as below: 

![alt text][image2]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I created several color and and gradient functions including sobel, magnitude, direction, HSL space and LAB space. The code is in the 5th code cell in the IPython notebook. And after some experiments the final decision is using HSL L channel and LAB B channel combined, and a function `combined_binary()` was created. The reason is I found LAB B channel works best for yellow line detection while HSL L channel works best for white line detection. 
Below is example of output for this step. 

![alt text][image3]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform includes a function called `warpPerspective_from_img()`, which is in the 7th code cell in the IPython notebook. The `warpPerspective_from_img()` function takes as inputs an image, and the source (`src`) and destination (`dst`) points are defined inside the function.  I chose the hardcode the source and destination points in the following manner:

```python
    src = np.float32(
        [[(img_size[0] / 2) - 58, img_size[1] / 2 + 100],
        [((img_size[0] / 6) - 0), img_size[1]-10],
        [(img_size[0] * 5 / 6) + 48, img_size[1]-10],
        [(img_size[0] / 2 + 64), img_size[1] / 2 + 100]])
    dst = np.float32(
        [[(img_size[0] / 4), 0],
        [(img_size[0] / 4), img_size[1]],
        [(img_size[0] * 3 / 4), img_size[1]],
        [(img_size[0] * 3 / 4), 0]])
```

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![alt text][image4]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

I defined a function `new_fit()` to identify laneline pixels and fit their positions, which is in in the 9th code cell in the IPython notebook.

The first step in this function is to use histogram to fine the base points of both lines. Then use sliding window technic to all laneline pixels. The last step is to use polynominal function `np.polyfit()` to calculate the fit coefficients of both lines.  
 
I fit my lane lines with a 2nd order polynomial kinda like this:

![alt text][image5]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

I defined a function `cal_curverad_offset()` to calculate the radius of curvature of the lane and the position of the vehicle with respect to center, which is in in the 12th code cell in the IPython notebook.

This functions takes the image and the fit coefficients of both lines, as wells as the laneline pixels from function `new_fit()` as inputs. 
Left and right line curvatures are calculated separately, and the final lane curvature is the average of the two.

Assuming the center of the image is the center of the vehicle, while the average bottom pixel point of left and right line is the center of the lane, I can get the position of the vehicle with respect to lane center by the error of these 2 values, and its direction.

The most important thing in this step is to convert from pixels space to meters.

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I defined a function `warpPerspective_to_img()` to plot the detected lane mask back to the origianl image, which is in in the 11th code cell in the IPython notebook.

Here is an example of my result on a test image:

![alt text][image6]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

After finishing the pipeline for processing an image, I moved forward to work on video. Most steps are the same, but several are different.

First, after the 2 lines are well detected in one frame, the fit lines can be used in the next frame as start point, and skip the calculation of running sliding window again. Based on this, a function called `carry_over_fit()` is created, which takes previous fit coefficients and detect the lines in new frame. 

Second, a `line()` class was created to store important history data of line if line is detected in current frame or not, the last 5 fit coefficients, and the average best fit which can be used to avoid the lines jumping around, or be used when lane failed to be detected in certain frames.

Here's a [link to my video output easy project_video](./output_project_video.mp4)

Here's a [link to my video output challenge_video](./output_challenge_video.mp4)

To diagnose the performance of the code, I created a second pipeline called `process_diag()`, which shows the lane pixels detected and the fit lines with both average coefficients and current coefficient. From there, I can see the average fit is still working properly even when no line can be detected in a certain frame, or the dtected line flies away.

Here's a [link to my video output challenge_video_diag](./output_challenge_video_diag.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

In this project, two issues I spent most time on.

One, I had a hard time using color binary and HLS combined to fine the line, until I knew about LAB space, which works very well in finding yellow lines.

Two, when I was first working on video using `cv2.VideoCapture()`, I always failed in finding left line in challenge_video.mp4. I extracted all frames in the video and they worked fine as a single image. With the help from diagnose process, I finally find the problem: cv2 is using GBR space, instead of RGB... After making conversion before and after cv2.VideoCapture(), the video works pretty good! 

The harder_challenge_video is really hard... I gave a try, but the performance is still far away from acceptable. From diagnose processing mode, it looks in 1/3 of frames, the line can be seen clear, which may be a good start. And I need to spend time on it later. Here is the [line to my video output_harder_challenge_video_diag](./output_harder_challenge_video_diag.mp4)

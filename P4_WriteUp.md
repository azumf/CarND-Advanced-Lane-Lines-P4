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

### Image processing steps

#### 1. Camera Calibration

The code for this step is contained in the section 1.1 - 1.3 of the IPython notebook located in "CarND-Advanced-Lane-Lines-P4.ipynb".

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

![alt text][image1]
<img src="/output_images/chess_raw.png " width="600"/>

#### 2. Undistort an example image

I distortion-corrected the image by applying the `cv2.undistort` function by using distortion matrix mxt and dist.
Here an example of an undistorted image is shown:
<img src="/output_images/img_undist.png " width="600"/>

#### 3. Getting the binary image

I used a combination of color and gradient thresholds to generate a binary image. Here's an example of my output for this step.

#### 4. Gray-scale it
First, the image was converted to grayscale by appyling the cvtColor function of opencv.

<img src="/output_images/img_gray.png " width="600"/>

#### 5. Get the gradients
To identify bright points in the image the Sobel Operator is applyed to the image. Thereby the gradient between the pixels of the image is evaluated. The kernelsize describes the amount of pixels that are evaluated. By increasing the kernelsize, a smoother result could be achieved. Therefore, openCV has the Sobel() function. The thresholds for min and max gradient were 
```python
thresh_min = 50
thresh_max = 100
```
<img src="/output_images/img_sobel.png " width="600"/>

#### 6. Convert to HLS

To achieve better results the initial image is converted from RGB to HLS color space. The HLS color space has a color channel for lighting, that is way more suitable to detected lines.
To convert the initial image to HLS the `cvtColor()` function is applied.
The thresholds for min and max of the S-channel were (example image):
```python
s_thresh_min = 150
s_thresh_max = 255
``` 
<img src="/output_images/img_hls.png " width="600"/>

#### 7. Stack the filters

To combine the advantages of both, gradient and HLS-convertion, both filters are stacked to gain the binary output.
This step is described in 1.6 of the jupyter notebook.

<img src="/output_images/img_stacked.png " width="600"/>


#### 8. Perspective transformation

To identify the lane lines the perspective is transformed from "driver view" to a bird view. Therefore in a first step a proper section of the image that will be transformed is defined.
I chose the following pixel values as source (src):
```python 
x1 = 260 # 200
y1 = 720
x2 = 630 
y2 = 440 
x3 = 710 
y3 = 440 
x4 = 1130
y4 = 720

Drawn into the image:
<img src="/output_images/img_section.png " width="600"/>

The corresponding code can be found in the code cells of section 1.7 of the jupyter notebook.

As destination points, a rectangle was defined:
```python
dst = np.float32([[950, 0], [950, 720], [370, 720], [370, 0]])

First, the warp matrix M was calculated by applying the getPerspectiveTransform() function with src and dst as inputs.
To warp the image the warpPerspective() function was used. 
I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

<img src="/output_images/img_bird.png " width="600"/>

#### 9. Region of interest
TO filter noise from the images that was generated by appyling the gradient and color filter to the image a region on interest is defined. Therefore vertices are defined that edge the road lane. By using the fillPoly() and bitwise_and() functions of opencv the section defined by the vertices is kept as it was, but the surrounding pixels are set to zero. The result can be seen here:
<img src="/output_images/img_masked.png " width="600"/>

#### 10. Finding the lane lines
To find the lane lines on the warped image the image data is converted to a histogram classification using the pixel values. The maxima of the data plot are the two lane lines.
<img src="/output_images/lane_hist.png " width="600"/>

#### 11. Measuring the curvature
To measure the curvature the sliding windows approach is used. The corresponding code can be found in section 1.10 of the jupyter notebook. The image is scanned with a defined amount of windows from in inverted direction of the y-axis (max ==> min) and a polynomial fit is calculated for the lane line section in each window. The fit is calculated using the polyfit() function of NumPy.

<img src="/output_images/img_windows.png " width="600"/>

#### 12. Visualize the lane lines
The lane lines are visualized by getting the lane_line points coming from the polynomial fits.
<img src="/output_images/img_lanelines.png " width="600"/>

#### 13. calculate the curvature
In the next step, 1.12 in the jupyter notebook, the curvature of the lane lines is calculated. This can be achieved by appyling the function that was thaught in the lesson to the left and the right plot respectively. The results are the curvatures in pixel values. To convert pixel to metres, the parameters
```python
lane_length = 30
lane_width = 3.7 
ym_per_pix = 30/720
xm_per_pix = 3.7/580

are used.
As well the distance of the vehicle from the middle of the lane can be calculated by using the midpoint of the x-axis.

#### 14. Warp back to driver view
To display the lane lines and the region between them to the street, the image is warped back to the driver view by appyling the warpPerspective() function of opencv again.
<img src="/output_images/img_warpback.png " width="600"/>


### Building the pipeline
In section 2 of the jupyter notebook the built pipeline can be found. The pipeline contains all of the steps described above coded as functions.
The pipeline() function itself applies all of the other functions in the right order and additionally takes into account global variables to smooth and average the calculated lane lines of the last n frames.


---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./project_video.mp4)

PIPELINE POINTS
| Source        | Destination   | 
|:-------------:|:-------------:| 
| 585, 460      | 320, 0        | 
| 203, 720      | 320, 720      |
| 1127, 720     | 960, 720      |
| 695, 460      | 960, 0        |



---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

Here I'll talk about the approach I took, what techniques I used, what worked and why, where the pipeline might fail and how I might improve it if I were going to pursue this project further.  

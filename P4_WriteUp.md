## Writeup Project 4:
## Advanced Lane Finding

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

## Image processing
The workflow for the image processing contains the following steps:
1. Camera Calibration
2. Image distortion
3. Getting the binary image
4. Gray-scale the image
5. Apply Sobel Operator to calculate the gradients
6. Convert the image in HLS color space
7. Stack both image processing steps
8. Warp the perspective
9. Filter the region of interest
10. Find Lane Lines by pixel evaluation long the x-axis
11. Measuring the curvature and fit polynomial
12. Re-project lines region back to the road
13. Calculate curvature and distance from midpoint of the lane

#### 1. Camera Calibration

The code for this step is contained in the section 1.1 - 1.3 of the IPython notebook located in "CarND-Advanced-Lane-Lines-P4.ipynb".

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

![chessboard](/output_images/chess_raw.png)


#### 2. Undistort an example image

I distortion-corrected the image by applying the `cv2.undistort` function by using distortion matrix mxt and dist.
Here an example of an undistorted image is shown:
![undistorted chess board](/output_images/img_undist.png)

#### 3. Getting the binary image

I used a combination of color and gradient thresholds to generate a binary image. In the next steps an explanation of the applied functions is decribed.

#### 4. Gray-scale it
First, the image was converted to grayscale by appyling the cvtColor function of opencv.


![gray-scale](/output_images/img_gray.png)

#### 5. Get the gradients
To identify bright points in the image the Sobel Operator is applyed to the image. Thereby the gradient between the pixels of the image is evaluated. The kernelsize describes the amount of pixels that are evaluated at one time. By increasing the kernelsize, a smoother result could be achieved. Therefore, openCV has the Sobel() function.  Here, the thresholds for min and max gradient were: 
```python
thresh_min = 50
thresh_max = 100
```

![sobel operator](/output_images/img_sobel.png)

#### 6. Convert to HLS

To achieve better results the initial image is converted from RGB to HLS color space. The cylindrical color space contains a color channel for the luminence, that is way more suitable to detected lines.
To convert the initial image to HLS the `cvtColor()` function is applied.
The thresholds for min and max of the S-channel were (example image):
```python
s_thresh_min = 150
s_thresh_max = 255
``` 

![hls filter](/output_images/img_hls.png)

Here, the lane lines are quite good visible and distinguishable from the surroundings.

#### 7. Stack the filters

To combine the advantages of both, gradient detection by applying the Sopel Operator and HLS-convertion with evaluation of the luminence color channel, both filters are stacked to gain the binary output.
This step is described in 1.6 of the corresponding jupyter notebook.

![Binary output](/output_images/img_stacked.png)


#### 8. Perspective transformation

To identify the lane lines the perspective is transformed from "driver view" to a bird view. Therefore in a first step a proper section of the image that will be transformed is defined.
I chose the following pixel values as source (src) values for transformation:
```python 
x1 = 260 # 200
y1 = 720
x2 = 630 
y2 = 440 
x3 = 710 
y3 = 440 
x4 = 1130
y4 = 720
```

Drawn into the image:
![region to transform](/output_images/img_section.png)

The corresponding code can be found in the code cells of section 1.7 of the jupyter notebook.

As destination points, a rectangle `dst` was defined:
```python
dst = np.float32([[950, 0], [950, 720], [370, 720], [370, 0]])
```

First, the warp matrix `M` was calculated by applying the `getPerspectiveTransform()` function with `src` and `dst` as inputs.
To warp the image the `warpPerspective()` function was used. 
I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![image transformation](/output_images/img_bird.png)

#### 9. Region of interest
To filter noise from the images that was generated by appyling the gradient and color filter to the image a region on interest is defined. Therefore vertices are defined that edge the road lane. By using the `fillPoly()` and `bitwise_and()` functions of opencv the section defined by the vertices is kept as it was, but the surrounding pixels are set to zero. The result can be seen here:
![region of interest](/output_images/img_masked.png)

This step can be very helpful to filter noise like changes of road surface. The following image shows a situation in which this region of interest filter increased the performance.
![region of interest2](/output_images/img_interest.png)
The red arrows show a change of road surface due to construction work or whatever. If this region around the midpoint of the lane is not filtered, the algorithm will evaluate this surface change as a lane line since the gradient of the pixels between left and right of the arrows is quite high.

#### 10. Finding the lane lines
To find the lane lines on the warped image the image data is converted to a histogram classification using the pixel values along the x-axis. The maxima of the data plot are the two lane lines.

![histogram](/output_images/lane_hist.png)

#### 11. Measuring the curvature
To measure the curvature and fit a polynomial the sliding windows approach is used. The corresponding code can be found in section 1.10 of the jupyter notebook. The image is scanned with a defined amount of windows from in inverted direction of the y-axis `max --> min` and a polynomial fit is calculated for the lane line section in each window. The fit is calculated using the `polyfit()` function of NumPy.

![sliding windows](/output_images/img_windows.png)

#### 12. Visualize the lane lines
The lane lines are visualized by getting the `lane_line` points coming from the polynomial fits.

![lane lines visualization](/output_images/img_lanelines.png)

#### 13. Calculate the curvature
In the next step, 1.12 in the jupyter notebook, the curvature of the lane lines is calculated. This can be achieved by appyling the function that was thaught in the lesson to the left and the right plot respectively. The results are the curvatures in pixel values. To convert pixel to metres, the parameters
```python
lane_length = 30
lane_width = 3.7 
ym_per_pix = 30/720
xm_per_pix = 3.7/580
```

are used.
As well the distance of the vehicle from the middle of the lane can be calculated by using the midpoint of the x-axis.

#### 14. Warp back to driver view
To display the lane lines and the region between them to the street, the image is warped back to the driver view by appyling the `warpPerspective()` function of opencv with the inverted Warp-Matrix `M_inv`.
![warp back](/output_images/img_warpback.png)
___
## Building the pipeline
In section 2 of the jupyter notebook the built pipeline can be found. The pipeline contains all of the steps described above coded as functions.
The `pipeline()` function itself applies all of the other functions in the right order and additionally takes into account global variables to smooth and average the calculated lane lines of the last n frames.
The global variables are:
```python
filter_size #amount of previous frames to average
old_img_lines #lane lines of previous frame
l_fit_buffer #buffer storage for polynomial fits for left line
r_fit_buffer #buffer storage for polynomial fits for right line
```
The `old_img_lines` and buffer storage is updated if the new frame and the previous frame match. The percentage of similarity is determined by applying the `matchShapes()` function of opencv.

To check the pipeline on test images, I used the existing test images but as well created new ones from the provided videos that might be a little bit more tricky than the existing ones. This step can be found in section 4 of the jupyter notebook.  
For example this one with changes of the light and shadows projected to the lane by surroundings:
![pipeline test](/output_images/img_pipeline.png)

---

## Apply pipeline to project video

The chosen thresholds for the video processing were:
```python
filter_size = 9
old_img_lines = None
l_fit_buffer = None
r_fit_buffer = None
def process_image(image):
    # kernel_size, thresh_min, thresh_max, s_thresh_min, s_thresh_max
    processed_img = pipeline(image, mtx, dist, 5, 35, 100, 115, 255)
    return processed_img
```


The used `src` and `dst` points for the pipeline were:


| Source        | Destination   | 
|:-------------:|:-------------:| 
| 620, 455      | 370, 0        | 
| 240, 720      | 370, 720      |
| 1130, 720     | 950, 720      |
| 730, 455      | 950, 0        |

Here's a [link to my video result](./project_video_processed.mp4)

---

## Further Discussion

##### Problems / issues
During the project I got stuck at finding the right values for the thresholds for different situations. Some will work great at sunlight but they will fail in driving situations with shadows on the road.  
As well the defintion of proper `src` and `dst` points for image transformation was a crucial point to achieve good results with the polynomial fits through the lines.  
Another tricky part I did not manage to implement in this submission was the `Class()` object to average and smooth the found lane lines over the last n frames. I checked the discussion forums but did not mange it. However,  I will try to implement it in the next time after the submission.  
**Do you have some kind of example code or detailed explanation how to implement the class the right way?**  
Would be a great help! Instead of the class object I used the global variables. I know that a class object would be preferred.

##### Improvements
It was a great project, I learnt a lot of stuff! However, my pipeline worked quite well on the project video. On the challenge videos the used pipeline has weaknesses and is not robust enough to find the lane lines in each situation. In the next time after this project submission I will continue to work on this to achieve better results on the challenge videos as well.
To enhance the pipeline performance and make it more robust I will try to implemented the following improvements:

- Implement the class object
- Implement a dynamic determination of threshold values depending on the actual lighting / illumination of the road
- tweak the `src` and `dst` points and the vertices for the region of interest filter
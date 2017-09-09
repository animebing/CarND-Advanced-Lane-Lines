# Advanced lane line detection

## Project summary
In this project, first, I find the way to do calibration with opencv python version and review the process about how to do camera calibration. Second, I find why to use horizontal gradient and S(Saturation) channel in HLS space to threshold the image. Then do perspective transformation and window searching to find left and right lane line. At last, transform back to the original image

---
## Camera calibration
Camera calibration is the process to get camera intrinsic parameter, distortion parameter and extrinsic paramter, in this project, computed intrinsic parameter and distortion parameter are used to distort image, the shape, size and location of objects are distorted because of distortion, so it is important to do image distortion before any other operation.

the classic camera calibration method is proposed by Zhengyou Zhang, you can find it in [http://www.vision.caltech.edu/bouguetj/calib_doc/papers/zhan99.pdf](http://www.vision.caltech.edu/bouguetj/calib_doc/papers/zhan99.pdf), the recommended calibration pipeline are 

* Print a pattern(chessboard here) and attach it to a planar surface
* Take a few images of the pattern plane under different orientations by moving camera or the patter plane
* Detect the feature points(chessboard corners here) in the images
* Estimate the five intrinsic parameters and six extrinsic parameters
* Estimate the coefficient of the radial distortion
* Refine all parameters by minimizing the projection error

the corresponding opencv python tutorial can be found in [http://docs.opencv.org/3.3.0/dc/dbb/tutorial_py_calibration.html](http://docs.opencv.org/3.3.0/dc/dbb/tutorial_py_calibration.html), below is the original chessboard and undistored chessboard using calibration results

![chessboard](https://github.com/animebing/CarND-Advanced-Lane-Lines/tree/master/output_images/original_distorted.jpg "Original and Undistorted Chessboard")

## Pipeline(single image)

### undistort image
now we can undistort a road image using computed calibration results as below-it is not quite as obvious of a change as the chessboard is, but the same undistortion is being done to it.

![road](https://github.com/animebing/CarND-Advanced-Lane-Lines/tree/master/output_images/undistort_road.jpg "Original and Undistorted road image")

### threshold image

Here I am going to threshold image to find lane line, in above image, the left lane line is yellow and right laneline is white
---
#### R(Red), G(Green), B(Blue) channel image

![RGB channels](https://github.com/animebing/CarND-Advanced-Lane-Lines/tree/master/output_images/RGBchannels.jpg "R G B channel images")

for the white lane line, the R,G,B channels are similar, but for yellow lane line, it is more obvious in R channel image than other two channels
---
#### H(Hue), L(lightness), S(Saturation) channel image

Althogh yellow lane line is more obvious in R channel, but the contrast between lane line and surrounding road is not very high, so here we can try to convert image from RGB space to HLS space, below is corresponding image for each channel

![HLS channels](https://github.com/animebing/CarND-Advanced-Lane-Lines/tree/master/output_images/HLSchannels.jpg "H L S channel images")

in S channel images, the yellow lane can be easily recoginized, compared to that in R channel image, it has much higher constrast with surrounding road, so next I use S channel to find yellow lane line
---
#### Sobel operator for gradient

here we use Sobel gradient to compute the gradient in horizontal and vertical direction, here I will use another image(**I2, the former image is I1**) to demostrate the result of Sobel operator, because in the gray scale image of I1, it is hard to tell road and lane line as below, that is also the reason why to use other color info to find lane line

![I1](https://github.com/animebing/CarND-Advanced-Lane-Lines/tree/master/output_images/scale_1.jpg "scale image")

I2 and its gray scale image is as below

![I2](https://github.com/animebing/CarND-Advanced-Lane-Lines/tree/master/output_images/scale_2.jpg "scale image")

then I use (40, 100) to threshold horizontal, vertical and total gradient magnitude, use (pi/6, pi/3) to threshold abs gradient direction, the resulting image is below

![sobel](https://github.com/animebing/CarND-Advanced-Lane-Lines/tree/master/output_images/sobel.jpg "sobel image")

we can get these three magnitude image is similar except for some horizontal lines in vertical and total magtitude images, and the gradient direction image is more like noise map, based on above, horizontal gradient is use to find lane line
---
#### combine horizontal gradient and S channel 
use (40, 100) to threshold horizontal gradient magnitude and (180, 255) to threshold S channel, then combine these two mask by "or" operation, separating mask and combined mask is as below

![combined mask](https://github.com/animebing/CarND-Advanced-Lane-Lines/tree/master/output_images/combined.jpg "combined mask image")

in combined mask image, there is a red polygon, the area within it is choosed and other parts in the image is abandoned, this trick is used to filter noise points and the area is hand designed
---
#### perspective transformation
transform choosed area to bird-eye view (from top to down), in this view, the left and right lane line is approximately parallel, the points used to compute perspective matrix is as below

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 120, 720      | 200, 720        | 
| 550, 470     | 200, 0      |
| 780, 470      | 1080, 0     |
| 1200, 720      | 1080, 720        |

then image I2, warped image, and warped mask is as below

![warped](https://github.com/animebing/CarND-Advanced-Lane-Lines/tree/master/output_images/warped.jpg "warped image")
---
#### histogram, window serching and lane line fitting
for the warped mask, compute the number of points in each horizontal position in a histgram, find the peak in left and right part as the initial position to find points in corresponding lane line, then find all left lane points and right lane points along vertical direction, at last, fit these points with quadratic curve. The histgram, searching windows and fitting curve is as below

![hist_search_fit](https://github.com/animebing/CarND-Advanced-Lane-Lines/tree/master/output_images/hist_search_fit.jpg "histogram, searching windows and fitting lane line images")
---
#### draw lane line area and transfrom back to undistored image
draw lane line area in warpped mask in green, then tranform this area back and combine with undistored image, before and after transformation image is as below

![laneline](https://github.com/animebing/CarND-Advanced-Lane-Lines/tree/master/output_images/laneline.jpg "lane line images")

### Pipeline(Video)
for project_video.mp4, I process each frame using the pipeline above for single image, in most frame, it is okay, but when there is shadow, the pipeline fails because shadow area in S channel image has large value, which intorduces many noise points, the S channel mask, warped mask, searching window and fitting lane are as below










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

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

![alt text][image1]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

To demonstrate this step, I will describe how I apply the distortion correction to one of the test images like this one:
![alt text][image2]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I used a combination of color and gradient thresholds to generate a binary image (thresholding steps at lines # through # in `another_file.py`).  Here's an example of my output for this step.  (note: this is not actually from one of the test images)

![alt text][image3]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform includes a function called `warper()`, which appears in lines 1 through 8 in the file `example.py` (output_images/examples/example.py) (or, for example, in the 3rd code cell of the IPython notebook).  The `warper()` function takes as inputs an image (`img`), as well as source (`src`) and destination (`dst`) points.  I chose the hardcode the source and destination points in the following manner:

```python
src = np.float32(
    [[(img_size[0] / 2) - 55, img_size[1] / 2 + 100],
    [((img_size[0] / 6) - 10), img_size[1]],
    [(img_size[0] * 5 / 6) + 60, img_size[1]],
    [(img_size[0] / 2 + 55), img_size[1] / 2 + 100]])
dst = np.float32(
    [[(img_size[0] / 4), 0],
    [(img_size[0] / 4), img_size[1]],
    [(img_size[0] * 3 / 4), img_size[1]],
    [(img_size[0] * 3 / 4), 0]])
```

This resulted in the following source and destination points:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 585, 460      | 320, 0        | 
| 203, 720      | 320, 720      |
| 1127, 720     | 960, 720      |
| 695, 460      | 960, 0        |

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![alt text][image4]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

Then I did some other stuff and fit my lane lines with a 2nd order polynomial kinda like this:

![alt text][image5]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

I did this in lines # through # in my code in `my_other_file.py`

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

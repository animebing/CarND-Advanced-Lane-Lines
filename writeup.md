# Advanced lane line detection

## Project summary
In this project, first, I find the way to do calibration with opencv python version and review the process about how to do camera calibration. Second, I find why to use horizontal gradient and S(Saturation) channel in HLS space to threshold the image. Then do perspective transformation and window searching to find left and right lane line. At last, transform back to the original image


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

#### R(Red), G(Green), B(Blue) channel image

![RGB channels](https://github.com/animebing/CarND-Advanced-Lane-Lines/tree/master/output_images/RGBchannels.jpg "R G B channel images")

for the white lane line, the R,G,B channels are similar, but for yellow lane line, it is more obvious in R channel image than other two channels

#### H(Hue), L(lightness), S(Saturation) channel image

Althogh yellow lane line is more obvious in R channel, but the contrast between lane line and surrounding road is not very high, so here we can try to convert image from RGB space to HLS space, below is corresponding image for each channel

![HLS channels](https://github.com/animebing/CarND-Advanced-Lane-Lines/tree/master/output_images/HLSchannels.jpg "H L S channel images")

in S channel images, the yellow lane can be easily recoginized, compared to that in R channel image, it has much higher constrast with surrounding road, so next I use S channel to find yellow lane line

#### Sobel operator for gradient

here we use Sobel gradient to compute the gradient in horizontal and vertical direction, here I will use another image(**I2, the former image is I1**) to demostrate the result of Sobel operator, because in the gray scale image of I1, it is hard to tell road and lane line as below, that is also the reason why to use other color info to find lane line

![I1](https://github.com/animebing/CarND-Advanced-Lane-Lines/tree/master/output_images/scale_1.jpg "scale image")

I2 and its gray scale image is as below

![I2](https://github.com/animebing/CarND-Advanced-Lane-Lines/tree/master/output_images/scale_2.jpg "scale image")

then I use (40, 100) to threshold horizontal, vertical and total gradient magnitude, use (pi/6, pi/3) to threshold abs gradient direction, the resulting image is below

![sobel](https://github.com/animebing/CarND-Advanced-Lane-Lines/tree/master/output_images/sobel.jpg "sobel image")

we can get these three magnitude image is similar except for some horizontal lines in vertical and total magtitude images, and the gradient direction image is more like noise map, based on above, horizontal gradient is use to find lane line

#### combine horizontal gradient and S channel 
use (40, 100) to threshold horizontal gradient magnitude and (180, 255) to threshold S channel, then combine these two mask by "or" operation, separating mask and combined mask is as below

![combined mask](https://github.com/animebing/CarND-Advanced-Lane-Lines/tree/master/output_images/combined.jpg "combined mask image")

in combined mask image, there is a red polygon, the area within it is choosed and other parts in the image is abandoned, this trick is used to filter noise points and the area is hand designed

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

#### histogram, window serching and lane line fitting
for the warped mask, compute the number of points in each horizontal position in a histgram, find the peak in left and right part as the initial position to find points in corresponding lane line, then find all left lane points and right lane points along vertical direction, at last, fit these points with quadratic curve. The histgram, searching windows and fitting curve is as below

![hist_search_fit](https://github.com/animebing/CarND-Advanced-Lane-Lines/tree/master/output_images/hist_search_fit.jpg "histogram, searching windows and fitting lane line images")
---
#### draw lane line area and transfrom back to undistored image
draw lane line area in warpped mask in green, then tranform this area back and combine with undistored image, before and after transformation image is as below

![laneline](https://github.com/animebing/CarND-Advanced-Lane-Lines/tree/master/output_images/laneline.jpg "lane line images")

### Pipeline(Video)
for project_video.mp4, I process each frame using the pipeline above for single image, in most frame, it is okay, but when there is shadow, the pipeline fails because shadow area in S channel image has large value, which intorduces many noise points, the undistorted image and S channel image are as below

![shadow_s](https://github.com/animebing/CarND-Advanced-Lane-Lines/tree/master/output_images/shadow_s.jpg "shadow and s channel images")

the resulting mask image, searching window and final lane line images are as below

![shadow_final](https://github.com/animebing/CarND-Advanced-Lane-Lines/tree/master/output_images/shadow_final.jpg)

because shadow causes lightness change, so I am trying to solve it using L(Lightness) channel in HLS space, corresponding L and R channel images are as below

![ls_channel](https://github.com/animebing/CarND-Advanced-Lane-Lines/tree/master/output_images/ls_channel.jpg "Lightness and Saturation channel images")

the shadow part in L channel image is much darker, I use (100, 200) to threshold L channel image, then do "and" operation to L mask ans S mask, at last do "or" operation with horizontal gradient to get the final mask, corresponding masks are as below

![final_ls_mask](https://github.com/animebing/CarND-Advanced-Lane-Lines/tree/master/output_images/final_ls_mask.jpg)

there isn't too much points in ls_mask, but the horizontal mask contains enough points to find lane line, by using L channel image, the shadow points are filtered out, this strategy works well in image without shadow. After this process, the searching window and final lane line image are as below

![shadow_ls_mask](https://github.com/animebing/CarND-Advanced-Lane-Lines/tree/master/output_images/shadow_ls_final.jpg)

Here's the [link] for the result of project_video.mp4(./project_video_color.mp4)

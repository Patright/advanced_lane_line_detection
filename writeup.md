## Writeup Advanced Lane Finding Project

---

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


[image1]: ./examples/chessboard_image.png "Example image"
[image2]: ./examples/chessboard_image_with_corners_drawn_on_it.png "Chessboard"
[image3]: ./examples/chessboard_image_undistorted.png "Undistorted"
[image4]: ./examples/perspective_transform.png "Warp Example"
[image5]: ./examples/sample_straight_lines1.jpg
[image6]: ./examples/sample_straight_lines2.jpg
[image7]: ./examples/sample_test1.jpg
[image8]: ./examples/sample_test2.jpg
[image9]: ./examples/sample_test3.jpg
[image10]: ./examples/sample_test4.jpg
[image11]: ./examples/sample_test5.jpg
[image12]: ./examples/sample_test6.jpg
[image13]: ./examples/histogram.png
[image14]: ./examples/sliding_windows.png
[image15]: ./examples/search_around_poly.png
[image16]: ./examples/curvature_formula.png
[image17]: ./examples/projected_lane_lines.png
[image18]: ./examples/new_thresholds.png

[video1]: ./project_video.mp4 "Video"

---

### STEP 1: Camera Calibration

Real cameras use curved lenses. With curved lenses the light arrays often bend to much at the edges. That leads to radial distortion. Radial distortion describes the effect that objects appear more or less curved than they really are. Tangential distortion occurs when the camera lens is not alligned perfectly parallel to the imaging plane where the camera sensor is placed. Thus objects may appear farther away or closer than they really are. To address that issues the camera has to be calibrated.

The calibration pipline consists of the following steps: 

0. Take at least 10 pictures of a well defined object (i.e. a Chessboard) with the camera from different angels and distances. For this project these images can be found in the `/camera_cal` folder 
1. Convert images to grayscale with opencv's function `cv2.cvtColor(img,cv2.COLOR_BGR2GRAY)`

![alt text][image1]

2. Find Chessboardcorners with `cv2.findChessboardCorners(gray, (nx,ny),None)`

![alt text][image2]

3. Compute the distortion coefficients and the camera matrix. The camera matrix is needed to transform 3D object points into 2D image points by using `cv2.calibrateCamera(objpoints, imgpoints, image.shape[::-1],None,None)`
4. Both, the distortion coefficients and the camera matrix are then used to undistort the picture by using the function `cv2.undistort(image, mtx, dist, None, mtx)`  

![alt text][image3]

The program code for the camera calibration step can be found in the second cell of the main jupyter notebook.

---

### STEP 2: Perspective Transform, warp undistorted images to birds-eye-view

My goal is to measure the curvature of the lane lines.  
To reach that goal I first have to warp the undistorted images to birds-eye-view.  
The pipeline includes the following steps:  
1. Choose four points on original image (-> source points `src`)
2. Determine four points for the transformed image (-> destination points `dst`)
|src points  | dst points |
|:-----------|:-----------|
|(579, 460)  |(320,   0)  |
|(193  720)  |(320, 720)  |
|(1127,720)  |(960, 720)  |
|(703, 460)  |(960,   0)  |

3. Use that points to compute the perspective transform matrix `M` with `cv2.getPerspectiveTransform(src, dst)`
4. Finally create the warped image using linear interpolation to fill in the missing points with the openCV function `cv2.warpPerspective(undist, M, img_size, flags=cv2.INTER_LINEAR)` 

Here is are example images to illustrate the pipeline:  

![alt text][image4]

---

### STEP 3: Lane line detection  

Now it is time to detect the lane lines.  
I did that by writing the function `combined_lane_line_detection()` it takes as input the warped image and outputs a binary image with the pixels that belong to the lane lines.
At first I used a Sobel x Operator on the grayscaled image to take the derivative in x of every pixel. To accentuate lines away from horizontal I used a minimal threshold of 20 and a maximal threshold of 100 for the x gradient. Secondly I separated the Saturation-Channel of the image of the HSL color space with a threshold from 80 to 255 in order to identify pixels in that range. A full saturation means that the pure base hue is used. 0 saturation will always be black. And as a third step I took the L-Channel with minimal threshold of 191 and a maximal threshold of 255 in order to identify the pixels with high value for Lightness.  
I applied that function on all test images in order to see how good it works and where I have to adjust the parameters.  
In the left column is the original image, then follows the warped image. Next I placed an image with the stacked thresholds: The green pixels identify the pixels detected by the sobel x Operator. The red pixels are the ones identified by the Lightnessfilter and the blue ones are from the Saturationchannel. The image all to the right shows the final result, the binary image with the detected lane line pixels. This results worked well on the first video. 

![alt text][image5]

![alt text][image6]

![alt text][image7]

![alt text][image8]

![alt text][image9]

![alt text][image10]

![alt text][image11]

![alt text][image12]

---

### STEP 4: Find lane lines and fit a polynomial for measuring the curvature

In this step I identified the pixels that really belong to the lane lines. To achive that I used two strategies. A first one to identify the pixels initially on the hole image, just for the first image in the movie. And than the sceond one, once we have a detection, I used that information to search around the previous detection with some offset to the left and to the right of the preceding lane line, to speed up and to focus on the most likely area.
For the intial detection I used the function `fit_poly_with_windows()`. It takes the binary image as input and outputs the identified pixels, the polynomials for the line fit in pixels and in meters.  
It works as follows:
1. Take bottom half of the image and get the histogram with the pixelcount
2. Identify the region on the left half and on the right half of the image with the highest amount of pixels. These peaks are used as starting points of the left and the right lane line. 

![alt text][image13]

3. Once the starting points are clear, I used windows to search for pixels in that area and from that first finding upwards with sliding windows from pixel cloud to pixel cloud until the upper edge of the picture is reached. Than I took the detected pixels and fitted a second order polynomial for getting the curvature of the lane lines. Here is an example of the result. The red pixels are the detection for the left lane line, the blue ones for right lane line and the fitted polynomials are shown in yellow.

![alt text][image14]

4. For detection of the following lines I searched around the fitted polynomials with `fit_poly_around_last_poly()` from the previous image:

![alt text][image15]

For the conversion from pixels to meters, I used:  
Meters per pixel in x dimension: xm_per_pix = 3.7/700  
Meters per pixel in y dimension: ym_per_pix = 30 /720  

---

### STEP 5: Calculate the radius of curvature in meters for both lane lines

To calculate the radius of curvature in meters for both lane lines I created the function `measure_curvature()`. In that function I used this formular to compute the curvature:  

![alt text][image16]

---

### STEP 6: Draw lane lines area and informations about curvature

With the function `project_lane_lines()` I project the detected lane lines onto the image. Together with the calculated curvature and the offset of the detection from the center of the image. This helps to visually evaluate the quality of the detection.

![alt text][image17]

---

### STEP 7: Implement a Class for tracking each lane line 

I wrote the class `Line()` for tracking the characteristics of each line detection such as:  
- Values for detected line pixel values
- Polynomial coefficeints in pixel and in meter
- Radius of curvature of the line
- Distance in meters of vehicle center from the line
- Difference in polynomial fit between last and new fits
- flag for line detection  

I also added some setter methods and integrated the methods `measure_curvature()`, `fit_polynomial()`, `search_pix_around_last_detection()`, `search_pix_with_windows()`.

---

### STEP 8: Pipline for the lane line detection

As first step (*STEP  1*) I initialised the source and destination points for the warp function and instantiated two objects of the `Line()` Class:   
- one for the left lane line and 
- one for the right lane line.  

Then I programmed the pipeline function `pipeline_lane_detection()` which will be applied to each frame of the video including the following steps:  

*STEP  2* : Perspective Transform to birds-eye view  
*STEP  3* : Combined Color and Gradient Threshold for lane line detection  
*STEP  4* : Collect pixels from lane line detection  
*STEP  5* : Fit polynomials  
*STEP  6* : Calculate the current radius of the lane line curvature in meters  
*STEP  7* : Set distance of vehicle center from the line  
*STEP  8* : Check position compared to image center   
*STEP  9* : Draw lane lines area onto the road with additional informations

---

### STEP 9: Use workflow on video

I used the pipeline on the video `project_video.mp4`. Here's a [link to my video result](./project_video_output.mp4).

---

### STEP 10: Use workflow on challenge video

Here is how I modified the pipline in order to get an accetable result on the `challenge_video.mp4`:  
- Update the source points to destination points mapping:  
|dst       |src       |
|:--------:|:--------:|
|(320,   0)|(625, 490)|
|(320, 720)|(428, 720)|
|(960, 720)|(937, 720)|
|(960,   0)|(715, 490)|

 
- Reduction of the threshold for the sobel x operator to a range between 20 and 23, because of interference contours that would otherwise have been detected as lane lines  
- Widening of the threshold range for the saturation value, which leads to a significantly better detection of the left yellow lane line   

To find the best threshold I wrote a pipeline to store frames from the video. The pipeline can be found in the jupyter notebook `Video2Frame.ipynb`. Then I selected some of the images to use different values for the sobel x, the lightness and the saturation on them in order to identify the most helpful ones.  
Here are some sample images with the final adjustement of the thresholds:   

![alt text][image18]




Here's a [link to my challenge video result](./challenge_video_output.mp4).

---

### Discussion

Under the given circumstances the workflow for the lane line detection in both videos worked well.   

But my pipeline will likely fail under different weather conditions like snow or rain, at night, with more traffic and roads with interrupted or very pale lane lines.  

To adress these issues more checks and different kinds of detection can be inplemented. With these informations better matching thresholds and filters for the lane line detection can be chosen.  
Another even more promising idea is to use machine learning, especially supervised learning. By feeding the computer with images of all kinds of lane lines, weather and light conditions, the computer should be able to fit an algorithm for a reliable lane line detection.  



## Advanced Lane Finding
### In this project, various computer vision techniques are used to track lanes in different weather conditions and approximate the radii of curvature and offsets of the vehicle to the center of the lane. This project serves as an essential component in the final self-driving car software pipeline as it provides the necessary information for controlling the steering angle of the car.
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
* Produce final video based on dashcam recording

[//]: # "Image References"

[image0]: ./output_images/undistorted_cal.jpg "Undistorted Calibration"
[image1]: ./output_images/original.jpg "Original"
[image2]: ./output_images/undistorted.jpg "Undistorted"
[image3]: ./output_images/thresholds.png "Thresholded Images"
[image4]: ./output_images/binary_warped.jpg "Binary War"
[image5]: ./output_images/polyfit_img.jpg "Polynomial Fit In Progress"
[image6]: ./output_images/polyfit.jpg "Polynomial Fit Final"
[image7]: ./output_images/result.jpg "Final Image"
[video1]: ./processed_out.mp4 "Video"
[]: 

###Camera Calibration

####1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The first step is to prepare the object and image points, with the function `get_obj_and_img_points_from_calibration_images()` in Advanced-Lane-Lines.ipynb. For every calibration chessboard image, the object points are the (x, y, z) coordinates of the chessboard corners in the world. Assuming the chessboard is in the (x, y) plane with z=0, the object points are the same for all calibration images. Then, using OpenCV's `cv2.findChessboardCorners()` function, we find all the corners from the chessboard images, and save their (x, y) pixel positions in the image plane as image points.

Next, the resulting `objpoints` and `imgpoints` are used with the`cv2.calibrateCamera()` function to get the camera matrix and distortion coefficients.

Using the `cv2.undistort()` function, we test the calibration on one of the chessboard images, and confirm that the original image's distortions have been corrected, as shown in Figure 1.

![alt text][image0]

**Figure 1. Undistorted Chessboard Image After Camera Calibration**



###Pipeline (single images)

####1. Provide an example of a distortion-corrected image.
Similarly to the chessboard image above, we call the `cv2.undistort` function on a test image, with the camera matrix and distortion coefficients previously calculated.
![alt text][image1]

**Figure 2. Original Dashcam Image**

![alt text][image2]

**Figure 3. Undistorted Dashcam Image**

We can notice the white car on the right side has been stretched longer in the undistorted image, which suggests that the car was compressed in the original image.



####2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

The thresholded binary image was generated using a combination of gradient and color thresholds. The helper functions `abs_sobel_thresh()`, `mag_thresh()`, `dir_threshold()`, `hls_select_s()` generate the different binary images which we combine in `get_binary()` to obtain the final binary image.

![alt text][image3]

**Figure 4. Different Thresholded Binary Images**

The thresholds are chosen by trying a variety of road and lighting conditions found in the test images. For most examples, the gradient based images work quite well. However, for this specific test image, the brightness of the yellow line and of the road are almost indistinguishable once converted to grayscale. In order to take advantage of the color information in the image, we convert the image into HLS color-space and perform a threshold filtering on the Saturation channel. As shown in figure 4, the resulting combined binary image shows the lanes quite clearly.

####3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The perspective transform is done in the function called `get_warped()`. 

First, the source and destination points are generated using the following formulae, found from tracing points on a straight lane so the trapezoid in the original image can be transformed into a rectangle.

```
    src = np.float32(
        [[(img_size[0] / 2) - 55, img_size[1] / 2 + 100],
        [((img_size[0] / 6) + 10), img_size[1]],
        [(img_size[0] * 5 / 6) + 55, img_size[1]],
        [(img_size[0] / 2 + 75), img_size[1] / 2 + 100]])
    dst = np.float32(
        [[(img_size[0] / 5), 0],
        [(img_size[0] / 5), img_size[1]],
        [(img_size[0] * 4 / 5), img_size[1]],
        [(img_size[0] * 4 / 5), 0]])
```
This resulted in the following source and destination points:

|  Source   | Destination |
| :-------: | :---------: |
| 585, 460  |   256, 0    |
| 223, 720  |  256, 720   |
| 1121, 720 |  1024, 720  |
| 715, 460  |   1024, 0   |

The source points are verified by plotting the trapezoid they form onto the undistorted image, as shown in figure 3 above. The destination points can be confirmed to form a rectangle by looking at their coordinates.

Using these points with the `cv2.getPerspectiveTransform()` function, we get the transform matrix M, which we then use with `cv2.warpPerspective()` to get the warped image, shown in figure 5.

![alt text][image4]

**Figure 5. Warped Binary Image Showing the Lanes**



####4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

The `find_poly_fit_using_histogram()` function takes a binary warped image, finds the lane pixels, and fits polynomial lines through them.

First, a histogram taken along the x-axis to find the pixel columns that have the most white pixels. Those are most likely to be the lanes we are looking for. Next, the image height is divided by 9 to fit nine 100-pixel-wide windows for searching lanes. Starting from the bottom of the image, we keep track of all the non-zero pixels within the windows. If the number of non-zero pixels per window is greater than a threshold of 50, we re-center the window to their mean x-position and continue the lane searching.

The resulting image is shown in figure 6.

![alt text][image5]

**Figure 6. Using Histogram and Window Search to Find Lane Pixels**

Then, using all the good pixels in the windows, we find the second-degree polynomial fits using the numpy function `np.polyfit()`.

![alt text][image6]

**Figure 7. The Second-Degree Polynomial Fits**

####5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

The function `get_radius_of_curvature_in_meters()` takes the x and y fit points as inputs and returns the radius of curvature in meters. From observing the images, the conversion rates were estimated to be 3.7 meters (standard US highway width) per 750 pixels in x, and 30 meters per 720 pixels in y. Using these rates, the function re-generates the second degree fit lines in meters and implements the curvature calculation formula from the lectures to give the final results.

Similarly, the function `get_offset()` uses the same pixel-to-meters conversion in its offset calculation. Given the base pixel coordinates of both left and right lanes, it computes the center of the lanes and returns the difference between that and the center of the image, assuming the camera was place at the center of the vehicle. A negative offset means the vehicle is to the left of the lane center, and a positive offsite means it is to the right.

Looking at the values for different test images and the video, the radii of curvature range from high-hundreds to low-thousands on a turn, and tens of thousands on a relatively straight path. The offsets mostly fall within +/- 0.5 m. These values make sense with the given information about the road.

####6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

After chaining all the previous steps in the function `pipeline()`, we perform one extra step and calls the function `draw_lane_zone()`, which takes the previously computed inverse perspective transform, and plot the lane region back onto the undistorted image with `cv2.fillPoly()`. In another function `annotate()`, we display the curvature and offset information as texts at the top of the image. The final image looks like the following:

![alt text][image7]

---

###Pipeline (video)

####1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./processed_out.mp4)

---

###Discussion

####1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

Using different kinds of thresholding techniques definitely helped increasing recognition accuracy under different lighting and road conditions. However, since the thresholds are tweaked manually, the process may fail for an unseen situation with drastically different conditions. One way to avoid this is to get even more training data in different situations, and use a Machine Learning approach to help find the thresholds and recognize the lane pixels.

The current pipeline is also a bit too slow when processing the video. In order to use it as part of a self-driving car vision system, it needs to be able to process images much faster, ideally at close to real-time speed. This can be helped with the use of information from previous searches so we do not have to do a full search for every frame. For example, we can save the previous polynomial fit locations, and do a much quicker search within their proximity, since we do not expect the lanes' positions to move much from one frame to the next.  


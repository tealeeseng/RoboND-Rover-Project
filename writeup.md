## Project: Search and Sample Return
### Writeup Template: You can use this file as a template for your writeup if you want to submit it as a markdown file, but feel free to use some other method and submit a pdf if you prefer.

---


**The goals / steps of this project are the following:**  

**Training / Calibration**  

* Download the simulator and take data in "Training Mode"
* Test out the functions in the Jupyter Notebook provided
* Add functions to detect obstacles and samples of interest (golden rocks)
* Fill in the `process_image()` function with the appropriate image processing steps (perspective transform, color threshold etc.) to get from raw images to a map.  The `output_image` you create in this step should demonstrate that your mapping pipeline works.
* Use `moviepy` to process the images in your saved dataset with the `process_image()` function.  Include the video you produce as part of your submission.

**Autonomous Navigation / Mapping**

* Fill in the `perception_step()` function within the `perception.py` script with the appropriate image processing functions to create a map and update `Rover()` data (similar to what you did with `process_image()` in the notebook).
* Fill in the `decision_step()` function within the `decision.py` script with conditional statements that take into consideration the outputs of the `perception_step()` in deciding how to issue throttle, brake and steering commands.
* Iterate on your perception and decision function until your rover does a reasonable (need to define metric) job of navigating and mapping.  

[//]: # (Image References)

[image1]: ./misc/rover_image.jpg
[image2]: ./calibration_images/example_grid1.jpg
[image3]: ./calibration_images/example_rock1.jpg
[samples]: ./misc/samples.png
[test_video]: https://img.youtube.com/vi/zY1S1NgmNZc/0.jpg

## [Rubric](https://review.udacity.com/#!/rubrics/916/view) Points
### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---
### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  

You're reading it!

### Notebook Analysis
#### 1. Run the functions provided in the notebook on test images (first with the test data provided, next on data you have recorded). Add/modify functions to allow for color selection of obstacles and rock samples.

We preview samples images for perspective transformation and color threshold analysis.

![alt text][samples]

From the left image, I capture the x,y co-ordinates of the whole grid box as source of perspective transformation. Destination is defined as below.

```

dst_size = 5
bottom_offset = 6
source = np.float32([[14, 140], [301 ,140],[200, 96], [118, 96]])
destination = np.float32([[image.shape[1]/2 - dst_size, image.shape[0] - bottom_offset],
                  [image.shape[1]/2 + dst_size, image.shape[0] - bottom_offset],
                  [image.shape[1]/2 + dst_size, image.shape[0] - 2*dst_size - bottom_offset],
                  [image.shape[1]/2 - dst_size, image.shape[0] - 2*dst_size - bottom_offset],
                  ])

```

From the right image, I use color picker to find out the RGB color range and defined following golden rock threshold

```
def color_between_thresh(img, rgb_low_thresh=(170, 150, 50), rgb_high_thresh=(230,210,110)):
    # Create an array of zeros same xy size as img, but single channel
    color_select = np.zeros_like(img[:,:,0])
    # Require that each pixel be above all three threshold values in RGB
    # above_thresh will now contain a boolean array with "True"
    # where threshold was met
    below_thresh = (img[:,:,0] >= rgb_low_thresh[0]) \
                & (img[:,:,1] >= rgb_low_thresh[1]) \
                & (img[:,:,2] >= rgb_low_thresh[2]) \
                & (img[:,:,0] <= rgb_high_thresh[0]) \
                & (img[:,:,1] <= rgb_high_thresh[1]) \
                & (img[:,:,2] <= rgb_high_thresh[2])
    # Index the array of zeros with the boolean array and set to 1
    color_select[below_thresh] = 1
    # Return the binary image
    return color_select

```



#### 1. Populate the `process_image()` function with the appropriate analysis steps to map pixels identifying navigable terrain, obstacles and rock samples into a worldmap.  Run `process_image()` on your test data using the `moviepy` functions provided to create video output of your result.

With process_image() populated with steps of perspective transformation, color threshold and Coordinate Transformations on golden rock, navigable area and obstacles, `moviepy` produce following video.

![alt text][test_video]

click (https://youtu.be/zY1S1NgmNZc) to watch.


```
def process_image(img):
    # Example of how to use the Databucket() object defined above
    # to print the current x, y and yaw values
    # print(data.xpos[data.count], data.ypos[data.count], data.yaw[data.count])

    # TODO:
    # 1) Define source and destination points for perspective transform
    # 2) Apply perspective transform
    # 3) Apply color threshold to identify navigable terrain/obstacles/rock samples
    # 4) Convert thresholded image pixel values to rover-centric coords, DOING.
    # 5) Convert rover-centric pixel values to world coords
    # 6) Update worldmap (to be displayed on right side of screen)
        # Example: data.worldmap[obstacle_y_world, obstacle_x_world, 0] += 1
        #          data.worldmap[rock_y_world, rock_x_world, 1] += 1
        #          data.worldmap[navigable_y_world, navigable_x_world, 2] += 1

    # 7) Make a mosaic image, below is some example code
        # First create a blank image (can be whatever shape you like)
    output_image = np.zeros((img.shape[0] + data.worldmap.shape[0], img.shape[1]*2, 3))
        # Next you can populate regions of the image with various output
        # Here I'm putting the original image in the upper left hand corner
    output_image[0:img.shape[0], 0:img.shape[1]] = img

        # Let's create more images to add to the mosaic, first a warped image
    warped = perspect_transform(img, source, destination)
        # Add the warped image in the upper right hand corner
        # output_image[y:x]
    threshed = color_thresh(warped)
    obstacleTh = color_below_thresh(warped)

    threshed3d = np.zeros((threshed.shape[0],threshed.shape[1],3))
    threshed3d[:,:,0]=threshed
    threshed3d[:,:,1]=threshed
    threshed3d[:,:,2]=threshed
    rockTh = color_between_thresh(warped)
    rockTh3d = np.zeros((rockTh.shape[0], rockTh.shape[1],3))
    rockTh3d[:,:,0] = rockTh
    rockTh3d[:,:,1] = -1*rockTh
    rockTh3d[:,:,2] = -1*rockTh

    threshed3d=(np.clip(threshed3d+rockTh3d, 0,1))*255
    output_image[0:img.shape[0], img.shape[1]:] = threshed3d

        # Overlay worldmap with ground truth map
    map_add = cv2.addWeighted(data.worldmap, 1, data.ground_truth, 0.5, 0)
        # Flip map overlay so y-axis points upward and add to output_image
    output_image[img.shape[0]:, 0:data.worldmap.shape[1]] = np.flipud(map_add)


#     nav_map = np.zeros((data.worldmap.shape[1],440,3))
    xpix, ypix = rover_coords(threshed)
    xrpix, yrpix = rover_coords(rockThresh)
    xopix, yopix = rover_coords(obstacleTh)

    i = data.count
    navigable_x_world, navigable_y_world = pix_to_world(xpix=xpix, ypix=ypix, xpos=data.xpos[i], ypos=data.ypos[i], \
                                    yaw=data.yaw[i], scale=10, world_size=data.worldmap.size)

    rock_x_world, rock_y_world = pix_to_world(xpix=xrpix, ypix=yrpix, xpos=data.xpos[i], ypos=data.ypos[i], \
                                    yaw=data.yaw[i], scale=10, world_size=data.worldmap.size)
#     print('rock world:',rock_x_world, rock_y_world)

    obstacle_x_world, obstacle_y_world = pix_to_world(xpix=xopix, ypix=yopix, xpos=data.xpos[i], ypos=data.ypos[i], \
                                    yaw=data.yaw[i], scale=10, world_size=data.worldmap.size)


    data.worldmap[obstacle_y_world, obstacle_x_world, 0] += 1
    data.worldmap[rock_y_world, rock_x_world, 1] += 1
    data.worldmap[navigable_y_world, navigable_x_world, 2] += 1
    # add navigation trace.
#     print(nav_map.shape)
#     print(data.worldmap.shape)
    output_image[img.shape[0]:, img.shape[1]:img.shape[1]+data.worldmap.shape[1]] = data.worldmap*255


        # Then putting some text over the image
    cv2.putText(output_image,"Populate this image with your analyses to make a video!", (20, 20),
                cv2.FONT_HERSHEY_COMPLEX, 0.4, (255, 255, 255), 1)
    data.count += 1 # Keep track of the index in the Databucket()

    return output_image
```



### Autonomous Navigation and Mapping

#### 1. Fill in the `perception_step()` (at the bottom of the `perception.py` script) and `decision_step()` (in `decision.py`) functions in the autonomous mapping scripts and an explanation is provided in the writeup of how and why these functions were modified as they were.

With previous practice and learning, I fill up `perception_step()` with scriplets learnt in `process_image()`. `decision_step()` prefilled with working sample.

#### 2. Launching in autonomous mode your rover can navigate and map autonomously.  Explain your results and how you might improve them in your writeup.  

**Note: running the simulator with different choices of resolution and graphics quality may produce different results, particularly on different machines!  Make a note of your simulator settings (resolution and graphics quality set on launch) and frames per second (FPS output to terminal by `drive_rover.py`) in your writeup when you submit the project so your reviewer can reproduce your results.**

I run the Simulator in resolution of 800x600 with "Good" graphics quality. The frame rate fluctuating in between 8 to 15 frames.

I did following points during the autonomous simulation.

- Double throttle_set to 0.4 to speed up simulation.

- Double stop_forward and go_forward to compensate for faster throttle_set.
```
        self.stop_forward = 100 # Threshold to initiate stopping
        self.go_forward = 1000 # Threshold to go forward again
```

- Filter obstacle x,y co-ordinates to avoid program error which stops the drive_rover.py

```
    oFilter = obstacle_y_world<200

    obstacle_y_world = obstacle_y_world[oFilter]
    obstacle_x_world = obstacle_x_world[oFilter]
```

- Remove framerate and data keys log to have easier tracing.

The Project is definitely unfinished and I wonder how behavior cloning deep learning technique can optimize `decision_step()`.  

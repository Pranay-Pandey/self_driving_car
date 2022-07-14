# self driving car

This development was for the qualification round of <a href='https://nxpaimindia.com/'><b>NXP-AIM-INDIA</b></a>

The task is to move the car using only the pixy camera provided along with the car in the arena on the laid out track which includes under the over bridge, over the over bridge, zig zag tracks and some speed bumps.

![image](https://user-images.githubusercontent.com/79053599/179040830-9c331b0b-118d-47cb-a063-781e238e5e18.png)

The complete development is done in ros2 enviroment and gazebo simulation in Ubuntu 20.04
<hr>
For the installation of ros2 and all the required packages pls refer <a href='https://nxp.gitbook.io/nxp-aim'>here</a>

The script nxp_track_vision.py contains the code for the extraction of 2 vectors(num_vectors) giving the endlines of the track usig computer vision on the image captured from the pixy camera.<br>
The script aim_line_follow.py contains the script for controlling and steering the car along the road using the information passed from the nxp_track_vision.

Algorithm details:-

1. A variable ‘state’ checks if the car is on the over-bridge using the following condition: (when state==0)
(msg.m0_x1>26 and msg.m1_x1<52 and msg.m0_x0<msg.m0_x1 
and msg.m1_x1<msg.m1_x0 and (msg.m0_x0-window_center)*(msg.m1_x0-window_center)<0) 
(parameters were experimentally observed)

2. for num_vectors == 2
if both the end tracks are observed on the same side from the center then, the car should turn away from that side (was required in the initial track)

3. for num_vectors==2 and state==1 (on the bridge)
Steer angle is decided from the average of the slope of the 2 vectors obtained or the difference in y values of the tails of vectors (observed to steer in the correct direction over the bridge) 

4. The state changes back to 0 (implies ‘not on overbridge’) after some time calculated from the datetime.now().timestamp()

5. reducing speed at sharp steers using the exponential formula 
speed = speed*e^(-a*|steer|)  

We have imporved the steering mechanism for the car. This steering is directly related to the white pixel density on right and left side of the image window. A higher white pixel density means that area is on the road. First we used colour segmentation to get the mask image of the road and then a left and right slice image is taken from the masked image window. The number of white pixels is found in both of them respectively, and then the number of white pixels is divided by the total number of pixels in that slice image to get left and right pixel density. This value will always be between 0 and 1. we publish this values in an encoded message and receive it in aim_line_follow.py algorithm. Then we just add the left pixel density to the net steering and subtract right density from it. Since if left density is high the car should move left and hence steer should go +ve, and if right density is high the car should go right and steer should go -ve. We also apply a PID to the density values so that they don’t change too rapidly. This trick ensures that the car take smooth turns.<br> 
self.steer_vector.z = self.steer_vector.z + (pid(L) - pid(R))/2.0<br>
We also used the odom topic to access car’s velocity. Since during uphill, car velocity decreases and during downhill increases than its supplied value. To counter this we used a controll structure for car whenever it reaches the bridge. We supply velocity as<br>
self.speed_vector.x = self.speed_vector.x + kp*error <br>
Where, error = self.speed_vector.x – (car velocity as determined by odom), kp is a constant <br>
In this way we ensure that the car doesn’t go too slow during uphill and not too fast during downhill than the required value.<br>

## Introduction

In this Blog, I **Omar Garrido Martin**, student of the Master in Computer Vision from the university URJC, Madrid, Spain, will describe all the steps Ive gone through to make a Formula 1 car follow the line in the [JdeRobot Academy Simulator](https://academy.jderobot.org/)

This is part of an exercise for the subject Robotic Vision.

Foto of the exercise board (TODO)

The goal of the exercise is to make the car go as fast as we can while following the line in a circuit. For this we have a front camera of the car and a red line on the road of the track.

## First Steps

### Getting to know the environment
The first time that I arrive at the simulator, my goal was to get the image from the camera, and learn how the actuators from the car worked. This was a simple task due to the documentation within the Jupyter notebook provided by the JdeRobot Academy within the exercise.

- **Sensors**: Get the image from the camera using *input_image = self.getImage()*
- **Actuators**: Checking the speed and turn with the following code
  - *self.motors.sendV(10)*: Set the speed. On m/s
  - *self.motors.sendW(5)*: Set the steering wheel angle. To turn right negative values, while positive to left turns.
  
### Process the image
Once we know hot to make the car move, we need to process the image given by the front camera in order to get the red line segmented.
This was easy as the documentation or help within the simulator offers the needed instructions in opencv-python for this task. The steps are described next:
  1. Using a light independent color space as HSV: *image_HSV = cv2.cvtColor(myRGBImage)*
  2. Finding the right values for the red color in the HSV space:
    - *value_min_HSV = np.array([0, 150, 150])*
    - *value_max_HSV = np.array([20, 255, 255])*
  3. Segmentation with opencv with: *mask = cv2.inRange(img_hsv, value_min_HSV, value_max_HSV)*
  
  ![alt text](https://github.com/omar-ogm/FollowLine_F1/edit/master/resources/segmentation.PNG "Optional Title")
  
### First approximation to make it follow the line
In order to make it turn right or left to keep the line on the center, a metric to know the position of the line has to be found.
Here Ill use the centroid of the segmented line. To find the centroid or center of mass of the line, we must do it on the contour of the line segmented. 
If we pay attention to the images we will see that more than one contour may be retrieved since the line on a turn may be discontinuous. However, this doesnt represent any problem since we will focus on the biggest contour, the one with the biggest area, the one that gives us better information about the line itself, and seems to work pretty well.

The error or metric use here will be called **deviation**. Deviation is the length in pixel from the center of the image to the centroid, using only the x coordinate, width of the image. This deviation will have sign (negative for a left deviation, positive for a right deviation), this is important since we will oppose the deviation.

Using this deviation we can use it to turn based on it. For a first approximation of what a PID controller will be, I created a P controller. Fixing the speed to a low value, we only have to worry about turning right or left when the deviation appears. For this I created a variable kp that multiply the deviation error and this is used to turn in the opposite direction of the deviation to try to keep the car on the line.

What we see here is that is or kp is too big the car will go from one side to the other, and we will never be able to stay on the center. Adjusting experimentally the kp variable we find that the car could complete a lap "following" the line. Increasing the speed and adjusting again the kp value we can make it better and better. A trick was applied here so the car wont go from one side to the other too abruptly. I applied a range from the center where the car wont turn. This seemed to improve the performance on straight lines.

But this approach keeps having a problem, the car always goes from one side to the other instead of going in a straight line when possible. This means that we are losing time on the straight lines and to solve this we need to go to a more complex controller

## On the way to a PID controller
A PID controller is a control loop feedback mechanism really useful when we want to modulate a certain parameter around a reference point. This means that our reference will be to have a **deviation** of 0 (we are in the center of the line). As in the previous approach we still use the deviation as the error that we try to minimize, so the **kp** value is still there , a P controller, so now we add a **kd** parameter, a Derivative parameter. So now we have a PD controller, but how does this work? The derivative will be the difference between the actual_error and the previous_error divided by the time between samples. A simple derivative deviation. But what is this for? This parameter will oppose to the change, so if we are turning right and we get closer to the center(reference point), on the next iteration the derivative controller will oppose the direction we are turning on. But wait a second, is this what we want? If you think it at first it doesnt make sense, since we still want to go right, but we also want to do it smoothly so when we get close to the center we keep there. So the derivative term allows us to control how we get close to the reference point. 
Once again the kp and kd values must be found experimentally, trying to have enough strength to turn on curves (kp) but enough smoothness to stay in the middle of the line and stop going from one side to another (kd).

The integral value that will give us a PID controller is used to help close the gap between in the cases where our PD has an offset and cant get to the reference point. But since the PD works pretty well we wont be doing it at least for now.

At this moment I have a PD which controls the turn strength and a fix speed. Increasing the speed while adjusting the PD controller parameters was done here and allows me to complete the circuit in 1:06 minutes.

This seems to have a good performance but on a different circuit it may not have enough flexibility since the speed cannot be controlled. So in another circuit with a strong curve it may lose the line since it cannot regulate the speed and the angle we can apply to the steering wheel has a maximum value.

### Getting a P controller for the speed
If we think of speed, a P controller will be more than enough since the steering wheel is already trying to smooth the turns. In the speed controller we start with a P controller, which combined with the PD controller should give a good performance. 

Obtain a P that controls the speed is not an easy task. A lot of approaches can be made. We have to make assumptions here for our problem, based on the behavior that we want. The behavior desired for the speed will be:
  - We want the maximum speed possible when the deviation is close to 0. 
  - Once we start to have more deviation we want to readjust the speed to lower values, so we help to stay on the line.
  - In this case we dont mind about the sign of the deviation since we only care if we are close to the center or far, so we will use the **absolute value** of the deviation.
  - Also, we want to inverse someway the deviation, so when we have a low deviation we get high speed, and when the deviation is big we get low speed values.
  
For the last task the first attempt was to use *1/abs(deviation)* **(insert mathematical formula)**. This inverse the value as we wanted but it has a problem, we can only get extreme values because the steps are too abrupt. For example for a Kp=9 for a deviation of 1 the speed will be 9m/s which seems fine but when deviation is 2 the speed goes to 4.5m/s and with 10 pixels of deviation (which is a very small offset) the speed will be 0.9m/s. So this doesnt seem a good approach. More attempts where made to try to find a way to make the reduction in speed smooth using the deviation.

At the end a simple formula was applied that give good performance and the desired behavior. The maximum deviation is set to 320 (which is the maximum value we can get since the width of the image is 640 pixels) and we subtract the deviation to the maximum deviation -> 320-deviation=error. This means that when deviation=0 the error is 320, for deviation=1 the error is 319, and for a 320 deviation the speed will be 0. This way we have inverse the error as we wanted while keeping the linearity (so the speed is changed smooth). 

With this approach and tuning the parameters we get times of up to 1:02 min. Not a great improvement, but now the car is more flexible and will adapt better to changes in the circuit, is more robust.

Before trying to get a PD controller for speed and try how it works I think that getting to know if the car is on a straight line or a curve and applying a different controller for each one will improve more the times. Since a good controller on straight lines will allow to get the maximum speed, gaining a lot of time and having a controller or curves exclusively will let to tuned better the parameters and improve the speed on curves too.

### Attempt to change the cycle time
Before starting with the next step, I made a try to see if I could change the execution time of our loop. Right now and by default the cycle_time is fix to 80ms. This means that our algorithm of decision is being executed each 80ms, but what if we could go faster? Iw we could go faster we could improve the performance by having more reaction time or responsiveness. So I check how many time was taking my algorithm to execute on each loop. I get an average value of 0.15ms and some peaks of 0.3 maximum and 0.005 minimum. So based on this we could reduce the time of the cycle. I did the test and set the cycle time to 40ms but it didnt improve as expected...it performance went down by 10-15 seconds! The car started to go from one side to another on the straight lines. This has an explanation, the tuning parameters needed to readjust since now the algorithm was being executed more times, and the distance traveled on each iteration has been halved. So the PID that controls the turn was being too aggressive, this means we could increase the speed of our speed controller or make the turn controller less aggressive. But since I still have to add more code to be executed I will leave this to the end of the exercise, since the execution time will be probably increased

## Two controller for Two different situations

The first step if we want to apply two different controllers for each case is finding a way to know if the car is on a straight line or on a curve.

### Finding straight lines and curves
In order to find if the car is seeing a straight line or a curve, the next steps were done:
  1. Create points in the center(x coordinate) of the line on different heights(y coord) of the line.
  2. Using this points, each two consecutive points get the parameters of the line that formed this two points,
  3. Compare all the lines obtained (number_lines = number_points - 1). If all the lines has similar line equation parameters, similar scope(I dont compare the b in "y=mx+b", since in this problem parallel lines shouldnt be found). If the scope for all is similar then the car is on a Straight line while if the scope is different between two or more lines, then the car is on a curve.

Once the car can distinguish between straight lines and curves, all that is left is to create another instance of a controller and fine tune the parameters. Since Im using a PD controller for turning and a P controller for speed, Ill have 3x2=6 parameters to tune.

But in reality I tuned only 3 at each moment, since each controller is separated.




### Jekyll Themes

Your Pages site will use the layout and styles from the Jekyll theme you have selected in your [repository settings](https://github.com/omar-ogm/FollowLine_F1/settings). The name of this theme is saved in the Jekyll `_config.yml` configuration file.

### Support or Contact

Having trouble with Pages? Check out our [documentation](https://help.github.com/categories/github-pages-basics/) or [contact support](https://github.com/contact) and we’ll help you sort it out.

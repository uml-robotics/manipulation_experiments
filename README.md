# Description
This repo contains ROS packages in separate branches for testing two grasping algorithms (**GPD 2.0.0** [1] and GQCNN 1.1.0) on the Kinova Gen3. Specifically, for grasping comparison experiments.
All testing was done on Ubuntu 16.04 w/ ROS Kinetic in Simulation and with a Real workstation. 

## Required ROS packages for the Gen3:
* [uml_hri_nerve_armada_workstation](https://github.com/uml-robotics/uml_hri_nerve_armada_workstation) - More robot package hyperlinks can be found here
* [uml-robotics/ros-kortex](https://github.com/uml-robotics/uml_hri_nerve_armada_workstation) - This package is used internally at the NERVE Center
* [kinovarobotics/ros_kortex](https://github.com/Kinovarobotics/ros_kortex) - This package is the original ros_kortex package
* [ros_kortex_vision](https://github.com/Kinovarobotics/ros_kortex_vision)
* [kinova_ros](https://github.com/Kinovarobotics/kinova-ros)  
More details here: https://github.com/uml-robotics/uml_hri_nerve_pick_and_place

## Setting Up the Gen3 through Ethernet:  
[Gen3 Manual](https://www.kinovarobotics.com/sites/default/files/UG-014_KINOVA_Gen3_Ultra_lightweight_robot-User_guide_EN_R01.pdf)
1. Reset robot to factory settings (Hold power button for 10 seconds)
2. Connect Ethernet cable from arm to PC
2. On Ubuntu: Add Ethernet Network Connection -> IPv4 Settings -> Address = 192.168.1.11, Mask = 255.255.255.0
3. Go to 192.168.1.10 on a browser to confirm that the arm is connected
4. Use the arm in RVIZ: `roslaunch kortex_driver kortex_driver.launch arm:=gen3 gripper:=robotiq_2f_85 ip_address:=192.168.1.10`  

## Setting up GPD_ROS
1. [Follow the GPD Install Instructions](https://github.com/atenpas/gpd/tree/2.0.0#install)
   1. Make sure to pull the GPD 2.0.0 release
   1. Use PCL version 1.7.2 (Solves PCL error when using GPD_ROS)
   1. Use OpenCV version 3.4.0.14
   1. Had to comment out lines 769:773 in `src/gpd/util/plot.cpp` to solve a "not found" error
   1. Might have to remove the `improc.h` include that causes a build error
1. Change paths in `cfg/ros_eigen_params.cfg` Example:
    ```
    weights_file = /home/kyassini/gpd-2.0.0/models/lenet/15channels/params/
    ```
1. Make sure `./detect_grasps ../cfg/eigen_params.cfg ../tutorials/krylon.pcd` works without any errors and the GUI pops up. (Press E to see the grasps)
   1. If anything fails then make sure to resolve it before moving on
1. Clone [GPD_ROS](https://github.com/atenpas/gpd_ros/) to your catkin_ws/src
1. Replace `ur5.launch` with desired params for the arm and your install location of GPD 2.0.0.
   1. `Gen3.launch` example included in this repo
1. `Catkin build`
   1. NOTE: You may have to `catkin clean`, then build ONLY gpd_ros first. After, you can build everything else. Doing this solved a "bad date error" for me.   

## Running the GPD Grasping node
1. Need a running kortex diver:
   1. For simulation (Can toggle sim_workstation):  
   `roslaunch kortex_gazebo spawn_kortex_robot.launch arm:=gen3 gripper:=robotiq_2f_85 sim_workstation:=false dof:=7`
   1. For the real Arm:  
   `roslaunch kortex_driver kortex_driver.launch arm:=gen3 gripper:=robotiq_2f_85 ip_address:=192.168.1.10`  
   `roslaunch kinova_vision kinova_vision.launch device:=192.168.1.10`
1. Build and launch the appropriate launch file
   1. `gpd_grasping_node.launch`: launches GPD and the grasping node
   1. `real_gen3_grasping`: launches the above and the Kinova vision node  

If everything runs correctly, you will get the following behavior:
   
<p align="center">
<img src="imgs/real_gpd_example.gif" width="250"><img src="imgs/gpd_example.gif" width="250"><img src="imgs/gpd_example.png" width="150"> 
</p>

## Troubleshooting
First start with the simulated arm and the sim_workstation disabled to reduced other variables. View the published `Combined_Cloud` in rviz. 
Occasionally in simulation the cloud will not be transformed correctly, not sure how to fix this as the same code works 100% of the time with the real depth camera.
* Combined_Cloud is empty or not correct:
  * Make sure the correct pass-through filter values are being used for your workspace and that the correct topic is being used in the cloud transform. 
  Use `perception_test.launch` to test the perception code and visualize the published cloud.  
  **NOTE:** the clouds generated in simulation are unreliable sometimes and not as consistent as the clouds using the real arm. Sometimes they will be transformed in wierd ways and can get totally cutoff due to the pass-through values.
* The cloud looks fine and only contains the object, but no grasps are being generated.
  * Enable `plot_selected_grasps = 1` in `gpd/cfg/ros_eigen_params.cfg`. Make sure the GPD grasps look good. You will have to play around with the config values for the arm to get it right.
* GPD shows OK grasps, but the arm isn't finding a plan to the grasp.
  * Make sure the transform topic is correct and that there are no inaccurate collision objects added.

## Basic Grasping Node Structure
1. Move to a set position
2. Take a "screenshot" of the point cloud from the wrist depth camera
3. Filter and potentially concatenate the clouds
4. Publish cloud to GPD_ROS node, get a grasp msg in return
5. Calculate pre and post grasp positions along approach vector from the grasp msg
6. Plan to Pre-grasp, move in to grasp object, and then move out to a post-grasp
7. Move to a "hold" position for 5 seconds to verify that the object was adequately grasped

## Notes
* Ran into several issues installing GPD. The versions and details specified above should fix them all, but it is possible that something is missing since I installed multiple versions.
Pay attention to any build errors.
* Could not get the [MoveIt pick() pipeline](http://docs.ros.org/en/kinetic/api/moveit_tutorials/html/doc/pick_place/pick_place_tutorial.html) to work correctly. 
Had to call move() to the pose_sample which removed the ability to use a pre and post grasp as needed for actual grasping.
So I Replaced it with using the pose generated from the GPD grasp message directly and [this `createPickingEEFPose` function](https://gist.github.com/tkelestemur/60401be131344dae98671b95d46060f8#file-hsr_gpd_sample-cpp-L9)
to generate a pre and grasp pose directly from the approach vector.
* Had an issue with the group names in the `gen3_robotiq_2f_85.srdf.xacro`. This was fixed by editing the srdf which can be found in this repo under GPD_Files. However, might not be needed since pick() isn't being used anymore. 

## References
[1] Andreas ten Pas, Marcus Gualtieri, Kate Saenko, and Robert Platt. Grasp Pose Detection in Point Clouds. The International Journal of Robotics Research, Vol 36, Issue 13-14, pp. 1455-1473. October 2017.

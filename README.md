# This project is merged with [BeamNG_MATLAB_integration](https://github.com/BeamNG/BeamNG-MATLAB-integration) into [BeamNG-MATLAB-Simulink-integration](https://github.com/BeamNG/BeamNG-MATLAB-Simulink-integration)

<img src="https://raw.githubusercontent.com/BeamNG/BeamNG-MATLAB-integration/main/media/announcement.webp">


<a name="About"></a>

## About

We are excited to announce that the highly requested feature of interfacing [BeamNG.Tech](https://documentation.beamng.com/beamng_tech/) in [Simulink](https://www.mathworks.com/products/simulink.html) is here. The purpose of this document is to provide details/instructions on how to use the BeamNG Simulink Vehicle Systems Interface.  The user can connect the BeamNG simulator with a specially designed Simulink [S-function](https://www.mathworks.com/help/simulink/sfg/what-is-an-s-function.html), which will allow management of a tightly coupled two-way communication between these two environments.  The major focus here is on allowing native Simulink code to control and query a vehicle in BeamNG.  This includes various powertrain properties, including wheel torques (both drive and brake), or pedal inputs.


## Table of Contents

 - [Documentation](#docs)
 - [Features](#features) 
 - [Execution](#execution)
 - [Setup](#setup)
 - [S-Function Block](#sfunc)
 - [Simulink Memory Block](#memory)
 - [Instructions](#instructions)
 - [Examples](#examples)
 - [Compatibility](#compatibility)
 - [License](#license)
 
<a name="docs"></a>

## Documentation
[![](https://github.com/BeamNG/BeamNG-Simulink_generic_interface/blob/main/media/readthedocs.png)](https://beamngpy.readthedocs.io/en/latest/MATLAB.html)



<a name="features"></a>

## Features

This document provides details and contains instructions for using the prototype UDP tight-coupling system for Simulink model with the BeamNG simulator. This prototype solution demonstrates typical communication between both ends of the system.

### Tight-Coupling:

#### BeamNG Architecture Considerations:

BeamNG operates within a multi-threaded architecture.  Graphics rendering, collision detection, UI etc. all run on separate threads.  Also running on its own thread are the physics computations.

Figure 1 shows a simplified overview of this, where the physics steps are shown to be running concurrently with various other processes.

Wall-clock time refers to the time which has passed in the real world.  Simulation time refers to the the amount of time which has been simulated; note that we can simulate ahead in time, so the current simulation time could ahead of the current wall clock time.

BeamNG has both graphics steps (frames) and physics steps:  

Frames describe how often the user receives a visual update on screen, and can be measured in Frames-per-Second (FPS).  Eg for 30 FPS, we update the image on screen 30 times every second.  This quantity varies a lot of time, but is guaranteed not to drop below 20 FPS at any time.  

At the start of a frame, predictions are made as to when the next frame is expected in (wall-clock time).  Within each frame, BeamNG computes enough physics steps to take the simulation time up to this predicted wall-clock time, at which time the next frame will start to execute.

Each physics step computes exactly 0.5ms of simulation time.  This quantity is fixed, and can also be expressed as 2000hz.  Within any frame, BeamNG knows how many physics steps it must compute to take the simulation time up to the next frame start time (predicted).  However, there is no guarantee as to when – inside the current frame – these steps will be computed; only that they must be computed ahead of wall-clock time, in order to keep the simulation real-time.




<img src="https://github.com/BeamNG/BeamNG-Simulink_generic_interface/blob/main/media/1_BeamNG_Multithreading.png" alt="Figure 1: BeamNG Multithreaded Architecture">

Figure 1: BeamNG Multithreaded Architecture

In Figure 1, two possible cases are highlighted:

i) In frame 1 (left), we have the case where the physics steps are computed earlier than the work being performed on the other concurrent threads.  This results in a gap on the physics thread towards the end of the frame.  The other threads are the frame bottleneck here, not the physics.

ii) In frame 2 (right), we have the second case where the other threads finish earlier than the physics steps; we say here that the physics steps are the frame bottleneck.

A main thing to note here, particularly when looking over multiple frames, is the irregular nature of when physics steps occur. We cannot predict when future physics steps will be, since this depends on many non-deterministic factors (especially within Human-In-The-Loop environments).  Further, depending on what needs to be computed, some physics steps can take longer than others to compute.

However, we can guarantee a fixed number of physics steps being performed on average, over reasonable lengths of time. This is important to note, especially since we are typically simulating ahead of time.


<a name="execution"></a>

## Execution:

In order to execute efficient coupling, the user must provide accurate measurements for two things:

### i) The Simulink computation time:

This is the time required for Simulink to process a message sent to it from BeamNG.  If this varies, then the user could choose either the maximum or the average time and see which provides more optimal results.  if it is regular, more optimal coupling can be made between the two.

The simulinkTime property in the BeamNG controller should be set to this time.

### ii) The UDP round-trip time:

This is the time required for the UDP infrastructure to send a message from the BeamNG machine to the Simulink machine, and back again.  Even if they are on the same machine, this should still be measured.  

A standard ping test from a terminal is sufficient for this. For example, on windows:

1. Type “cmd” to bring up the Command Prompt.
2. Open the Command Prompt.
3. Type “ping” in the black box and hit the space bar.
4. Type the IP address you'd like to ping (e.g., 192.XXX.X.X).
5. Review the ping results displayed.

When this value has been computed, the pingTime property in the BeamNG controller should be set to this.



### iii) V-Sync:

The user should ensure that the V-Sync mode is switched off in BeamNG.  If it is switched on, optimal coupling will not be achieved since V-Sync will add extra latency.  This option can be found in the Display menu on the main options screen (press ESC).


<a name="setup"></a>

## Setting up Simulink:

In order to set the simulation time in Simulink to match the simulation time in BeamNG, the user should use the formula:

```ceil(simulinkDt / physicsDt) * physicsDt ```

where simulinkDt is the Simulink computation time, physicsDt is the BeamNG physics step time (fixed at 0.0005 seconds), and ceil is the ceiling operator.

Figure 2 shows where this is set (highlighted in yellow).

<img src="https://github.com/BeamNG/BeamNG-Simulink_generic_interface/blob/main/media/2_Setting_The_Simulink_Simulation_Time.png">

Figure 2: Setting The Simulink Simulation Time




### Coupling Case #1:

In Figure 3, we have the case where the Simulink computation time is similar in length to the physics steps in BeamNG.  However, the UDP round-trip time is significantly larger.

For efficient coupling, we need to have multiple messages sent out before any are received back in BeamNG.  Internally, BeamNG will use the two given time measurements to compute the optimal coupling management, which will send, receive and block execution at the appropriate times.


<img src="https://github.com/BeamNG/BeamNG-Simulink_generic_interface/blob/main/media/3_Coupling_Case_1.png" alt="Figure 1: BeamNG Multithreaded Architecture">

Figure 3: Coupling Case #1

### Coupling Case #2:

In Figure 4, we have the opposite case; the Simulink computation time is much slower than the BeamNG physics step time, but the UDP round-trip time is quite fast.

Here, it is optimal to have the coupled system skip sending messages on every second physics step.  If messages were sent at this time, then Simulink would still be processing the previous step and would need to buffer them, and this buffering would lead to sync problems in a short amount of time.  The clear bottleneck here is the Simulink computation time.


<img src="https://github.com/BeamNG/BeamNG-Simulink_generic_interface/blob/main/media/4_Coupling_Case_2.png" alt="Figure 1: BeamNG Multithreaded Architecture">


Figure 4: Coupling Case #2.



<a name="sfunc"></a>

## S-Function Block:

Let us now look at the Simulink [S-Function](https://www.mathworks.com/help/simulink/sfg/what-is-an-s-function.html) in some detail.  This is the part of Simulink which handles communication with BeamNG and controls execution in SImulink appropriately.  Figure 5 gives an overview of how the S-function has been implemented.



<img src="https://github.com/BeamNG/BeamNG-Simulink_generic_interface/blob/main/media/5_The_Simulink_S_Function.png">

Figure 5: The Simulink S-Function



### Inputs and Outputs:

The BeamNG S-function block is shown in Figure 6.  The inputs are controlled with a message of fixed format, and the outputs are controlled with another message with a different fixed format.  These are both described below in detail.

The input section is split into a core block (containing the core powertrain/vehicle system properties) and a custom block, which can be used by the user to bring any other desired data from BeamNG to Simulink.

The output section contains eight blocks:  

The driver controls section has signals for; throttle, brake, clutch, parking brake, and steering.  

The body state section has signals for; position, velocity, acceleration, ground speed, roll, pitch, yaw, and altitude. 

The status section has signals for; ignition level, gear, fuel, engine load, high beam, low beam, maximum RPM, reverse, RPM, signal L, signal R, and wheel speed. 

The wheel sections have signals for each wheel of the vehicle; including angular velocity, wheel speed, braking torque, propulsion torque, friction torque, and downforce. 

Finally, the custom section (on the input and output sides) has up to 50 user-defined signals. 

The S-function is designed to transfer data between Simulink and BeamNG using these fixed messaged, where every variable always exists at the same position in the message for every send/receive.  These signals are contiguous arrays of double-precision numbers.



<img src="https://github.com/BeamNG/BeamNG-Simulink_generic_interface/blob/main/media/6_S_function_block.png">

 
Figure 6: S-function block


### BeamNG → Simulink Message (Fixed Format):

Bank A: Core Driver Control
|Bank|  	Name	| Range  |
|---|-------------------|--------|
|A1	|throttle	        | [0,1]  |
|A2	|throttle_input	    | [0,1]  |
|A3	|brake	            | [0,1]  |
|A4	|brake_input	    | [0,1]  |
|A5	|clutch	            | [0,1]  |
|A6	|clutch_input	    | [0,1]  |
|A7	|parkingbrake	    | [0,1]  |
|A8	|parkingbrake_input | [0,1]  |
|A9	|steering	        | [-1,1] |
|A10	|steering_input	| [-1, 1]|

Bank B: Vehicle Kinematics
|Bank|  	Name	| Units  |
|---|-------------------|--------|
|B1 |	posX 	|	m | 
|B2 |	posY 	|	m |
|B3 |	posZ 	|	m |
|B4 |	velX 	|	m/s |
|B5 |	velY 	|	m/s |
|B6 |	velZ 	|	m/s |
|B7 |	groundspeed |	m/s|
|B8 |	accX 	|	m/s² |
|B9 |	accY 	|	m/s² |
|B10 |	accZ 	|	m/s² |
|B11 |	roll 	|	rad |
|B12 |	pitch 	|	rad |
|B13 |	yaw   	|	rad |
|B14 |	altitude|	m   |



Bank C: Secondary Vehicle Properties
|Bank|  	Name	| Range or Units |
|---|-------------------|--------|
|C1 | ignitionLevel | 	[0, 1, 2, 3] |
|C2 | gear | 	TBD |
|C3 | fuel | 	[0,1] |
|C4 | engineLoad | 	[0,1] |
|C5 | highbeam | 	[0,1] |
|C6 | lowbeam | 	[0,1] |
|C7 | maxrpm | 	1/min |
|C8 | reverse | 	[0,1] |
|C9 | rpm | 	1/min |
|C10 | signal_L | 	[0 or 1] |
|C11 | signal_R | 	[0 or 1] |
|C12 | wheelspeed | 	m/s |



Bank D: Wheel FL (Front-Left)
|Bank|  	Name	| Units |
|---|-------------------|--------|
|D1 |	wheelFL_angularVelocity |	rad/s |
|D2 |	wheelFL_wheelSpeed |	m/s |
|D3 |	wheelFL_brakingTorque |	Nm |
|D4 |	wheelFL_propulsiontorque |	Nm |
|D5 |	wheelFL_frictionTorque |	Nm |
|D6 |	wheelFL_downForce |	Nm |



Bank E: Wheel FR (Front-Right)
|Bank|  	Name	| Units  |
|---|-------------------|--------|
|E1	|wheelFR_angularVelocity |	rad/s | 
|E2	|wheelFR_wheelSpeed |	m/s | 
|E3	|wheelFR_brakingTorque |	Nm| 
|E4	|wheelFR_propulsiontorque |	Nm| 
|E5	|wheelFR_frictionTorque |	Nm| 
|E6	|wheelFR_downForce |	Nm| 

Bank F: Wheel RL (Rear-Left)
|Bank|  	Name	| Units  |
|---|-------------------|--------|
|F1	|wheelRL_angularVelocity |	rad/s |
|F2	|wheelRL_wheelSpeed |	m/s |
|F3	|wheelRL_brakingTorque |	Nm|
|F4	|wheelRL_propulsiontorque |	Nm|
|F5	|wheelRL_frictionTorque |	Nm|
|F6	|wheelRL_downForce| 	Nm |



Bank G: Wheel RR (Rear-Right)
|Bank|  	Name	| Units  |
|---|-------------------|--------|
|G1	|wheelRL_angularVelocity |	rad/s |
|G2	|wheelRL_wheelSpeed |	m/s |
|G3	|wheelRL_brakingTorque |	Nm|
|G4	|wheelRL_propulsiontorque |	Nm|
|G5	|wheelRL_frictionTorque |	Nm|
|G6	|wheelRL_downForce |	Nm |

Bank H: Custom User Values
|Bank|  	Name	| 
|---|---------------------------|
|H1	 | 	custom user values (up to 50)|
|..|		                     |
|H50	 |  |


Note: Bank H contains the custom user values.  These are values over which the user can manually choose properties in BeamNG and send them to Simulink.  With some implementation, this could involve readings from sensors, environmental information, or anything else available in BeamNG.  We leave this up to the user to decide on what to add, if required. 


### Simulink → BeamNG Message (Fixed Format):


Bank A: Core Vehicle Data
|Bank|  	Name	| Range or Units |
|---|-------------------|--------|
|A1	|engine throttle	| [0, 1] |
|A2	|brake pedal	| [0, 1] |
|A3	|steering	| [-1, 1] |
|A4	|RESERVED	|N/A|
|A5 |wheelFL_brakingTorque |	Nm|
|A6 |wheelFR_brakingTorque |	Nm|
|A7 |wheelRL_brakingTorque |	Nm|
|A8 |wheelRR_brakingTorque |	Nm|
|A9 |wheelFL_propulsionTorque |	Nm|
|A10 |wheelFR_propulsionTorque |	Nm|
|A11 |wheelRL_propulsionTorque |	Nm|
|A12 |wheelRR_propulsionTorque |	Nm|
|A13 |drive mode				  |bool|

Bank B: Custom User Values
|Bank|  	Name	| 
|---|-------------------|
|B1	|custom user values (up to 50)|
|..|		                     |
|B50 |    |	


Bank B contains space to allow the user to send any properties from Simulink to BeamNG.  Such data could then be processed within BeamNG and used to control some custom code. 



Note: for both messages, we expect all values to be double precision (8 bytes). If the user wishes to send other values (eg integer or boolean), they should be converted to double-precision before forming the message.  For example, a boolean flag could be sent as 0.0 or 1.0. This is an important consideration to note since some properties are not naturally double-valued.  An standard integer, for example, is only 4 bytes - adding this to the message would alter the makeup of the contiguous data in the message, and would lead to errors.


<a name="memory"></a>

## Simulink Memory Block:

We have introduced a memory block as shown in Figure 7.  In Simulink, memory blocks are used to store the previous value of a signal or variable, so that it can be accessed in a subsequent iteration of the simulation. They are necessary when modeling systems with delays or feedback loops, where different parts of the model may not process at the same time.  Memory blocks enable the storage and retrieval of values across multiple time steps, allowing for the implementation of feedback loops and the handling of delayed responses.


 
<img src="https://github.com/BeamNG/BeamNG-Simulink_generic_interface/blob/main/media/7_The_Memory_Block.png">

Figure 7: The Memory Block

<a name="instructions"></a>

## Instructions:

The Lua controller must be loaded in order to start the tight coupling.  We can do this with the following three steps:

1. First, bring up the Lua console debug window with the ' key.  
2. The vehicle should then be selected at the bottom-left of this screen (usually this will be “BeamNG - Current Vehicle”).
3. The following command should be typed into the command bar, to load the controller:  “controller.loadControllerExternal('tech/vehicleSystemsCoupling', 'vehicleSystemsCoupling', {})”

Figure 8 shows the bar at the bottom of the console window in detail.  Note the vehicle selection menu on the left, and the command bar on the right, where one can enter commands.

The Simulink process should also be started.  If BeamNG is not running, Simulink will block its execution until it receives a message from BeamNG.  The reverse is also true; if Simulink is not executing, BeamNG will block execution.

When communication has been established over the UDP send and recieve sockets (after both ends of the communication have start executing), the tight coupling process will commence.


<img src="https://github.com/BeamNG/BeamNG-Simulink_generic_interface/blob/main/media/8_The_Console_Window_Command_Bar.png">


Figure 8: The Console Window Command Bar

<a name="exmaples"></a>

## Examples:

We have provided some Simulink code examples to help the user see the BeamNG-Simulink coupling in action.  If the user wishes to execute these examples, the three control parameters described in this document (window width, send wait, send offset) should be set up appropriately.  The examples can be found in the repository, and are briefly described below: 

 
<img src="https://github.com/BeamNG/BeamNG-Simulink_generic_interface/blob/main/media/9_The_controller_function_of_the_Simulink_model.png">

Figure 9: The controller function of the Simulink model


### Example #1:

The user is able to test a basic controller to maintain the speed limit at using pedals of the vehicle.  The speed limit can be edited speed_input constant as shown in Figure 9.  Switching the vehicle control from torque to pedal by the toggle switch at the bottom of the model. 



### Example #2:

The user is able to test a basic controller to maintain the speed limit using only the wheel torque. User can switch from the torque control by using the toggle switch at the bottom of the model in Figure 9.



### Example #3:

The user is able to test a basic controller to maintain an angle of the vehicle in the map using Desired_steering_angle_input constant as shown in Figure 9. switching the vehicle control from torque to pedal by the toggle switch at the bottom of the model. 


 <a name="compatibility"></a>

## Compatibility  

Running the BeamNG-Simulink generic interface requires three individual software components, here is a list of compatible versions.

| BeamNG.tech | BeamNG-Simulink generic interface | MATLAB & Simulink | 
|-------------|-------------------------------------|--------|
| [0.28](https://beamng.tech/blog/beamng-tech-028/)    | [0.1.0](https://github.com/BeamNG/BeamNG-Simulink_generic_interface/releases/tag/v0.1.0)                     | [R2018b & later](https://www.mathworks.com/downloads) |



<a name="license"></a>

## License

This project is licensed under the MIT License - see the [LICENSE](https://github.com/BeamNG/BeamNG-Simulink_generic_interface/blob/main/LICENSE.txt) file for details.


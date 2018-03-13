# SGVHAK Rover
Main UI server and associated software for SGVHAK rover project.

Setup for development & testing
---
Written under `python 2.7.14` with associated `pip 9.0.1`
- Python version dictated by Ion Motion Control's RoboClaw Python API, which is written for 2.7. See http://forums.ionmc.com/viewtopic.php?f=2&t=542

### `virtualenv` recommended to help keep Python libraries separate.
- Install virtualenv `pip install virtualenv`
- Switch to SGVHAK_Rover directory
- Create new virtual environment `python -m virtualenv venv`
- Activate virtual environment `. venv/bin/activate`. Prompt should now be prepended with `(venv)`
  
### Install dependencies
- All dependencies are described in setup.py and can be installed with `pip install -e .`

### Start Flask
- `export FLASK_APP=SGVHAK_Rover`
- To enable debugging (warning: development only) `export FLASK_DEBUG=1`
- `flask run`
- Open UI by pointing web browser to `http://localhost:5000`

Setup for Rover Raspberry Pi
---

- Clone this repository and set up python, pip, and virtualenv as described above. Manually launch Flask and a web browser to verify the web app launched successfully.
- Configure flask for launch on startup by editing `/etc/rc.local` and add the following just above `exit 0` at the end of that file.
```
cd /home/pi/SGVHAK_Rover
export FLASK_APP=SGVHAK_Rover
. venv/bin/activate
flask run --host=0.0.0.0 &
```
- Configure Pi to be a wireless access point by following instructions at https://www.raspberrypi.org/documentation/configuration/wireless/access-point.md
- Once complete, connect to the new Pi-hosted wireless access point and open a web browser (default URL is `http://192.168.4.1:5000`) to use rover UI.

Overview
---

This project was implemented to control a six-wheeled rover with kinematics similar to rovers NASA JPL sent to Mars.
This meant we need to control the relative velocities of its six driven wheels and the steering angles for each of the four corner steerable wheels. This calculation is the core of the project, and can be found in `roverchassis.py`.

After the calculations have been made, their results can be sent to one of several motor control control module that will drive the actual mechanical parts. This implementation has the following motor control module implementations to match hardware installed on the rover:

1. A low-cost low-end servo motor solution controlled with the [Adafruit PWM HAT](https://learn.adafruit.com/adafruit-16-channel-pwm-servo-hat-for-raspberry-pi). Adafruit provides a Python library which is wrapped via `adafruit_servo_wrapper.py`.
2. A midrange solution with brushed DC motor matched with a quadrature encoder for closed-loop feedback. Controlled via [Ion Motion Control's RoboClaw](http://www.ionmc.com/Standard_c_18.html) modules. Ion Motion Control likewise provides a Python library, which is wrapped via `roboclaw_wrapper.py`.
3. For testing purposes, a nonoperative solution that serves as placeholder when running on a system that has none of the above controllers installed. All operations return success code with no actual effect. This is implemented by `roboclaw_wrapper.py` calling into `roboclaw_stub.py` instead of the actual RoboClaw Python API.

The HTML/CSS/JavaScript files in this project present the user interface for driving this rover. The HTML menu system is centralized in `menu.py` and the root menu is in `index.html`. The flexibility of HTML allows quick experimentation for different methods to present a rover user interface to the user. Several experimental UI are included and they all use the common `move_velocity_radius` API of `roverchassis.py`.


Configurations and Modifications
---
**Physical Geometry**
`roverchassis.py` requires knowing the physical layout of rover's wheels in order to properly calculate velocity and angle. Physical layout is described by specifying each wheel's (x,y) coordinate inside `config_roverchassis.json`. The coordinate system used is: Looking down on the rover from above, the front of the rover is the +Y axis and the right side of the rover is the +X axis. The center of the rover is the origin. The example length values in the repository are in inches, but any unit (either metric or imperial) may be used as long as they are used consistently. Since `roverchassis.py` calculations are made on their relative ratios.

**Wheel Control**
Aside from physical geometry, `config_roverchassis.json` also specifies two motor controlls for each wheel. One for the rolling travel motion, and the other for steering control.
* A freely rolling, undriven wheel will have `null` as its rolling control.
* A wheel that has no steering motor will have `null` as its steering control.
* It is valid to have a wheel that has `null` for both values. For example, a caster wheel.

**RoboClaw Parameters**
When RoboClaw controller is used, relevant parameters must be present in `config_roboclaw.json`. See Ion Motion Control's RoboClaw documentation for details.
* Connection parameters: serial port, baudrate, etc.
* Velocity PID values must be present if RoboClaw is controlling any rolling travel motors.
* Position PID values must be present if RoboClaw is controlling any steering motors.

**Adafruit Servo HAT Parameters**
When Adafruit PWM HAT is used, relevant parameters must be present in `config_adafruit_servo.json`.
* Connection parameters: I2C address, I2C bus, PWM frequency.
* The following parameter for each of 16 servo addresses:
  * PWM value for center position.
  * Maximum travel range, expressed in degrees off center.
  * PWM value for the maximum positive travel. (Minimum is assumed symmetric and will be calculated from other parameters.)

**UI Replacement** 
The web-based UI (HTML/CSS/JavaScript served by Flask) can be completely replaced by another system if desired. One example is to use a gaming controller communicating over Bluetooth. This Bluetooth communication module can call `move_velocity_radius` API on `roverchassis.py` to utilize all the same code calculating velocity/angle and sending them to the motor controllers.

**Additional Motor Controllers**
Other motor control classes may be added as peers of `roboclaw_wrapper.py` and `adafruit_servo_wrapper.py`. The new motor control module must be initialized in `roverchassis.py` method `init_motorcontrollers()`. Then its name may be used in `config_roverchassis.json` to specify its usage as wheel rolling or steering control.

# CarND-Controls-MPC
Self-Driving Car Engineer Nanodegree Program

---

### The Vehicle Model
In this project the kinematic model was used. This was the primairy model taught in class and is more applicable to fit the constraints of the simulator, it doesn't really take into account the physics of the car. This model uses the following equations:
```
x_[t+1] = x[t] + v[t] * cos(psi[t]) * dt
y_[t+1] = y[t] + v[t] * sin(psi[t]) * dt
psi_[t+1] = psi[t] + v[t] / Lf * delta[t] * dt
v_[t+1] = v[t] + a[t] * dt
cte[t+1] = f(x[t]) - y[t] + v[t] * sin(epsi[t]) * dt
epsi[t+1] = psi[t] - psides[t] + v[t] * delta[t] / Lf * dt
```
Where `x` & `y` reflect the cars position, `psi` is the orientation of the car, `v` is the velocity, `cte` is the cross-track-error (distance from the center of the lane), & `epsi` is the error in the orientation.

### Timestep Length and Elapsed Duration (N & dt)
`N` represents the number of steps into the future the car analyses, and `dt` is the duration of each of those steps. Ideally I would like to have used a combination that produced a prediction horizon `T` of 2-3 seconds, but combinations more than 1.5 seconds had less desirable results in the simulator. The values that produced an optimal result were N = 10, dt = 0.1, witch results in a `T` = 1sec. With the goal of my model being the ability to drive as fast as possible around the track, this produces a horizon long enough to cover the given waypoints, without predicting too far down the track (beyond the given waypoints).

### Polynomial Fitting and MPC Preprocessing
This simulator runs by providing the car with waypoints that represent the center of the track. These points are sent to the controller in the global coordinate system. For these point to be most usefull they need to first be convirted to the local coordinate system of the vehicle. This is done with the following code.
```
for (auto i=0; i<len ; ++i){
  t_points(0,i) =   cos(psi) * (ptsx[i] - px) + sin(psi) * (ptsy[i] - py);
  t_points(1,i) =  -sin(psi) * (ptsx[i] - px) + cos(psi) * (ptsy[i] - py);
}
```
Where `t_points(0,i)` is for the x component in the vehicle coordinate system, and `t_points(1,i)` represents the y direction. This brings the state of the vehicle to be:
```
state << 0, 0, 0, v, cte, epsi;
```
in the vehicle's coordinate system.

Once the waypoints have been converted to the vehicle's coordinate system we fit a polynomial:
```
coeffs = polyfit(ptsx_transform, ptsy_transform, 3);
```

Using the obtained coefficients, we are then able to determine the cross-track-error and orientation error.
```
double cte = polyeval(coeffs, 0);
double epsi = -atan(coeffs[1]);
```

We can then use this information to solve for a path based on the current position and orientation of the vehicle.
```
auto vars = mpc.Solve(state, coeffs);
```

### Model Predictive Control with Latency
To account for the 100ms delay, the calculations for the next actuator state is predicted for the cars state 100ms into the future. The following equations are used to predict where the car will be 100ms from its current state and then the future control predictions can be made.
```
double Lf = 2.67;
double delay = 0.1;
px = px + v * cos(psi) * delay;
py = py + v * sin(psi) * delay;
psi = psi - v * steer_value / Lf * delay; // negative steering!
v += throttle_value * delay;
```

## Dependencies

* cmake >= 3.5
 * All OSes: [click here for installation instructions](https://cmake.org/install/)
* make >= 4.1
  * Linux: make is installed by default on most Linux distros
  * Mac: [install Xcode command line tools to get make](https://developer.apple.com/xcode/features/)
  * Windows: [Click here for installation instructions](http://gnuwin32.sourceforge.net/packages/make.htm)
* gcc/g++ >= 5.4
  * Linux: gcc / g++ is installed by default on most Linux distros
  * Mac: same deal as make - [install Xcode command line tools]((https://developer.apple.com/xcode/features/)
  * Windows: recommend using [MinGW](http://www.mingw.org/)
* [uWebSockets](https://github.com/uWebSockets/uWebSockets)
  * Run either `install-mac.sh` or `install-ubuntu.sh`.
  * If you install from source, checkout to commit `e94b6e1`, i.e.
    ```
    git clone https://github.com/uWebSockets/uWebSockets
    cd uWebSockets
    git checkout e94b6e1
    ```
    Some function signatures have changed in v0.14.x. See [this PR](https://github.com/udacity/CarND-MPC-Project/pull/3) for more details.
* Fortran Compiler
  * Mac: `brew install gcc` (might not be required)
  * Linux: `sudo apt-get install gfortran`. Additionall you have also have to install gcc and g++, `sudo apt-get install gcc g++`. Look in [this Dockerfile](https://github.com/udacity/CarND-MPC-Quizzes/blob/master/Dockerfile) for more info.
* [Ipopt](https://projects.coin-or.org/Ipopt)
  * Mac: `brew install ipopt`
  * Linux
    * You will need a version of Ipopt 3.12.1 or higher. The version available through `apt-get` is 3.11.x. If you can get that version to work great but if not there's a script `install_ipopt.sh` that will install Ipopt. You just need to download the source from the Ipopt [releases page](https://www.coin-or.org/download/source/Ipopt/) or the [Github releases](https://github.com/coin-or/Ipopt/releases) page.
    * Then call `install_ipopt.sh` with the source directory as the first argument, ex: `bash install_ipopt.sh Ipopt-3.12.1`.
  * Windows: TODO. If you can use the Linux subsystem and follow the Linux instructions.
* [CppAD](https://www.coin-or.org/CppAD/)
  * Mac: `brew install cppad`
  * Linux `sudo apt-get install cppad` or equivalent.
  * Windows: TODO. If you can use the Linux subsystem and follow the Linux instructions.
* [Eigen](http://eigen.tuxfamily.org/index.php?title=Main_Page). This is already part of the repo so you shouldn't have to worry about it.
* Simulator. You can download these from the [releases tab](https://github.com/udacity/self-driving-car-sim/releases).
* Not a dependency but read the [DATA.md](./DATA.md) for a description of the data sent back from the simulator.


## Basic Build Instructions


1. Clone this repo.
2. Make a build directory: `mkdir build && cd build`
3. Compile: `cmake .. && make`
4. Run it: `./mpc`.

## Tips

1. It's recommended to test the MPC on basic examples to see if your implementation behaves as desired. One possible example
is the vehicle starting offset of a straight line (reference). If the MPC implementation is correct, after some number of timesteps
(not too many) it should find and track the reference line.
2. The `lake_track_waypoints.csv` file has the waypoints of the lake track. You could use this to fit polynomials and points and see of how well your model tracks curve. NOTE: This file might be not completely in sync with the simulator so your solution should NOT depend on it.
3. For visualization this C++ [matplotlib wrapper](https://github.com/lava/matplotlib-cpp) could be helpful.

## Editor Settings

We've purposefully kept editor configuration files out of this repo in order to
keep it as simple and environment agnostic as possible. However, we recommend
using the following settings:

* indent using spaces
* set tab width to 2 spaces (keeps the matrices in source code aligned)

## Code Style

Please (do your best to) stick to [Google's C++ style guide](https://google.github.io/styleguide/cppguide.html).

## Project Instructions and Rubric

Note: regardless of the changes you make, your project must be buildable using
cmake and make!

More information is only accessible by people who are already enrolled in Term 2
of CarND. If you are enrolled, see [the project page](https://classroom.udacity.com/nanodegrees/nd013/parts/40f38239-66b6-46ec-ae68-03afd8a601c8/modules/f1820894-8322-4bb3-81aa-b26b3c6dcbaf/lessons/b1ff3be0-c904-438e-aad3-2b5379f0e0c3/concepts/1a2255a0-e23c-44cf-8d41-39b8a3c8264a)
for instructions and the project rubric.

## Hints!

* You don't have to follow this directory structure, but if you do, your work
  will span all of the .cpp files here. Keep an eye out for TODOs.

## Call for IDE Profiles Pull Requests

Help your fellow students!

We decided to create Makefiles with cmake to keep this project as platform
agnostic as possible. Similarly, we omitted IDE profiles in order to we ensure
that students don't feel pressured to use one IDE or another.

However! I'd love to help people get up and running with their IDEs of choice.
If you've created a profile for an IDE that you think other students would
appreciate, we'd love to have you add the requisite profile files and
instructions to ide_profiles/. For example if you wanted to add a VS Code
profile, you'd add:

* /ide_profiles/vscode/.vscode
* /ide_profiles/vscode/README.md

The README should explain what the profile does, how to take advantage of it,
and how to install it.

Frankly, I've never been involved in a project with multiple IDE profiles
before. I believe the best way to handle this would be to keep them out of the
repo root to avoid clutter. My expectation is that most profiles will include
instructions to copy files to a new location to get picked up by the IDE, but
that's just a guess.

One last note here: regardless of the IDE used, every submitted project must
still be compilable with cmake and make./

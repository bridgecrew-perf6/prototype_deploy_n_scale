# Prototype, deploy and scale robotics solutions with balena and ROS

In [my last post](https://www.balena.io/blog/build-one-or-many-robots-balena-ros-tech-stack-block/), I walked through a high-level look at how one could use balena and ROS  a platform to create one or many edge device robots. Since then, I’ve published several balena Blocks that help bring this vision to life, and in this post, I’d like to walk through how anyone can use these blocks to prototype, deploy, and scale an IoT robot fleet.

All around the world, people use ROS to develop robotics applications. The ROS ecosystem is a large and diverse group of organizations and individuals, including academia, enterprises such as ABB and Boston Dynamics, government and institutions like NASA and ESA, and thousands of robotics enthusiasts and makers all around the world.

## Demonstrating how Docker and ROS *can* work well together
Last year, Canonical published an article on [6 reasons](https://ubuntu.com/blog/ros-docker) why Docker and ROS **are not a good fit**. 

To summarize, here are the main points of the article:
* No transactional updates for ROS on Docker
* No delta updates for applications
* Privileged containers are a security risk
* Configuring networking is challenging
* Fleet management is a pain.
* No notifications for updating vulnerable software packages

Let’s go through some of these concerns and see how we could help.

### Transactional updates
Updates that happen in a single step and can be rolled back in case of failure are called atomic, or transactional. While docker doesn’t support this functionality, the way releases are structured in balenaCloud, enables the same functionality, you can simply set the target release to an earlier, working version.

### Delta updates and preloading
Edge devices can be connected to the internet through various methods, from wired ethernet to LoRa and 3G Cellular. Robotics applications tend to be large in size compared to other edge-computing use cases. Balena can help you keep your robots up to date no matter how they are connected using [**delta updates**](https://www.balena.io/docs/learn/deploy/delta/) and [**preloading**](https://github.com/balena-io-modules/balena-preload)

### Connecting to external hardware resources
Using GPIO, I2C, USB, and other similar interfaces usually requires administrative rights on any Linux box. A common solution to this when using docker is to add `privileged: True` to your `docker-compose.yaml` file for that specific container. However, this opens the door to a bunch of potential security issues. That’s why everybody will tell you that [privileged containers are a bad idea](https://www.trendmicro.com/en_us/research/19/l/why-running-a-privileged-container-in-docker-is-a-bad-idea.html).

Adding a single device by its path: `/dev/i2c-1` instead of giving access to the whole bus or tree: `/dev/i2c` doesn't require a container to be privileged. In the case of I2C this is, but what about a USB-to-UART converter like CH340 which changes its name at every reconnection?

In essence, `privileged: True` enables all the [**Linux capabilities**](https://man7.org/linux/man-pages/man7/capabilities.7.html) for a specific container. A more fine-grained way of accessing external hardware is to only enable `CAP_RAWIO` capability. [Here](https://www.balena.io/docs/learn/develop/hardware/)'s more info on how to do that.

Additionally, if you are using a [balena base image](https://www.balena.io/docs/reference/base-images/base-images/#how-the-images-work-at-runtime) for your container you can make use of UDEV to connect to dynamically named devices like the aforementioned USB-to-serial bridge.

### Networking
ROS indeed has a few strict requirements when it comes to multi-device networking, mainly a few environment variables that you have to set. You find out more [here](http://wiki.ros.org/ROS/Tutorials/MultipleMachines) 

On balena devices, services can be resolved by their hostname, no need to find out their IP or anything else. Check out the readme for [ros-core](https://github.com/cristidragomir97/ros-core) to see how easy it is to get multiple ros-based containers to talk to each other. 
Balena can furthermore help you by allowing you to SSH into your robots remotely, either from the dashboard or our [balena-cli](https://www.balena.io/docs/learn/manage/ssh-access/#using-balena-ssh-from-the-cli) tool.

Additionally, if you enable the [Public URL](https://www.balena.io/docs/learn/manage/actions/#enable-public-device-url) option on the dashboard, you can expose a web-ui to the web. 

Oh, and as for **fleet management**, that’s literally what we do. :)

That being said, we beg to differ and we won’t only show you why docker and ROS are a good fit, but also how we can help you grow your robotics solution from the prototyping stage to mass deployment and scaling using the same tools and principles.
  
## Prototyping
Let’s take a concrete example out of my personal experience. For a former project, I was considering a couple of different single-board computers (SBCs). My goal was to find the cheapest, smallest, most powerful option that could support the Intel RealSense camera.

I bought three different options, BananaPi Zero, NanoPi Duo2 and another one I can't remember. I had to compile the RealSense SDK and its ROS wrapper on the boards themselves, with build settings corresponding to the kernel version each vendor offered. 

Anyway, the whole operation took several weeks, not to mention the compile time for the SDK only, was around 12h+ for each board.
This is exactly the kind of situation we mean when we say our mission is to “reduce friction for fleet owners”.

Let’s explore the ways in which balena can help you drastically shorten your prototyping time.

### Blocks
Per their official definition balena-blocks are “intelligent, drop-in chunks of functionality built to handle the basics, allowing you to focus on solving the hard problems”.

Let’s take the former example of the realsense camera. You can wrap the installation of realsense SDK and its ROS package into a dockerfile, upload it to balenaHub, and target the architectures you want. Then, it’s ready to run on every board we support.

You are not only saving time for yourself. But also for everyone who might be using this part. You’ll only need to reference the block in your docker-compose.yaml file and you are ready to go.

We have prepared a few ROS blocks, both to test the way ros-core would plug in with other blocks, and provide you with a neat robotics starter kit. 


Let’s explore some of the ros-centric blocks we have prepared for you:

#### ros-core
ros-core is the beating heart of the whole balena-ros ecosystem. It’s a full installation of ROS(1) Noetic wrapped inside a block. On runtime, it runs the roscore process. This process keeps track of the nodes that are running, how to reach them, and what their message types are.

Instead of having a full copy of ROS for each service, we define a volume mount share and let the other ROS blocks access the same binaries, this reduces the amount of space needed considerably and allows for layered ros-packages.

Additionally, ros-core makes it possible that ROS nodes can all talk to each other even if they are part of different services/containers. For security reasons, there's a separate internal network for the services. However, you can open whatever port you want to the external world.

To include ros-core in your balena solution, you simply need to add it to your docker-compose.yaml file like this:

You can find more information about how roscore works on its [github repository](https://github.com/cristidragomir97/ros-core). Also, there’s a guide on how to write your own ros-core compatible images [here](https://github.com/cristidragomir97/robotics-images)

```yaml
version: "2.1"

volumes:
ros-bin:

services:
  - roscore:
        image: cristidragomir97/ros-core
        environment:
            - ROS_HOSTNAME=roscore
            - ROS_MASTER_URI=http://roscore:11311
        links:
            - service0
        volumes:
          - ros-bin:/opt/ros/noetic

  - service0:
        build: ./service0
        environment: 
            - ROS_HOSTNAME=service0
            - ROS_MASTER_URI=http://roscore:11311
        volumes:
          - ros-bin:/opt/ros/noetic
``` 


#### ros-io

In a world where ICs, modules, and electronics are going through long waiting times we are trying to lend a hand to makers to prototype much faster, and allow robotics fleet owners the flexibility to replace the hardware quickly and deploy fleets with mixed hardware configurations.

ros-io aims to solve that by creating a hardware abstraction layer for modules and chips that are connected to I2C, UART, and GPIO interfaces. This reduces friction by eliminating the need to write low-level code, firmware, or ROS nodes and replaces it with a config file similar to the way docker-compose works.

Here's how this all works:

* You define your hardware in a configuration file and upload it to a git repository
ros-io downloads your config file parses its contents, and installs the proper packages and 3rd party dependencies
* Package code gets imported into the ros-io code, and worker threads are deployed:
* Workers wrap the part object, create ROS messages and map the read/update functions to rospy Subscriber and Publisher objects

Add this to your docker-compose.yaml file to use this block:
```yaml
ros-io:
  image: cristidragomir97/ros-io
  environment:
    - ROS_HOSTNAME=ros-io 
    - ROS_MASTER_URI=http://ros-core:11311
    - CONFIG_REPO=https://github.com/<repo_containing_your_config>
  volumes:
    - ros-bin:/opt/ros/noetic
  devices:
    - "/dev/i2c-1:/dev/i2c-1"
  cap_add:
      - SYS_RAWIO
```

More info at [ros-io](https://github.com/cristidragomir97/ros-io)
#### ros-camera

This block adds plug-and-play ROS support for the official Raspberry Pi cameras. On runtime, if everything is connected properly, you’ll be able to see the camera feed from this sensor on the `/image/compressed` topic.

This block wraps the most popular ROS package for the Raspberry Pi camera. Check out [their GitHub repository](https://github.com/UbiquityRobotics/raspicam_node) for more information.

Add this to your docker-compose.yaml file to use this block:
```yaml
ros-camera:
    depends_on:
      - ros-core
    image: cristidragomir97/ros-camera
    environment: 
       - UDEV=1
       - ROS_HOSTNAME=ros-camera
       - ROS_MASTER_URI=http://ros-core:11311
    volumes:
      - ros-bin:/opt/ros/noetic
    devices:
      - "/dev/vchiq:/dev/vchiq"
    cap_add:
      - SYS_RAWIO
```

More info at [ros-camera](https://github.com/cristidragomir97/ros-camera)

#### ros-lidar 

Slamtec’s RPLidar series is one of the most affordable 2D LIDAR hardware solutions on the market. 
This block adds plug-and-play ROS support for these devices, by wrapping [their official ROS package](https://github.com/Slamtec/rplidar_ros). On runtime, if everything is connected properly, you’ll be able to see the laser scan data from this sensor on the `/scan` topic.

```yaml
ros-rplidar:
    depends_on:
      - ros-core
    image: cristidragomir97/ros-rplidar
    environment: 
       - UDEV=1
       - ROS_HOSTNAME=ros-rplidar
       - ROS_MASTER_URI=http://ros-core:11311
    volumes:
      - ros-bin:/opt/ros/noetic
    devices:
      - "/dev/ttyUSB0:/dev/ttyUSB0"
    cap_add:
      - SYS_RAWIO
```

More info at [ros-rplidar](https://github.com/cristidragomir97/ros-rplidar)

#### ros-board
The standard ROS installation contains RViz, a powerful configurable tool that creates visual representations of your ros topics. Maps, image streams, and even complex 3D robot models. 

This is great for development, but in our scope, we are targeting headless embedded computers. There’s a way to stream your topics to another machine that has a desktop environment, but the network configuration is pretty complex, and it tends to be laggy even in a local area network, not to mention miles away over a VPN.

This is where [dheera’s rosboard](https://github.com/dheera/rosboard) comes in handy. This is a great piece of software that provides a browser-based visual representation of the topics published on your robot. It’s kind of like Grafana, but for robotics.

Add this to your docker-compose.yaml file to use this block:
```yaml
ros-board:
    depends_on:
      - ros-core
    image: cristidragomir97/ros-board
    environment: 
       - ROS_HOSTNAME=ros-board
       - ROS_MASTER_URI=http://ros-core:11311
    ports:
      - "8888:8888"
    volumes:
      - ros-bin:/opt/ros/noetic
```
More info at [ros-board](https://github.com/cristidragomir97/ros-board)

#### ros-joystick
This is a very simple block that allows you to use any linux compatible gamepad to remotely control your robot. It turns information from `/dev/input/js0` to ROS messages on the `/cmd_vel` topic. 

```yaml
ros-joystick:
    depends_on:
      - ros-core
    image: cristidragomir97/ros-joystick
    environment: 
       - UDEV=1
       - ROS_HOSTNAME=ros-joystick
       - ROS_MASTER_URI=http://ros-core:11311
    volumes:
      - ros-bin:/opt/ros/noetic
    devices:
      - "/dev/input/js0:/dev/input/js0"
    cap_add:
      - SYS_RAWIO
```

More info at [ros-joystick](https://github.com/cristidragomir97/ros-joystick)

#### arduino-block
While not a ros-block per se, this is a useful tool that helps you to compile, flash and update firmware using arduino-cli and avrdude. For now, this is only compatible with 8-bit Arduinos that use USB/UART.

```yaml
arduino-block:
    image: cristidragomir97/arduino-block
    devices:
        - "/dev/ttyAMA0:/dev/ttyAMA0"
    environment:
        - AVRDUDE_CPU=atmega328p
        - MONITOR_BAUD=115200
        - SERIAL_PORT=/dev/ttyAMA0
        - REPO="https://github.com/cristidragomir97/motorhead"
        - REPO_NAME="motorhead"
        - SKETCH_FOLDER=/firmware
        - ARDUINO_FQBN=arduino:avr:nano
```


Upon runtime, the block pulls the repository found at REPO, compiles the sketch for the board you have set with ARDUINO_FQBN, and flashes it to SERIAL_PORT using `avrdude`.


## Deployment
You can find most of the popular ROS packages on their official repository and install them with your distro’s package manager. The true power of ROS however comes with its build system called catkin. You can configure very complex builds, that include code from many sources and written in different programming languages.

There is one major downside to this, however: builds tend to get large, really quickly. Complex packages like 3D SLAM, or advanced physics engines required for kinematics can take a long time to build on powerful x64 workstations, so building them on a Raspberry Pi-like device is close to impossible.

That’s where the balena cloud builder comes in handy.

### Take advantage of the cloud builder
The cloud build can take care of this load for you, just configure your packages, create a docker file, and let the cloud builder take care of the heavy lifting for you. In the end, you’ll get an image containing just the artifacts that are required. If you want to keep image size to a minimum, consider using multistage docker builds. 

### Target multiple architectures
Apart from some very specific cases, for example, the ROS package for the Raspberry Pi camera, which uses both the physical CSI interface and some binary blobs from Broadcom, most ROS packages can be built for all the popular platforms out there, mainly armhf, arm64, and amd64. 

Using balena you can also just create a fleet for each device type you are targeting and push your release to it. This is extremely useful as your solution grows.

 Another aspect of this is the ability to move between different SBCs with the same CPU architecture. You can prototype on a Raspberry Pi or Jetson Nano and then move to something more production-ready, like [Variscite’s series](https://www.variscite.com/products/system-on-module-som/) of SOMs (System-on-a-Module).
 
### Preloading & Delta Updates
Since robotics tools, software and libraries are considerably more space hungry than other edge device use-cases, having the bulk of an image already loaded before the device is provisioned is crucial for robotics applications. Once it leaves the factory, it can be deployed in the wild, where no assumptions about the internet speed can be made. Any further update must be as slim in size as possible, and this is where [**delta updates**](https://www.balena.io/docs/learn/deploy/delta/) come in handy. Learn more about preloading your application [here](https://github.com/balena-io-modules/balena-preload)


## Scaling
All the techniques, blocks, and aspects of the balena ecosystem we presented in this article are an integral part of making sure that deploying ROS-powered software to 100 robots is as easy as deploying software to one unit. The only thing you need to take care of is flashing the SD cards, but hey, we have [Etcher Pro](https://www.balena.io/etcher/pro/) for that. 

## What’s next?
Next up, we are going to pretend we are the fictional company Paws Inc. which is building the world's most advanced cat companion robot. Armed with a laser pointer and powerful machine learning, it can keep your pet busy for hours. 

We are going to use most of the blocks presented in this article to create a working prototype and then show you how to get this ready for mass deployment. Additionally, we are going to show you some tips and tricks on how to make your robot development workflow as fast as possible. 

While I build out said fictional robot prototype, please check out the recently published ROS blocks and experiment on your own:
* [ros-core](https://github.com/cristidragomir97/ros-core)
* [ros-io](https://github.com/cristidragomir97/ros-io)
* [ros-camera](https://github.com/cristidragomir97/ros-camera)
* [ros-rplidar](https://github.com/cristidragomir97/ros-rplidar)
* [ros-board](https://github.com/cristidragomir97/ros-board)
* [ros-joystick](https://github.com/cristidragomir97/ros-joystick)
* [arduino-block](https://github.com/cristidragomir97/arduino-block)

Also, please leave any questions and comments below!


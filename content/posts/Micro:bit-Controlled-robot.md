---
title: "Micro:bit Controlled Robot"
date: 2020-01-26T16:21:47Z
draft: false
toc: true
images:
tags:
  - microbit
  - robot
  - raspberry pi
---

## Introduction

Recently I found myself in Cambridge, England.
This meant that I *had* to visit the [Raspberry Pi store](https://raspberrypi.org/raspberry-pi-store).
While I was there I picked up a CamJam Edukit 3 amoungs other things.

## The Project

### Demo

{{< vimeo 387285083 >}}

### Checking that everything works

I started off by following the [provided worksheets](https://camjam.me/?page_id=1035).
These give you a guideline of what you need to do while forcing you to be creative and come up with some solutions and ideas of your own.

I decided to use the provided box for the chase.
This seems to be a very popular choice.
Once I had connected the motors to my Pi and to the chase, and bolted the 'ball castor' to the chase, I cloned the [git repo](https://github.com/CamJam-EduKit/EduKit3).

```bash
git clone https://github.com/CamJam-EduKit/EduKit3
```

On my Raspberry Pi I then ran `python3 EduKit3/CamJam Edukit 3 - GPIO Zero/Code/3-motors.py` and saw my robot drive forwards.
If we look at the file we see the code we just executed.

```python
# CamJam EduKit 3 - Robotics
# Worksheet 3 - Motor Test Code

import time  # Import the Time library
from gpiozero import CamJamKitRobot  # Import the GPIO Zero Library CamJam library

robot = CamJamKitRobot()

# Turn the motors on
robot.forward()

# Wait for 1 seconds
time.sleep(1)

# Turn the motors off
robot.stop()
```

Here we can see that the script first imports some libraries. `from gpiozero import CamJamKitRobot` allows imports some code that allows us the inferface with our robot's motors.
`import time` will allow us to make our program pause (also known as sleeping) for a given amount of time.

We then create our robot and make it drive forwards.
The program will then wait (or sleep) for 1 second then stop moving.

### Adding micro:bit control

After making the robot move forwards I decided that I wanted to use a [micro:bit](https://microbit.org/) to control the robot.
This project actually required two micro:bits.
One micro:bit is used as a remote control.
It uses accelerator data to work out in which direction it is tilting.
It then sends that data over the built in radio to another micro:bit that sends the received direction over a USB cable to Raspberry Pi.
The Raspberry Pi should then move the robot accordingly using code similarly to what we just ran.

For this project I am going to create a folder named `edukit3_microbit`. All of the code for this project will be kept in here.
Firstly I am going to create the code for the micro:bit controller.

```python
# mb_controler.py

from microbit import *
import radio

radio.on()
# 9 = make led bright. 0 = turn off led
x_image = Image(
        "90009:"\
        "09090:"\
        "00900:"\
        "09090:"\
        "90009:")  # This displays an x on the screen
images = {
"u": Image.ARROW_N,
"d": Image.ARROW_S,
"l": Image.ARROW_W,
"r": Image.ARROW_E,
"f": x_image
}

def get_dir():
  x = accelerometer.get_x()
  y = accelerometer.get_y()
  abs_x = abs(x)
  abs_y = abs(y)
  amount = 300
  if abs_x &gt; abs_y:
    if x &gt; amount:
      return "r"
    if x &lt; -amount:
        return "l"
  elif abs_y &gt; abs_x:
    if y &gt; amount:
      return "d"
    if y &lt; -amount:
      return "u"
  return "f"

while True:
  d = get_dir()  # get the current direction from the acceleromter
  radio.send(d)  # send the direction over the radio to the other micro:bit
  display.show(images[d])  # display the image for the direction
  sleep(250)  # wait for 250ms (1/4 second) as to not spam the radio
```

We now need our second micro:bit to revieve this radio data and send it to our Raspberry Pi over a micro USB cable.

```python
# mb_dongle.py

from microbit import *
import radio

radio.on()


while True:
    data = radio.receive()
    if data:
        print(data)
```

Firstly we import the microbit library and the radio module.
The radio module will allows us to recieve the data from the other micro:bit.

Everything beneath `while True:` will carry on running until the micro:bit turns off.
Finally we then recieve the data over the radio, check if some data was actually sent, and if it was we then print the data.
When the micro:bit runs `print(data)` it sends the data over serial (the USB connection) to the Rapsberry Pi.

```python
# main.py

from gpiozero import CamJamKitRobot
from microbitdongle import Dongle


def main():
    mb = Dongle(debug=True)
    robot = CamJamKitRobot()
    commands = {
        "l": robot.left,
        "u": robot.forward,
        "d": robot.backward,
        "r": robot.right,
        "f": robot.stop,
    }
    while True:
        data = mb.recv()
        if data:
            print(data)
            if data in commands:
                commands[data]()


if __name__ == "__main__":
    main()
```

`main.py`is the script that will run on the actual Raspberry Pi.
`from gpiozero import CamJamKitRobot` imports the code needed to interface with the robots motors.
The line `from microbitdongle import Dongle` allows us to revieve the data from the micro:bit over the serial connection.

Next we define the `main()` function.
`mb = Dongle(debug=True)` makes a connection with the micro:bit what we can access through the variable `mb`.
`debug=True` outputs some useful information about the connecton with the micro:bit.

`robot = CamJamKitRobot()` allows to to later control the motors of the robot.
We will do this through the `robot` variable.

Writeup to be completed. In the mean time find the source code for the project [here](https://gitlab.com/mokytis/edukit3_microbit/).

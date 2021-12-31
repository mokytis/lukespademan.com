+++
title = "Tutorial: Thermal Printer"
date = 2019-12-26T15:33:53Z
aliases = ["/posts/2019/12/thermal-printer/"]
tags = ["christmas", "thermal-printer", "escpos", "tutorial"]
+++


For Christmas this year one of the gifts I recieved was a thermal printer.
A thermal printer is a printer that does not use ink; it uses heat to mark the special paper.
The printer I have prints receipts and can be controled using the ESC/POS protcol.

<!-- more -->

## First Program

To control the printer we need the `idVendor` and `idProduct`.
To do this we can plug in the printer, turn it on, and run `dmesg`.

```bash
$ dmseg
...
[ 4699.659181] usb 1-2: New USB device found, idVendor=0416, idProduct=5011, bcdDevice= 1.00
...
```

I am going to be using python to control the printer.
One library that allows us to do this is `python-escpos`.
To install it run `pip install python-escpos`.

A "Hello, World!" script may look like this.

```python
from escpos.printer import Usb


p = Usb(0x416, 0x5011, 0)

p.text("Hello, World!")  # prints on the printer
p.cut()  # prints a few empty lines and cuts the paper (if the printer supports it)
```

## Printing Images

The printer and python library also allow you to print black and white images.
You can do this with a small ammendment to your previous code.

```python
from escpos.printer import Usb


p = Usb(0x416, 0x5011, 0)

p.text("This code prints an image")
p.image("/path/to/image.gif")
p.cut()
```

## Self Printing Program

One of the first programs I decided to make was a program that prints itself.
This should be quite easy.
In python the `__file__` variable stores the location of the currently running script.

```python
from escpos.printer import Usb


p = Usb(0x416, 0x5011, 0)

with open(__file__, "r") as f:  # opens the running script file for reading
    content = f.read()  # reads the content of the file
    p.text(content)
    p.cut()

```

## Replacing The Python Print Function

After tweeting about my self printing program @ben_nuttall asked if I could replace the python print function with a function that prints to the printer.

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">Can you implement a print function that prints to the printer? Then just import print (overwriting the built-in print function) in some other code</p>&mdash; Ben Nuttall (@ben_nuttall) <a href="https://twitter.com/ben_nuttall/status/1210223090086694917?ref_src=twsrc%5Etfw">December 26, 2019</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script> 

At first I was not sure if it would work but it was easier than I first thought.
I first created a file `print.py` which I then imported in my script `main.py`.

As we want to print line my line we need to tell the printer to print each line using `self._raw(b"\x1bd\x00")`.
This is normally done for us by `self.cut()`.

```python
# print.py

from escpos.printer import Usb

class Printer(Usb):
    def print(self, txt):
        self.text(txt)  # sends text to printer

        self.text("\n" * 5)  # sends new lines to printer (so you can see the printed text)
        self._raw(b"\x1bd\x00")  # sends the command to print the text.

p = Printer(0x416, 0x5011, 0)

def print(txt):
    p.print(txt)
```

```python
# main.py

from print import print

print("Hello world!")
name = input("what is your name?")
print(f"Your name is {name}")%
```

I was suprised to see that this worked first time.

<blockquote class="twitter-tweet"><p lang="und" dir="ltr">yes <a href="https://t.co/Xl5jWuKzln">pic.twitter.com/Xl5jWuKzln</a></p>&mdash; Luke (@mokytis_) <a href="https://twitter.com/mokytis_/status/1210225756896407556?ref_src=twsrc%5Etfw">December 26, 2019</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script> 

## DSL

At the moment if I want to print a receipt I would have to write a python script.
I think that this is very exsessive and time consuming.
To fix this I made a parser in python that interpets a DSL I have just created and prints the receipt.

### Example Script

I want to DSL (Domain Specific Language) to be simple (relativley to other markup languages like HTML).
Here is an exmaple of what I want a script to look like

```plain
CENTER
BOLD
TEXT Hello, World
UNBOLD
TEXT this is centered but not bold
LEFT
TEXT this is left aligned
RIGHT
TEXT this is right aligned
TEXT
TEXT this is right aligned with an empty line above it
IMG /path/to/img.gif
TEXT this text has an image above it
```

### The Parser

I have never created a DSL or parser before so any advice on how to improve this would be appriciated.

```python
import sys
from escpos.printer import Usb


class Printer(Usb):
    def print(self):
        self._raw(b"\x1bd\x00")


class Config:
    def __init__(self, printer):
        self.printer = printer
        self.config = {"align": "left", "text_type": "normal"}
        self.align = "left"
        self.text_type = "normal"

    def update(self, key, value):
        self.config[key] = value
        self.printer.set(text_type=self.config["text_type"], align=self.config["align"])


p = Printer(0x416, 0x5011, 0)
config = Config(p)


commands = {
    "LEFT": lambda: config.update("align", "left"),
    "RIGHT": lambda: config.update("align", "right"),
    "CENTER": lambda: config.update("align", "center"),
    "BOLD": lambda: config.update("text_type", "b"),
    "UNBOLD": lambda: config.update("text_type", "normal"),
    "CUT": lambda: p.cut(),
    "TEXT": lambda: p.text("\n"),
}



file_name = sys.argv[2]  # path to file passed in command line ([1] will be the scipt name)
p.text(" ")

with open(file_name) as f:
    content = f.read()
    lines = content.split("\n")

for line in lines:
    line = line.strip()
    parts = line.split(" ")
    print(line)
    if len(parts) == 1:
        cmd = parts[0]
        if cmd in commands:
            print("    " + cmd)
            commands[cmd]()
    elif len(parts) > 1:
        if parts[0] == "TEXT":
            text = " ".join(parts[1:])
            print("    " + text)
            p.text(text + "\n")
        elif parts[0] == "IMG":
            p.image(parts[1])

```

We can print a receipt using the following:

```bash
python parser.py /path/to/receipt/file
```

## Ideas For The Future

There are a couple of ideas I have for future uses of my printer.

### Twitter Bot

One idea is to create a twitter bot.
When you tweet to the bot record itself printing your tweet and reply to the tweet with the footage.

### Morning Briefing

Every morning my printer could print me the days weather forcast and any news updates (from an RSS feed) and other information I care about (e.g. sports results from the previous day).

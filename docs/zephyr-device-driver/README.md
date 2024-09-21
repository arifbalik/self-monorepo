# A Practical Guide to Zephyr Device Driver Development <!-- omit from toc -->

This is a guide on how to write a production grade device driver in Zephyr. This includes driver code, testing, CI and many more. Do not consider this document as a tutorial to Zephyr or device drivers, I'll just showcase my workflow and the environment I use to develop device drivers in Zephyr. Some of this may not be suited for your workflow, but I'll try to keep each section on it's own so you can skip the parts you don't need or don't want to use.

I'll not talk about some details and assume you have some knowledge about Zephyr and device drivers. If you don't, or want to refresh your memory check out the [official documentation](https://docs.zephyrproject.org/latest/).

> [!NOTE]
> Zephyr still makes big changes to the framework, so some of the information here might be outdated or not the best practice anymore. I'll try to keep this up to date as much as possible, and give version information anywhere I can. Also I'll run inside a development container for you readers to easily follow along.
>

## Table of Contents <!-- omit from toc -->

- [Introduction](#introduction)
- [Prerequisites and Setup](#prerequisites-and-setup)
  - [Setup](#setup)
    - [Using the container](#using-the-container)
    - [Building the container](#building-the-container)
    - [VSCode Remote Containers](#vscode-remote-containers)
- [Getting Started](#getting-started)
  - [West Workspaces](#west-workspaces)
    - [West Manifest](#west-manifest)
- [Device Driver Development](#device-driver-development)
  - [Device Tree Nodes and Bindings](#device-tree-nodes-and-bindings)
  - [Peripheral APIs](#peripheral-apis)
  - [Device Driver Implementation](#device-driver-implementation)
  - [Integration into the Build System](#integration-into-the-build-system)
- [How Do We Test This?](#how-do-we-test-this)

## Introduction

I've known about Zephyr for a while now but only had the chance to work with it by the beginning of this year when I listened the [this Amp Hour podcast](https://theamphour.com/653-benjamin-cabe-nose-zephyr/). I've quickly come to realize the enormous benefits of using Zephyr for any kind of embedded project. From full blown RTOS running on x86 with network stacks and PCIe drivers to a microcontroller with 32KB of flash, Zephyr is a great choice for any project. It was surreal to edit [devicetree](https://www.devicetree.org/) files to configure a 50-cent microcontroller.

Zephyr gives you a nice framework for everything embedded. You have device trees, device drivers, networking stacks, filesystems, a testing framework, a nice RTOS, a nice build system and anything else you can name. It abstracts low level hardware so nicely that you only need to write what your application ultimately wants to do in a hardware-agnostic way and have it run on anything without touching a single line of code.

I especially liked the Zephyr because I was working on a range of similar products with an unorganized and distributed codebase, we needed to run our business logic on slightly different hardware platforms, so the device driver model and device trees was the perfect solution in this case. We initially got concerned about the size of the firmware as we were on very tight budget, we could only get 32KB and 64KB flash microcontrollers, and Zephyr seemed overkill for this, but it was not. It seems like the Zephyr team has a lot of experience in the embedded world and Linux in general, so they've made the right choice of making the entire thing customizable down to the detail. If you strip down everything you can get down to a ~3KB binary, which is amazing. But most of the time you want to use OS services and other features so you end up with a [~7KB-10KB binary for a minimal application](https://docs.zephyrproject.org/latest/samples/basic/minimal/README.html).

Most of the drivers are already in the Zephyr upstream, but you eventually have to write your own device drivers for your custom hardware. That's what I ended up doing, writing some device drivers, generating my custom board configurations and device trees, and wrote my application logic, and here I'll show you how I did it.

## Prerequisites and Setup

First of all we need an environment to work in. In this environment we'll have all the tools we need to develop, test and debug our device driver. I usually prefer working in docker containers for reproducibility and ease of deployment. Therefore you need some tools installed in your host to get started.

- [Docker](https://www.docker.com/) (v26.0.0)
- [VSCode](https://code.visualstudio.com/) (v1.92.0)

> [!NOTE]
> I'm running on a x86 Windows machine, so I can use things like [USBIP](https://usbip.sourceforge.net/) to debug my hardware connected to my host but this feature is not yet available in macOS, but we will also discuss about remote debugging in later sections.

I assume most of you already have those installed, so lets quickly get started.

### Setup

If you are using VSCode, you can use devcontainers feature to get your development environment up and running in no time. But you can also use any other editor or a terminal to do everything I'll show you here.

If you want to work on your host machine, you can check out the [official Zephyr documentation](https://docs.zephyrproject.org/latest/getting_started/index.html) to setup your environment.

For the container we run on Ubuntu Noble and install the following:

- Zephyr SDK (0.16.8)
- Zephyr ARM and Xtensa toolchains
- All the system packages needed for Zephyr
- Python3 and pip packages (including [west](https://docs.zephyrproject.org/latest/develop/west/index.html))
- A Zephyr user
- ZSH and Oh My Zsh

Check out the [Dockerfile](.devcontainer/Dockerfile) if you want to see the exact setup. To keep it compatible with non-vscode users, I kept devcontainer configuration minimal and only relied on the Dockerfile.

#### Using the container

The fastest way to start is to pull and run the image I prepared from my GitHub Container Registry with the following command:

```bash
docker run -it --name zephyr-dev -v zephyr-volume:/home/zephyr/west-workspace ghcr.io/arifbalik/zephyr-dev:latest
```

This should drop you into a shell with the Zephyr SDK and all the tools installed.

#### Building the container

The building process is simple, go to the directory of the Dockerfile and run the following command to build and run the container:

```bash
docker run -it --name zephyr-dev -v zephyr-volume:/home/zephyr/west-workspace $(docker build -t zephyr-dev -q .)
```

> [!TIP]
> If you run into problems try building the container separately to see and fix the error messages.
>
> `docker build -t zephyr-dev .`
>
> then run the container with the following command:
>
> `docker run -it --name zephyr-dev -v zephyr-volume:/home/zephyr/west-workspace zephyr-dev`

#### VSCode Remote Containers

Easiest method to work in a container is to use the `Remote - Containers` extension in VSCode. You can open the project folder in VSCode and click on the green icon in the bottom left corner or with `ctrl + shift + p` and select `Attach to Running Container...` and select the container you just started.

It is important to do this inside a workspace with a `.devcontainer` folder, so VSCode can detect the configuration and start the container for you.

> [!TIP]
> You can also use the [@devcontainers/cli](https://www.npmjs.com/package/@devcontainers/cli) npm package to automate this process.

## Getting Started

Now that we have a system we can build on. We'll use the `west` tool to create a new workspace and configure our project.

### West Workspaces

Zephyr calls its workspaces `west workspaces`. And there are 3 main workspace types:

- Zephyr source as a workspace (T1)
- Zephyr application as a workspace (T2)
- Custom workspace with multiple Zephyr applications (you guessed it..)

More information on them can be found [here](https://docs.zephyrproject.org/latest/guides/west/workspaces.html).

We will go with the third option as it is the most customizable and flexible option. But the other two is even simpler to set up. Zephyr has an [example repository](https://github.com/zephyrproject-rtos/example-application.git) for a T2 type workspace that will work just as fine. You can also directly work in the [Zephyr source repository](https://github.com/zephyrproject-rtos/zephyr.git). With T3 topology option we can import all zephyr source code as a module and configure our workspace however we like, but of course we will go with the best practices and use a similar structure to the Zephyr codebase.

#### West Manifest

To create a workspace we will use a west manifest file `west.yml`. This file will contain the information about the repositories we want to include in our workspace. For now we will only go with zephyr source code as our only module.

Firstly lets create new directories for our workspace.

```bash
mkdir -p west-workspace/manifest
cd west-workspace
```

Then we will create a `west.yml` file inside the `west-workspace/manifest` directory with the following content:

```yaml
manifest:
  remotes:
    - name: zephyrproject-rtos
      url-base: https://github.com/zephyrproject-rtos
  projects:
    - name: zephyr
      remote: zephyrproject-rtos
      revision: v3.7.0
```

This is the simplest standalone workspace you can create. It only includes the Zephyr source code and nothing else. You can add more repositories to the `projects` section to include more modules in your workspace (check out [west manifest documentation](https://docs.zephyrproject.org/latest/develop/west/manifest.html)) but for now we will only use upstream Zephyr.

## Device Driver Development

Zephyr uses Kconfig and CMake to decide if it will compile and link our source code. Also to make our code work on virtually any hardware, we can not directly access hardware registers and HAL libraries, Zephyr has two main abstractions for this, device tree nodes and peripheral APIs.

### Device Tree Nodes and Bindings

One of the first things to do is to create a device tree binding to describe our hardware. The concept of device trees are not new and is used in Linux for many years, so it may be familiar to some of you, but for those who are unfamiliar, it is a JSON-like file that describes a hardware node in a system called `device-tree`, this allows for decoupling the hardware from the source code so that software may run independent of the hardware it is running on, meaning the API won't change if you change the hardware.

To know more about device trees check out the [official documentation](https://www.devicetree.org/).

To sketch a design lets thinker about a node for the button driver.

```dts
button0: button0 {
    compatible = "custom,digital-button";
    label = "BUTTON";
    gpios = <&gpio0 0 GPIO_ACTIVE_HIGH>;
    interrupt-driven;
    debounce-interval = <100>;
};
```

This simple node seems pretty self-explanatory, it describes a button with the label `BUTTON` connected to the port `gpio0` and pin 0 with an active high signal. It is interrupt driven and has a debounce interval of 100ms. The key is in the `compatible` property, this is a string that describes the driver that will handle this node, in other words, this node information will be passed to a driver that declares itself as `custom,digital-button` compatible. Very neat!

A node `binding` is any property of a node, described in a file, a `yaml` file in our case, so to make our `custom,digital-button` node work we need to create a binding file for it. And there are strict rules to follow about the location of these files and the naming conventions. The binding files must be in the `dts/bindings` directory of workspace or application root, board directories or in modules. We will create our own in the workspace root.

The naming convention for the binding files is usually `vendor,driver.yaml` and usually they reside in subfolders for better organization. So we will create an `input` folder inside and put the binding there.

```bash
mkdir -p dts/binding/input
```

And create the `custom,digital-button.yaml` file with the following content:

```yaml
description: A Basic Digital Button Driver

compatible: "custom,digital-button"

properties:
  label:
    type: string
    required: true
  gpios:
    type: phandle-array
    required: true
  interrupt-driven:
    type: boolean
  debounce-interval:
    type: int
    default: 100
```

Again these documents are quite verbose and easy to read and maintain, so that's good. They can also get pretty messy if you don't do it right, so please follow the [official documentation](https://docs.zephyrproject.org/latest/guides/dts/bindings.html) for more information.

Now our node can be described in the device tree and the driver can be written to handle this node.

### Peripheral APIs

Peripheral APIs are generic Zephyr APIs that defines a common interface for general "peripherals", like GPIO, PWM, I2C, SPI, etc. These APIs are implemented by the SoC specific HALs so we don't have to change our application code, or driver code, if we change the hardware. We will be only dealing with the Zephyr APIs and it will do the plumbing for us.

These peripheral APIs are usually pretty nice and complete, for example check out the comprehensive GPIO Peripheral API [documentation](https://docs.zephyrproject.org/latest/doxygen/html/group__gpio__interface.html).

To use any GPIO in Zephyr we just need to include the header file and use the API.

```c
#include <zepryr/drivers/gpio.h>

void main(void)
{
    struct device *dev = NULL;
    int ret = 0;

    dev = device_get_binding("GPIO_0");
    if (!dev) {
        k_panic("Could not get GPIO device\n");
        return;
    }

    ret = gpio_pin_configure(dev, 0, GPIO_INPUT | GPIO_ACTIVE_HIGH);
    if (ret) {
        k_panic("Could not configure GPIO pin 0\n");
        return;
    }

    ...
}
```

Once we are able to get a `device` struct, we can be sure that it was initialized at boot time and can be used with its respective API. Notice how `struct device` is just a generic struct, in fact if we look at the definition of it we see a really abstract structre:

```c
/**
 * @brief Runtime device structure (in ROM) per driver instance
 */
struct device {
 /** Name of the device instance */
 const char *name;
 /** Address of device instance config information */
 const void *config;
 /** Address of the API structure exposed by the device instance */
 const void *api;
 /** Address of the common device state */
 struct device_state *state;
 /** Address of the device instance private data */
 void *data;

  ...
};
```

`config`, `api` and `data` are all adapters (void pointers) to whatever any "device" wants. And if we look at the `gpio_pin_configure` function you see after some checking it calls the actual driver that was described in the device tree of the hardware we are using, which was connected to this adapter at boot time, by the device tree macros. We will see this in detail in the next section.

```c
int gpio_pin_configure(const struct device *port,
     gpio_pin_t pin,
     gpio_flags_t flags)
{
  const struct gpio_driver_api *api =
    (const struct gpio_driver_api *)port->api;

  ...
  
 return api->pin_configure(port, pin, flags);
}
```

So the `void *api` was assigned to the device structure by the actual vendor driver at boot time and again dereferenced here by the API to get the address of the actual driver function which does the actual configuration. So probably the vendor driver will look into the `config` field or the `data` field to get the actual hardware registers and do the configuration.

Strong opinion time, C allowed this kind of abstraction all along, but nobody dared to implement it. I think this was the framework industry needed for a long time, and Zephyr did it right. So hats off to the Zephyr team for this.

### Device Driver Implementation

Now that we have our device tree node and binding ready, we can start writing our driver.

We will again start by creating a directory for our driver in the workspace root. And again in the `input` folder for a cohesive folder structure. These are not mandatory but it is good practice to keep things organized.

```bash
mkdir -p drivers/input/digital_button
```

### Integration into the Build System

## How Do We Test This?

We made all kinds of abstractions in every step, surely testing it on a real hardware will not be our only option. Zephyr has an extensive testing and emulation framework, allowing us to do all kinds of tricks to confidently say our driver works. And without even running it on a real hardware, at least for simple drivers like these ones.

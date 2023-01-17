# STM32-DMA-GPIO
This repository was create to follow the content of the course 'ARM Cortex M Microcontroller DMA Programming Demystified'.

I am using:
* NUCLEO-F401RE Board
* STM32Cube FW_F4 V1.27.1
* STM32CubeIDE version 1.10.1

## Intro

Basically, DMA work as master together with the processor, so the memory functions (such as access and storing info) can be performed by DMA, and not generating interrupts to the processor. Also, DMA saves power when compared to processor functions.

![image](https://user-images.githubusercontent.com/58916022/212678646-9b3b67fd-e5ab-492b-86df-bc6e848dfea7.png)

**Generic steps to follow while using DMA:**
* 1. Identify the DMAx Controller to use in application;
* 2. Initialize the DMA;
* 3. Trigger the DMA (can be automatic or manual trigger);
* 4. Wait for TC (poll) or get the callback from DMA driver (interrupt);

## GPIO with DMA

First, we are opening STM32CubeIDE and creating a new project. Then we are going to use STM32CubeMX to help with the APIs.

We are going to take data (0x00 and then 0xFF) from SRAM1 and send to PORT A.

To identify the DMAx Controller to be used, we follow the 'STM32F401xD/xE block diagram' on the microcontroller datasheet to check in which bus GPIO port A (LD2 is connect on PA5). GPIOs are connected to AHB1 bus (max clock up 84 MHz). According to the 'Figure 1. System Archtecture', only DMA2 can reach the AHB1 bus.

### CubeMX

In CubeMX, we will set the configuration we want for the DMA. On 'Pinout & Configuration' we go to 'System Core' then DMA2 option. We Set the 'DMA2 Stream 0' and uncheck the 'Increment Address' option (not useful for our application). Just click 'Add' to complete this part.

![image](https://user-images.githubusercontent.com/58916022/212888806-1053f7ea-f5f1-4e8c-9588-f4963afe96c2.png)

Then, we need to configure the NVIC. Since first code we will be using polling methode, we will no set the Interrupt Table for the DMA2. 

![image](https://user-images.githubusercontent.com/58916022/212906810-2038b83d-df9d-43a8-8fc6-47374005b14a.png)

In 'Code Generation' tab, we set to 'Generate IRQ Handler' for the SysTick Timer. It will help us to perform tasks that required time.

![image](https://user-images.githubusercontent.com/58916022/212910468-fe906f85-347b-48b2-a675-988ca0d4438c.png)

Then justo go on 'Project' -> 'Generate Code' or 'Alt-K' to generate code.

### Code

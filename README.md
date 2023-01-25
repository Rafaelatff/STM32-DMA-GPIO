# _HAL-STM32-DMA-Poll-GPIO
This repository was create to follow the content of the course 'ARM Cortex M Microcontroller DMA Programming Demystified'.

I am using:
* NUCLEO-F401RE Board
* STM32Cube FW_F4 V1.27.1
* STM32CubeIDE version 1.10.1

## Intro

Basically, DMA work as master together with the processor, so the memory functions (such as access and storing info) can be performed by DMA, and not generating interrupts to the processor. Also, DMA saves power when compared to processor functions.

![image](https://user-images.githubusercontent.com/58916022/212678646-9b3b67fd-e5ab-492b-86df-bc6e848dfea7.png)

Each FIFO has room for 4 bytes. 

Advantage 1: memory port is less accessed allowing other pending DMA request to access memory port.

Advantage 2: allowing other bus masters to access the memory, thus decreasing the waiting time for other masters.

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

As we can see, the DMA was initialized. In main() we have tha calling for the function *MX_DMA_Init();* that bassicaly does (as set on CubeMX): 

```c
static void MX_DMA_Init(void)
{

  /* DMA controller clock enable */
  __HAL_RCC_DMA2_CLK_ENABLE();

  /* Configure DMA request hdma_memtomem_dma2_stream0 on DMA2_Stream0 */
  hdma_memtomem_dma2_stream0.Instance = DMA2_Stream0;
  hdma_memtomem_dma2_stream0.Init.Channel = DMA_CHANNEL_0;
  hdma_memtomem_dma2_stream0.Init.Direction = DMA_MEMORY_TO_MEMORY;
  hdma_memtomem_dma2_stream0.Init.PeriphInc = DMA_PINC_DISABLE;
  hdma_memtomem_dma2_stream0.Init.MemInc = DMA_MINC_DISABLE;
  hdma_memtomem_dma2_stream0.Init.PeriphDataAlignment = DMA_PDATAALIGN_BYTE;
  hdma_memtomem_dma2_stream0.Init.MemDataAlignment = DMA_MDATAALIGN_BYTE;
  hdma_memtomem_dma2_stream0.Init.Mode = DMA_NORMAL;
  hdma_memtomem_dma2_stream0.Init.Priority = DMA_PRIORITY_LOW;
  hdma_memtomem_dma2_stream0.Init.FIFOMode = DMA_FIFOMODE_ENABLE;
  hdma_memtomem_dma2_stream0.Init.FIFOThreshold = DMA_FIFO_THRESHOLD_FULL;
  hdma_memtomem_dma2_stream0.Init.MemBurst = DMA_MBURST_SINGLE;
  hdma_memtomem_dma2_stream0.Init.PeriphBurst = DMA_PBURST_SINGLE;
  if (HAL_DMA_Init(&hdma_memtomem_dma2_stream0) != HAL_OK)
  {
    Error_Handler( );
  }
}
```

Then I added the data to be transferred from SRAM to GPIO port A.

![image](https://user-images.githubusercontent.com/58916022/212914272-99a3b64e-3232-4507-af6f-9b951779ea71.png)

Then, we need to see how to trigger the DMA. In the project tree, we can see that there are two HAL drivers regarding DMA API's:

![image](https://user-images.githubusercontent.com/58916022/212914938-e8ca5bd4-9f9c-4383-98da-2c74f22d2177.png)

**Note: static functions are not able to be called by user application.**

The function *HAL_DMA_Start* uses polling methode meanwhile the *HAL_DMA_START**_IT*** uses interruption mode.

![image](https://user-images.githubusercontent.com/58916022/212972136-7a1cd906-0070-4bd5-9081-3c7be5daa30a.png)

Besides, the API's return the type *HAL_StatusTypeDef*. 

![image](https://user-images.githubusercontent.com/58916022/212972759-7ace76e8-3a94-4389-a884-91798fb7f5d1.png)

For both API's, we need to send the following parameters:

* DMA_HandleTypeDef \*hdma: &hdma_memtomem_dma2_stream0 (address of the variable for DMA stream 0).
* uint32_t SrcAddress: Source address, (uint32_t) &led_data[0] (address of the led_data variable in first index, typecasted to uint32_t).
* uint32_t DstAddress: Destiny address, (uint32_t) &GPIOA->ODR (address of output data register of port A, typecasted to uint32_t).
* uint32_t DataLength: Data Length in bytes.

And for the last step, we need to wait for the transfer to be complete. We use the function *HAL_DMA_PollForTransfer*. This function also returns a type *HAL_StatusTypeDef*.

![image](https://user-images.githubusercontent.com/58916022/212982280-327b6433-3772-471f-8ee4-665fee97f936.png)

Then we send the following parameters:

* DMA_HandleTypeDef \*hdma:
* HAL_DMA_LevelCompleteTypeDef CompleteLevel: can be 'HAL_DMA_FULL_TRANSFER' or 'HAL_DMA_HALF_TRANSFER' (enum).
* uint32_t Timeout: uint32_t type or macro HAL_MAX_DELAY (function will wait for the DMA flax or timeout).

Final code (without non important comments):

```c
int main(void)
{
  /* MCU Configuration--------------------------------------------------------*/
  /* Reset of all peripherals, Initializes the Flash interface and the Systick. */
  HAL_Init();
  
  /* Configure the system clock */
  SystemClock_Config();

  /* Initialize all configured peripherals */
  MX_GPIO_Init();
  MX_DMA_Init();
  MX_USART2_UART_Init();

  /* Infinite loop */
  while (1)
  {
	  HAL_DMA_Start (&hdma_memtomem_dma2_stream0, (uint32_t) &led_data[0], (uint32_t) &GPIOA->ODR, 1);
	  HAL_DMA_PollForTransfer(&hdma_memtomem_dma2_stream0, HAL_DMA_FULL_TRANSFER, HAL_MAX_DELAY);

	  /*delay of 1 sec*/
	  current_ticks = HAL_GetTick();
	  while ((current_ticks +1000) >= HAL_GetTick());

	  HAL_DMA_Start (&hdma_memtomem_dma2_stream0, (uint32_t) &led_data[1], (uint32_t) &GPIOA->ODR, 1);
	  HAL_DMA_PollForTransfer(&hdma_memtomem_dma2_stream0, HAL_DMA_FULL_TRANSFER, HAL_MAX_DELAY);

	  /*delay of 1 sec*/
	  current_ticks = HAL_GetTick();
	  while ((current_ticks +1000) >= HAL_GetTick());
  }
}
```

Now let's do by interrupt.

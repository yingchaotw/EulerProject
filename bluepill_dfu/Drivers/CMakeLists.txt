cmake_minimum_required(VERSION 3.13)

project(Drivers)

add_library(${PROJECT_NAME}
STATIC
    STM32F1xx_HAL_Driver/Src/stm32f1xx_hal_cortex.c
    STM32F1xx_HAL_Driver/Src/stm32f1xx_hal_dma.c
    STM32F1xx_HAL_Driver/Src/stm32f1xx_hal_exti.c
    STM32F1xx_HAL_Driver/Src/stm32f1xx_hal_flash_ex.c
    STM32F1xx_HAL_Driver/Src/stm32f1xx_hal_flash.c
    STM32F1xx_HAL_Driver/Src/stm32f1xx_hal_gpio_ex.c
    STM32F1xx_HAL_Driver/Src/stm32f1xx_hal_gpio.c
    STM32F1xx_HAL_Driver/Src/stm32f1xx_hal_pcd.c
    STM32F1xx_HAL_Driver/Src/stm32f1xx_hal_pcd_ex.c
    STM32F1xx_HAL_Driver/Src/stm32f1xx_hal_pwr.c
    STM32F1xx_HAL_Driver/Src/stm32f1xx_hal_rcc_ex.c
    STM32F1xx_HAL_Driver/Src/stm32f1xx_hal_rcc.c
    STM32F1xx_HAL_Driver/Src/stm32f1xx_hal_tim_ex.c
    STM32F1xx_HAL_Driver/Src/stm32f1xx_hal_tim.c
    STM32F1xx_HAL_Driver/Src/stm32f1xx_hal.c
    STM32F1xx_HAL_Driver/Src/stm32f1xx_ll_exti.c
    STM32F1xx_HAL_Driver/Src/stm32f1xx_ll_gpio.c
    STM32F1xx_HAL_Driver/Src/stm32f1xx_ll_rcc.c
    STM32F1xx_HAL_Driver/Src/stm32f1xx_ll_usb.c
    STM32F1xx_HAL_Driver/Src/stm32f1xx_ll_utils.c
)

target_compile_definitions(${PROJECT_NAME}
PUBLIC
    USE_HAL_DRIVER
    USE_FULL_LL_DRIVER
    STM32F103xB
)

target_include_directories(${PROJECT_NAME}
PUBLIC
    STM32F1xx_HAL_Driver/Inc
    STM32F1xx_HAL_Driver/Legacy
    CMSIS/Device/ST/STM32F1xx/Include
    CMSIS/Include
)
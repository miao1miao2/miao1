# stm32学习

本项目学习使用的是 **STM32F103C8T6** 、**Keil uVision5** 开发，本仓库主要为自己学习使用。

### GPIO

这一节主要需要实现的功能 **点灯** 、**开关点灯** 

**所以你需要准备**

- [x] LED 小灯
- [x] 开关（没有开关用两个杜邦线也能代替）

GPIO一共有8种输入4种输出4种

| 输入模式 |                           作用特点                           |                    适用场景                     |
| :------: | :----------------------------------------------------------: | :---------------------------------------------: |
| 浮空输入 |            引脚“完全悬空”，外部给什么电平就读什么            |               外部已经有稳定驱动                |
| 上拉输入 |                 默认高电平 1 （外部拉低变0）                 |  开关按下 收到0 （VCC---按键----GPIO----GND）   |
| 下拉输入 |                 默认低电平 0 （外部拉高变1）                 |   开关按下 收到1（VCC---按键---GPIO----GND）    |
| 模拟输入 | 关闭 Schmitt Trigger、输入缓冲器、断开上拉/下拉 **模拟电压 → ADC采样电路** | ADC采样、电压检测、电流采样、电位器、传感器输出 |

| 模式     | 默认状态 | 是否稳定 | 是否需要外部电阻 | 典型用途       |
| -------- | -------- | -------- | ---------------- | -------------- |
| 浮空输入 | 不确定   | ❌不稳定  | ✔建议外部        | 外部强驱动     |
| 上拉输入 | 1        | ✔稳定    | ❌不需要          | 按键（最常用） |
| 下拉输入 | 0        | ✔稳定    | ❌不需要          | 按键/信号输入  |
| 模拟输入 | 电压值   | ✔稳定    | ❌不需要          | ADC            |

**输出模式**

|   输出模式   |                           作用特点                           |                           适用场景                           |
| :----------: | :----------------------------------------------------------: | :----------------------------------------------------------: |
|   推挽输出   | 推挽输出内部是两个晶体管：<br/>✔ 可以“主动输出高/低”<br/> ✔ 驱动能力强<br/> ✔ 速度快（ns级翻转）<br/> ✔ 适合高速数字信号 | VCC<br/>       │<br/>      P-MOS（上拉）<br/>       │<br/>GPIO ───────→ 引脚<br/>       │<br/>      N-MOS（下拉）<br/>       │<br/>      GND |
|   开漏输出   |                                                              | GPIO ─────→ 引脚<br/>   │<br/>  N-MOS（只能拉低）<br/>   │<br/>  GND |
| 复用推挽输出 |                                                              |                                                              |
| 复用开漏输出 |                                                              |                                                              |

| 特性         | 推挽     | 开漏         |
| ------------ | -------- | ------------ |
| 输出高       | 主动驱动 | 依赖上拉     |
| 输出低       | 主动驱动 | 主动驱动     |
| 是否需要电阻 | 不需要   | 需要         |
| 是否可共线   | ❌危险    | ✔安全        |
| 速度         | 快       | 慢（上升沿） |
| 应用         | LED/通信 | I2C/总线     |

下面代码我是结合 **freeRtos** （移植freeRtos后面再写吧）

```c
#include "stm32f10x.h"
#include "FreeRTOS.h"
#include "task.h"

TaskHandle_t LED0_Task_Handle = NULL;
TaskHandle_t LED1_Task_Handle = NULL;
TaskHandle_t LED2_Task_Handle = NULL;

/*================ LED0：PA0 500ms闪烁 ================*/
void LED0_Task(void *pvParameters)
{
    while(1)
    {
        /* ODR：输出数据寄存器，bit0控制PA0 */
        GPIOA->ODR |= (1 << 0);                 // PA0=1 输出高电平点亮LED
        vTaskDelay(pdMS_TO_TICKS(500));         // 延时500ms

        GPIOA->ODR &= ~(1 << 0);                // PA0=0 输出低电平熄灭LED
        vTaskDelay(pdMS_TO_TICKS(500));         // 延时500ms
    }
}

/*================ LED1：PA1 200ms闪烁 ================*/
void LED1_Task(void *pvParameters)
{
    while(1)
    {
        GPIOA->ODR |= (1 << 1);                 // PA1输出高电平
        vTaskDelay(pdMS_TO_TICKS(200));         // 延时200ms

        GPIOA->ODR &= ~(1 << 1);                // PA1输出低电平
        vTaskDelay(pdMS_TO_TICKS(200));         // 延时200ms
    }
}

/*================ LED2：PA3按键控制PA2 ================*/
void LED2_Task(void *pvParameters)
{
    uint8_t led_state = 0;
    uint8_t last_key = 1;

    while(1)
    {
        /* IDR：输入数据寄存器，bit3读取PA3按键 */
        uint8_t now_key = (GPIOA->IDR & (1 << 3)) ? 1 : 0;

        /* 检测按键下降沿：1->0（按下瞬间） */
        if(last_key == 1 && now_key == 0)
        {
            led_state ^= 1;

            if(led_state)
                GPIOA->BSRR = (1 << 2);         // PA2置1（点亮）
            else
                GPIOA->BSRR = (1 << (2 + 16));  // PA2复位（熄灭）
        }

        last_key = now_key;
        vTaskDelay(pdMS_TO_TICKS(20));          // 消抖+轮询
    }
}

int main(void)
{
    /* RCC->APB2ENR：开启GPIOA时钟 bit2=1 */
    RCC->APB2ENR |= (1 << 2);

    /* GPIOA->CRL：配置PA0为推挽输出2MHz */
    GPIOA->CRL &= ~(0xF << 0);
    GPIOA->CRL |=  (0x2 << 0);  // MODE=10(2MHz) CNF=00(推挽)

    /* PA1推挽输出2MHz */
    GPIOA->CRL &= ~(0xF << 4);
    GPIOA->CRL |=  (0x2 << 4);

    /* PA2推挽输出2MHz */
    GPIOA->CRL &= ~(0xF << 8);
    GPIOA->CRL |=  (0x2 << 8);

    /* PA3上拉输入 MODE=00 CNF=10 => 0x8 */
    GPIOA->CRL &= ~(0xF << 12);
    GPIOA->CRL |=  (0x8 << 12);

    /* 上拉输入：ODR=1 */
    GPIOA->ODR |= (1 << 3);

    /* 创建任务 */
    xTaskCreate(LED0_Task, "LED0", 128, NULL, 2, &LED0_Task_Handle);
    xTaskCreate(LED1_Task, "LED1", 128, NULL, 2, &LED1_Task_Handle);
    xTaskCreate(LED2_Task, "LED2", 128, NULL, 2, &LED2_Task_Handle);

    /* 启动调度器 */
    vTaskStartScheduler();

    while(1);
}
```


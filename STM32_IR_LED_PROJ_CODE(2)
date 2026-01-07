#include "stm32f10x.h"
#include "stm32f10x_gpio.h"
#include "stm32f10x_tim.h"
#include "stm32f10x_rcc.h"
#include "OLED.h"

#define IR_RX_PIN      GPIO_Pin_0
#define BTN_L1_PIN     GPIO_Pin_1
#define BTN_S1_PIN     GPIO_Pin_2
#define BTN_L2_PIN     GPIO_Pin_3
#define BTN_S2_PIN     GPIO_Pin_4

#define MAX_PULSES     200
#define NUM_SLOTS     2

volatile uint16_t ir_buffer[NUM_SLOTS][MAX_PULSES];
volatile uint16_t ir_len[NUM_SLOTS];
volatile uint8_t  ir_start_level[NUM_SLOTS];

void TIM2_Init(void){
    TIM_TimeBaseInitTypeDef tim;

    RCC_APB1PeriphClockCmd(RCC_APB1Periph_TIM2, ENABLE);

    tim.TIM_Prescaler     = 72 - 1;
    tim.TIM_CounterMode   = TIM_CounterMode_Up;
    tim.TIM_Period        = 0xFFFF;
    tim.TIM_ClockDivision = TIM_CKD_DIV1;
    tim.TIM_RepetitionCounter = 0;

    TIM_TimeBaseInit(TIM2, &tim);
    TIM_Cmd(TIM2, ENABLE);
}

void delay_us(uint16_t us)
{
    TIM_SetCounter(TIM2, 0);
    while (TIM_GetCounter(TIM2) < us);
}

#define IR_ON()   TIM_Cmd(TIM1, ENABLE)
#define IR_OFF()  TIM_Cmd(TIM1, DISABLE)

void IR_PWM_Init(void){
    GPIO_InitTypeDef gpio;
    TIM_TimeBaseInitTypeDef tim;
    TIM_OCInitTypeDef oc;

    RCC_APB2PeriphClockCmd(
        RCC_APB2Periph_GPIOA |
        RCC_APB2Periph_TIM1,
        ENABLE
    );

    gpio.GPIO_Pin   = GPIO_Pin_8;
    gpio.GPIO_Mode  = GPIO_Mode_AF_PP;
    gpio.GPIO_Speed = GPIO_Speed_50MHz;
    GPIO_Init(GPIOA, &gpio);

    tim.TIM_Prescaler     = 0;
    tim.TIM_CounterMode   = TIM_CounterMode_Up;
    tim.TIM_Period        = 1799;
    tim.TIM_ClockDivision = TIM_CKD_DIV1;
    tim.TIM_RepetitionCounter = 0;
    TIM_TimeBaseInit(TIM1, &tim);

    oc.TIM_OCMode      = TIM_OCMode_PWM1;
    oc.TIM_OutputState = TIM_OutputState_Enable;
    oc.TIM_Pulse       = 900;
    oc.TIM_OCPolarity  = TIM_OCPolarity_High;
    TIM_OC1Init(TIM1, &oc);

    TIM_OC1PreloadConfig(TIM1, TIM_OCPreload_Enable);
    TIM_ARRPreloadConfig(TIM1, ENABLE);
    TIM_CtrlPWMOutputs(TIM1, ENABLE);

    IR_OFF();
}

void GPIO_Init_All(void){
    GPIO_InitTypeDef gpio;

    RCC_APB2PeriphClockCmd(
        RCC_APB2Periph_GPIOA |
        RCC_APB2Periph_GPIOB |
        RCC_APB2Periph_AFIO,
        ENABLE
    );

    gpio.GPIO_Pin   = IR_RX_PIN | BTN_L1_PIN | BTN_S1_PIN | BTN_L2_PIN | BTN_S2_PIN;
    gpio.GPIO_Mode  = GPIO_Mode_IPU;
    gpio.GPIO_Speed = GPIO_Speed_50MHz;
    GPIO_Init(GPIOA, &gpio);
}


void IR_Learn(uint8_t slot){
    uint8_t last, now;

    ir_len[slot] = 0;
    ir_start_level[slot] =
        GPIO_ReadInputDataBit(GPIOA, IR_RX_PIN);

    last = ir_start_level[slot];
    TIM_SetCounter(TIM2, 0);

    while (ir_len[slot] < MAX_PULSES){
        now = GPIO_ReadInputDataBit(GPIOA, IR_RX_PIN);

        if (now != last){
            ir_buffer[slot][ir_len[slot]++] =
                TIM_GetCounter(TIM2);
            TIM_SetCounter(TIM2, 0);
            last = now;
        }

        if (TIM_GetCounter(TIM2) > 50000 && ir_len[slot] > 10)
            break;
    }

    if (ir_len[slot] > 0)
        OLED_ShowString(slot + 2, 9, "O");
}

void IR_Send(uint8_t slot){
    uint16_t i;
    uint8_t level = ir_start_level[slot];

    OLED_ShowString(slot + 2, 9, "Sending");

    for (i = 0; i < ir_len[slot]; i++){
        if (level == 0){
            IR_ON();
            delay_us(ir_buffer[slot][i]);
            IR_OFF();
        }
        else{
            IR_OFF();
            delay_us(ir_buffer[slot][i]);
        }
        level ^= 1;
    }

    IR_OFF();
    OLED_ShowString(slot + 2, 9, "O      ");
}

void IR_Send_Sony(uint8_t slot){
		int r;
    for (r = 0; r < 3; r++)
    {
        IR_Send(slot);
        delay_us(30000);
    }
}

int main(void){
    TIM2_Init();
    IR_PWM_Init();
    GPIO_Init_All();
    OLED_Init();

    delay_us(50000);
    OLED_Clear();

    OLED_ShowString(1, 1, "Slots");
    OLED_ShowString(1, 8, "Status");
    OLED_ShowString(2, 1, "1");
    OLED_ShowString(3, 1, "2");
    OLED_ShowString(2, 9, "E");
    OLED_ShowString(3, 9, "E");

    while (1){
        if (!GPIO_ReadInputDataBit(GPIOA, BTN_L1_PIN)){
            delay_us(20000);
            IR_Learn(0);
            while (!GPIO_ReadInputDataBit(GPIOA, BTN_L1_PIN));
        }

        if (!GPIO_ReadInputDataBit(GPIOA, BTN_S1_PIN)){
            delay_us(20000);
            IR_Send_Sony(0);
            while (!GPIO_ReadInputDataBit(GPIOA, BTN_S1_PIN));
        }

        if (!GPIO_ReadInputDataBit(GPIOA, BTN_L2_PIN)){
            delay_us(20000);
            IR_Learn(1);
            while (!GPIO_ReadInputDataBit(GPIOA, BTN_L2_PIN));
        }

        if (!GPIO_ReadInputDataBit(GPIOA, BTN_S2_PIN)){
            delay_us(20000);
            IR_Send_Sony(1);
            while (!GPIO_ReadInputDataBit(GPIOA, BTN_S2_PIN));
        }
    }
}

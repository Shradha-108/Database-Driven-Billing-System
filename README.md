// main.c
#include "main.h"
#include "lcd.h"
#include "keypad.h"
#include "rfid.h"
#include <string.h>
#include <stdio.h>

UART_HandleTypeDef huart1; // PC UART
UART_HandleTypeDef huart2; // RFID UART

char rfid_uid[16];
char uart_tx_buf[64];
char key;
uint8_t quantity = 0;
uint16_t total_amount = 0;

void SystemClock_Config(void);
static void MX_GPIO_Init(void);
static void MX_USART1_UART_Init(void);
static void MX_USART2_UART_Init(void);

int main(void)
{
  HAL_Init();
  SystemClock_Config();
  MX_GPIO_Init();
  MX_USART1_UART_Init();
  MX_USART2_UART_Init();

  LCD_Init();
  Keypad_Init();
  RFID_Init(&huart2);

  LCD_Clear();
  LCD_Print("Scan RFID Card");

  while (1)
  {
    if (RFID_ReadUID(rfid_uid))
    {
      LCD_Clear();
      LCD_Print("Card Detected");
      LCD_SetCursor(2, 0);
      LCD_Print(rfid_uid);

      HAL_Delay(1000);

      LCD_Clear();
      LCD_Print("Enter Qty:");

      quantity = 0;
      while (1)
      {
        key = Keypad_GetKey();
        if (key >= '0' && key <= '9')
        {
          quantity = quantity * 10 + (key - '0');
          LCD_SetCursor(2, 0);
          LCD_Print("Qty: ");
          LCD_PrintNum(quantity);
        }
        else if (key == '#') // Confirm
        {
          break;
        }
      }

      total_amount = quantity * 50; // Example price

      sprintf(uart_tx_buf,
              "UID:%s,QTY:%d,AMT:%d\r\n",
              rfid_uid, quantity, total_amount);

      HAL_UART_Transmit(&huart1,
                        (uint8_t *)uart_tx_buf,
                        strlen(uart_tx_buf),
                        HAL_MAX_DELAY);

      LCD_Clear();
      LCD_Print("Bill Sent!");
      HAL_Delay(2000);

      LCD_Clear();
      LCD_Print("Scan RFID Card");
    }
  }

  
 // rfid.h
#ifndef __RFID_H
#define __RFID_H

#include "stm32f1xx_hal.h"

void RFID_Init(UART_HandleTypeDef *huart);
uint8_t RFID_ReadUID(char *uid);

#endif

//rfid.c
#include "rfid.h"
#include <string.h>

static UART_HandleTypeDef *rfid_uart;
static uint8_t rx_byte;
static uint8_t index = 0;
static char buffer[16];

void RFID_Init(UART_HandleTypeDef *huart)
{
    rfid_uart = huart;
    HAL_UART_Receive_IT(rfid_uart, &rx_byte, 1);
}

uint8_t RFID_ReadUID(char *uid)
{
    if (index >= 12)  // Typical RFID UID length
    {
        buffer[index] = '\0';
        strcpy(uid, buffer);
        index = 0;
        return 1;
    }
    return 0;
}

void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
    if (huart == rfid_uart)
    {
        buffer[index++] = rx_byte;
        HAL_UART_Receive_IT(rfid_uart, &rx_byte, 1);
    }
}

//lcd.h
#ifndef __LCD_H
#define __LCD_H

#include "stm32f1xx_hal.h"

void LCD_Init(void);
void LCD_Clear(void);
void LCD_Print(char *str);
void LCD_PrintNum(uint16_t num);
void LCD_SetCursor(uint8_t row, uint8_t col);

#endif

//lcd.c
#include "lcd.h"
#include <stdio.h>

#define RS_PIN GPIO_PIN_0
#define EN_PIN GPIO_PIN_1
#define D4_PIN GPIO_PIN_2
#define D5_PIN GPIO_PIN_3
#define D6_PIN GPIO_PIN_4
#define D7_PIN GPIO_PIN_5
#define LCD_PORT GPIOB

void LCD_Enable(void)
{
    HAL_GPIO_WritePin(LCD_PORT, EN_PIN, GPIO_PIN_SET);
    HAL_Delay(1);
    HAL_GPIO_WritePin(LCD_PORT, EN_PIN, GPIO_PIN_RESET);
}

void LCD_Send4Bit(uint8_t data)
{
    HAL_GPIO_WritePin(LCD_PORT, D4_PIN, data & 1);
    HAL_GPIO_WritePin(LCD_PORT, D5_PIN, (data >> 1) & 1);
    HAL_GPIO_WritePin(LCD_PORT, D6_PIN, (data >> 2) & 1);
    HAL_GPIO_WritePin(LCD_PORT, D7_PIN, (data >> 3) & 1);
    LCD_Enable();
}

void LCD_Command(uint8_t cmd)
{
    HAL_GPIO_WritePin(LCD_PORT, RS_PIN, GPIO_PIN_RESET);
    LCD_Send4Bit(cmd >> 4);
    LCD_Send4Bit(cmd & 0x0F);
    HAL_Delay(2);
}

void LCD_Data(uint8_t data)
{
    HAL_GPIO_WritePin(LCD_PORT, RS_PIN, GPIO_PIN_SET);
    LCD_Send4Bit(data >> 4);
    LCD_Send4Bit(data & 0x0F);
}

void LCD_Init(void)
{
    HAL_Delay(50);
    LCD_Command(0x28);
    LCD_Command(0x0C);
    LCD_Command(0x06);
    LCD_Command(0x01);
}

void LCD_Clear(void)
{
    LCD_Command(0x01);
}

void LCD_Print(char *str)
{
    while (*str)
        LCD_Data(*str++);
}

void LCD_PrintNum(uint16_t num)
{
    char buf[6];
    sprintf(buf, "%d", num);
    LCD_Print(buf);
}

void LCD_SetCursor(uint8_t row, uint8_t col)
{
    uint8_t addr = (row == 1) ? 0x80 : 0xC0;
    LCD_Command(addr + col);
}


}

//keypad.h
#ifndef __KEYPAD_H
#define __KEYPAD_H

#include "stm32f1xx_hal.h"

void Keypad_Init(void);
char Keypad_GetKey(void);

#endif

//kaypad.c
#include "keypad.h"

const char keymap[4][4] =
{
  {'1','2','3','A'},
  {'4','5','6','B'},
  {'7','8','9','C'},
  {'*','0','#','D'}
};

void Keypad_Init(void)
{
    // GPIO already configured via CubeMX
}

char Keypad_GetKey(void)
{
    for (int r = 0; r < 4; r++)
    {
        HAL_GPIO_WritePin(GPIOA, 0x0F, GPIO_PIN_RESET);
        HAL_GPIO_WritePin(GPIOA, (1 << r), GPIO_PIN_SET);

        for (int c = 0; c < 4; c++)
        {
            if (HAL_GPIO_ReadPin(GPIOA, (1 << (c + 4))) == GPIO_PIN_SET)
            {
                HAL_Delay(200);
                return keymap[r][c];
            }
        }
    }
    return 0;
}



//output
UID:123456789ABC,QTY:2,AMT:100

# PhaseOffset

/*
 * Phase Offset Detection with Period Measurment Example
 * Written by Matthew Bon for Digikey Electronics
 */

#include <stdint.h>
#include <stdbool.h>
#include <stdio.h>
#include "C:\ti\TivaWare_C_Series-2.1.4.178\inc\hw_ints.h"
#include "C:\ti\TivaWare_C_Series-2.1.4.178\inc\hw_memmap.h"
#include "C:\ti\TivaWare_C_Series-2.1.4.178\inc\hw_nvic.h"
#include "C:\ti\TivaWare_C_Series-2.1.4.178\inc\hw_types.h"
#include "C:\ti\TivaWare_C_Series-2.1.4.178\inc\hw_gpio.h"
#include "C:\ti\TivaWare_C_Series-2.1.4.178\driverlib\sysctl.h"
#include "C:\ti\TivaWare_C_Series-2.1.4.178\driverlib\interrupt.h"
#include "C:\ti\TivaWare_C_Series-2.1.4.178\driverlib\systick.h"
#include "C:\ti\TivaWare_C_Series-2.1.4.178\driverlib\rom.h"
#include "C:\ti\TivaWare_C_Series-2.1.4.178\driverlib\pin_map.h"
#include "C:\ti\TivaWare_C_Series-2.1.4.178\driverlib\uart.h"
#include "grlib/grlib.h"
#include "utils/uartstdio.h"
#include "driverlib/gpio.h"
#include "driverlib/gpio.c"
#include "driverlib/timer.h"

int Start_flag = 0;
int Lead_Lag_Flag = 0;
uint32_t DeltaT = 0;
uint32_t Phase_offset = 0;
uint32_t Period = 0;
int status = 0;


/*Check if Driver Library Encounters an error */
#ifdef DEBUG
void
__error__(char *pcFilename, uint32_t ui32Line)
{
}
#endif

/*Functions*/
extern void IntHandler_Phase(void);
extern void IntHandler_Period(void);
void System_Init();


void IntHandler_Phase()
{
	/*Clear Interrupt*/
	status = GPIOIntStatus(GPIO_PORTC_BASE,true);
	GPIOIntClear(GPIO_PORTC_BASE,status);


		if (GPIOPinRead(GPIO_PORTC_BASE, GPIO_PIN_7) == 128) //Check if it was a rising edge.
		{

			TimerEnable(TIMER0_BASE, TIMER_A); //If it was a rising edge, start timer.

		}

		else //If falling edge, perform measurment.
		{

			/*Calculate DeltaT*/
			DeltaT = (65535 - TimerValueGet(TIMER0_BASE, TIMER_A));

			/*Reset and Reload Timer*/
			TimerDisable(TIMER0_BASE, TIMER_A);
			SysCtlPeripheralEnable(SYSCTL_PERIPH_TIMER0);
			TimerConfigure(TIMER0_BASE, TIMER_CFG_SPLIT_PAIR | TIMER_CFG_A_ONE_SHOT);
			TimerLoadSet(TIMER0_BASE, TIMER_A, 65535);


		}


}

void IntHandler_Period()
{
	//Clear Interrupt
    status = GPIOIntStatus(GPIO_PORTD_BASE,true);
	GPIOIntClear(GPIO_PORTD_BASE,status);

	/*check to see if this is the first falling edge of the program */
	if (Start_flag ==  0)
	{
		Start_flag = 1;
	    TimerEnable(TIMER1_BASE, TIMER_B);
	}

	else
	{
		/*Measure Period and calculate Phase Offset */
		Period = 65535-TimerValueGet(TIMER1_BASE, TIMER_B);
		Phase_offset = (((DeltaT * 10000)/(Period)) * 360u )/10000;

		/*Reset, Reload and Re-Enable Timer */
		TimerDisable(TIMER1_BASE, TIMER_B);
		SysCtlPeripheralEnable(SYSCTL_PERIPH_TIMER1);
		TimerConfigure(TIMER1_BASE, TIMER_CFG_SPLIT_PAIR | TIMER_CFG_B_ONE_SHOT);
	    TimerLoadSet(TIMER1_BASE, TIMER_B, 65535);
	    TimerEnable(TIMER1_BASE, TIMER_B);

	}

	if (GPIOPinRead(GPIO_PORTC_BASE, GPIO_PIN_7) == 128)//Check if XOR output is High
		{
			Lead_Lag_Flag = 1;
		}
	else
		{
			Lead_Lag_Flag = 0;
		}

}


//*****************************************************************************
//
// Configure the UART
//
//*****************************************************************************
void
Configure_UART(void)
{
	/*Enable GPIO and UART*/
    ROM_SysCtlPeripheralEnable(SYSCTL_PERIPH_GPIOA);
    ROM_SysCtlPeripheralEnable(SYSCTL_PERIPH_UART0);

    /*Configure GPIO A0 and A1 as UARTS*/
    ROM_GPIOPinConfigure(GPIO_PA0_U0RX);
    ROM_GPIOPinConfigure(GPIO_PA1_U0TX);
    ROM_GPIOPinTypeUART(GPIO_PORTA_BASE, GPIO_PIN_0 | GPIO_PIN_1);

    /*Designate UART Clock Source*/
    UARTClockSourceSet(UART0_BASE, UART_CLOCK_SYSTEM);

    UARTStdioConfig(0, 115200, 3125000);
}


//*****************************************************************************
//
//Initiate System, wait for interrupts and print out phase offset value
//
//*****************************************************************************
int
main(void)
{


     System_Init();

	 while(1)
	 {
		 	if (Lead_Lag_Flag == 1)
		 	{
			 UARTprintf("Phase offset = %i Lagging \n", Phase_offset); //Measured Signal Lagging Reference Signal
		 	}

		 	else
		 	{
		 	UARTprintf("Phase offset = %i Leading \n", Phase_offset); //Measured Signal Leading Reference Signal
		 	}

	 }


}

void System_Init()
{
		SysCtlClockSet(SYSCTL_SYSDIV_64 | SYSCTL_USE_PLL | SYSCTL_XTAL_16MHZ |
			                       SYSCTL_OSC_MAIN); //  200 MHz / sysdiv = 3.125 MHZ

		/*Init Port C*/
		SysCtlPeripheralEnable(SYSCTL_PERIPH_GPIOC);
		SysCtlDelay(3);

		GPIOPinTypeGPIOInput(GPIO_PORTC_BASE, GPIO_INT_PIN_7);
		GPIOPadConfigSet(GPIO_PORTC_BASE ,GPIO_INT_PIN_7,GPIO_STRENGTH_2MA,GPIO_PIN_TYPE_STD_WPD);
		GPIOIntTypeSet(GPIO_PORTC_BASE,GPIO_PIN_7,GPIO_BOTH_EDGES); //Set Interrupt Type on PORT C, pin 7

		/*Init Port D*/
		SysCtlPeripheralEnable(SYSCTL_PERIPH_GPIOD);
		SysCtlDelay(3);

		GPIOPinTypeGPIOInput(GPIO_PORTD_BASE, GPIO_INT_PIN_1);
		GPIOPadConfigSet(GPIO_PORTD_BASE ,GPIO_INT_PIN_1,GPIO_STRENGTH_2MA,GPIO_PIN_TYPE_STD_WPD);
		GPIOIntTypeSet(GPIO_PORTD_BASE,GPIO_PIN_1,GPIO_RISING_EDGE);//Set Interrupt Type on PORT B, pin 1

		/*Register Interrupts*/
		GPIOIntRegister(GPIO_PORTC_BASE,IntHandler_Phase);
		GPIOIntRegister(GPIO_PORTD_BASE,IntHandler_Period);

		/*Configure UART */
		Configure_UART();

		/*Configure Timers*/
		SysCtlPeripheralEnable(SYSCTL_PERIPH_TIMER0);
		SysCtlPeripheralEnable(SYSCTL_PERIPH_TIMER1);

		TimerConfigure(TIMER0_BASE, TIMER_CFG_SPLIT_PAIR | TIMER_CFG_A_ONE_SHOT);
		TimerConfigure(TIMER1_BASE, TIMER_CFG_SPLIT_PAIR | TIMER_CFG_B_ONE_SHOT);

		TimerLoadSet(TIMER0_BASE, TIMER_A, 65535);
		TimerLoadSet(TIMER1_BASE, TIMER_B, 65535);

		IntPrioritySet(INT_GPIOC_TM4C123, 0); //Set Period Interrupt at highest priority.

		/*Enable Interrupts*/
		GPIOIntEnable(GPIO_PORTC_BASE, GPIO_INT_PIN_7);
		GPIOIntEnable(GPIO_PORTD_BASE, GPIO_INT_PIN_1);

}


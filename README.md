# BPMProject
Developing an embedded system to read heartbeat using a micro-controller 

#include <hidef.h>      /* common defines and macros */
#include "derivative.h"      /* derivative-specific definitions */
#include "SCI.h"

//initialize global variables used throughout functions and main 
char string[20];
unsigned short n;  
int counter = 0;
unsigned short val;

int upper = 0;
int prebpm = 0; 
int bpm = 0;
int increment = 0;

int hundreds = 0;
int tens = 0;
int ones = 0;
int remainder = 0;


void SetClk14(void);

void OutCRLF(void);

void delayby1ms(int);

void BCDConvert(int);


void main(void) {

  EnableInterrupts;	
	SetClk14();
 
//statements below configure the Timer Input Capture                                                   
           
  TSCR1 = 0x90;    //Timer System Control Register 1
                    // TSCR1[7] = TEN:  Timer Enable (1-enable)
                    // TSCR1[4] = TFFCA:  Timer Fast Flag Clear All (1-read/write clears interrupt flags)
                    
  TSCR2 = 0x01;    //Timer System Control Register 2
                    // TSCR2[2:0] = Timer Prescaler Select: table22-12 of MC9S12G Family Reference Manual r1.25 (set for bus/1)
                    
  TIOS = 0xFE;     //Timer Input Capture or Output capture set TIC[0] and input (similar to DDR)
  
  PERT = 0x01;     //Enable Pull-Up resistor on TIC[0]

  TCTL3 = 0x00;    //TCTL3 & TCTL4 configure which edge(s) to capture
  
  TCTL4 = 0x02;    //Configured for falling edge on TIC[0]
  
  TIE = 0x01;      //Timer Interrupt Enable
  

//LED DDR configuration  

  DDR1AD = 0x7F;  //lower bits set for output except for AN7(signal input)
  DDRP = 0x03;    //upper bits set for output except for PP2, used as AN7(signal input)
     
   

// Setup and enable ADC channel 7
// Refer to Chapter 14 in S12G Reference Manual for details
		
	ATDCTL1 = 0x4F;		// set for 12-bit resolution
	ATDCTL3 = 0x88;		// right justified, one sample per sequence
	ATDCTL4 = 0x00;		// prescaler = 0; ATD clock = 14MHz / (2 * (0 + 1)) == 7 MHz
	ATDCTL5 = 0x27;		// continuous conversion on channel 7
	
// Setup LED and SCI
  DDRJ |= 0x01;     // PortJ bit 0 is output to LED D2 on DIG13
  SCI_Init(57600);
    
//program execution of button, SCI, BPM calculation and BCD conversion   
  for(;;) {
  
//debounced button    
    if(counter == 1){
      delayby1ms(30);  
      if(counter == 1){  

//toggle LED to show serial communication is active  
      PTJ ^= 0x01;         
      val=ATDDR0;
      SCI_OutUDec(val);
      OutCRLF();
      
// delay for sampling time      
      delayby1ms(69); 
      increment = increment + 1; 
       
//threshold method, threshold set for 2.8V CHANGE THRESHOLD TO LARGER NUMBER TO ACCOUNT FOR NOISE/ POOR SIGNAL       
//if voltage equals threshold        
        
      if((val > 3960) && (upper == 0) ){
        upper = 1;     
      }
      if ((val<3960) && (upper == 1)){
        prebpm = prebpm + 1; 
        upper = 0;
      }
      if (increment == 100){
        bpm = prebpm*6;
        increment = 0;
        prebpm = 0;
      } 
        
//convert bpm to bcd
      BCDConvert(bpm);     
                                                                                                                                                                                                                   
      }  
    } 
  
  
  }   



}


interrupt  VectorNumber_Vtimch0 void ISR_Vtimch0(void)
{
  unsigned int temp; //DON'T EDIT THIS
  int i=0;
 
  DDRJ = 0x01;
  
  if(counter==1){
    SCI_OutString("Goodbye!");OutCRLF();
    counter =0;
  }
  
  else if(counter==0){
    SCI_OutString("Hello!");OutCRLF();  
    counter =1;
  }
   
  temp = TC0;       //Refer back to TFFCA, we enabled FastFlagClear, thus by reading the Timer Capture input we automatically clear the flag, allowing another TIC interrupt
  }



void SetClk14(void){
CPMUREFDIV = 0x83;	//ratio for setting e clk 
CPMUSYNR = 0x06;	//ratio for setting e clk 
CPMUPOSTDIV = 0x00;	//controls the frequency btwn VCOCLK and PLLCLK
CPMUCLKS = 0x80;	//the CPMUCLK register is part of the CPMU register setting CPMUCLK to 0x80 sets the first bit (PLLSEL) whcih selects the PLLCLK 
CPMUOSC = 0x80;	//register to configure external oscillator 
while(!(CPMUFLG & 0x08));
}



void OutCRLF(void){
  SCI_OutChar(CR);
  SCI_OutChar(LF);
  PTJ ^= 0x20;          // toggle LED D2
}



void delayby1ms(int k){
	 int ix; 
//enable timer and fast flag clear
	 TSCR1 = 0x90;
	 
// disable timer interrupt, select prescaler to 1
	 TSCR2 = 0x00;
	 
// timer input capture, output compare register bits set to 1 are input capture and set to 0 are output compare
	 TIOS |= 0x02;  //'or' the value in TIOS, initially setting TIOS to IOS1, for input capture
	 
	 TC1 = TCNT + 14000;//Time capture 1 is set to capture the timer count register plus the bus speed of 14Khz 
	 
	 for(ix= 0; ix<k; ix++){
		 while(!(TFLG1_C1F));
			 
		 TC1+= 14000; //increment time capture 1 by 1000th of the bus clk 
	 } 
	 
	 TIOS &= ~0x01; //'and' the value of the TIOS register with 'FE'
 }
 
 void BCDConvert(int m){
 if(m/100 >0){
        hundreds = m/100;
        m = m - 100;
      }else {
        hundreds = 0;
      }
      
      
      if (m/10 > 0){
        tens = m/10;
        m = m - (tens*10);
      }else {
        tens = 0;  
      }
      
      if (m>0){
        ones = m; 
      }else{
        ones = 0;
      }
      
      
      
      if(hundreds != 0){
        PT1AD_PT1AD0 = 1;
      }else {
        PT1AD_PT1AD0 = 0; 
      }
      
      
      
      if(tens/8 > 0){
        PT1AD_PT1AD1 = 1; 
        remainder = tens%8;
      } else {
          PT1AD_PT1AD1 = 0;
          remainder =tens%8;
      }
      
      
      if (remainder/4 >0){
        PT1AD_PT1AD2 = 1;
        remainder = remainder%4;
      } else {
          PT1AD_PT1AD2 = 0;
          remainder = remainder%4;
      }
      
      
      if(remainder/2 > 0){
        PT1AD_PT1AD3 = 1;
        remainder = remainder%2;   
      } else {
          PT1AD_PT1AD3 = 0;
          remainder = remainder%2;
      }
      
      if(remainder > 0){
        PT1AD_PT1AD4 = 1; 
      } else {
          PT1AD_PT1AD4 = 0; 
      }
      
      
      
            
      if(ones/8 > 0){
        PT1AD_PT1AD5 = 1;
        remainder = ones%8;
      } else {
          PT1AD_PT1AD5 = 0;
          remainder = ones%8;
      }
      
      
      if(remainder/4 > 0){
        PT1AD_PT1AD6 = 1;
        remainder = remainder%4;
      } else {
          PT1AD_PT1AD6 = 0;
          remainder =remainder%4;
      }
      
      
      if(remainder/2 > 0){
        PTP_PTP0 = 1;
        remainder = remainder%2;
      } else {
          PTP_PTP0 = 0;
          remainder = remainder%2;           
      }
      
      if(remainder > 0){
        PTP_PTP1 = 1;
      } else {
          PTP_PTP1 = 0;          
      }
     

}

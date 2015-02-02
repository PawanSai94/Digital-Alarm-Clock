# Digital-Alarm-Clock

// Digital Alarm Clock using DS1307RTC and 8051

LCD DATA P2 ; define LCD data port on port 1
Busy BIT LCD.7 ; define LCD busy flag
EN BIT P3.5 ; define LCD enable pin on port 2.2
RW BIT P3.6 ; define LCD register select pin on port 2.0
RS BIT P3.7 ; define LCD read/write pin on port 2.1
;**************************************************************
; KEYS CONNECTIONS
UP BIT P1.3
DN BIT P1.2
Rel_Set BIT P3.4
T_Set BIT P3.2
A_Set BIT P3.3
;**************************************************************
Rel_Out BIT P1.4
;**************************************************************
; I2C
SCL BIT P1.0 ; I2C serial clock line
SDA BIT P1.1 ; I2C serial data line
;**************************************************************
; Slave Address
SAW EQU 0D0H ; Slave address for write (DS1307)
SAR EQU 0D1H ; Slave address for read (DS1307)
;**************************************************************
BitCnt DATA 20H ; BIT COUNTER FOR I2C ROUTINES
Sec DATA 21H ; SECONDS STORAGE RAM
Min DATA 22H ; MINUTES STORAGE RAM
Hour DATA 23H ; HOURS STORAGE RAM
Day DATA 24H ; DAY STORAGE RAM
Date DATA 25H ; DATE STORAGE RAM
Month DATA 26H ; MONTH STORAGE RAM
Year DATA 27H ; YEAR STORAGE RAM
SQW DATA 28H ; SQUARE-WAVE CONTROL
AL_Hour DATA 29H ; ALARM HOURS STORAGE RAM SPACE
AL_Min DATA 2AH ; ALARM MINUTES STORAGE RAM SPACE
Rel_H DATA 2BH ; RELAY HOURS STORAGE RAM SPACE (FOR ON)
Rel_M DATA 2CH ; RELAY MINUTES STORAGE RAM SPACE (FOR ON)
RH_Off DATA 2DH ; RELAY HOURS STORAGE RAM SPACE (FOR OFF)
RM_Off DATA 2EH ; RELAY MINUTES STORAGE RAM SPACE (FOR OFF)
Flags DATA 2FH ; FLAGS
AL_Flag BIT Flags.0 ; ALARM FLAG
RL_Flag BIT Flags.1 ; RELAY TIMER FLAG
RL_On BIT FlagS.2
;**************************************************************
; ***MACRO'S***
I2C_Delay MACRO
NOP
ENDM
;--------------------------------------------------------------
SCLHigh MACRO
SETB SCL
JNB SCL,$
ENDM
;--------------------------------------------------------------
D_Str MACRO
ACALL Command
LCALL Disp_Char
ENDM
;--------------------------------------------------------------
CLR_LCD MACRO
MOV A,#01H
ACALL Command
ENDM
;**************************************************************
ORG 0000H
;--------------------------------------------------------------
CLR AL_Flag
CLR RL_Flag
CLR RL_On
MOV SP,#2FH
MOV SQW,#80H
ACALL SQW_W
ACALL I2C_Start
MOV A,#SAW
ACALL I2C_Write
MOV A,#3FH
ACALL I2C_Write
ACALL I2C_Start
MOV A,#SAR
ACALL I2C_Write
ACALL I2C_Read_Last
ACALL I2C_Stop
CJNE A,#'~',INIT
LJMP LCD_I
;--------------------------------------------------------------
INIT: ACALL I2C_Start
MOV A,#SAW
ACALL I2C_Write
MOV A,#3FH
ACALL I2C_Write
MOV A,#'~'
ACALL I2C_Write
MOV A,#00H
ACALL I2C_Write
ACALL I2C_Stop
SJMP LCD_I
;--------------------------------------------------------------
RTC_Reset:
MOV R0,#21H
MOV R6,#07H
ACALL I2C_Start
MOV A,#SAW
ACALL I2C_Write
MOV A,#00H
ACALL I2C_Write
Loop1: MOV A,@R0
ACALL I2C_Write
INC R0
DJNZ R6,Loop1
ACALL I2C_Stop
RET
;--------------------------------------------------------------
AL_Off: CLR AL_Flag
MOV SQW,#80H
ACALL SQW_W
CLR_LCD
MOV A,#84H
D_Str
DB 'Alarm Off!',0
MOV A,#82H
ACALL Command
MOV A,#01H
ACALL Data_Disp
ACALL Delay
ACALL RL_Logo
SJMP Main
;--------------------------------------------------------------
Alarm_Set:
JB AL_Flag,AL_Off
CLR_LCD
MOV A,#84H
D_Str
DB 'Set Alarm',0
MOV A,#82H
ACALL Command
MOV A,#01H
ACALL Data_Disp
ACALL Delay
AJMP AH
;--------------------------------------------------------------
LCD_I: ACALL LCD_Initial
ACALL CGR
;--------------------------------------------------------------
Main: ACALL Disp_C
Start: ACALL Key_Scan
Back1: MOV R0,#21H
MOV R1,#0DH
ACALL I2C_Start
MOV A,#SAW
ACALL I2C_Write
MOV A,#00H
ACALL I2C_Write
ACALL I2C_Start
MOV A,#SAR
ACALL I2C_Write
Loop: ACALL I2C_Read
DJNZ R1,Loop
ACALL I2C_Read_Last
ACALL I2C_Stop
ACALL Display
ACALL Alarm_Check
ACALL Relay_Check
SJMP Start
;--------------------------------------------------------------
Key_Scan:
JNB T_Set,Jump1
JNB A_Set,Alarm_Set
JNB Rel_Set,Jump2
RET
;--------------------------------------------------------------
Jump1: LJMP Time_Set
Jump2: SJMP Relay_Set
;--------------------------------------------------------------
Alarm_Check:
JNB AL_Flag,CH
MOV A,AL_Min
CJNE A,Min,Alarm_Off
MOV A,AL_Hour
CJNE A,Hour,Alarm_Off
ACALL Alarm_On
CH: RET
;--------------------------------------------------------------
Alarm_Off:
MOV SQW,#80H
ACALL SQW_W
RET
;--------------------------------------------------------------
RL_Off: CLR RL_Flag
SETB Rel_Out
CLR RL_On
CLR_LCD
MOV A,#84H
D_Str
DB 'Relay Off!',0
MOV A,#82H
ACALL Command
MOV A,#00H
ACALL Data_Disp
ACALL Delay
ACALL AL_Logo
SJMP Main
;--------------------------------------------------------------
SQW_W: ACALL I2C_Start
MOV A,#SAW
ACALL I2C_Write
MOV A,#07H
ACALL I2C_Write
MOV A,SQW
ACALL I2C_Write
ACALL I2C_Stop
RET
;--------------------------------------------------------------
Alarm_On:
MOV SQW,#10H
ACALL SQW_W
RET
;--------------------------------------------------------------
Relay_Check:
JNB RL_Flag,CH1
JB RL_On,Rel_Off
MOV A,Rel_M
CJNE A,Min,Relay_Off
MOV A,Rel_H
CJNE A,Hour,Relay_Off
SETB RL_On
ACALL Relay_On
CH1: RET
;--------------------------------------------------------------
Relay_Off:
SETB Rel_Out
RET
;--------------------------------------------------------------
Relay_On:
CLR Rel_Out
RET
;--------------------------------------------------------------
Relay_Set:
JB RL_Flag,RL_Off
LJMP R_Set
;--------------------------------------------------------------
Rel_Off:MOV A,RM_Off
CJNE A,Min,Relay_On
MOV A,RH_Off
CJNE A,Hour,Relay_On
CLR RL_On
ACALL Relay_Off
CH2: RET
;--------------------------------------------------------------
Time_Set:
CLR_LCD
MOV A,#83H
D_Str
DB 'Set Hours:',0
MOV A,#0C7H
ACALL Command
MOV A,Hour
MOV R7,A
ACALL Disp_BCD
JNB T_Set,$
I24: PUSH ACC
MOV A,#0C7H
ACALL Command
POP ACC
KS24: JNB UP,Inc_Hour
JNB DN,Dec_Hour
JNB T_Set,Done_H
SJMP KS24
Inc_Hour:
NOP
ACALL BCD_2_Hex
INC A
CJNE A,#18H,SH
MOV A,#00H
SH: ACALL Hex_2_BCD
ACALL Disp_BCD
JNB UP,$
JNB DN,$
SJMP I24
Dec_Hour:
NOP
ACALL BCD_2_Hex
DEC A
CJNE A,#00H-1,SH
MOV A,#17H
SJMP SH
Done_H: JNB T_Set,$
MOV Hour,R7
;--------------------------------------------------------------
Mint: CLR_LCD
MOV A,#82H
D_Str
DB 'Set Minutes:',0
MOV A,#0C7H
ACALL Command
MOV A,Min
MOV R7,A
ACALL Disp_BCD
JNB T_Set,$
M1: PUSH ACC
MOV A,#0C7H
ACALL Command
POP ACC
KSMIN: JNB UP,Inc_Min
JNB DN,Dec_Min
JNB T_Set,DoneMin
SJMP KSMIN
Inc_Min:NOP
ACALL BCD_2_Hex
INC A
CJNE A,#3CH,SMin
MOV A,#00H
SMin: ACALL Hex_2_BCD
ACALL Disp_BCD
JNB UP,$
JNB DN,$
SJMP M1
Dec_Min:NOP
ACALL BCD_2_Hex
DEC A
CJNE A,#00H-1,SMin
MOV A,#3BH
SJMP SMin
DoneMin:JNB T_Set,$
MOV Min,R7
;--------------------------------------------------------------
CLR_LCD
MOV A,#83H
D_Str
DB 'Set Days:',0
MOV A,#0C5H
D_Str
DB '* *',0
MOV A,#0C6H
ACALL Command
MOV A,Day
PUSH ACC
LCALL W_Day
POP ACC
JNB T_Set,$
D1: PUSH ACC
MOV A,#0C6H
ACALL Command
POP ACC
KSDAY: JNB UP,Inc_Day
JNB DN,Dec_Day
JNB T_Set,DoneDay
SJMP KSDAY
Inc_Day:NOP
INC A
CJNE A,#08H,SDay
MOV A,#01H
SDay: PUSH ACC
LCALL W_Day
POP ACC
JNB UP,$
JNB DN,$
SJMP D1
Dec_Day:NOP
DEC A
CJNE A,#00H,SDay
MOV A,#07H
SJMP SDay
DoneDay:JNB T_Set,$
MOV Day,A
;--------------------------------------------------------------
CLR_LCD
MOV A,#83H
D_Str
DB 'Set Date:',0
MOV A,#0C7H
ACALL Command
MOV A,Date
MOV R7,A
ACALL Disp_BCD
JNB T_Set,$
DA1: PUSH ACC
MOV A,#0C7H
ACALL Command
POP ACC
KSDAT: JNB UP,Inc_DAT
JNB DN,Dec_DAT
JNB T_Set,DoneDAT
SJMP KSDAT
Inc_DAT:NOP
ACALL BCD_2_Hex
INC A
CJNE A,#20H,SDAT
MOV A,#01H
SDAT: ACALL Hex_2_BCD
ACALL Disp_BCD
JNB UP,$
JNB DN,$
SJMP DA1
Dec_DAT:NOP
ACALL BCD_2_Hex
DEC A
CJNE A,#00H,SDAT
MOV A,#1FH
SJMP SDAT
DoneDAT:JNB T_Set,$
MOV Date,R7
;--------------------------------------------------------------
CLR_LCD
MOV A,#83H
D_Str
DB 'Set Month:',0
MOV A,#0C7H
ACALL Command
MOV A,Month
MOV R7,A
ACALL Disp_BCD
JNB T_Set,$
MM1: PUSH ACC
MOV A,#0C7H
ACALL Command
POP ACC
KSMON: JNB UP,Inc_MON
JNB DN,Dec_MON
JNB T_Set,DoneMON
SJMP KSMON
Inc_MON:NOP
ACALL BCD_2_Hex
INC A
CJNE A,#0DH,SMON
MOV A,#01H
SMON: ACALL Hex_2_BCD
ACALL Disp_BCD
JNB UP,$
JNB DN,$
SJMP MM1
Dec_MON:NOP
ACALL BCD_2_Hex
DEC A
CJNE A,#00H,SMON
MOV A,#0CH
SJMP SMON
DoneMON:JNB T_Set,$
MOV Month,R7
;--------------------------------------------------------------
CLR_LCD
MOV A,#83H
D_Str
DB 'Set Year:',0
MOV A,#0C6H
D_Str
DB '20',0
MOV A,#0C8H
ACALL Command
MOV A,Year
MOV R7,A
ACALL Disp_BCD
JNB T_Set,$
YY1: PUSH ACC
MOV A,#0C8H
ACALL Command
POP ACC
KSYY: JNB UP,Inc_YY
JNB DN,Dec_YY
JNB T_Set,DoneYY
SJMP KSYY
Inc_YY: NOP
ACALL BCD_2_Hex
INC A
CJNE A,#64H,SYY
MOV A,#00H
SYY: ACALL Hex_2_BCD
ACALL Disp_BCD
JNB UP,$
JNB DN,$
SJMP YY1
Dec_YY: NOP
ACALL BCD_2_Hex
DEC A
CJNE A,#00H-1,SYY
MOV A,#63H
SJMP SYY
DoneYY: JNB T_Set,$
MOV Year,R7
MOV Sec,#00H
ACALL RTC_Reset
ACALL Done
ACALL Disp_C
ACALL RL_Logo
ACALL AL_Logo
LJMP Main
;--------------------------------------------------------------
Done:
CLR_LCD
MOV A,#86H
D_Str
DB 'Done!',0
ACALL Delay
RET
;--------------------------------------------------------------
Delay: MOV R2,#0FFH
MOV R3,#14H
LP3: MOV R2,#0FFH
LP2: MOV R5,#0FFH
LP1: DJNZ R5,LP1
DJNZ R2,LP2
DJNZ R3,LP3
RET
;--------------------------------------------------------------
AH: CLR_LCD
MOV A,#83H
D_Str
DB 'Set Hours:',0
MOV A,#0C7H
ACALL Command
MOV A,AL_Hour
MOV R7,A
ACALL Disp_BCD
ALH1: PUSH ACC
MOV A,#0C7H
ACALL Command
POP ACC
ALH2: JNB UP,Inc_AL_Hour
JNB DN,Dec_AL_Hour
JNB A_Set,DoneALH
SJMP ALH2
Inc_AL_Hour:
NOP
ACALL BCD_2_Hex
INC A
CJNE A,#18H,Z_AL_H
MOV A,#00H
Z_AL_H: ACALL Hex_2_BCD
ACALL Disp_BCD
JNB UP,$
JNB DN,$
SJMP ALH1
Dec_AL_Hour:
NOP
ACALL BCD_2_Hex
DEC A
CJNE A,#00H-1,Z_AL_H
MOV A,#17H
SJMP Z_AL_H
DoneALH:JNB A_Set,$
MOV AL_Hour,R7
CLR_LCD
MOV A,#82H
D_Str
DB 'Set Minutes:',0
MOV A,#0C7H
ACALL Command
MOV A,AL_Min
MOV R7,A
ACALL Disp_BCD
JNB A_Set,$
ALM1: PUSH ACC
MOV A,#0C7H
ACALL Command
POP ACC
ALM2: JNB UP,Inc_AL_Min
JNB DN,Dec_AL_Min
JNB A_Set,DoneAL_Min
SJMP ALM2
Inc_AL_Min:
NOP
ACALL BCD_2_Hex
INC A
CJNE A,#3CH,SAL_Min
MOV A,#00H
SAL_Min:ACALL Hex_2_BCD
ACALL Disp_BCD
JNB UP,$
JNB DN,$
SJMP ALM1
Dec_AL_Min:
NOP
ACALL BCD_2_Hex
DEC A
CJNE A,#00H-1,SAL_Min
MOV A,#3BH
SJMP SAL_Min
DoneAL_Min:
JNB A_Set,$
MOV AL_Min,R7
SETB AL_Flag
ACALL ALT_Done
ACALL Done
ACALL RL_Logo
ACALL AL_Logo
LJMP Main
;--------------------------------------------------------------
R_Set: CLR_LCD
MOV A,#81H
D_Str
DB ' Set Timer(ON)',0
MOV A,#80H
ACALL Command
MOV A,#00H
ACALL Data_Disp
ACALL Delay
CLR_LCD
MOV A,#83H
D_Str
DB 'Set Hours:',0
MOV A,#0C7H
ACALL Command
MOV A,Rel_H
MOV R7,A
ACALL Disp_BCD
JNB Rel_Set,$
RTH1: PUSH ACC
MOV A,#0C7H
ACALL Command
POP ACC
KHREL: JNB UP,Inc_Rel_H
JNB DN,Dec_Rel_H
JNB Rel_Set,DoneRel_H
SJMP KHREL
Inc_Rel_H:
NOP
ACALL BCD_2_Hex
INC A
CJNE A,#18H,HRel
MOV A,#00H
HRel: ACALL Hex_2_BCD
ACALL Disp_BCD
JNB UP,$
JNB DN,$
SJMP RTH1
Dec_Rel_H:
NOP
ACALL BCD_2_Hex
DEC A
CJNE A,#00H-1,HRel
MOV A,#17H
SJMP HRel
DoneRel_H:
JNB Rel_Set,$
MOV Rel_H,R7
CLR_LCD
MOV A,#82H
D_Str
DB 'Set Minutes:',0
MOV A,#0C7H
ACALL Command
MOV A,Rel_M
MOV R7,A
ACALL Disp_BCD
JNB Rel_Set,$
RTM1: PUSH ACC
MOV A,#0C7H
ACALL Command
POP ACC
KMREL: JNB UP,Inc_Rel_M
JNB DN,Dec_Rel_M
JNB Rel_Set,DoneRel_M
SJMP KMREL
Inc_Rel_M:
NOP
ACALL BCD_2_Hex
INC A
CJNE A,#3CH,MRel
MOV A,#00H
MRel: ACALL Hex_2_BCD
ACALL Disp_BCD
JNB UP,$
JNB DN,$
SJMP RTM1
Dec_Rel_M:
NOP
ACALL BCD_2_Hex
DEC A
CJNE A,#00H-1,MRel
MOV A,#3BH
SJMP MRel
DoneRel_M:
JNB Rel_Set,$
MOV Rel_M,R7
Off: CLR_LCD
MOV A,#81H
D_Str
DB ' Set Timer(OFF)',0
MOV A,#80H
ACALL Command
MOV A,#00H
ACALL Data_Disp
ACALL Delay
CLR_LCD
MOV A,#83H
D_Str
DB 'Set Hours:',0
MOV A,#0C7H
ACALL Command
MOV A,RH_Off
MOV R7,A
ACALL Disp_BCD
JNB Rel_Set,$
RTHF1: PUSH ACC
MOV A,#0C7H
ACALL Command
POP ACC
KHFREL: JNB UP,Inc_Rel_HF
JNB DN,Dec_Rel_HF
JNB Rel_Set,DoneRel_HF
SJMP KHFREL
Inc_Rel_HF:
NOP
ACALL BCD_2_Hex
INC A
CJNE A,#18H,HFRel
MOV A,#00H
HFRel: ACALL Hex_2_BCD
ACALL Disp_BCD
JNB UP,$
JNB DN,$
SJMP RTHF1
Dec_Rel_HF:
NOP
ACALL BCD_2_Hex
DEC A
CJNE A,#00H-1,HFRel
MOV A,#17H
SJMP HFRel
DoneRel_HF:
JNB Rel_Set,$
MOV RH_Off,R7
CLR_LCD
MOV A,#82H
D_Str
DB 'Set Minutes:',0
MOV A,#0C7H
ACALL Command
MOV A,RM_Off
MOV R7,A
ACALL Disp_BCD
JNB Rel_Set,$
RFTM1: PUSH ACC
MOV A,#0C7H
ACALL Command
POP ACC
KFMREL: JNB UP,Inc_Rel_MF
JNB DN,Dec_Rel_MF
JNB Rel_Set,DoneRel_MF
SJMP KFMREL
Inc_Rel_MF:
NOP
ACALL BCD_2_Hex
INC A
CJNE A,#3CH,MFRel
MOV A,#00H
MFRel: ACALL Hex_2_BCD
ACALL Disp_BCD
JNB UP,$
JNB DN,$
SJMP RFTM1
Dec_Rel_MF:
NOP
ACALL BCD_2_Hex
DEC A
CJNE A,#00H-1,MFRel
MOV A,#3BH
SJMP MFRel
DoneRel_MF:
JNB Rel_Set,$
MOV RM_Off,R7
SETB RL_Flag
ACALL ALT_Done
ACALL Done
ACALL RL_Logo
ACALL AL_Logo
LJMP Main
;--------------------------------------------------------------
RL_Logo:JNB RL_Flag,WE1
MOV A,#0CDH
ACALL Command
MOV A,#00H
ACALL Data_Disp
WE1: RET
;--------------------------------------------------------------
AL_Logo:JNB AL_Flag,WE2
MOV A,#0C2H
ACALL Command
MOV A,#01H
ACALL Data_Disp
WE2: RET
;--------------------------------------------------------------
ALT_Done:
MOV R1,#29H
MOV R3,#07H
ACALL I2C_Start
MOV A,#SAW
ACALL I2C_Write
MOV A,#08H
ACALL I2C_Write
LOOP4: MOV A,@R1
ACALL I2C_Write
INC R1
DJNZ R3,LOOP4
ACALL I2C_Stop
RET
;--------------------------------------------------------------
LCD_Initial:
MOV A,#38H
ACALL Command
MOV A,#0CH
ACALL Command
CLR_LCD
MOV A,#06H
ACALL Command
RET
;--------------------------------------------------------------
Display:MOV R1,#21H
MOV A,#0CAH
ACALL Command
MOV A,@R1
ACALL Disp_BCD
;
INC R1
MOV A,#0C7H
ACALL Command
MOV A,@R1
ACALL Disp_BCD
;
INC R1
MOV A,#0C4H
ACALL Command
MOV A,@R1
ACALL Disp_BCD
;
INC R1
MOV A,#80H
ACALL Command
MOV A,@R1
LCALL W_Day
;
INC R1
MOV A,#86H
ACALL Command
MOV A,@R1
ACALL Disp_BCD
;
INC R1
MOV A,#89H
ACALL Command
MOV A,@R1
ACALL Disp_BCD
;
INC R1
MOV A,#8EH
ACALL Command
MOV A,@R1
ACALL Disp_BCD
RET
;--------------------------------------------------------------
Hex_2_BCD:
MOV B,#00001010B
DIV AB
MOV R3,B
MOV B,#00010000B
MUL AB
ADD A,R3
MOV R7,A
RET
;--------------------------------------------------------------
BCD_2_Hex:
MOV B,#00010000B
DIV AB
MOV R3,B
MOV B,#00001010B
MUL AB
ADD A,R3
RET
;--------------------------------------------------------------
Disp_BCD:
PUSH ACC
MOV R5,A
ANL A,#11110000B
SWAP A
MOV DPTR,#Ascii_Code
MOVC A,@A+DPTR
ACALL Data_Disp
MOV A,R5
ANL A,#00001111B
MOVC A,@A+DPTR
ACALL Data_Disp
POP ACC
RET
;--------------------------------------------------------------
Disp_C: MOV A,#80H
D_Str
DB ' / /20 ',0
MOV A,#0C0H
ACALL Command
MOV A,#'*'
ACALL Data_Disp
MOV A,#0C6H
ACALL Command
MOV A,#':'
ACALL Data_Disp
MOV A,#0C9H
ACALL Command
MOV A,#':'
ACALL Data_Disp
MOV A,#0CFH
ACALL Command
MOV A,#'*'
ACALL Data_Disp
RET
;--------------------------------------------------------------
CGR: MOV R4,#08H
MOV R5,#40H
MOV DPTR,#Clock
ACALL WRI
MOV R4,#08H
MOV R5,#48H
MOV DPTR,#Bell
ACALL WRI
RET
;--------------------------------------------------------------
WRI: CLR A
ACALL Get_Ready
MOV LCD,R5
CLR RS
CLR RW
SETB EN
CLR EN
INC R5
MOVC A,@A+DPTR
ACALL Data_Disp
INC DPTR
DJNZ R4,WRI
RET
;---------------------------------------;
; ************I2C Commands************* ;
;---------------------------------------;
I2C_Start:
SETB SCL
SETB SDA
I2C_Delay
CLR SDA
I2C_Delay
CLR SCL
RET
;--------------------------------------------------------------
I2C_Stop:
CLR SDA
SETB SCL
I2C_Delay
SETB SDA
RET
;--------------------------------------------------------------
I2C_Write:
MOV BitCnt,#08H
I2C_Write_Loop:
RLC A
MOV SDA,C
NOP
SCLHigh
CLR SCL
DJNZ BitCnt,I2C_Write_Loop
NOP
SETB SDA
NOP
SETB SCL
I2C_Delay
MOV C,SDA
CLR SCL
NOP
JNC Label
ACALL I2C_Stop
ACALL I2C_Start
SJMP I2C_Write
Label: RET
;--------------------------------------------------------------
I2C_Read_Dummy:
SETB SDA
CLR A
MOV BitCnt,#08H
I2C_Read_Loop:
CLR SCL
I2C_Delay
SCLHigh
MOV C,SDA
RLC A
DJNZ BitCnt,I2C_Read_Loop
CLR SCL
MOV @R0,A
INC R0
RET
;--------------------------------------------------------------
I2C_Ack_Write:
CLR SDA
NOP
SETB SCL
I2C_Delay
CLR SCL
SETB SDA
I2C_Delay
RET
;--------------------------------------------------------------
I2C_Nack_Write:
SETB SDA
NOP
SETB SCL
I2C_Delay
CLR SCL
I2C_Delay
RET
;--------------------------------------------------------------
I2C_Read:
ACALL I2C_Read_Dummy
ACALL I2C_Ack_Write
RET
;--------------------------------------------------------------
I2C_Read_Last:
ACALL I2C_Read_Dummy
ACALL I2C_Nack_Write
RET
;--------------------------------------;
; ************LCD COMMANDS*************;
;--------------------------------------;
Command:ACALL Get_Ready
MOV LCD,A
CLR RS
CLR RW
SETB EN
CLR EN
RET
;--------------------------------------------------------------
Data_Disp:
ACALL Get_Ready
MOV LCD,A
SETB RS
CLR RW
SETB EN
CLR EN
RET
;--------------------------------------------------------------
Get_Ready:
SETB Busy
CLR RS
SETB RW
Back: CLR EN
SETB EN
JB Busy,BACK
RET
;--------------------------------------------------------------
Disp_Char:
POP DPH
POP DPL
Print_Text:
CLR A
MOVC A,@A+DPTR
CJNE A,#00H,Loop2
SJMP Return
Loop2: MOV R4, A
LCALL Data_Disp
INC DPTR
LJMP Print_Text
Return: MOV A,#01H
JMP @A+DPTR
;--------------------------------------------------------------
W_Day: CJNE A,#01H,MON
LCALL Disp_Char
DB 'Sun',0
RET
MON: CJNE A,#02H,TUE
LCALL Disp_Char
DB 'Mon',0
RET
TUE: CJNE A,#03H,WED
LCALL Disp_Char
DB 'Tue',0
RET
WED: CJNE A,#04H,THU
LCALL Disp_Char
DB 'Wed',0
RET
THU: CJNE A,#05H,FRI
LCALL Disp_Char
DB 'Thu',0
RET
FRI: CJNE A,#06H,SAT
LCALL Disp_Char
DB 'Fri',0
RET
SAT: CJNE A,#07H,WHAT
LCALL Disp_Char
DB 'Sat',0
RET
WHAT: RET
;--------------------------------------------------------------
Ascii_Code:
DB 30H,31H,32H,33H,34H,35H,36H,37H,38H,39H
;--------------------------------------------------------------
;Icons
Clock: DB 00H,0EH,15H,17H,11H,0EH,00H,00H
Bell: DB 04H,0EH,0EH,0EH,1FH,00H,04H,00H
;--------------------------------------------------------------
END

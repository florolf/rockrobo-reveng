# Test points

TPC37: 3V3 (STM32)
TPC38: SWDIO
TPC39: SWCLK
TPC40: NRST (ueber widerstand rechts neben YC1)
TPC41: GND
TPC42: BOOT0
BOOT1 -> inner layer

TPI1: einschaltbutton
TPI2: GND

TPL4: switched gnd for lidar
TPL7: GND

TPC21: RX
TPC22: TX
TPC23: GND

TPP8: connected to Out of QP14
TPP11: supply for AXP (before 0 ohm R)
TPP19: Vbat
TPP21: 5V supply fuer UP8 (3.3V LDO)

TPA6: not serial?
TPA7: not serial?
TPA9: not serial?
TPA10: not serial?

TPA22: GND
TPA17: (uboot?) not serial?

TPA24: not serial?
TPA20: not serial?
TPA21: GND
TPA26: (power?)
TPA27: GND
TPA28: not serial?

TPA23: supply for AXP
TPA25: AXP boot trigger (active low)
TPA11: power?

TP-RA16: GND
TP-CA83: activity, bursty. 400 khz square wave in the beginning. wtf?
TP-CA17: ?

# Misc

RP21/22 -> divider for Vbat measurement, low sinde also connected to more resistors
   top: 140k
   bot: 100k || (3.01M + 10k)

   -> pin 13 of up8 (CMPIN)

vbat -> linker 0ohm unter up8

bq2477x connected to i2c1 of stm32

up56 -> LDO nach 3.3V, output am R links davon, oder am grossen testpad links unten

QA1 schaltet PWRON vom AXP223 gegen gnd
 - an pin46 vom stm32 (PE15)
 - others?

vor power button press ist der axp nicht bestromt, danach mit 5V

power on button (der obere) geht auf die transistorgruppe ganz oben rechts unter cpb1

YV7VD: mosfet driver?
  - 2: input
  - 3 gnd
  - 4: out
  - 5: supply

pin 35 vom STM geht an interen linken ferrit(?) von UM5
pin 61 vom STM geht an unteren rechten ferrit von UM5
pin 36 geht an unteren linken ferrit von UM6

PB4 (pin 90) geht an RI31 (rueckseite)
 - R31 pullup towards Vopamp?!
 - pulldown durch button 2
PD7 (pin 88) pulled down by button 1

## QP 28
P channel mosfet, Drain -> Vbat

# Power distribution

SSEHC: power switch?
 - 1 -> out
 - 2 -> GND
 - 4 -> enable? (connected via 10k to 5 in QP17)
 - 5 -> supply


## QP14

In: 5V
Out: Supply for ultrasonic and lidar motor
Enable: pulldown -> stm32 39 (PE8)

 also required by bumpers

## QP15

In: 5V
Out: Supply for comperators (at least UM5 and UM6)
En: pulldown, STM32: 70 (PA11)

## QP16

In: 3v3
Out: Supplies opamps
En: pulldown, STM32: 65 (PC8)

## QP17

In: 5V
Out: Power to AXP223
En: via QP31 (NPN), which is pulled down (-> internal pullup in SSEHC?), which is connected to STM pin 95 (PB8)

## QP18

pulldown -> ?

## UA8

USB power enable?

Out: connector
In: 5V supply of AXP
En: pulled down, connected into allwinner

# Connectors

QF3:
 Invertiert gate von pkanal fet fuer supply vom sauger
  -> high an base -> sauger an
  -> pin 1 von stm32 -> PE2
JF1:
 - 1: TPF5 -> pin 66 vom stm32 (PC9)
 - 2: TPF4 -> RF4 (10k) -> 
 - 3: GND
 - 4: Supply, switched

 JM1:
  - Hbridge. Supply gleiche wie JF (messpunkt z.b. CPF2) und JL
  - irgendwas  (widerstaende links oben ueber qs4) geht an 3 vom STM

JL1:
 - 1: Motor supply?, 5V (via QP14)
 - 2: GND, switched
 - 3: GND
 - 6: 5V via QP14

 custom-kabel
 ------------

 seriell: schwarz->gnd, weiss -> tx, grau->rx
 swd: schwarz -> clk, weiss->io, grau->Vcc

firmware
=========

 tim5 ccr2 -> vacuum duty cycle

 guesses based on athnd_pMOTORCUROFFq:
  - tim3_ccr4 -> left wheel
   - PE7 relates to this, see sub_800D5F8
  - tim3_ccr3 -> right wheel
   - PE4 relates to this, see sub_800D624
  - tim4_ccr2 -> main brush
  - tim4_ccr3 -> small brush (latter 2 might be swapped)

guessed on handle_irq_tim8_cc2:
 - tim8_ccr2 -> ultrasonic
  - PC6 also seems to be related, see enable_ultrasonic
   - connects directly to pin 3 of ultrasonic module
   - probably enables ultrasonic output

guess based on get_fand_speed_rpm:
 - tim8_ccr4: fan speed input

guesses based on athnd_pLEDe:
 - tim2_ccr2, tim2_ccr3, tim2_ccr4 -> leds

tim4_ccr4 is configured separately in setup_tim4_ch4. also connected to wheels (the corresponding pin is PD15)
 - but this is only used in AT commands?!

get_board_id suggests PC11 and PC10 are static board rev identicators

tim7 is in one pulse mode
 - seems to be used for various one-shot tasks?
  - monitor_ultrasonic enables it and PC6 (sound generation?) in monitor_ultrasonic with ARR=9 and handle_tim7_update disables PC6 iff ARR is 9 and does cliff stuff otherwise (more precisely, if ARR==11)
tim6 overflows with 1 kHz

exti map
--------

1 -> PD1 -> right drop
2 -> PD2 -> left drop (dropping: high)
3 -> PE3 (rising/falling) -> dock
4 -> PB4 -> key home (pressed: low)
7 -> PD7 -> key start (pressed: low)
8 -> PD8 -> bumper right (bumping: high)
9 -> PE9 (rising)
 (the output PE7 relates to this, see handle_exti9)
 left odometry
10 -> PD10 -> bumper left (bumping: high)
11 -> PE11 (rising)
 (related output: PE4, handle_exti11)
 right odometry
12 -> PA12 -> AP running status? (1 -> asleep)
13 -> PC13 -> dustbin (removed: high)
15 -> PA15
 (sends an event to PID, probably INT1 of UC4/BMI160)

setup_one_exti also configures each of those as a floating input

GPIO setup
==========

init_some_gpios
---------------

### GPIOA
 - out push-pull: PA8, PA11
### GPIOB
 - out push-pull: PB8, PB12
### GPIOC
 - in floating: PC10, PC11
 - out push-pull: PC6, PC8, PC12
### GPIOD
 - in floating: PD3
 - out push-pull: PD4, PD9, PD11, PD15
### GPIOE
- in floating: PE13
- out push-pull: PE2, PE4, PE6, PE7, PE8, PE15

usart1_setup
------------

PA9: out push-pull
PA10: in floating

usart2_setup
------------

PD5: AF push-pull
PD6: in floating

setup_spi2
----------

PB13, PB14, PB15: AF push-pull
PB14 is MISO, wtf?

PB12 is CS of BMI160 (from context and looking at the board)

setup_i2c1
----------
PB6, PB7: AF open drain

setup_tim4
-----------

r0 != 1

PD15: out push-pull

r0 == 0

PD15: AF push-pull

PD15 seems to be related to the cliff sensors? (see: cliff_something)
 - empirically tested: high -> cliff sensor leds are on

tim_gpio_init
-------------

via the various timer descriptor structs

in floating:
 - PC7 (tim8 / cc2)
  - connects directly to pin 4 of ultrasonic module
 - PC9 (tim8 / cc4)

af push-pull:
 - PB3 (tim2 / cc2) -> red led
 - PB10 (tim2 / cc3) -> blue led
 - PB11 (tim2 / cc4) -> yellow led
 - PB0 (tim3 / cc3)
 - PB1 (tim3 / cc4)
 - PD13 (tim4 / cc2)
 - PD14 (tim4 / cc3)
 - PA1 (tim5 / cc2)

setup_adc
---------

ADC1 (sample_time == 3 unless noted):
rank offset: 1
 - rank 0: ch 15 (PC5)
 - rank 1: ch 4 (PA4)
 - rank 2: ch 14 (PC4)
 - rank 3: ch 7 (PA7)
 - rank 4: ch 5 (PA5)
 - rank 5: ch 6 (PA6)
 - rank 6: ch 16 (temperature sensor), sample_time: 7

ADC3 (sample_time == 7):
rank offset: 250
 - rank 7: ch 11 (PC1)
 - rank 8: ch 12 (PC2)
 - rank 9: ch 10 (PC0)
 - rank 10: ch 13 (PC13)
 - rank 11: ch 3 (PA3)
 - rank 12: ch 0 (PA0, connected to the output of the UC3 analog mux)
   - control inputs A and B are PD9 and PD11 (in some order)
 - rank 13: ch 2 (PA2)

channel guesses:
 - 2, 5, 6, extmux_F: battery-related, see setup_battery_inner
  - ch2: battery temperature
  - ch5: battery current
   - conversion depends on dock status, see conv_batt_current
  - ch6: battery voltage (conversion factor: 17679/4096)
  - extmux_F: battery identification

 - 4, 7, 14, 15: cliff sensors? (handle_tim7_update)

 - 3, 10, 11, 12, 13: current sensing (athnd_pCUq)
  - 3: sweep
  - 10: fan current (athnd_hFANe)
  - 11: left wheel current
  - 12: right wheel current
  - 13: brush

 - extmux_E: wall sensor? (via athnd_pWSq)
 - extmux_10: adapter current? (see 8016eec)
  - conversion: (raw * 8250) / 4096
 - extmux_11: adapter voltage (see 8017b1c)
  - conversion: (raw * 25740) / 4096

active remappings
-----------------

 - GPIO_Remap_SWJ_JTAGDisable (in setup())
 - GPIO_Remap_USART2 (in usart2_setup())

setup_timers:
 - GPIO_FullRemap_TIM2
 - GPIO_Remap_TIM4

misc guesses
------------

 PE7 and PE4 might be the wheel motor direction inputs? they're used in various places to negate stuff that looks like odometry

 PA8 is the brush direction
 PC12 is the sweep direction
 PB8 is the AP reset pin

 PE6 might be related to charging
  -> setting powers down the platform
     (maybe changes AXP223 power on trigger? suddenly, the allwinner boots on pressing the boot button)

 PD3, PD4 -> sleeping, AP handshaking

Allwinner pinout
================

SWD towards STM32
PB6 -> SWDIO
PB7 -> SWCLK

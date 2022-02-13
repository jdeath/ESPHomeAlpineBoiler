# ESPHomeAlpineBoiler
Documentation for ESPHome Modbus Boiler Reader

This is documenation to add an [Alpine Burnham (US Boiler) Boiler](https://www.usboiler.net/product/alpine-high-efficiency-condensing-gas-boiler.html) to homeassistant. Tested on model number: ALP150BW-4T02 but should work on most similar models.
Putting this togeather leveraged research from many people online, amazon comments, etc:
1. https://github.com/alanmitchell/mini-monitor/blob/master/readers/sage_boiler.py
1. https://github.com/jdleslie/sage2-boiler/blob/master/sage_boiler.py
1. https://www.ccontrols.com/support/dp/Sage2.doc
1. https://mini-monitor-documentation.readthedocs.io/en/latest/hardware.html#parts-cv1-and-j1-burnham-alpine-boiler-interface

Components Needed (Just a few bucks for all of this):
1. TTL to RS485 Module 485 . https://www.amazon.com/gp/product/B07YZTGHGG Could use the more common Max485, but this board is a little easier as does not need flow control.(but may need a resister)
2. Wemos Mini D1. https://www.amazon.com/gp/product/B081PX9YFV . Could use any ESP module you like, but these are cheap
3. Cat 5/6 Ethernet Cable (which you will destroy)
4. 120 Ohm Resitor. (I used a 100 Ohm and still worked) This was the trick, without this I was beating my head against the wall!

Prep the Boiler:

You must set the “Boiler Address” to 1 in order to read data from the boiler. This is done on the “Adjust”, “Sequence Slave”, “Boiler Address” menu item available through the touch screen control on the boiler. The Factory default for this setting is “None”, so it must be changed in order for the monitoring system to work. The default password is 76 , if it asks you for one.

Install ESPHome:

Load ESPHome onto the Wemos. Do this first. Just load a standard firmware, so you can update it later without needing to program it manually. Make sure you also have the ESPHome integration loaded.

Solder the Components:

You need to solder the Wemos to the RS485 Adapter and the RS485 Adapter to the CAT 5/6 Cable and resister. 

First for the Wemos to RS485:
| Wemos | RS485 | Comment |
| - | - | - | 
| 5V | Vcc | Works best with 5 V |
| RX | RX | Yes, RX to RX. Not a mistake. This 485 board is labled this way! |
| TX | TX | Yes, TX to TX. Not a mistake.This 485 board is labled this way! |
| G  | G |

Next for the RS485 to the Cat 5/6 Cable
| RS485 | Cat 5/6 | Comment |
| - | - | - | 
| A+ | Pin 8 | Often this is the brown cable . Will be twisted with brown/white cable |
| B- | Pin 7 | Often this is the brown/white cable . Will be writed with brown cable |
| G Labled with Chinese Character  | Pin 6 | Often this is green . Might be optional, but I did it |

Now the big trick is if your cable is short (mine was) you need a 120 Ohm resitor between A+ and B- wires near the adapter. I soldered it across the two (Brown and Brown/White) CAT5 wires, a few mm before the 485 Module. Without this, you will get reflections and get error messaages in ESPHome and nothing worked. Once I did this, things worked perfectly.

Now connect other side of CAT 5/6 cable to the boiler. The boiler controller has a MODBUS RS485 interface that is accessed through a standard RJ45 jack on the side of the boiler. You want the top jack labeled "Boiler to Boiler".

Now, you can go back into ESPHome and load the following YAML onto the WEMOS. These are the sensor's I am using. If you want more sensors, you can look at the code linked at the top of the page, to add more holding register sensors. I will update this code as I learn more. The temperature sensors will be wrong if the temp goes below OC. I need to figure out how to do the 2s compliment in an if statement in ESPHome. But this should give you the most important sensors. There are two ways to determine the firing rate. I used the method that worked for mine. I also scale the firing rate to Btu/h to match what is displayed on the boiler display panel.
```
esphome:
  name: boiler

esp8266:
  board: d1_mini

uart:
  id: mod_bus
  tx_pin: GPIO1
  rx_pin: GPIO3
  baud_rate: 38400
  

modbus:
  id: modbus1

modbus_controller:
  - id: vlb
    ## the Modbus device addr
    address: 0x1
    modbus_id: modbus1
    setup_priority: -10
    command_throttle: 100ms

sensor:
  - platform: modbus_controller
    modbus_controller_id: vlb
    name: "outlet_temp"
    id: outlet_temp
    register_type: holding
    address: 0x7
    value_type: U_WORD
    accuracy_decimals: 1
    filters:
    - lambda: return x * 0.18 + 32.0;
    unit_of_measurement: "°F"
  - platform: modbus_controller
    modbus_controller_id: vlb
    name: "fan_rate"
    id: fan_rate
    register_type: holding
    address: 0x8
    value_type: U_WORD
    accuracy_decimals: 1
    bitmask: 0x7FFF
    unit_of_measurement: "rpm"
  - platform: modbus_controller
    modbus_controller_id: vlb
    name: "max_rpm"
    id: max_rpm
    register_type: holding
    address: 193
    value_type: U_WORD
    accuracy_decimals: 1
    unit_of_measurement: "rpm"  
  - platform: template
    name: "firing_rate"
    lambda: |-
      return 118000.0 * id(fan_rate).state / id(max_rpm).state; 
    unit_of_measurement: "Btu/h"
  - platform: modbus_controller
    modbus_controller_id: vlb
    name: "outdoor_temp"
    id: outdoor_temp
    register_type: holding
    address: 170
    value_type: U_WORD
    accuracy_decimals: 1
    unit_of_measurement: "°F"
    filters:
    - lambda: |-
        if (x > 32767 ) {
           return (x - 65536.0) * 0.18 + 32.0;
        } else {
           return x * 0.18 + 32.0;
        } 
# Alpine 150 = BTU Heat Rating: 118000 Btu/h
binary_sensor:
  - platform: modbus_controller
    modbus_controller_id: vlb
    id: ch_demand
    name: "Heat Demand"
    register_type: holding
    address: 66
  - platform: modbus_controller
    modbus_controller_id: vlb
    id: water_demand
    name: "Water Demand"
    register_type: holding
    address: 83 

# Enable logging
logger:
  level: VERBOSE
  baud_rate: 0
# Enable Home Assistant API
api:

ota:
  password: "UseAGoodPassword"

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

  # Enable fallback hotspot (captive portal) in case wifi connection fails
  ap:
    ssid: "Boiler Fallback Hotspot"
    password: "UseAGoodPassword"

captive_portal:

web_server:
  port: 80

```
 

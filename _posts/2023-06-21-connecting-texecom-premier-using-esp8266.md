![PXL_20230621_132420915](https://github.com/leocrawford/leocrawford.github.io/assets/915016/1fd32687-e637-4233-84fb-bf53f34fa555)

For some time I have been using a Raspberry Pi Zero W to run `ser2net` relaying the COM port of my Texecom Premier Elite 48 over wifi
so that I can connect wintex remotely. Latterly I have moved to using home assistant to automate my alarm using [this excellent HA add-on](https://hub.docker.com/r/dchesterton/texecom2mqtt). 
I must say this is awesome; now I can alert my phone if the alarm goes off, and auto arm (and disarm) as I enter/leave the house.

However I have been dis-satisfied with the raspberry pi solution for a few reasons:

1. I currently power it with a trailing USB cable which is ugly, and sometimes gets yanked.
2. I'm using the UART pins directly with 5v IO (which they aren't rated for) and whilst it works I feel uncomfortable about this solution.
3. Patching the pi takes work, and sometimes requires changes to be made
4. The pi seems overkill for the job (and I can use it for other things)
5. Ideally I'd be able to connect two COM ports so I can use wintex and HA at the same time

Inspired by Rogan Dawes in various posts including [this]([url](https://community.home-assistant.io/t/integrating-texecom-premier-alarm-panels-via-esphome-using-wintex-protocol/330396)https://community.home-assistant.io/t/integrating-texecom-premier-alarm-panels-via-esphome-using-wintex-protocol/330396)
and [this]([url](https://community.home-assistant.io/t/texecom-alarm-panel/40561/54)https://community.home-assistant.io/t/texecom-alarm-panel/40561/54) I decided to dip my toe into 
embedded devices. I bought a few ESP2866 (because they were cheap) and quickly got started with [esp-link]([url](https://github.com/jeelabs/esp-link)https://github.com/jeelabs/esp-link) which was
really easy and effective. However the little dev boards I had were still powered over USB, weren't specified for 5v UART (though they did seem to work) and 
I still only had one COM port connected.

Via a lot of web searching (and not a little trial and error with aliexpress orders, not all of which I fried) I finally settled on this amazing piece of gear.

![esp12e](https://github.com/leocrawford/leocrawford.github.io/assets/915016/e3941e86-c25b-4178-912a-a555a75da22d)

It only [costs £4.60]([url](https://www.ebay.co.uk/itm/203202954420)https://www.ebay.co.uk/itm/203202954420) but it has an onboard power converter allowing it to be powered over 12V (which the texecom premier provides on the COM port) 
and also has a built in level shifter allowing the UART pins to be 3v or 5v. Now I have a single piece of hardware that can be connected directly into my alarm with no trailing cables, power adapaters or level shifter.

However I was aware that I still only had one COM port connected (and the ESP8266 can only run one UART using hardware) so what to do? I could have looked at 
ESP32s but then my hunt for the perfect piece of gear would start over (and I must have already spent more with aliexpress than just buying the official Wifi adapter from texecom) so I turned to software. I found some bit banging code and was
contemplating merging it into esp-link (which I realised hadn't bene updated for three years) when I came across [esphome]([url](https://esphome.io/)https://esphome.io/) which has 
built in [UART support]([url](https://esphome.io/components/uart.html)https://esphome.io/components/uart.html) and a number of different stream-servers that
relay this serial connection over wifi. I experimented with a few, which either didn't support ESP2866 or didn't seem to permit multiple UART connections - but finally found [this one]([url](https://github.com/2QT-Lexi/esphome-stream-server-v2)).
I used HASS to create a new esphome device, and configured it like this..

````
esphome:
  name: alarm-link
  friendly_name: alarm-link

esp8266:
  board: esp12e

external_components:
  - source: github://2QT-Lexi/esphome-stream-server-v2

uart:
 - id: uart_bus
   tx_pin: GPIO1
   rx_pin: GPIO3
   baud_rate: 19200 
   data_bits: 8
   stop_bits: 2
 - id: uart_bus_2
   tx_pin: GPIO4
   rx_pin: GPIO5
   data_bits: 8
   stop_bits: 2
   baud_rate: 19200 

stream_server:
 - uart_id: uart_bus
   port: 6638
   id: "b1"
 - uart_id: uart_bus_2
   port: 6639
   id: "b2"

binary_sensor:
  - platform: stream_server
    stream_server: "b1"
    name: "serial_server_1"
  - platform: stream_server
    stream_server: "b2"
    name: "serial_server_2"
       
# Enable logging
logger:
  baud_rate: 0


# Enable Home Assistant API
api:
  encryption:
    key: "...."

ota:
  password: "...."

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

  # Enable fallback hotspot (captive portal) in case wifi connection fails
  ap:
    ssid: "Alarm-Link Fallback Hotspot"
    password: "...."

captive_portal:
    
````

It seems that esphome is smart enough to drop back to software (bit banging) for the second UART connection (which I chose to be the one I use infrequently for wintex, not the always connected HASS connection).  Part of my logs now show my device has started up, and using the UART pins I specified has exposed both over two different ports

````
[13:42:41][C][uart.arduino_esp8266:103]:   TX Pin: GPIO1
[13:42:41][C][uart.arduino_esp8266:104]:   RX Pin: GPIO3
[13:42:41][C][uart.arduino_esp8266:106]:   RX Buffer Size: 256
[13:42:41][C][uart.arduino_esp8266:108]:   Baud Rate: 19200 baud
[13:42:41][C][uart.arduino_esp8266:109]:   Data Bits: 8
[13:42:41][C][uart.arduino_esp8266:110]:   Parity: NONE
[13:42:41][C][uart.arduino_esp8266:111]:   Stop bits: 2
[13:42:41][C][uart.arduino_esp8266:113]:   Using hardware serial interface.
[13:42:41][C][uart.arduino_esp8266:102]: UART Bus:
[13:42:41][C][uart.arduino_esp8266:103]:   TX Pin: GPIO4
[13:42:41][C][uart.arduino_esp8266:104]:   RX Pin: GPIO5
[13:42:41][C][uart.arduino_esp8266:106]:   RX Buffer Size: 256
[13:42:41][C][uart.arduino_esp8266:108]:   Baud Rate: 19200 baud
[13:42:41][C][uart.arduino_esp8266:109]:   Data Bits: 8
[13:42:41][C][uart.arduino_esp8266:110]:   Parity: NONE
[13:42:41][C][uart.arduino_esp8266:111]:   Stop bits: 2
[13:42:41][C][uart.arduino_esp8266:115]:   Using software serial
[13:42:41][C][captive_portal:088]: Captive Portal:
[13:42:41][C][mdns:108]: mDNS:
[13:42:41][C][mdns:109]:   Hostname: alarm-link
[13:42:41][C][ota:093]: Over-The-Air Updates:
[13:42:41][C][ota:094]:   Address: alarm-link.local:8266
[13:42:41][C][ota:097]:   Using Password.
[13:42:41][C][api:138]: API Server:
[13:42:41][C][api:139]:   Address: alarm-link.local:6053
[13:42:41][C][api:141]:   Using noise encryption: YES
[13:42:41][C][streamserver:110]: Stream Server:
[13:42:41][C][streamserver:111]:   Address: 192.168.1.115:6638
[13:42:41][C][streamserver:110]: Stream Server:
[13:42:41][C][streamserver:111]:   Address: 192.168.1.115:6639
````

Success. I now have a single cheap device tucked inside the alarm case, with two COM ports wired up, and power drawn from the alarm itself. So far it has been 100% stable and I can manage it remotely (including reflashing) if I need. 

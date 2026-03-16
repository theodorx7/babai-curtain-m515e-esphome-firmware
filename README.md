# ESPHome firmware: Zemismart curtain / blind M515E   babai.curtain.m515e

[English](./README.md) | [Russian](./README_RU.md)


> [!TIP]
> This method of modification and/or firmware configuration can also be used for other Zemismart motor models in the Tuya and Xiaomi ecosystems.

<br/>
The first version of my smart home was built on cloud technologies. However, I quickly realized that a smart home must be based exclusively on local control. Only local devices are capable of providing reliability and comfort. Otherwise, you risk disappointment.

As it turned out, the Zemismart M515E can be integrated into Home Assistant using [hass-xiaomi-miot](https://github.com/al-one/hass-xiaomi-miot) and the official [ha_xiaomi_home](https://github.com/XiaoMi/ha_xiaomi_home) integration. The devices can operate in local mode, but only support one-way control, with no feedback. Unfortunately, all 3 of my devices periodically experienced failures: they stopped responding and accepting commands.

The device is based on the MHCWB4P-B - RTL8720CN, and at the moment there is a method [LibreTiny](https://github.com/libretiny-eu/libretiny/issues/44#issuecomment-2514974466) - [ltchiptool FIX](https://github.com/prokoma/ltchiptool/tree/ambz2-fix-release), which allows custom firmware to be installed on this chip.  

But in order to switch the chip into firmware flashing mode, GPIO0 must be connected to 3.3v. But it turned out that this contact is hard-wired to GND, and if 3.3v is applied to it, a short circuit will occur.  
Out of the 3 devices I had, I managed to remove the solder on only one in such a way that a gap formed between the main board and the chip board, allowing the GPIO0 contact to be disconnected from GND. On the other two boards, the components were too tightly adjacent to each other to completely remove the solder and break the contact. And only one device I was able to switch into firmware flashing mode. However, after connection, the connection would drop almost immediately, as I understood it, due to the chip continuously rebooting. Since on 2 devices I still could not disconnect GPIO0 and GND, I decided to disable the built-in chip and replace it with an ESP.

I already had ESP8266 modules, and they fit inside the communication module (dongle) housing, so I chose them (they can be found for 1 dollar on Aliexpress). You can also use an ESP32-C3 **Super Mini**, and in that case the initial flashing process will be simpler and you will not need a USB-TTL adapter.    



<br/>
<details>
  <summary>FIRMWARE COMPILATION</summary><br/>

  Compile the firmware in [ESPHome Device Builder](https://esphome.io/), using the ready-made [babai.curtain.m515e](https://github.com/theodorx7/babai-curtain-m515e-esphome-firmware/tree/main/config) configuration, created with the individual characteristics of the device in mind. It creates a full-fledged `cover` entity in Home Assistant and extends the capabilities compared to the manufacturer's stock firmware.


### Problems of the original device and implemented fixes

- The device does not provide a full set of states: movement statuses are absent, and the motor transmits only the final position after the command is completed. Also, interaction with the motor goes through an asynchronous chain: HA ⟷ ESP ⟷ MCU ←RF connection→ Motor.  
  **Solution:** a custom intermediate control layer has been implemented, which fully takes over the `cover` logic, state generation, and the state machine at the firmware level.

- If RF commands are sent too quickly, the motor does not have time to process them all.  
  **Solution:** a delayed sending mechanism with an interval between commands has been implemented, in which a new command overrides the previous one, so only the latest relevant command reaches the MCU.

- During movement, the motor does not provide feedback, does not stream the current position, and does not publish `Opening` / `Closing` statuses.   
  **Solution:** `Opening` / `Closing` statuses are formed locally in the firmware based on the sent command and the last known position, while movement completion is determined by the final status from the motor. If the final position status does not arrive, a watchdog moves the `cover` state to `IDLE` after a timeout so that it does not remain stuck in `Opening` / `Closing`.

- After the ESP is power-cycled, ESP boots up and connects to Home Assistant faster than the MCU manages to transmit the actual status. And Home Assistant sets the default status to `Open`. The actual status arrives later, which causes state jitter.  
  **Solution:** position synchronization with the motor is performed first. Only after that are Wi-Fi and the API enabled. The `cover` becomes available in Home Assistant already with the correct state. If no status arrives within 3 minutes after startup, Wi-Fi is enabled, and connection to Home Assistant is established regardless of whether a state is available.

- After commands are sent, the MCU publishes an echo of the previous position instead of the new target one, which causes rollback and state jitter.  
  **Solution:** filtering of early incoming statuses has been implemented so that they are not published as the actual result of the command just sent. When the motor is controlled manually using the buttons on the housing, this filtering does not interfere with status processing, since it is applied only to scenarios initiated from HA.

- The stock polling at the `esphome-miot` component level publishes the status from the MCU cache, which may not match the actual one. For example, if the motor started moving from 0 to 100, then during movement polling returns the original position rather than an intermediate or target one. Only after movement is completed does the motor send the actual position status, which causes state jitter.  
  **Solution:** a custom fork of the `esphome-miot` component is used, in which the stock polling is disabled. Instead, a custom polling script is made, which runs only in the safe `IDLE` state.

- In 2 out of my 3 units, jitter of the final position status within ±1% relative to the actual value is observed.  
  **Solution:** if the final position status differs from the target by no more than `±1%`, it is forcibly brought to the target value in the firmware.

If you look at the logs, the motor does not send regular reports to the MCU. It transmits only the position and only after receiving the `STOP` command or after movement is completed following `open/close/target` commands or after completion of commands initiated by pressing the buttons on the motor housing. If it is necessary to reliably obtain an uncached but actual status of the current position, you can send the `STOP` command. And if the RF signal from the motor reaches the MCU, the state will update to the actual one.  

For Roller Mode (siid2 piid4), the motor does not report state.  

<br/>
</details>



<details>
  <summary>FLASHING ESP8266</summary><br/>
  
  - Activate the chip: connect EN ⟷ VCC

  - Connect the chip to a USB-TTL adapter in firmware flashing mode: 
    - TX → RX TTL
    - RX → TX TTL
    - GND → GND TTL
    - IO0 → GND TTL (enables firmware flashing mode)
    - VCC → 3.3V TTL (**3.3V only**, otherwise the chip will burn out)

       <img src="photo/flashing-1.jpg" height="400">
       <br/>
       <br/>

  - Flash the ESP<br/>
    A convenient option is to use the web uploader [web.esphome.io](https://web.esphome.io/)

  - Switch the chip to normal boot mode: connect IO0 ⟷ 3.3V
      
      <img src="photo/flashing-2.jpg" height="400">
      <br/>
      <br/>

  - Supply power for the first startup and firmware check:
    - GND → GND TTL
    - VCC → 3.3V TTL

  - If the chip boots successfully and is detected in Home Assistant, add the device through the ESPHome integration.  
    IMPORTANT: due to the specifics of the firmware/device, in the absence of communication with the MCU and the motor, the ESP enables Wi-Fi and establishes a connection to HA only 3 minutes after boot.  
    
<br/>
</details>



<details>
  <summary>OPENING THE COMMUNICATION MODULE HOUSING</summary><br/>
  
  Open the housing specifically from the right side relative to the "SET" label and approximately in the middle with a slight downward offset. Any damage from opening will later be hidden by the hole in that spot for routing the antenna outside. 

  <img src="photo/opening-1.png" height="400"> 
  <br/>
  <br/>

  Inside there are two boards connected by a ribbon cable. Pull them out gradually. Slightly pull one board, then the other, and repeat this alternately. Use tweezers, but do not tear off the SMD components with them.

  <img src="photo/opening-2.jpg" height="400">
  <img src="photo/opening-3.jpg" height="400">
  <br/>
  <br/>

<br/>
</details>



<details>
  <summary>DISABLING THE BUILT-IN RTL8720CN CHIP</summary><br/>
  Parallel operation of two chips on the same TX line is impossible. In addition, the built-in chip must not consume power or create interference for communication.
  
  - On the MHCWB4P-B connect CHIP_EN ⟷ GND  

      <img src="photo/deactivate-1.png" height="400">
      <img src="photo/deactivate-2.jpg" height="350">
      <br/>
      <br/>

<br/>
</details>



<details>
  <summary>MANDATORY RF ANTENNA MODIFICATION</summary><br/>

  Before the modification, my dongles could send commands reliably, but receiving feedback from the motor was sometimes unstable.  
  After I built the ESP into the housing, command sending remained stable, but feedback almost did not come through at all!  
  
  <img src="photo/antenna-1.jpg" height="400">
  <br/>
  <br/>
  
  The problem was easily solved by moving the antenna outside. For this it is preferable to use thin copper wire, but I used stranded copper wire in rigid insulation. After that, reception became stable and the range increased.  
  
  <img src="photo/antenna-2.jpg" height="400">
  <img src="photo/antenna-3.jpg" height="300">
  <br/>
  <br/>

<br/>
</details>



<details>
  <summary>INTEGRATING THE ESP INTO THE COMMUNICATION MODULE</summary><br/>
  
  - Power the ESP from the communication module board
    - VCC → 3.3V
    - GND → GND  
    
      <img src="photo/integration-1.png" height="300">
      <br/>
      <br/>

  - Connect the ESP communication lines to the TX (GPIO14) and RX (GPIO13) pins on the MHCWB4P-B  
    **Important: the built-in RTL8720CN chip must ALREADY BE DISABLED**
 
    - If you use ESP8266:  
      - TX → TX (GPIO14)
      - RX → RX (GPIO13)  
    - If you use ESP32-C3 Super Mini:  
      - GPIO21 → TX (GPIO14)
      - GPIO20 → RX (GPIO13)  
      
        <img src="photo/integration-2.png" height="500">
        <br/>
        <br/>

  - **Ensure complete insulation of the ESP board, as it will come into contact with other components inside the housing**
    
  -  Assemble the device and enjoy the result!

<br/>
</details>



<br/>
If this material was useful or interesting to you, please leave a star 🙂

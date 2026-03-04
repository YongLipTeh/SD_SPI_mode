# SD Card reader in SPI Mode
![image9](https://github.com/user-attachments/assets/cd2c8a3c-b922-4bad-918d-a22fe5e7e90a)

One of the applications of SPI protocol is we can use it to read SD cards. SD cards are very sensitive because it requires robust communication before it can hand over the data stored in its sectors. First of all, we connect all the wires to the correct pins.

| SD Card Module | STM32 (F446RE) |
| -------- | ------- |
| SCK | PA5 (SPI1) |
| MISO | PA6 |
| MOSI | PA7 |
| CS | PB10 (Pulled Up) |

We have also utilized on of the pin PC2 as a Logic Analyzer trigger so we can control its capture timing perfectly. We have set PB10 to always pull up so that it returns to its default state, SET, which means the SD card stops listening to the data line.
# Initialization
## Waking up the SD card
Before we start reading, we need to wake the SD card up by sending a pulse to it. First of all, we set CS pin so that it stops listening, this is the only special case where you need to set the CS pin to high so that the CPU can be woken up.
We send 20 pulses of 0xFF (minimum 10, or 74 clock pulses) so that it can wake up properly for reading.
## Configuring Initialization
The SD card defaults at SDIO mode, we need to tell the MCU in the SD card to switch to SPI mode, it can be done by first resetting the CS pin so the data lines (MISO, MOSI) can start working. We send CMD0 (0x40, 0x00, 0x00, 0x00, 0x00, 0x95) through MOSI line, it will expect an R1 response of 0x01 from the slave (SD card). We have created four drivers to automate this

CMD0: 0x40 = 0 (start bit), 1 (transmission bit), 0x00 = 0b0000, 0 to 31 are stuff bits, 0x95 = 0b1001010 (CRC7) and 1 (end bit)

## Drivers
1.	_SD_Init_CMD()_, send CMD bytes and returns status.
2.	_getResponseTerminate()_, get response from slave and terminate the communication. We use transmitReceive, so that the master will send 0xFF and receive an output from the slave in return. _Receive_  will fail because the slave will assume the master wants to restart a session with 0x00 sent to the slave. It contains the terminateSPISession() function as well.
3.	_terminateSPISession()_, transmits an extra 0xFF to end the conversation and pull the pin to high again (not necessary for this example but it works well even if you forgot to pull it high).
4.	_getResponse()_, similar to to getResponseTerminate() but without the terminate function. _getResponse()_ and _getResponseTerminate()_ functions contain a dump function that allows debugging through UART. These functions will compare the the response to the first byte, known as expectedFirst, it is known as R1 responses in the [specsheet](https://www.sdcard.org/downloads/pls/). It will also receive extra bytes if needed.
# High Speed Transmission (SDHC Card)
SDHC card supports high speed data transmission. We should also send CMD8 to initialize and confirm it. The bytes are 0x48, 0x00, 0x00, 0x01, 0xAA, 0x87. 

CMD8: 0x48 = 0 (start bit),1 (transmission bit), 0x08 = 0b001000, 0b0000 (reserved), 0x01 = (0b0001, 3.3V), 0xAA (ping message, it must ping back 0xAA), 0x87 (CRC7 check), 1 (end bit)

CMD8 will respond with 5 bytes (R7), we will need to verify the first response before receiving the others. The expected first response is 0x01, we must keep trying so that our master receive 0x01 as the first input. If this is successful, meaning that the SD card is ready, we can receive the remaining 4 bytes. If we get 0xAA in index 4, it means the slave is pinging 0xAA back to the master (STM32), the SD card is in high speed SDHC mode.
# Powering the SD Card and validation
SDHC requires the correct voltage to function, thus we need to inform the card the host’s operating voltage (3.3V). If the card responds with 0x00, it means that the card is ready (completed its initialization). However, if it responds with 0x01, it is still initializing, so we need to keep trying until it returns 0x00.

Because ACMD41 is an application specific command, it must be followed by CMD55, without termination byte 0xFF in between. Thus, we use getResponse() function instead).
## Cyclic Redundancy Check
CRC is a non-cryptographic hash function that acts as a checksum, like a barcode to ensure the first 40 bits are correctly sent.
The formula for cyclic redundancy check

<img width="490" height="68" alt="image2" src="https://github.com/user-attachments/assets/b25e5a03-b629-4cbc-9afb-ef98b90f8a05" />

Figure 1, formula for Cyclic Redundancy Check, this formula can be found in SD Physical Layer Simplified Datasheet, x = 0x0000000000 for CMD0.

Once CRC[6…0] is calculated, we will shift it 7 bits forward and end it with a final 1, for the end bit. Since M(x) is power dependent, any switching of bits will give a different CRC7 hash (the avalanche effect).

CMD55 = 0x77, 0x00, 0x00, 0x00, 0x00, 0xFF;

CMD55 = 0 (start bit), 1 (transmission bit), (Index 17 in base 2), (0 to 31 stuff bits, 0), 0xFF (CRC7), 1 (end bit)

ACMD41 = 0x69, 0x40, 0xFF, 0x80, 0x00, 0xFF

ACMD41 = 0 (start bit), 1 (end bit), (index 41 in base 2), (0x40FF8000, 0:29 are reserved bits, 30 is HCS bit, 31 is also reserved), 0xFF (CRC7, can be anything), 1 (end bit).

CRC formula is provided in the code for those who needs it, just pass in the array directly, with length = 5 and will output array[5] (CRC with the end bit) for you. _array[5] = SD_CRC(array, 5)_;
# SD Card
We have prepared a simple SD card formatted in FAT32, a text file (named test.txt) with hello world message is placed in the SD card. The file is inspected under a hex editor (HxD).
## HxD
Open the hex editor, click on the open disk icon and select physical disk, SD Card. It will show the hex data, use _Ctrl + F_ to search for the word you are looking for, it should take about 1s. For the filename, we searched _TEST_ and it came up immediately.

<img width="1920" height="1030" alt="image3" src="https://github.com/user-attachments/assets/edea66b2-fa5b-441b-b5d4-00a70b73a68f" />

Figure 2 shows the filename displayed on HxD, please take note that it is located at sector 24576 as well as the 06 on the 26th offset from 54 (T in hex).

<img width="1920" height="1029" alt="image4" src="https://github.com/user-attachments/assets/9ace81b0-b672-4f28-a484-40f047d32435" />

Figure 3 shows the contents of the file test.txt on HxD. It is located in sector 24832.
## FAT32 File Format
If you subtract 24576 from 24832, you will get 256 directly, this is not a coincidence, this is how FAT32 file system work. 
The formula is 

$$\text{Target Sector}=\text{First Data Sector}+\left(\text{Cluster}-2\right)\times\text{Sectors Per Cluster}$$

Cluster number for this file system is 06 (from figure 2 earlier. Sectors per Cluster for FAT32 formatted SD card with 32KB is almost always 64 per cluster, we get (6-2)*64 = 256, which accounts for the offset we see in HxD.

24578 = 0x6000 and 24832 = 0x6100. In base 2, we just need to flip the 8th bit 

$$\text{First Data Sector}_{16}\ =\ \text{Target Sector}_{16}\ \ |\ \ (1\ \ll\ 8)$$

# Jump to Sector
Once we have the correct sector locations (0x6000 and 0x6100), and converted into hex. We are ready to read the SD card.  We need CMD17 to initiate reading of sector, it allows the slave to jump to the given address and prepare for reading the SD card.

CMD17: 0 (start bit), 1 (transmission bit), 0x51 (Index 17), 0 to 31 are address bits, 0xFF (CRC7, can be any hex) and 1 (end bit)

The fillCMD17() function will handle the conversion from decimal system to hexadecimal system. The ready response for CMD17 is 0x01

Just like between CMD55 and ACMD41, we cannot allow the CMD17 to end. In contrast, we need to receive a response from CMD17 called the token, this token is 0xFE; once we receive this signal, then we can initiate the reading part.
# Reading the Sector
The size of a sector is 512KB, so we need to prepare enough room to fit these data. After that we receive 2 bytes from CRC as a checksum for our data. Simply use _HAL_SPI_Receive()_ to receive the data you requested. However, this polling technique is quite taxing on the CPU and requires the CPUs full attention.
# DMA mode
We can reduce the CPU’s workload by delegating the task of moving the SD data to DMA. Since have permissions from all the CMD before, it means the slave is fully ready for data retrieval. We use _HAL_SPI_Receive_DMA()_ to initiate the data transmission. When DMA is done receiving data, we call _HAL_SPI_RxCpltCallback()_ (polling) to receive the CRC bytes.

Once we are done receiving only then can we terminate the SPI session. Otherwise, the data you received would not be what you requested (undefined behavior).

With the DMA handling all the work, we can let the CPU go into sleep mode, and wait for interrupts in the main loop using
_HAL_PWR_EnterSLEEPMode(PWR_MAINREGULATOR_ON, PWR_SLEEPENTRY_WFI)_;

This will significantly reduce the power consumption of the MCU, so it is very suitable for low power or even battery applications.
# Boost Mode
Due to the fact of how sensitive the SD card is initially; we have to run it at a lower clock speed. We have set the SPI baud rate prescaler to _SPI_BAUDRATEPRESCALER_256_. It lowers the 90MHz PCLK2 clock speed to 351.5kHz, which is ideal for polling and waking the MCU in the SD card up. CPOL is set to 0 and CPHA is set to the 1 edge (instead of second) in CubeMX.

We have added the TURN_ON_BOOST preprocessor directive so that we can control when to probe the data by commenting out or undo comment #define TURN_ON_BOOST. The boost sets the baud rate prescaler to 4: _SPI_BAUDRATEPRESCALER_4_. It increases the frequency to 22.5MHz. 
## Trigger Pin
To simplified the task of capturing high frequency signals, we have configured pin PC2 to be the trigger pin for logic analyzer capture, this method of capturing signal is more atomic and precise to capture the signal instead of relying on the CLK signal or reset button. The trigger pins are placed in the ifdef ifndef preprocessor directives.
## PRIMASK
The transmission between CMD17 and receiving data is very sensitive, we do not want to interrupt the CPU and cause any bits to be missed; thus, we need the CPU’s full attention. We can protect this sensitive code by putting ___disable_irq()_ before jumpToSector and ___enable_irq()_ after _HAL_SPI_ReceiveDMA()_. After that, the CPU is free and is able to receive other interrupts.
# Logic Analyzer Analysis
Using a high-speed Logic Analyzer, we can observe the high-speed communication between the master and slave. We will first verify that the initialization part works properly before moving on to the high-speed signals. 

Our logic analyzer has a memory depth limit of 393216 samples, so we have to limit the measurements appropriately, otherwise we wouldn’t be able to observe anything.
## Capturing Everything
When the boost mode is turned off, we could measure all the signals. We measured it at 2MHz so that every signals fit into our window of measurement.

<img width="1430" height="677" alt="image5" src="https://github.com/user-attachments/assets/2ecef5ae-55d6-4279-b644-8f7ff882d7ae" />

Figure 4 shows the full signal of the logic analyzer for the file sector; the rough locations of the commands are tagged in green.

The tags are: Wake, CMD0, CMD8, CMD55, ACMD41, CMD55, ACMD41, CMD55, ACMD41, CMD17, Data and CRC. Wake is the initial pulse command (CS as you can see is high). The CS pin is high when MOSI and MISO are transmitting. 

Noticed that CMD55 and ACMD41 are repeated for three times, we can investigate further by zooming in and see the response.

<img width="1429" height="630" alt="image6" src="https://github.com/user-attachments/assets/23667e65-a82c-4ace-8e84-c31949d2e963" />
<img width="1429" height="621" alt="image7" src="https://github.com/user-attachments/assets/dbe30dae-8c28-4a48-924d-bb7b636ec97f" />

Figure 5 shows ACMD41 signals, the R1 response for the first two ACMD41 are 0x01 (top figure), which means it is still idle, we must continue sending the signal pairs to make sure the SD card is fully powered. Eventually it is powered and ready: R1=0x00 in the bottom figure.

Once it is ready, CMD17 was sent and MISO responded with tokens.

<img width="1431" height="549" alt="image8" src="https://github.com/user-attachments/assets/3c54b06a-3c40-48ef-b4fc-424e8c61d34a" />

Figure 6 shows MOSI keeps pushing 0xFF to the slave, but the slave responds with 0xFF (it is not ready). It only becomes ready on the third signal, the byte 0xFE, that is the token we are looking for, the slave has jumped to the corresponding sector and is ready to read the data.
# Boosted Communication
With boosted communication, we are entering the domain of high-speed communication. At 22.5MHz, any sample rate less than 45MHz is considered too low to be accurate (according to **Nyquist-Shannon Sampling** Theorem). Therefore, we have decided to increase the frequency to 200MHz to capture the signal at high fidelity.
## High Signal Fidelity
Unfortunately at high frequencies, the wires become transmission lines, regular Dupont wires will not suffice, we need to use coaxial cables with shielding to reduce EMI interference and crosstalk. These coaxial cables (RG316) is ideal because they are quite thin and the ground shields shortens the loop and reduces parasitic inductance. A series resistor is added at the Device Under Test (DUT) end. 

![image9](https://github.com/user-attachments/assets/12723cf7-6e85-4df1-a04d-781252e9f56a)

Figure 7 shows a logic analyzer connected through coaxial cable to the SD card module and it is controlled by STM32F446RE, the bottom left is an SD card reader.
## Measurement

![image10](https://github.com/user-attachments/assets/44260571-d2d5-4b3d-b389-ab18f9c31b80)

Figure 8 shows an annotated PulseView signal for the hello world message captured by the logic analyzer. If you look closely, CLK signals high and low are of different in length (High is slightly longer than Low).

The asymmetric high and low length CLK signal is due to the frequency of the signal, it is 22.5MHz, which lies in a very embarrassing spot, it is not perfectly divisible by 5 (1/200MHz = 5ns). The high time is 25ns while the low is 20ns, the high is slightly too long while the low is too short, and it is quantized, so the high will be sampled one extra time. To avoid this effect, we must observe it at higher sample rates to increase its resolution.

<img width="1427" height="702" alt="image11" src="https://github.com/user-attachments/assets/5a1893f1-c6cf-42d4-8a3b-ff7135526cc2" />

Figure 9 shows the two CRC signals received by MISO: 06 and 6D, it acts as a checksum to our 512KB signal.



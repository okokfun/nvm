***请注意，此纳伏特计设计已被新的更优设计所取代，该设计将很快在[this Github repository](https://github.com/jaromir-sukuba/nvm2)此存储库中公开。
 ***


## 这个资料库里有什么？
这里包含了整套的设计文件、制造数据、源文件、测试结果以及其他我在构建纳伏特计的过程中所使用过的资源。这一过程并非偶然，而是对TiN在[他的网站](https://xdevs.com/article/nvm_comp/)上提出的纳伏特计挑战的回应。
![外部视图](/media/IMG_0002_.jpg?raw=true)


## 什么是纳伏特表，它有什么用？
好吧，纳伏是一种灵敏度、噪声和稳定性都能分辨低至纳伏特的电压的电压表。虽然许多万用表确实具有相当敏感的低压范围， 分辨率(最低有效数字)通常在数百纳伏特左右，而最后一个数字的值通常会因仪表的噪声和非线性而降低。因此，对于适当的低信号测量，必须有更高的灵敏度。
测量如此低的电压对许多应用都很有用，例如解决各种差分和桥式电路中的小电压差，以及使用适当的电流源测量非常小的电阻。
商用纳米电压表确实存在，如Keithley Model 2182A或Keysight 34420A，但价格高昂，因此DIY设计肯定有其一席之地。

## XDevs 挑战
尽管我一直在鼓捣DIY的测试和测量工具，但我从未萌生过探索纳伏特世界的冲动。这一契机发生在2021年9月2日，当时xdevs网站上的TiN发起了[纳伏特挑战](https://xdevs.com/article/nvm_comp/)。简而言之，该挑战呼吁设计一款开源纳伏特计，其参数需与商业设计相当，且需在256天内完成。这是一项非比寻常的任务，而对我而言，更是如此，因为在此之前我从未设计过低信号电路，也从未操作过纳伏特计。这确实是一个颇为不利的起始点，但我决定迎难而上，而不是浅尝辄止。为了让自己安心，我等待了几天，观察其他人在[eevblog论坛](https://www.eevblog.com/forum/metrology/nanovolt-design-challenge-build-and-show-your-own-nv-meter-in-256-days/?all)上的反应。我不敢说这完全鼓舞了我。 
另一方面，我花了几天时间在网上查阅资料，研究知名供应商的测试设备服务手册。可惜他们在上个世纪90年代初就不再向我们提供原理图了；但即使是这些材料也非常有助于了解这种仪表的要求以及实现它的方法。

## NVM设计，第1部分--设计要求
有了新的知识，我开始分析挑战要求。

> 应具备机载本地电源调节功能。预计应提供单一的通用直流电源（+9至+24 VDC）或110/220VAC主电源输入接口。

好吧，我想我可以做到这一点。另一方面，任何合理的市电电压表都必须有一个隔离的PSU部分，为输入分频器/放大器和ADC供电，这样测量电位就可以与所有用户和通信IO的数字部分无关--需要接地或接近该电位。
这需要直流电或交流电，但它可以通过让一个PSU在适当的直流电压下工作(12V听起来像是一个很好的起点)并连接一个小型AC/DC SMPS来从市电中供电来结合。

> 提供±100 uV或以下的直流电压测量范围，包括±1V和±10VDC范围。

这种方式看起来很简单，但在相同的插孔上同时输入10V和100uV意味着要么切换输入放大器，要么在信号路径中插入分压器，最好是通过继电器。一些较旧的纳米电压表(Keithley Model 181)有两个不同的输入--一个用于更高的范围，另一个带有低TEMF连接器用于敏感范围。

> 至少有两个用户可访问的输入通道，用于测量信号。

哦，我的天，还有一个继电器。继电器很好，但当继电器从线圈功率中发热时，由于触点上产生的热电压(TEMF，热电动势的缩写)，可能会给信号带来偏移误差。这些电压可能在几微伏特到纳伏左右，这在设计中绝对需要考虑。
TEMF是纳伏特计的主要敌人，这个首字母缩略词将在后面的文本中使用几次。

> 具有低热连接接口，以最大限度地减少热电动势寄生误差。

再说一遍，我告诉过你。
  
> 为每个读数提供至少5½位的分辨率。

听起来很合理。我做过一些大规模的ADC项目，比如[电压表](https://www.eevblog.com/forum/metrology/diy-6-5-digit-voltmeter/)和[伏特欧姆表](https://www.eevblog.com/forum/metrology/diy-6-digit-handheld-volohmmeter/msg2978912/#msg2978912)，所以也许我可以回收一些东西，也可以带来一些新的东西。我认为商业sigma-delta A-D转换器也可以完成这项工作，但DIY ADC肯定需要更多。

> 能够在至少0.1-10 Hz的带宽下，以至少10 nV的分辨率和优于30 nV的噪声将输入直流信号数字化。

更多的数字要求是，10Hz带宽下的30nVP-P并不是一个非常宽松的要求--像Keithley 181型这样的老式仪表无法满足这一要求，目前的型号一般。

> Have autozero functionality to correct for static offsets.

That is a logical requirement. I will put in some more effort by introducing autocalibration.

> Have galvanic isolated analog front end, with isolation resistance to earth/chassis better than 10 GΩ.

That is what I proposed earlier, too. 

> Device should have ADC (any type) integrated.

This is likely to frame the project into standalone modus operandi.

> Have good long-term stability and use ovenized DC voltage reference (LM399, LTZ1000 or LTFLU with oven).

I wouldn't think of using anything worse than an LM399 (or ADR1399) anyway.

> Provide RJ45 Ethernet and/or IEEE-488 GPIB interface for communications with external world / external equipment.

Fair point. USB is simpler, though.

> 40W total input power budget (friendly to battery operation for sensitive experiments)

You wouldn't want your nanovoltmeter to consume more than 40W anyway. 40W is quite a bit of heat and it's not easy to dissipate it without introducing more TEMF errors.

> Device should be fully operational as standalone device (e.g. no debuggers or external equipment attached to make it work).

OK, fair.

After a bit of optimization, I came up with this block diagram.
![Block diagram](/media/block.png?raw=true)
It's basically done to fit the requirements above, with a few spicy parts I added for extra fun.
Input from either input connection is selected via an input multiplexor. There is also a third input, to be discussed later. This MUX passes the selected channel to a low noise amplifier (LNA) to be amplified and then routed via another MUX into the main amplifier. whose output is brought to the ADC. This ADC has a nominal 10V input range, so that the basic range with LNA is 10mV (10mV x 1000 = 10V). In order to have even more sensitive ranges, the main amplifier has option to switch to x1 (pass-through), x10 and x100 gain, combined into additional 1mV and 100uV ranges. When the LNA is bypassed, the input signal is brought directly to the main amplifier, allowing for 10V, 1V and 100mV ranges, seamlessly covering 100uV to 10V input range. That would make a nanovoltmeter already, but I added two more blocks - first is easy, a low-pass filter (also known as LPF) to cut down noise, especially on sensitive ranges. Another one is an auto-calibration block, which serves to decide the gain of both amplifiers. Ideally, the meter needs only to know its onboard 7V reference voltage and can derive lower ranges automatically without calibration/adjustment and keep it precise after resistors in dividers and amplifier drift due to age or temperature.
The entire upper portion of the circuit is held behind an isolation barrier in order to keep the input terminals separated from the power supply potential - either external DC or mains voltage. The bottom section consists mainly of the PSU and digital circuitry, including user IO and communication channels, like Ethernet and USB. For the added fun factor I also included GPIB as a low priority subproject.

## NVM design, part 2 - divide and conquer

Designing such an instrument, especially on a DIY base, is a lot of pingponging between mechanical and eletrical design. I could imagine how large the thing would be (say, an A4 page size as a footprint) and I knew I had to cram everything inside. Choosing a larger enclosure would give me much more freedom and other benefits (like easier dissipation of device heat, helping fight TEMF), but would be impractically large. Finding an appropriate enclosure wasn't easy - at first I thought of using G756 I used before in my [DIY SMU](https://github.com/jaromir-sukuba/J-SMU), but I preferred metal enclosures here - because of shielding from electrical interference, and with a steel enclosure even a bit of shielding from magnetic interference. Metal also helps with heat dissipation. Finally I settled on the 1EP802825 from Modushop.
That was the right time to take a look at the electrical domain. The Earthy and floating parts (separated by an isolation barrier) have to be physically separated in sub-enclosures (that calls for two PCBs minimum) and in order to have easier debugging and modification of the circuit, I decided to separate the floating part into two PCBs. One would hold FPGA control, the reference and ADC, while another one would consist of both amplifiers and ACAL circuits. Since the enclosure is 80mm tall, there is no problem stacking at least two PCBs on top of each other, via pin headers and metric spacers.
So, two PCBs for the floating part, one for the earthy part. The backside connectors have to be mounted somehow - here I added another PCB. The front panel pushbuttons also needed a PCB for mechanical reasons, that resulted in a total count of 5 PCBs. I named the PCBs with human names and gave them functionality:

**Bart** - I felt like this one will be tricky and there will be problems. Contains LNA, main amplifier, ACAL dividers and MUXes.  
**Homer** - largest PCB, contains FPGA, ADC, references. Closely related to Bart.  
**Lisa** - communication board, back panel PCB  
**Meggie** - simplest PCB, just to hold buttons  
**Marge** - contains PSU to feed all other PCBs, plus digital circuits.  

With the electronics roughly separated into basic blocks, I returned to mechanical design. By the time I finished first sketches of the schematics files, I received the enclosure, so I could start with more practical details. It's much easier to visualize potential problems having the real enclosure in hand compared to studying 3D files (if any, right). For circuit separation I opted for two sheet metal sub-enclosures.
![Internal metal parts](/media/mparts.png?raw=true). The smaller portion on right holding analog circuitry, the left portion holding earthy circuits - it's somehow shorter, to make room for a backpanel PCB.
I designed a bunch of holes into the enclosures to allow mounting PCBs via metric spacers, and now having physical constraints, I jumped back to PCB design.


## NVM design, part 3 - selected details of circuit operation
In this section I'll discuss a few selected design choices of this instrument.
#### LNA (Bart board)
I went through a few design iterations of the LNA and it took me roughly half of the development time. I tried DC-coupled JFET amplifiers, AC coupled JFET amplifiers in chopper configuration for DC operation as well as paralleling opamps. The parallel configuration brought me the least amount of headache and despite the insane amount of components used, it is not that expensive after all. As best in class, IF3602 JFETs are both expensive and suffer by huge parameter spread, making them quite an expensive solution. There are cheaper JFETs around, but their noise parameters mean more paralleled pieces - with problems they bring, for example binning and matching. More transistors or fewer low noise types also bring more gate capacitance, requiring more circuit support to diminish its influence. Enclosing the amplifier into chopper configuration brings more issues to solve - for example I had to deal with input current and temperature gradients increasing voltage offset. It's very likely that all those problems are solvable with more time and effort and would make an LNA with lower noise and drift than what I have in the current NVM, but at some point I decided to go a different way and built the LNA out of massively parallel off-the-shelf opamp amplifier. Integrated autozero/chopper opamps do have much better temperature stability and input current, so after a bit of searching and comparing I opted for MCP6V51 from Microchip, as the one with very good price/performance ratio. Typical input noise voltage is specified at 210nV in 10Hz bandwidth and I confirmed this experimentally by building a 35 piece opamp board first. 
![Amp35](/media/amp_t1.jpg?raw=true)
The noise parameters were in good agreement with the datasheet values, that gave me confidence to go this way. As calculated, I need at least 50 MCP6V51s to get under 30nV p-p criteria, so I opted for 100 pieces, in 10x10 matrix, as per the schematic below
![LNA diagram](/media/amp.PNG?raw=true)
Each of the first stage gain blocks (with 10 amplifiers in parallel noninverting configuration) is brought to a second stage summing amplifier, and all 10 of the second stage amplifiers are summed into a third, and final, amplifier, with an overall gain of 1000.
The last stage summing amplifier is also used to inject a voltage to null the offset voltage of LNA. The Bart board has a DAC and its amplifiers (U119 and U120) for this reason. The DAC's second channel is designed to compensate for input current - by injecting a current of opposite sign into the LNA input. I measured the uncompensated current to be around 2nA and in my current hardware the injection resistor isn't fitted on the board. By using this feature, I expect a current well below 100pA.
The LNA injects ground current into the ground net of the circuit, and the current is dependent on input voltage (and therefore output voltage, too). Ground current may cause unwanted voltage shifts between ground potentials of the circuit, especially at high gains. To counteract this ground current, amplifier with U121 is set up to source current of opposite polarity into the ground, into areas where the LNA resides.
The Bart board is 4-layer, with internal layers being ground nets, for both electrical and thermal reasons.

#### ADC (Homer board)
The ADC is of the integrating, charge balancing type with residual integrator voltage reading. SN74LV4053 is used for integrator current steering, the integrator is of the classic two opamp composite type. The FPGA does multiple jobs here, apart from orchestrating actions around the ADC, it also provides an interface between isolated serial link and multiple digital outputs used to switch relays, multiplexer and analog filter and does startup timing for smooth PSU startup.
I measured INL of ADC against a Solartron 7081 as reference. A 0-10V voltage sweep was provided by a DIY precision voltage source (LTZ1000A reference, AD5791B DAC). Measured INL graph:
![ADC INL graph](/media/lina.png?raw=true)
The INL of this magnitude may not be obviously needed for this application, but I wanted to employ ACAL functionality, where voltage transfer between ranges expects good INL of the ADC, so I was after good linearity and spent some time honing it.

#### PSU (Marge board)
My first idea for the PSU was to use two back-to-back 50Hz transformers. One would provide a few volts (providing first isolation barrier and ground reference) and the second would step it up to the level appropriate for +-16V and 5V DC outputs. Later I expended this idea by using an audio frequency amplifier circuit to feed the second transformer and omitting the first transformer. Using an AF amplifier (fed by an appropriate oscillator) to drive the transformer has a few advantages - the whole circuit could be powered from a single DC voltage and the oscillation frequency can be adjusted to find a good compromise between leakage and efficiency. Using classic linear amplifiers would bring way too much thermal dissipation into the enclosure, so I opted for the cheap and plentiful TPS3116 D-class amplifier. The oscillator signal - either from a local single opamp multivibrator or amplified signal provided by the MCU - is shaped by a series of lowpass filters, so that the amplifier is fed by a low harmonic content signal. Transformer secondary is brought to a fairly standard set of regulators - LM317/337 and 7805.
I measured ground leakage from the isolated PSU section and its efficiency as function of driving frequency.
![PSU graph](/media/psu-freq.PNG?raw=true)
As per expectations, a lower frequency also brings lower leakage current, but going too low causes efficiency to plummet at some point. I opted for 48Hz, where I measured leakage well below 200nA p-p. At this level, leakage measurement is very sensitive to nearby electric fields and conducted interference. After enclosing the test setup in a shielded box, the measured leakage fell to 40nA p-p, indicating the previous measurement was too pessimistic. I believe even the 40nA figure is pessimistic still and influenced by the shielding setup and the oscilloscope I used. For proper test results I'd need a larger shielded cage and a battery powered oscilloscope, but I didn't want to go that far.
The PSU design brought interesting detail to the game. In case of overload, TPS3116 will turn off the load drive. That sounds innocent, but may be problemtic in this case - because after powerup the reference is cold, so the heater takes a significant amount of current during powerup. To add insult to injury, all decoupling/bulk capacitors are discharged, even more increasing the powerup current spike. The spike causes TPS3116 to shut down for a moment, then it tries to restart the load, ending up in the same large spike, repeating again and again.
To mitigate this issue, two fixes were made:
 - PSU driving starts from higher frequency, slowly (over course of a few seconds) it descends to the final 48Hz value. Voltage on the isolated secondary ramps up much slower, allowing circuits to power up smoothly.  
 - Voltage reference heating current (large current spike when left untreated) and LNA power supply voltage is held off until 10 seconds after FPGA power gets stable, at which point heater current is enbled, and LNA power turns on two seconds later.

## Mechanical parts
While the enclosure is centered around the Modushop enclosure, a few more mechanical parts were needed.

#### Reference cover
This two-part 3D printed component prevents airflows around ADR1399 reference, decreasing its noise somehow.
![Ref cover](/media/refcover.jpg?raw=true)
![Ref cover](/media/refcover2.jpg?raw=true)

#### Lisa board holder
This 3D printed component keeps Lisa board in its place on back panel.
![Back holder](/media/backpcb.jpg?raw=true)
![Back panel](/media/backpanel.jpg?raw=true)

#### LNA cover
This 3D printed component provides insulation of LNA board area from airflows.
![LNA cover](/media/lnacover.jpg?raw=true)
![LNA cover 2](/media/lnacover2.jpg?raw=true)
The amount of influence on the readings was suprising to me. Here are two graphs of the shorted input measurement, one with cover, one without, the RMS noise voltage is nearly twofold for naked LNA.
![LNA cover graph](/media/lnacover_graph.PNG?raw=true)

#### Side PCB
The original modushop enclosure fitted the outer enclosure panel by screwing to side metal sheets by self tapping screws. I didn't like this, so I drilled the original holes for self-tappers into 3,5mm diameter and mounted dedicated mechanical PCB with nothing but holes and copper plane to solder M3 nuts. This PCB keeps the internal chassis and outer shields in single piece. I think this part would be somehow better made out of >3mm thick aluminium of the same size, with drilled and tapped holes, but PCB seems to work fine for now.
![Side PCB](/media/bok2.jpg?raw=true)

#### Push button rod
This is three piece 3D printed component to extend main switch shaft to front panel.  
TODO pictures.

#### Front panel construction
Display cover is epoxied into the front panel cutout, display is bolted to Meggie board and this board is bolted the to font panel.
TODO picture of perspex cutout
![Enclosure with metal parts](/media/frontpcb.jpg?raw=true)

#### Internal construction details
![Enclosure with metal parts](/media/enc_1.jpg?raw=true)
Complete internal construction without top cover
![Covers](/media/covers.jpg?raw=true)
Dump of construction photographs is in imgur gallery [here](https://imgur.com/a/fnK1Evj)

## NVM usage
#### Local interface
![Front panel](/media/fpanel.png?raw=true)
13 keys are provided for instrument operation, plus 2-line by 20 characters vacuum fluorescent display.
After startup the meter enters default (measurement) mode, with settings read from EEPROM. Typical display layout:
![Display example](/media/disp.png?raw=true)
The first line displays measured voltage and range in obvious way.  
Second line in this example indicates following states:  
CH1 - selected channel 1. Could be either 1 or 2.  
Z0 - zero function (see below) is not active on this range. Could be either 0 or 1.  
10NPLC - ADC integration time in powerline cycle multiplies. Could be either 1, 2, 5, 10 or 20.  
ETH1 - Ethernet is connected and IP address provided via DHCP. Could be either 0 or 1.  

Buttons functions:  
**INPUT** - by clicking this button, the instrument toggles between Input 1 and Input 2 on the 4-pin LEMO connector.  
**ZERO** - this function enables zeroing measured value (by subtracting constant voltage offset measured at the instant of enabling the function). Each range has its offset value and is independent of other ranges. Zeroing for given range is disabled by second button press.  
**FILT** - pressing this button displays analog filter, digital filter and NPLC settings. Each hit of the button moves focus to next setting (value is changed by pressing UP/DOWN controls), third hit returns to default measurement display.  
**ACAL** - pressing this button starts ACAL procedure and saves the calibration constants into EEPROM memory.  
**TRIG** - not used in this firmware revision.  
**STORE** - not used in this firmware revision.  
**RECALL** - not used in this firmware revision.  
**MENU** - enables menu. Moving across menu items is done by pressing UP and DOWN keys. Enter enables editing the item value - editing is performed using UP and DOWN keys. Escaping from the menu is possible via ESC button, saving and escaping via MENU button.  
**AUTO** - not used in this firmware revision.  
**ENTER** - apart from function in menu function, pressing ENTER in default mode displays IP address of the instrument (if attached to ethernet). Pressing ENTER again returns back to default state.  
**UP/DOWN** - in menu or filter editing it serves function described above, in default mode it selects higher or lower measurement range.  

#### Remote interface
Instrument could be controlled remotely via ethernet interface, by issuing SCPI-compatible commands.  
\*IDN? - returns instrument identification  
\*RST - resets instrument  
\*CLS - clear status  
SYSTem:ERRor[:NEXT] - lists system error  
SYSTem:ERRor:COUNt? - return amount of system errors  
SYSTem:VERSion? - returns instrument version string  
MEASure:VOLTage? - returns value of measured voltage  
CALibration:VREF - single float parameter, sets value of 7V reference voltage (the only input for ACAL procedure)  
CALibration:VREF? - returns value of 7V reference voltage  
CALibration:UNLock - single integer parameter, locks (0) or unlocks (1) access to calibration EEPROM  
CALibration:SAVe - if access to calibration EEPROM is unlocked, this command saves actual calibration into EEPROM  
CALibration:ACAL - starts ACAL procedure  
SENSe:CHANnel - single integer parameter, sets channel 1 or 2  
SENSe:CHANnel? - returns actual channel number  
SENSe:RANGe - single integer parameter, sets range 1 (100uV) to 6 (10V)  
SENSe:RANGe? - returns actual range number  
SENSe:DFILter - single integer parameter, sets digital filter to 1, 2, 4, 8, 16, 32 or 64 samples  
SENSe:DFILter? - returns digital filter setting  
SENSe:AFILter - single integer parameter, sets analog filter off (0) or on (1)  
SENSe:AFILter? - returns analog filter setting  
SENSe:DIGits - single integer parameter, sets nubmer of displayed digits - 3 to 6  
SENSe:DIGits? - returns nubmer of displayed digits  
SENSe:NPLC - single integer parameter, sets ADC integration period in NPLC cycles to 1NPLC (0), or 2NPLC (1), or 5NPLC (2), or 10NPLC (3), or 20NPLC (4)  
SENSe:NPLC? - returns ADC integration period  
SENSe:AZero - single integer parameter, sets ADC autozero off (0) or on (1)  
SENSe:AZero? - return ADC autozero setting  
SYSTem:BEEPer - single integer parameter, sets beeper off (0) or on (1)  
SYSTem:BEEPer? - return beeper state  

NVM is fully calibrated by ACAL feature. For this it needs to have its reference voltage setup in calibration memory before running ACAL function. The value is accessible only through remote interface and its CALibration:VREF command. If the reference value in calibration memory is outside 6,95-7,3V range, default value 7,05V is assumed. At the moment, individual range calibration isn't supported, but probably will be in future firmware revisions.

## NVM project goals
In this section I'll discuss the performance of this instrument and project goals. 
#### Noise
The main criteria of this project was to achieve low noise at the lowest voltage range. In order to assess this, I logged 20 seconds of shorted input measurements at 100uV range, with no autozero, 2NPLC (integration time 40ms) and 10Hz analog filter on.
![Noise graph](/media/noise.png?raw=true)
The noise is below 25nV p-p.
#### Stability
I recorded roughly 9 hours worth of data with shorted input, while logging ambient temperature too. Meter set to 10NPLC, autozero on, analog filter on. I separated the data into 64 seconds long chunks, averaged it and used as data points. I tried to control the room temperature to obtain some variation.
![Longer noise graph](/media/lnoise.png?raw=true)
There is no clear temperature dependency. I read the data as sensitivity to temperature changes is perhaps more prominent than the tempco of the instrument.
#### Linearity
I measured the same INL of the NVM against Keithley 2010 as reference meter.
![NVM INL graph](/media/linn.png?raw=true)
I believe it lies within acceptable margin for 6,5 digit meter.
#### Bandwidth and normal mode rejection
I measured ADC transfer function for sine input with 1Vp-p amplitude, stepped from 1Hz to 80Hz. Meter was set to 2NPLC, no autozero, analog filter on, 10V range.
![Frequency response graph](/media/freq.png?raw=true)
Resulting bandwidth for -3dB is between 9 and 10Hz, normal mode rejection for 25, 50 and 75Hz is around -90dB
#### Stepping the input with various voltages from Keithley 260 voltage source
Meter set to 20NPLC, autozero on, analog filter on, digital filter=2, 100uV range. Stepping in 10uV steps. The source has undergone massive repair and is out of calibration - but served well as test source to assess linearity and stability of NVM.
![10uV steps graph](/media/steps10uv.PNG?raw=true)
And similarly for 1uV steps. Note that 10uV step is missing, due to bad contact or resistor on Keithley 260 source.
![10uV steps graph](/media/steps1uv.PNG?raw=true)
#### Response to 10uV input step with various digital filter settings
Meter set to 20NPLC, autozero on, analog filter on, 100uV range. K260 induces 10uV step on input terminals, response time with various digital filter settings is observed.
![10uV steps graph](/media/dfilter.PNG?raw=true)
Response is very typical for boxcar average used as digital filter here.

#### Contest goals accomplishment
Now returning back to the contest call:

> Have local onboard power regulation. Single common DC (+9 to +24 VDC) or 110/220VAC mains input jack is expected.

Yes, it works on both DC 12V and 230VAC sources.

> Provide DC Voltage measurement ranges ±100 µV or below and include ±1V and ±10VDC range.

Yes, input ranges are bipolar 100uV, 1mV, 10mV, 100mV, 1V and 10V.

> Have at least two user-accessible input channels for signal to be measured.
> Have low-thermal connection interface to minimize thermal EMF parasitic errors.

Yes, two inputs with reasonably low TEMF LEMO connector.

> Provide at least 5½-digit resolution for each reading.

6,5digit provided.

> Ability to digitize input DC signal with resolution at least 10 nV and noise better than 30 nV peak to peak over at least 0.1-10 Hz bandwidth.

On lowest range and 6,5 digit readout the resolution is 100pV and noise in 10Hz bandwidth is ~20-25nV p-p.

> Have autozero functionality to correct for static offsets.

Yes, autozero and autocalibration provided.

> Have galvanic isolated analog front end, with isolation resistance to earth/chassis better than 10 GΩ.

Analog portion is isolated. Common mode current is apparently below 100nA p-p.

> Device should have ADC (any type) integrated.

I used my own integrating multislope ADC design.

> Have good long-term stability and use ovenized DC voltage reference (LM399, LTZ1000 or LTFLU with oven).

I used ADR1399. Can't assess long-term stability by now.

> Provide RJ45 Ethernet and/or IEEE-488 GPIB interface for communications with external world / external equipment.

RJ45 Ethernet provided, hardware ready for GPIB, too. Apart from that, USB interface is available on board.

> 40W total input power budget (friendly to battery operation for sensitive experiments)

Current draw is 12W.

> Device should be fully operational as standalone device (e.g. no debuggers or external equipment attached to make it work).

No external equipment needed for the meter to operate.

#### Cost and component availability/repairability
Total cost of the instrument components is around 400EUR. All parts I used are off-the-shelf components or are easy to order, with the exception of the power transformer. This is DIY-ed, but I believe many custom transformer manufacturers are able to deliver such a transformer.
Apart from that, I tried to use components with most generic footprints - like SOT23 or SOIC8 for opamps, SOT23 for transistors, most resistors do have a universal footprint able to accommodate MiniMELF (which I used because I wanted to), 1206 or 0805 resistor sizes, used a common display interface and so on. This is to help with alternative component sourcing, a hot topic in 2022, when the component crisis is influencing both professional as well as hobby electronics development and production. Despite my efforts, it's likely that some components in the BOM lists will be unavailable with usual vendors and it's hard to make future-proof designs in this respect.

## Future plans
There are a few features the meter hardware is supposedly able to do, but are not implemented at the moment and I consider this a TODO list for future work on this project.
- implement main frequency synchronous measurements. Zero-cross detector is already fitted on Marge board and ADC supports triggering by external signal, so it should be matter of firmware support.
- implement input current compensation. All parts except a large value resistor are already on the board, so after fitting it this feature should be again a matter of firmware.
- polish front panel user interface
- polish the SCPI commands implementation, add more commands and full manual calibration features.
- do more elaborate triggering, along with recording data into memory, to utilize all front panel buttons. 
- do more exploration and testing of the ACAL feature. As it is, works fine, but I think it could do with better repeatability.
- find lower power opamps types - this could decrease current consumption, which is always welcome in test gear.
- provide support for GPIB. Not that I need GPIB that much, but I think exploration of GPIB could be interesting project.


## Files contained in this repository
- **firmware**
	- **FPGA** - FPGA design entry files, Lattice Diamond project
	- **STM32** - STM32 on Marge board firmware, part of a workspace of STM32CubeIDE
- **hardware**
	- **PCB-design_files** - design files, one directory per board, each board is single Kicad v6 project
	- **PCB-production_files** - gerber and BOM data, one directory per board
	- **Schematics_pdf** - easy to read schematics in PDF format, one directory per board, plus top level interconnection diagram
- **measurement_data** - source data of selected measurements
- **media** - pictures and photos for this readme file.
- **mechanical**
	- **3DP_back_panel_pcb_holder** - component to keep Lisa board on back panel. Designed as single part, needs to be printed twice, one copy mirrored. Freecad design file and STL
	- **3DP_LNA_cover** - cover to keep LNA out of air turbulences. Designed as single part, needs to be printed twice, one copy mirrored. Freecad design file and STL
	- **3DP_pushbutton_assembly** - transfers front panel button to mains switch actuator. Three parts, each is needed once, contains freecad design file and STL files.
	- **3DP_reference_cover** - keeps voltage reference out of air turbulences. Two parts, each is needed once, contains freecad design file and STL files.
	- **acryllic_display_cover** - frame of display cover to be cut from gray acryllic.
	- **metal_parts_internal** - design files and manufacturing files for internal metal parts of the enclosure. 5 parts: four sheet metal (1mm thickness) components, one 2mm aluminium flat part. Each one is needed once.
	- **PCB_back_panel** - back panel, designed as PCB in Kicad. I let them to manufacture it out of FR4, but alumium boards are option, too.
	- **PCB_front_panel** - front panel, designed as PCB in Kicad. I let them to manufacture it out of FR4, but alumium boards are option, too.
	- **PCB_side_panel** - side panel, designed as PCB in Kicad. FR4 material works, aluminium would probably too, if reflowed. Two pieces needed for enclosure.
	
## Resume
This project was much fun and exploration, with the side effect of gaining a nanovoltmeter. I tried to make it as good as possible within a hobby budget, at the expense of not very cheap nor easy circuit. That gave me the idea of cherrypicking the good parts of this instrument, relaxing the design criteria somehow and making a new project with a significantly shorter and cheaper BOM. But that is the subject of a different project.

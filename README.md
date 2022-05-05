# APRS 434
Welcome to the home of **APRS&nbsp;434**,
the 434&nbsp;MHz LoRa APRS amateur radio project that **extends range by saving bytes.**

Unlike some other ham radio LoRa APRS projects,
this project aims at **deploying LoRa the way it was intended;**
namely by being frugal about the number of bytes put on air.
Doing so, reaps a number of benefits:

- less airtime,
- increased battery life,
- higher chances of good packet reception,
- hence, increased range,
- lower probability of packet collisions,
- therefore, more channel capacity.

In dense urban environments and/or on flat terrain, LoRa works best when the data payload is kept to a strict minimum.
This can be achieved taking full advantage of all 256 characters available for transmission with LoRa.
The APRS frame compression protocols presented below aim precisely at doing that;
for LoRa, or any other data link with an extended character set.

ESP32 [**tracker and i‑gate firmware**](#esp32-firmware-downloads) adhering to these compression protocols is provided as well.


## Index
- [An Open Standard for LoRa APRS Frame Compression](#an-open-standard-for-lora-aprs-frame-compression)
- [Measurable Benefits](#measurable-benefits)
    + [Reduced Packet Error Rate](#reduced-packet-error-rate)
    + [Airtime Reductions](#airtime-reductions)
- [LoRa Link Parameters](#lora-link-parameters)
    + [Considerations for Switching to SF11](#considerations-for-switching-to-sf11)
    + [LoRa ICs and Modules](#lora-ics-and-modules)
- [Callsign, SSID, Path and Data Type Compression](#callsign-ssid-path-and-data-type-compression)
    + [Encoding CCCC](#encoding-cccc)
    + [Decoding CCCC](#decoding-cccc)
    + [Encoding D](#encoding-d)
    + [Decoding D](#decoding-d)
    + [Codec Algorithms for CCCCD](#codec-algorithms-for-ccccd)
    + [SSID Recommendations](#ssid-recommendations)
    + [Data Type Codes](#data-type-codes)
- [Digipeating on LoRa Channels](#digipeating-on-lora-channels)
    + [Path Codes](#path-codes)
- [Compressed Geolocation Frames](#compressed-geolocation-frames)
- [Compressed Weather Report Frames](#compressed-weather-report-frames)
- [Compressed Text](#compressed-text)
    + [Encoding tttt](#encoding-tttt)
    + [Decoding tttt](#decoding-tttt)
    + [Codec Algorithms for tttt](#codec-algorithms-for-tttt)
- [Comments](#comments)
- [Compressed Status Report Frames](#compressed-status-report-frames)
- [Compressed Item Report Frames](#compressed-item-report-frames)
- [Compressed Addressed Message Frames](#compressed-addressed-message-frames)
    + [Encoding and Decoding EEEE](#encoding-and-decoding-eeee)
    + [Encoding F](#encoding-f)
    + [Decoding F](#decoding-f)
    + [Codec Algorithms for EEEEF](#codec-algorithms-for-eeeef)
- [Reducing Power Consumption](#reducing-power-consumption)
- [Recommended Hardware](#recommended-hardware)
    + [Tracker Hardware](#tracker-hardware)
    + [I-Gate Hardware](#i-gate-hardware)
- [ESP32 Firmware Downloads](#esp32-firmware-downloads)
    + [Tracker Firmware](#tracker-firmware)
    + [I-Gate Firmware](#i-gate-firmware)
- [Development Road Map](#development-road-map)
    + [Data Link Layer](#data-link-layer)
    + [Tracker Hardware](#tracker-hardware-1)
    + [Messaging](#messaging)
    + [WiFi Geolocation](#wifi-geolocation)
- [News, Social & Co-Development](#news-social--co-development)
- [Acknowledgements](#acknowledgements)


## An Open Standard for LoRa APRS Frame Compression
As a physical layer, LoRa permits sending any of the [256 characters](https://en.wikipedia.org/wiki/Extended_ASCII) from `\00` to `\ff`. This is double the amount of the [7‑bit, 128 ASCII character set](https://en.wikipedia.org/wiki/ASCII#Character_set). Hence, there are ample opportunities for compressing [AX.25](https://en.wikipedia.org/wiki/AX.25) ([packet radio](https://en.wikipedia.org/wiki/Packet_radio)) unnumbered information (UI) frames at the [data link layer](https://en.wikipedia.org/wiki/Data_link_layer), namely:

|[AX.25](https://en.wikipedia.org/wiki/AX.25) UI frame&nbsp;field|compression opportunities with&nbsp;LoRa|
|:-:|:-:|
|_Flag_|**not required**; explicit header provided by LoRa|
|_Destination Address_|**not required**; software version provided by the i‑gate|
|_Source Address_|any 6 out of **37** characters: 26 capital letters + 10 digits + space|
|_SSID_|1 out of [**16** hexadecimal numerals](https://en.wikipedia.org/wiki/Hexadecimal)|
|_Digipeater Address_|any out of [**5** recommended `n-N` paradigm paths](#path-codes)|
|_Control Field_|**not required**|
|_Protocol ID_|**not required**|
|_Information Field_|up to 256 out of [**95** printable ASCII characters](https://en.wikipedia.org/wiki/ASCII#Printable_characters)<br/>first character = [_Data Type ID_](#data-type-codes)|
|_Frame Check Sequence_|**not required**; [FEC](https://en.wikipedia.org/wiki/Error_correction_code#Forward_error_correction)&nbsp;& [CRC](https://en.wikipedia.org/wiki/Cyclic_redundancy_check) are provided by LoRa|
|_Flag_|**not required**|

- _Source Address, SSID_ and _Data Type ID_ can be compressed into only 5 payload bytes, compared to 26 payload bytes with OE5BPA firmware.
- It is customary to compress latitude, longitude, symbol, course and speed using [Base91](https://en.wikipedia.org/wiki/List_of_numeral_systems#Standard_positional_numeral_systems), which results in another 13 payload bytes; _Data Type ID_ not included. **APRS&nbsp;434** will not differ in this respect.
- If APRS Mic-E compression were to be used instead, that would require another 16 payload bytes to compress latitude, longitude, symbol, course and speed; 7&nbsp;bytes in the superfluous _Destination&nbsp;Address_ and 9&nbsp;bytes in the _Information&nbsp;Field; Data Type ID_ included. Hence, this is not a good option.

## Measurable Benefits

### Reduced Packet Error Rate
**APRS&nbsp;434** geolocation beacons will encode a total of **only 18 payload bytes** at a time, tremendously **increasing the chances of a flawless reception** by an [**APRS&nbsp;434&nbsp;LoRa&nbsp;i-gate**](https://github.com/aprs434/lora.igate). Other firmware tends to consume about six times as many LoRa payload bytes.

LoRa may receive up to 20&nbsp;dB under the noise floor, but keep in mind that [**the packet error rate (PER)**](https://en.wikipedia.org/wiki/Bit_error_rate#Packet_error_ratio) as a function of the bit error rate (BER) [increases with the number of transmitted bits](https://en.wikipedia.org/wiki/Bit_error_rate#Packet_error_ratio).

$$PER = 1 - (1 - BER)^n \approx n \cdot BER$$

approximately, when $BER$ is small and $n$ is large, and where:
- $(1-BER)$: the probability of receiving a bit correctly
- $n$: the number of bits in a packet; which is 8 times the number of bytes

#### PER Examples
When used with an explicit header, LoRa packets will have the following 36&nbsp;bit overhead:
a 16&nbsp;bit physical header `PHDR`, 4&nbsp;bits of header [CRC](https://en.wikipedia.org/wiki/Cyclic_redundancy_check) `PHDR_CRC` and another 16&nbsp;bits of payload `CRC`.

|payload|17 bytes|24 bytes|28 bytes|45 bytes|113 bytes|
|:-----:|:------:|:------:|:------:|:------:|:-------:|
|overhead|36 bits|36 bits|36 bits|36 bits|36 bits|
|n|172 bits|228 bits|260 bits|396 bits|940 bits|
|BER|0.1%|0.1%|0.1%|0.1%|0.1%|
|PER|15.8%|20.4%|22.9%|32.7%|61.0%|

By consequence, the chances of correctly receiving a 17&nbsp;byte payload are more than twice as high than with a 113&nbsp;payload:

$$\frac{1-0.158}{1-0.610} \approx 2.18$$

In reality, above calculations are more convoluted as LoRa employs symbols that are chip jumps or discontinuities in chirps to convey information.
Moreover a preamble, consisting out of a configurable length variable preamble, a set sync word, a start frame delimiter (SDF) and a small pause precede the explicit header.
The variable preamble is important as it trains the receiver at receiving the signal. Hence, the symbol length of this variable preamble also has an effect on the packet error rate.


### Airtime Reduction
An even more important reason for keeping the payload as small as possible, is the airtime required to send the LoRa frame.
AS will be shown in the next section, LoRa **TODO**

Due to the LoRa symbol encoding scheme, airtime reductions occur in steps of 5&nbsp;bytes when the spreading factor is SF12 and the bandwidth 125&nbsp;kHz (CR=1, explicit header, CRC=on). This is depicted as the stepped top trace on the figure below. (Adapted from [airtime-calculator](https://avbentem.github.io/airtime-calculator/ttn/eu868/4,14).)

![Figure 1: The top trace is for SF12BW125. The dot represents a total payload of 17 bytes as proposed for geolocation packets with compression.](lora.airtime-payload.18bytes.png)

|payload|17 bytes|24 bytes|28 bytes|45 bytes|113 bytes|
|:-----:|:------:|:------:|:------:|:------:|:-------:|
|airtime SF12||||||
|airtime SF11||||||

[The Things Network (TTN)](https://www.thethingsnetwork.org) organisation, albeit a global LoRaWAN, is exemplary in stressing [the importance of maintaining LoRa payloads small](https://www.thethingsnetwork.org/docs/devices/bytes/).


## LoRa Link Parameters
Currently, the following LoRa link parameters are commonly in use among amateur radio operators:
- In order to achieve a maximum range, [Semtech](https://en.wikipedia.org/wiki/Semtech) —&nbsp;the company that developed LoRa&nbsp;— recommends selecting the maximum spreading factor $SF = 12$. This corresponds to 12&nbsp;raw bits per symbol. Therefore, each symbol (or frequency chirp) holds $2^{12} = 4096\,\text{chips}$.
- Likewise, the bandwidth is set to the smallest commonly available bandwidth among all LoRa ICs, namely $BW = 125\,\text{kHz}$. This is by definition also the chip rate $R_c = BW$.
- To avoid any further overhead in an already slow mode, the [forward error correction (FEC)](https://en.wikipedia.org/wiki/Error_correction_code#Forward_error_correction) coding rate is kept at $CR = 1$, which corresponds to $\frac{data}{data + FEC} = \frac{4}{5}$.

With these settings, the symbol rate is:

$$R_s = \frac{R_c}{2^{SF}} = \frac{BW}{2^{SF}} = \frac{125\,000}{2^{12}} \approx 30.5\,\text{symbols/s}$$

Whereas the effective data rate $DR$ or bit rate $R_b$ can be calculated as follows:

$$DR = R_b =  \frac{BW}{2^{SF}} \cdot SF \cdot \frac{4}{4 + CR} = \frac{125\,000}{2^{12}} \cdot 12 \cdot \frac{4}{5} \approx 293\,\text{bits/s} \approx 36.6\,\text{byte/s}$$

Finally, it was observed that amateur radio predominantly employs the LoRa sync word '0x12'; which is manufacturer recommended for private networks, different from LoRaWAN.

Summarised, the following LoRa link parameters are proposed for APRS:

|LoRa parameter|uplink|downlink|
|:------------:|:----:|:------:|
|Region&nbsp;I frequency|443.775&nbsp;MHz|443.900&nbsp;MHz|
|SF|12|12|
|BW|125 000|125 000|
|CR|1 (5/4)|1 (5/4)|
|preamble sync length|8&nbsp;symbols|8&nbsp;symbols|
|sync word|`0x12`|`0x12`|
|header|explicit (20&nbsp;bits)|explicit (20&nbsp;bits)|
|[CRC](https://en.wikipedia.org/wiki/Cyclic_redundancy_check)|on (16&nbsp;bits)|on (16&nbsp;bits)|
|IQ|normal|inversed|

Above parameters seem adequate for sending LoRa frames with short, compressed payloads over the largest possible distance when the number of participant nodes is relatively low.
However, network simulations are deemed necessary to quantify the statistical capacity of a LoRa channel in different scenarios.

> For an in depth tutorial slide series about LoRa (and LoRaWAN), please refer to [Mobilefish.com](https://www.mobilefish.com/developer/lorawan/lorawan_quickguide_tutorial.html), also available in video format on [YouTube](https://youtube.com/playlist?list=PLmL13yqb6OxdeOi97EvI8QeO8o-PqeQ0g).

### Considerations for Switching to SF11
Depending on how popular APRS over LoRa becomes and on how intensely it will get used,
there might come a time when the LoRa channel gets saturated.
Unlike packet radio, LoRa has no carrier sensing capability.
Sending longer text messages, even when compressed, may aggravate the situation.
The same holds true for meshing or (emergency) `n-N` paradigm digipeating on the same channel.

When this situation occurs, the ham radio community should seriously **consider switching from SF12 to SF11.**
Doing so, would effectively double the data rate.

Not only would switching to SF11 reduce channel congestion;
It would also save 50% on airtime and batteries.
The range penalty from switching from SF12 to SF11 would in most circumstances be acceptable,
provided the availability of i‑gates in an area is sufficient.

With a payload of only 17&nbsp;bytes, the compressed geolocation frame is perfectly geared towards
taking advantage of the reduced airtime offered by SF11 (see [graph](#airtime-reductions)).

Unfortunately, most cheap i‑gates currently in use by ham operators are only capable of receiving one preset spreading factor.
Therefore, a choice needs to be made between SF12 and SF11.
Considering what some members of the amateur radio community have come to expect of LoRa,
the faster data rate offered by SF11 ought to be preferred.

### LoRa ICs and Modules
- [Semtech LoRa products](https://www.semtech.com/lora/lora-products)
- [Semtech SX1278](https://www.semtech.com/products/wireless-rf/lora-core/sx1278)

- [HopeRF LoRa modules](https://www.hoperf.com/modules/lora/index.html)
- [HopeRF RFM98W](https://www.hoperf.com/modules/lora/RFM98.html)


## Callsign, SSID, Path and Data Type Compression
Callsigns contain only capital letters and digits.
Up to six characters from such a 36 character set can easily be compressed into 4 `CCCC` bytes of an extended 256 character set.

Hence, all compressed APRS frames in this standard begin with 5 `CCCCD` bytes, irrespectively of the [_Data Type_](#data-type-codes).
Furthermore, the compressed frame length is limited by design to a maximum of 45 bytes, which leaves up to 40&nbsp; bytes for a payload.
For certain _Data Types,_ the maximum length is even significantly lower.

I‑gates should test whether the payload length of a received frame is in correspondence to the declared _Data Type_.
**Frames that do not comply, should be rejected.**

The combination of the declared _Data Type_ with the corresponding payload length forms the key so to speak to the i‑gate.
This is what allowed for a headerless frame design.
It prevents the i‑gate from relaying frames that are not intended for this compressed frame link.

|_Callsign_|_SSID_,<br/>_Path Code_&nbsp;&<br/>_Data Type Code_|_Compressed Data_|
|:--------:|:-------------------------------------------------:|:---------------:|
|4 bytes|1 byte|≤&nbsp;40&nbsp;bytes|
|`CCCC`|`D`||

where:
- `CCCC`: 4 bytes for the compressed 6 character _Callsign_
- `D`: compresses into 1 byte:
  + the [_SSID_](#ssid-recommendations) (between SSID 0 [none] and 15; included),
  + the [_Path Code_](#path-codes) (between path 0 [none] and 3; included), and
  + the [_Data Type Code_](#data-type-codes) (between type 0 and 3; included)

### Encoding CCCC
1. Treat the given 6&nbsp;character callsign string as a Base36 encoding. Decode it first to an integer.
2. Then, encode this integer as a 4&nbsp;byte Base256 `CCCC` bytestring.

### Decoding CCCC
1. First, decode the given 4&nbsp;byte Base256 `CCCC` bytestring to an integer.
2. Then, encode this integer as a 6&nbsp;character Base36 string, corresponding to the callsign.

### Encoding D
1. First, multiply the _SSID_ integer by&nbsp;16.
2. Multiply the _Path Code_  by&nbsp;4.
3. Then, algebraically add to these intermediate results to the _Data Type Code_ integer from below table.
4. Finally, convert the resulting integer to a single Base256 `D` byte.

### Decoding D
1. First, decode the given Base256 `D` byte to an integer.
2. The _SSID_ equals the integer quotient after [integer division](https://en.wikipedia.org/wiki/Division_(mathematics)#Of_integers) of the decoded integer by&nbsp;16.
3. The [remainder](https://en.wikipedia.org/wiki/Remainder) of above integer division is subjected to a second integer division by&nbsp;4.
4. The _Path Code_ equals the integer quotient of this second integer division.
5. Whereas the _Data Type Code_ equals the remainder this second integer division.

### Codec Algorithms for CCCCD
- [Python3](compression.py) compression algorithms and tests
- [MIT License](https://github.com/aprs434/aprs434.github.io/blob/main/LICENSE)

### SSID Recommendations
A secondary station identifier is a number in the range 0-15, as an adjunct to the station _Callsign_.
Similarly as with IEEE 802.11 wireless networks, an APRS SSID identifies a set of APRS station capabilities.

|_SSID_|APRS station type|
|:----:|:---------------:|
|0|primary station; usually fixed and message capable|
|1|generic additional station, digi, mobile, wx, etc.|
|2|generic additional station, digi, mobile, wx, etc.|
|3|generic additional station, digi, mobile, wx, etc.|
|4|generic additional station, digi, mobile, wx, etc.|
|5|other networks (D‑STAR, DMR, smartphones etc.)|
|6|special activity, satellite ops, camping, etc.|
|7|walkie talkies, HTs or other human portable|
|8|boats, sailboats, RVs or second main mobile|
|9|primary mobile (usually message capable)|
|10|Internet, (LoRa) i‑gates, echolink, Winlink, AVRS, APRN, etc.|
|11|balloons, aircraft, spacecraft, etc.|
|12|APRStt, DTMF, RFID, devices, one‑way (LoRa) trackers\*, etc.|
|13|weather stations|
|14|truckers or generally full time drivers|
|15|generic additional station, digi, mobile, wx, etc.|

\*: One-way trackers best use the 12 one‑way _SSID_ indicator,
whereas _SSID_ 9 usually means a ham with full communication capabilities; both APRS message and voice.

### Data Type Codes
Of all the _Data Types_ defined in the [APRS Protocol Reference](https://hamwaves.com/prs/doc/2000.aprs.1.01.pdf), a subset was selected, based on popularity and foremost suitability for LoRa.

|_Data Type_|_ID_|_Data Type Code_|payload|
|:---------:|:--:|:--------------:|:-----:|
|compressed [**geolocation**](#compressed-geolocation-frames) — no&nbsp;timestamp|`!`&nbsp;or&nbsp;`=`|0|17 or 19|
|complete [**weather report**](#compressed-weather-report-frames) — with compressed geolocation, no&nbsp;timestamp|`!`&nbsp;or&nbsp;`=`|0|28 or 29|
|[**status report**](#compressed-status-report-frames) (≤&nbsp;28&nbsp;characters)|`>`|1|6—24|
|[**item report**](#compressed-item-report-frames) — with compressed geolocation|`)`|2|20—24|
|[**addressed message**](#compressed-addressed-message-frames) (≤&nbsp;51&nbsp;characters)|`:`|3|10—45|

Note: Weather reports use the same _Data Type IDs_ as position reports but with a _Symbol Code_ `_` overlay.


## Digipeating on LoRa Channels
> **⚠ <u>REFRAIN</u> from digipeating on LoRa channels!**
> Because of LoRa being a slow data rate mode, digipeating on LoRa channels quickly leads to unwanted channel congestion.
> Unlike AX.25 packet radio, LoRa does not offer [carrier sensing](https://en.wikipedia.org/wiki/Carrier-sense_multiple_access).

Also consider that:
- Most LoRa gateways are connected to the APRS‑IS Internet server network and many users are merely interested in reaching APRS‑IS.
- There are hardly any, if any, low power portable LoRa devices displaying situational awareness in relation to other LoRa devices.
- In IARU Region&nbsp;I the central frequency of 433.900 MHz is proposed as a downlink channel from gateways to nodes. The proposal does not mention digipeating.

Hence, below `n-N` paradigm paths could be interpreted foremost as crossover AX.25 packet digipeating paths for any (VHF) digipeater co‑located with the LoRa (i‑)gate.

However, suppose meshing or `n-N` paradigm digipeating were to be allowed on a single LoRa channel; even for trackers. This would offer interesting emergency capabilities when no Internet is available. However, this would absolutely require switching from SF12 to the higher data rate offered by SF11 [as explained before](#considerations-for-switching-to-sf11). In such a scenario, below table represents the LoRa device communicating its digipeating requirements to the mesh network.

### Path Codes

|station|recommended `n-N` paradigm path|_Path Code_|
|:-----:|:-----------------------------:|:---------:|
|no digipeating||0|
|metropolitan fixed, mountain expeditions, balloons&nbsp;& aircraft|`WIDE2-1`|1|
|extremely remote fixed|`WIDE2-2`||
|metropolitan mobile|`WIDE1-1,WIDE2-1`|2|
|extremely remote mobile|`WIDE1-1,WIDE2-2`||
|space satellites|`ARISS,WIDE2-1`|3|

Note:
- The first `n` digit in `n-N` paradigm paths indicates the coverage level of the digipeater, whereby `1` is for domestic fill‑in digipeaters and `2` is for county-level digipeaters.
- The second `N` digit indicates the number of repeats at the indicated coverage level.


## Compressed Geolocation Frames
A compressed geolocation frame has a payload of either exactly 17 or 19 bytes.

|_Callsign_|_SSID_,<br/>_Path Code_&nbsp;&<br/>_Data Type Code_|_Compressed Data_|
|:--------:|:-------------------------------------------------:|:---------------:|
|4 bytes|1 byte|13 (or 15) bytes|
|`CCCC`|`D`|`/XXXXYYYY$cs`+`(aa)`|

where:
- `CCCC`: 4 bytes for the compressed 6 character _Callsign_
- `D`: compresses into 1 byte:
  + the [_SSID_](#ssid-recommendations) (between SSID 0 [none] and 15; included),
  + the [_Path Code_](#path-codes) (between path 0 [none] and 3; included), and
  + the [_Data Type Code_](#data-type-codes) (between type 0 and 3; included)
- `/`: the _Symbol Table Identifier_
- `XXXX`: the Base91 compressed longitude
- `YYYY`: the Base91 compressed latitude
- `$`: the _Symbol Code_
- `cs`: the compressed course (in degrees) and speed (in knots)
- `(aa)`: optionally, the compressed altitude (in feet)

Note:
- Terrestrial objects do not require sending altitude data. Anyhow, GPS height readings are notorious for being significantly inaccurate.
- In absence of `aa`, the i‑gate adds the _Compression Type Byte_ `T` right behind `cs`.
- When `aa` is present, the i‑gate will decompress the whole frame.


## Compressed Weather Report Frames
A compressed weather report frame has a payload of either exactly 28 or 29 bytes.

|_Callsign_|_SSID_,<br/>_Path Code_&nbsp;&<br/>_Data Type Code_|_Compressed Data_|
|:--------:|:-------------------------------------------------:|:---------------:|
|4 bytes|1 byte|23 (or 24) bytes|
|`CCCC`|`D`|`/XXXXYYYY_csgtrrppPPhbb(S)`|

where:
- `CCCC`: 4 bytes for the compressed 6 character _Callsign_
- `D`: compresses into 1 byte:
  + the [_SSID_](#ssid-recommendations) (between SSID 0 [none] and 15; included),
  + the [_Path Code_](#path-codes) (between path 0 [none] and 3; included), and
  + the [_Data Type Code_](#data-type-codes) (between type 0 and 3; included)
- `/`: the _Symbol Table Identifier_
- `XXXX`: the Base91 compressed longitude
- `YYYY`: the Base91 compressed latitude
- `_`: the weather report _Symbol Code_
- `cs`: the compressed wind direction (in degrees) and sustained one-minute wind speed (in knots)
- `g`: gust (half of peak wind speed in km/h in the last 5 minutes)
- `t`: temperature (in kelvin above 173.15 K)
- `rr`: rainfall (in mm) over the past hour
- `pp`: rainfall (in mm) over the past 24 hours
- `PP`: rainfall (in mm) since midnight
- `h`: humidity (in %)
- `bb`: barometric pressure (in Pa above 50000)
- `(S)`: optionally, snowfall (in cm) over the past 24 hours

Notes:
- All numerical encodings are one or two byte Base256 encodings.
- Here is a fascinating list of [weather records](https://en.wikipedia.org/wiki/List_of_weather_records).
- The i‑gate adds the _Compression Type Byte_ `T` right behind `cs`.


## Compressed Text
In order to prevent channel congestion, it is crucial to limit the character set of messages. This allows for more frame compression.
In resemblance to Morse code, the character set would contain only 26 Latin capital letters, the 10&nbsp;digits, space and a few punctuation marks and symbols. Limiting the set to 42 characters lets it fit 6 times in the 256 character set of LoRa.

|character set|amount|
|:-----------:|:----:|
|Latin capital letters|26|
|digits|10|
|space|1|
|punctuation `.`&nbsp;`?`|2|
|symbols `-`&nbsp;`/`&nbsp;`_`|3|
|**TOTAL**|**42**|

### Encoding tttt
1. Perform character replacement and filtering on the given string; only allow for charcters of the [42&nbsp;character set](#compressed-text).
2. Treat the resulting text string as a Base42 encoding. Decode it first to an integer.
3. Then, encode this integer as a Base256 `tttt` bytestring.

### Decoding tttt
1. First, decode the given Base256 `tttt` bytestring to an integer.
2. Then, encode this integer as a Base42 string, corresponding to the text.

### Codec Algorithms for tttt
- [Python3](compression.py) compression algorithms and tests
- [MIT License](https://github.com/aprs434/aprs434.github.io/blob/main/LICENSE)


## Comments
> **⚠ <u>REFRAIN</u> from adding any comments!**
> Adding more bytes to a LoRa frame only reduces the chances on successful reception.
> Rather, consider sending an occasional [status report](#compressed-status-report-frames).


## Compressed Status Report Frames
A compressed status report frame has a payload of between 6 and 24 bytes.

|_Callsign_|_SSID_,<br/>_Path Code_&nbsp;&<br/>_Data Type Code_|_Compressed Data_|
|:--------:|:-------------------------------------------------:|:---------------:|
|4 bytes|1 byte|≤&nbsp;19&nbsp;bytes|
|`CCCC`|`D`|`t(tttt…tttt)`|

where:
- `CCCC`: 4 bytes for the compressed 6 character _Callsign_
- `D`: compresses into 1 byte:
  + the [_SSID_](#ssid-recommendations) (between SSID 0 [none] and 15; included),
  + the [_Path Code_](#path-codes) (between path 0 [none] and 3; included), and
  + the [_Data Type Code_](#data-type-codes) (between type 0 and 3; included)
- `t(tttt…tttt)`: between 1 and 19 bytes of compressed text from a limited 42 character set, corresponding to 28 uncompressed characters


## Compressed Item Report Frames
A compressed item report frame has a payload of between 20 and 24 bytes.

|_Callsign_|_SSID_,<br/>_Path Code_&nbsp;&<br/>_Data Type Code_|_Compressed Data_|
|:--------:|:-------------------------------------------------:|:---------------:|
|4 bytes|1 byte|15—19 bytes|
|`CCCC`|`D`|`/XXXXYYYY$csTttt(tttt)`|

where:
- `CCCC`: 4 bytes for the compressed 6 character _Callsign_
- `D`: compresses into 1 byte:
  + the [_SSID_](#ssid-recommendations) (between SSID 0 [none] and 15; included),
  + the [_Path Code_](#path-codes) (between path 0 [none] and 3; included), and
  + the [_Data Type Code_](#data-type-codes) (between type 0 and 3; included)
- `/`: the _Symbol Table Identifier_
- `XXXX`: the Base91 compressed longitude
- `YYYY`: the Base91 compressed latitude
- `$`: the _Symbol Code_
- `cs`: the compressed course and speed
- `ttt(tttt)`: 3 to 7 bytes for the compressed _Item Name_ (between 3 and 9 characters of the limited 42 character set)

Note: The i‑gate adds the _Compression Type Byte_ `T` right behind `cs`.


## Compressed Addressed Message Frames
Up to now, APRS has been unduly considered to be predominantly a one-way localisation technology.
This went to the point that many mistakenly think the letter "P" in the acronym APRS would stand for "position."
[Bob Bruninga WB4APR (SK)](http://www.aprs.org), the spiritual father of APRS, deeply resented this situation.

> _"APRS is not a vehicle tracking system.
> It is a two-way tactical real-time digital communications system between all assets in a network
> sharing information about everything going on in the local area."_

In Bob's view of APRS as being foremost a real-time situational and tactical tool, messaging definitely merits its place.
One of the long-term goals is rendering APRS messaging more popular by offering messaging pager designs.

Below proposal for the compression of addressed message frames is primarily intended for uplink-only messaging or
for direct messaging without the aid of a digipeater.
The available message length of 51 characters is largely sufficient
for, for example, SOTA self-spotting using [APRS2SOTA](https://www.sotaspots.co.uk/Aprs2Sota_Info.php).

On the other hand, two‑way messaging over a digipeater would definitely require:
- the faster [SF11](#considerations-for-switching-to-sf11) data mode, and
- GPS-disciplined, dynamic [time division multiple access (TDMA)](https://en.wikipedia.org/wiki/Time-division_multiple_access) multiplexing.

The formulation of such a two‑way TDMA protocol is beyond the scope of this document.

A compressed addressed message frame has a payload of between 10 (for an empty ping) and 45 bytes.

|_Callsign_|_SSID_,<br/>_Path Code_&nbsp;&<br/>_Data Type Code_|_Compressed Data_|
|:--------:|:-------------------------------------------------:|:---------------:|
|4 bytes|1 byte|≤&nbsp;40&nbsp;bytes|
|`CCCC`|`D`|`EEEEF(tttt…tttt)`|

where:
- `CCCC`: 4 bytes for the compressed 6 character _Callsign_
- `D`: compresses into 1 byte:
  + the [_SSID_](#ssid-recommendations) (between SSID 0 [none] and 15; included),
  + the [_Path Code_](#path-codes) (between path 0 [none] and 3; included), and
  + the [_Data Type Code_](#data-type-codes) (between type 0 and 3; included)
- `EEEE`: 4 bytes for the compressed _Addressee_ (up to 6 character callsign)
- `F`: compresses into 1 byte:
  + the _Addressee SSID_ (between SSID 0 [none] and 15; included), and
  + the _Message No_ (from 0 to 15; included)
- `(tttt…tttt)`: up to 35 bytes of compressed text from a limited 42 character set, corresponding to 51 uncompressed characters

### Encoding and Decoding EEEE
The `EEEE` codec algorithms are identical to the [`CCCC` codec algorithms](#encoding-cccc).

### Encoding F
1. First, multiply the _SSID_ integer by&nbsp;16.
2. Then, algebraically add to this the _Message No_ integer.
3. Finally, convert the resulting integer to a single Base256 `F` byte.

### Decoding F
1. First, decode the given Base256 `F` byte to an integer.
2. The _SSID_ equals the integer quotient after [integer division](https://en.wikipedia.org/wiki/Division_(mathematics)#Of_integers) of the decoded integer by&nbsp;16.
3. Whereas the _Message No_ equals the [remainder](https://en.wikipedia.org/wiki/Remainder) of the decoded integer by&nbsp;16 ([modulo operation](https://en.wikipedia.org/wiki/Modulo_operation)).

### Codec Algorithms for EEEEF
- [Python3](compression.py) compression algorithms and tests
- [MIT License](https://github.com/aprs434/aprs434.github.io/blob/main/LICENSE)


## ITU Regulation
From a ITU regulatory point of view, long range communication —which, by definition, includes LoRa— is not allowed on ISM (Industrial, Scientific&nbsp;& Medical) bands. ISM&nbsp;bands are intended for local use only.

The amateur radio service forms a sole exception to this, as its 70&nbsp;cm UHF band happens to [overlap](https://hamwaves.com/lpd433/en/index.html#lpd433-channels) the [ITU&nbsp;Region&nbsp;1](https://en.wikipedia.org/wiki/ITU_Region) 434&nbsp;MHz ISM&nbsp;band as a primary service.
Moreover, ham radio is not restricted to a 20&nbsp;dBm (=&nbsp;100&nbsp;mW) power level, nor any 1% duty cycle limits on this band.

As a general rule, secondary users should always check whether a frequency is in use by a primary user before transmitting on air.
However, LoRa has no carrier sensing capability. Therefore, secondary ISM band users lack the ability to check whether an amateur radio operator is using the 434&nbsp;MHz band as a primary user.


## Reducing Power Consumption
1. OLED displays have a limited life span and consume quite a bit of power. An OLED screen and its driver [can be put to sleep](https://bengoncalves.wordpress.com/2015/10/01/oled-display-and-arduino-with-power-save-mode/) when the tracker is idle. The same holds true for the LoRa radio module and the ESP32. This needs to be investigated.
2. GPS modules are also power hogs. It may be conceivable to use the WLAN receiver aboard an ESP32 for localisation, whereby the three strongest WLAN SSIDs are transmitted to the i‑gate. The i‑gate would then guess the tracker location from a freely available [wardriving](https://en.wikipedia.org/wiki/Wardriving) data service from the Internet. This is comparable to how Google Android smartphone localisation works.


## Recommended Hardware

### Tracker Hardware:
- TTGO T-Beam 433&nbsp;MHz v0.7 or v1.1
- longer 433&nbsp;MHz antenna with [SMA male](https://en.wikipedia.org/wiki/SMA_connector) connector
- 16.9&nbsp;mm long tiger tail wire soldered to the female SMA socket
- 5&nbsp;V, 3&nbsp;A microUSB charge adapter
- Panasonic NCR18650B Li-ion cell, or quality equivalent
- glue gun to stick the GPS antenna to the cell holder
- SH1106 1.3" I²C (4‑pin) OLED display (slightly larger than the usual 0.8" displays often sold with the TTGO T-Beam)
- enclosure

### I-Gate Hardware:
- Either:
  + [TTGO LORA32 433&nbsp;MHz v2](http://www.lilygo.cn/prod_view.aspx?TypeId=50060&Id=1319&FId=t3:50060:3) ([U.FL](https://en.wikipedia.org/wiki/Hirose_U.FL) or [SMA female](https://en.wikipedia.org/wiki/SMA_connector) RF socket), or
  + maybe Heltec ESP32 LoRa 433&nbsp;MHz **v2** ([U.FL](https://en.wikipedia.org/wiki/Hirose_U.FL) female RF socket); subject to satisfactory receiver testing
  + **⚠ <u>DO NOT USE</u>** Heltec ESP32 LoRa 433&nbsp;MHz **v1** as it is as deaf as a post!
- 70&nbsp;cm amateur radio colinear groundplane antenna with coaxial cable and connectors
- 16.9&nbsp;mm long tiger tail wire soldered to the RF socket
- 5&nbsp;V, 1&nbsp;A microUSB power supply
- enclosure


## ESP32 Firmware Downloads

### Tracker Firmware
See: <https://github.com/aprs434/lora.tracker>

Please, note that the [`tracker.json`](https://github.com/aprs434/lora.tracker/blob/master/data/tracker.json) configuration file has been much simplified.

### I-Gate Firmware
See: <https://github.com/aprs434/lora.igate>

> Currently, the APRS&nbsp;434 tracker is still compatible with the i-gate developed by Peter Buchegger, OE5BPA. However, this will soon change as more LoRa frame compression is added.
>
> We feel confident that trackers with the proposed APRS&nbsp;434 compressed LoRa frame will eventually become dominant because of the longer range merit. To smooth out the transition, we are developing an **i‑gate capable of understanding both formats;** i.e. compressed APRS&nbsp;434&nbsp;and longer, legacy OE5BPA.
>
> It is strongly advised to install [**the accompanying APRS&nbsp;434 i-gate**](https://github.com/aprs434/lora.igate) as new releases will be automatically pulled over‑the‑air (OTA) via WiFi.


## Development Road Map

### Data Link Layer

|tracker<br/>firmware|completed|feature|LoRa payload|compatible with OE5BPA i‑gate|
|:------------------:|:-------:|:-----:|:----------:|:---------------------------:|
|v0.0.0|✓|original [OE5BPA tracker](https://github.com/lora-aprs/LoRa_APRS_Tracker)|113 bytes|✓|
|v0.1.0|✓|byte-saving [`tracker.json`](https://github.com/aprs434/lora.tracker/blob/master/data/tracker.json)|87 bytes|✓|
|v0.2.0|✓|fork of the [OE5BPA tracker](https://github.com/lora-aprs/LoRa_APRS_Tracker)<br/>with significantly less transmitted&nbsp;bytes|44 bytes|✓|
|v0.3.0|✓|[Base91](https://en.wikipedia.org/wiki/List_of_numeral_systems#Standard_positional_numeral_systems) compression of  location, course&nbsp;and speed&nbsp;data|31 bytes|✓|
|[v0.4.0](https://github.com/aprs434/lora.tracker)|✓|removal of the transmitted [newline](https://en.wikipedia.org/wiki/Newline) `\n`&nbsp;character at&nbsp;frame&nbsp;end|30 bytes|✓|
|||random time jitter between fixed interval packets to&nbsp;avoid repetitive&nbsp;[collisions](https://en.wikipedia.org/wiki/Collision_domain)|30 bytes|✓|
|||[tracker](https://github.com/aprs434/lora.tracker) and [i-gate](https://github.com/aprs434/lora.igate) with frame&nbsp;address&nbsp;compression,<br/>no custom&nbsp;header in&nbsp;payload|17 bytes|use the [APRS&nbsp;434 i‑gate](https://github.com/aprs434/lora.igate)|

> Currently, the APRS&nbsp;434 tracker is still compatible with the i-gate developed by Peter Buchegger, OE5BPA. However, this will soon change as more LoRa frame compression is added.
>
> We feel confident that trackers with the proposed APRS&nbsp;434 compressed LoRa frame will eventually become dominant because of the longer range merit. To smooth out the transition, we are developing an **i‑gate capable of understanding both formats;** i.e. compressed APRS&nbsp;434&nbsp;and longer, legacy OE5BPA.
>
> It is strongly advised to install [**the accompanying APRS&nbsp;434 i-gate**](https://github.com/aprs434/lora.igate) as new releases will be automatically pulled over‑the‑air (OTA) via WiFi.

### Tracker Hardware

|tracker<br/>firmware|completed|feature|
|:------------------:|:-------:|:-----:|
|v0.3.1|✓|coordinates displayed on screen|
|||reduced power consumption through [SH1106 OLED sleep](https://bengoncalves.wordpress.com/2015/10/01/oled-display-and-arduino-with-power-save-mode/)|
|||button press to activate OLED screen|
|||ESP32 power reduction|

### Messaging
At first, only uplink messaging to an i-gate will be considered. This is useful for status updates, [SOTA self‑spotting](https://www.sotaspots.co.uk/Aprs2Sota_Info.php), or even emergencies.

On the other hand, bidirectional messaging requires time division multiplexing between the up- and downlink, based on precise GPS timing. That is because channel isolation between different up- and downlink frequencies probably would require costly and bulky resonant cavities.

|tracker<br/>firmware|completed|feature|
|:------------------:|:-------:|:-----:|
|||add a [library](https://web.archive.org/web/20190316204938/http://cliffle.com/project/chatpad/arduino/) for the [Xbox 360 Chatpad](https://nuxx.net/gallery/v/acquired_stuff/xbox_360_chatpad/) keyboard|
|||[support](https://www.hackster.io/scottpowell69/lora-qwerty-messenger-c0eee6) for the [M5Stack CardKB Mini](https://shop.m5stack.com/products/cardkb-mini-keyboard) keyboard|

### WiFi Geolocation
TBD


## News, Social & Co-Development
Feel free to join our public [**Telegram Group**](https://t.me/aprs434) for the latest news and cordial discussions.

You are invited to contribute code improvements to [**this project on GitHub**](https://github.com/aprs434).
Here is a lightweight [video introduction to using GitHub](https://youtu.be/tCuPbW31vAw) by Andreas Spiess, HB9BLA.


## Acknowledgements
- Bernd Gasser, OE1ACM, for the earliest LoRa APRS experiments and code
- Christian Johann Bauer, OE3CJB, for the Base91 geolocation compression algorithm
- Peter Buchegger, OE5BPA, for providing the bulk of the tracker and i‑gate firmware as open source code, in a handy [PlatformIO](https://platformio.org) environment, with [over-the-air (OTA)](https://en.wikipedia.org/wiki/Over-the-air_programming) i‑gate updates. This was the ideal starting point for running LoRa frame compression experiments.
- Folkert Tijdens, PA0FOT, for asking the right questions, rendering this document more scholarly
- Pascal Schiks, PA3FKM, for providing insights about microcontroller stacks
- Greg Stroobandt, ON3GR, for cycling around the city with a privacy invading tracker
- Erwin Fiten, ON8AR, for testing firmware and reporting on long distance car approaches to the LoRa i‑gate
- Jan Engelen, DL6ZG, for testing firmware and providing feedback
- [Github.com](https://github.com/) for hosting this project, free of charge

May 2022<br/>
Serge Y. Stroobandt, ON4AA


<script>
MathJax = {
  tex: {
    inlineMath: [['$', '$']]
  }
};
</script>
<script src="https://polyfill.io/v3/polyfill.min.js?features=es6"></script>
<script id="MathJax-script" async src="https://cdn.jsdelivr.net/npm/mathjax@3/es5/tex-mml-chtml.js"></script>

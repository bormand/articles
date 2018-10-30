Taming the Slowpoke or fast JTAG link with DE0-Nano
===================================================

## Intro

Some of my DE0-Nano projects demanded fast and reliable communication with PC.
Of course, Cyclone IV can easily talk with UART, SPI or even USB FS.
But that will require additional hardware (some pins and wires at least).
So, I decided to use the only PC compatible connector — Micro-USB port of the USB-Blaster.
Using same port for both config upload and communication will be convenient too.

Quartus tools support TCL commands for JTAG communication over USB-Blaster.
But that tools aren't very portable (try to run them on latest Linux distros for example).
And installing twelve gigs just to handle 100 JTAG exchanges per second isn't very fun.

OpenOCD has drivers for USB-Blaster.
Unfortunately, their driver can handle at most 250 exchanges per second
(it probably suffers from the small packet problem that will be described below).
Actually, it's fine for most applications, but can we do better?

Let's try to communicate with DE0-Nano directly!

## First steps

OpenOCD code contains useful comments and links about USB-Blaster hardware and protocol:
[usb_blaster.c](https://sourceforge.net/p/openocd/code/ci/master/tree/src/jtag/drivers/usb_blaster/usb_blaster.c)

Now it's easy to locate USB-Blaster components on the board: FT245BL and 24MHz oscillator on the front side and EPM240 on the back side.
FT245BL is just a USB to 8-bit parallel converter. EPM240 bridges that converter to the JTAG pins of the FPGA and handles actual protocol.

![USB-Blaster parts](usb_blaster_parts.jpg "USB-Blaster parts")

Basic FT245BL protocol is very simple.
Just send outgoing data with bulk transfer to the endpoint 0x02 and receive incoming data from the endpoint 0x81.
Each incoming packet will be prefixed with two "modem status bytes" that are useless for parallel port and can be dropped.

USB-Blaster protocol (handled by EPM240) is simple too (quote from OpenOCD sources):

```c
/*
 * Actually, the USB-Blaster offers a byte-shift mode to transmit up to 504 data
 * bits (bidirectional) in a single USB packet. A header byte has to be sent as
 * the first byte in a packet with the following meaning:
 *
 *   Bit 7 (0x80): Must be set to indicate byte-shift mode.
 *   Bit 6 (0x40): If set, the USB-Blaster will also read data, not just write.
 *   Bit 5..0:     Define the number N of following bytes
 *
 * All N following bytes will then be clocked out serially on TDI. If Bit 6 was
 * set, it will afterwards return N bytes with TDO data read while clocking out
 * the TDI data. LSB of the first byte after the header byte will appear first
 * on TDI.
 */

/* Simple bit banging mode:
 *
 *   Bit 7 (0x80): Must be zero (see byte-shift mode above)
 *   Bit 6 (0x40): If set, you will receive a byte indicating the state of TDO
 *                 in return.
 *   Bit 5 (0x20): Output Enable/LED.
 *   Bit 4 (0x10): TDI Output.
 *   Bit 3 (0x08): nCS Output (not used in JTAG mode).
 *   Bit 2 (0x04): nCE Output (not used in JTAG mode).
 *   Bit 1 (0x02): TMS Output.
 *   Bit 0 (0x01): TCK Output.
 *
 * For transmitting a single data bit, you need to write two bytes (one for
 * setting up TDI/TMS/TCK=0, and one to trigger TCK high with same TDI/TMS
 * held). Up to 64 bytes can be combined in a single USB packet.
 * It isn't possible to read a data without transmitting data.
 */

```

Both EPM240 and Cyclone have BGA packages. It's hard to monitor signals on their pins unless they are connected to something more accessible.
Quartus library contains undocumented component `cycloneive_jtag`. We can use that component to forward JTAG signals to the external pins:

```verilog
module test(
    input tck,
    input tms,
    input tdi,
    output tdo,
    output tckutap,
    output tmsutap,
    output tdiutap
);
    cycloneive_jtag jtag(
        .tck(tck),
        .tms(tms),
        .tdi(tdi),
        .tdo(tdo),
        .tckutap(tckutap),
        .tmsutap(tmsutap),
        .tdiutap(tdiutap)
    );
endmodule
```

Let's send some bits in bit-bang mode:

```python
# TX only in the bit-bang mode
dev.bulkWrite(0x02, [TMS, TMS|TCK] * 5 + [0])
```

![TX only](tx_only.png "TX only")

```python
# TX+RX in the bit-bang mode
dev.bulkWrite(0x02, [TMS|READ, TMS|TCK|READ] * 5 + [0])
dev.bulkRead(0x81, 64)
```

![TX and RX](tx_rx.png "TX and RX")

Yellow is TCK, blue is TMS. As you can see, bit-banging mode is not very fast.
Single bit transmission consumes 830ns. Receiving bumps that to 1.33µs.
Moreover, we waste two bytes on USB per single JTAG bit (16x overhead!).

Now try byte-shift mode:

```python
# TX only in the byte-shift mode
dev.bulkWrite(0x02, [BYTES|4, 0xDE, 0xAD, 0xBE, 0xEF])
```

![TX only bytes](tx_only_b.png "TX only bytes")

Yellow is TCK, blue is TDI. Now transmission is four times faster (168ns per bit, 193ns taking account for gaps between bytes).
Therefore, transmission speed will be capped at 5.18MBit/s.

Note that TDI is switched in the middle between falling and rising edges of the TCK. Why not on the falling edge to give TDI more time to settle?
Probably USB-Blaster designers attempted to simplify shifting by changing TDI and reading TDO at the same time.

Also note that TCK remains high after byte transmission. We should account for that to properly combine bit-banging and byte-shifting code.

```python
# TX+RX in the byte-shift mode
dev.bulkWrite(0x02, [BYTES|READ|4, 0xDE, 0xAD, 0xBE, 0xEF])
dev.bulkRead(0x81, 64)
```

![TX and RX bytes](tx_rx_b.png "TX and RX bytes")

Clock speed during duplex transfer is same, but gaps are larger than for transmission. It gives 208ns/bit with accounting for gaps.
Therefore, duplex speed will be capped at 4.8MBit/s.

## Need for speed

Now we know physical limitations of the USB-Blaster. Can they be achieved?
Let's start from easiest part — long transmission-only stream.

```python
# TX only stream
for i in range(0, 100000):
    dev.bulkWrite(0x02, [BYTES|63] + [0x55] * 63)
```

![TX only stream](tx_only_s.png "TX only stream")

4.9MBit/s. Just 5% less than predicted 5.18MB/s even with small transfers!

Look at the large gaps with perfect 1ms interval between them. Those gaps are USB framing artifacts.
Full-speed USB host sends Start-of-Frame packet every millisecond.
It delays packets not fitting into the current frame and little FT245BL FIFO underflows.

Increasing transfer size was attempted, but it changes nothing.

Duplex transmission will be more difficult.
We need to balance between sending and receiving to keep both FT245BL FIFOs from overflow and underflow.
USB host can respond to overflows and underflows much faster than our program, so let's use async transfers here.

```python
# TX+RX stream
class Test:
    def __init__(self, dev, limit):
        self.total_rx = 0
        self.total_tx = 0
        self.limit = limit

    def run(self):
        self.tx = dev.getTransfer()
        self.tx.setBulk(0x02, [BYTES|READ|63] + [0x55] * 63, self.tx_callback)
        self.tx.submit()
        self.rx = dev.getTransfer()
        self.rx.setBulk(0x81, 64, self.rx_callback)
        self.rx.submit()
        while self.total_tx < self.limit or self.total_rx < self.limit:
            ctx.handleEventsTimeout(0)

    def tx_callback(self, t):
        assert(t.getStatus() == usb1.TRANSFER_COMPLETED)
        # ignore byte-shift mode header byte
        self.total_tx += t.getActualLength() - 1
        if self.total_tx < self.limit:
            t.submit()

    def rx_callback(self, t):
        assert(t.getStatus() == usb1.TRANSFER_COMPLETED)
        # don't forget about modem status bytes
        assert(t.getActualLength() >= 2)
        self.total_rx += t.getActualLength() - 2
        if self.total_rx < self.limit:
            t.submit()
```

![TX and RX stream](tx_rx_s.png "TX and RX stream")

This code gives us 4.5MBit/s, 7% less than predicted. Increasing transfer sizes doesn't help here too.

FT245BL is a full-speed USB device. Full-speed USB link works at 12Mbps, has 13 bytes overhead for bulk transfers
and 10% of bandwith is reserved for control transfers, so we have around 18 packets per 1ms frame. This perfectly
correlates with nine bars on the waveform above. So we run into USB FS limits here.

Long streams are fine, but what about small non-pipelined requests?

```python
for i in range(0, 100):
    tx = dev.getTransfer()
    tx.setBulk(0x02, [BYTES|READ|32] + [0x55] * 32)
    tx.submit()

    rx = dev.getTransfer()
    rx.setBulk(0x81, 64)
    rx.submit()

    while tx.isSubmitted() or rx.isSubmitted():
        ctx.handleEventsTimeout(0)

    assert(tx.getStatus() == usb1.TRANSFER_COMPLETED)
    assert(tx.getActualLength() == 33)
    assert(rx.getStatus() == usb1.TRANSFER_COMPLETED)
    assert(rx.getActualLength() == 34)
```

![Small packets](small_pkts.png "Small packets")

It's awful, only 16Kbit/s and 16ms per transfer (62.5 requests per second)! Waveform shows short bursts every 16ms. What happens? Looks like FT245BL delays responses for some reason.

After some googling [app note AN232B-04](https://www.ftdichip.com/Support/Documents/AppNotes/AN232B-04_DataLatencyFlow.pdf) from FTDI was found.
According to that note it's not a bug but a feature: `small amounts of data (or the end of large amounts of data), will be subject to a 16 millisecond delay when transferring into the PC`.

FTDI offers four solutions for that issue:

1. Fill buffer up to 64 bytes (2 status bytes and 62 bytes of payload). It works, as we can see in stream tests above.
2. Change RS232 status lines. FT245BL is a parallel converter, it has no such lines.
3. Send event character from the device side. We can persuade USB-Blaster to send us specific byte, but this will interfere with JTAG protocol.
4. Adjust delay for short packets. But it must be at least 1ms, so we can't completely eliminate delay.

Seems like the only practical way to achieve maximal speed is... just sending more data!

Remember that USB-Blaster responds with one byte for:
- each byte with READ (0x40) bit set in bit-bang more
- each byte in byte-shift mode if READ bit was set in the header byte (excluding header byte itself)

Delay should disappear if we align incoming packets to 62 bytes using rules above.
Non-pipelined request will contain at most one byte-shift header,
so we can just set READ bit for everything and append dummy bytes up to 62-63 bytes.

```python
for i in range(0, 100000):
    tx = dev.getTransfer()
    tx.setBulk(0x02, [BYTES|READ|32] + [0x55] * 32 + [READ] * 30)
    tx.submit()

    rx = dev.getTransfer()
    rx.setBulk(0x81, 64)
    rx.submit()

    while tx.isSubmitted() or rx.isSubmitted():
        ctx.handleEventsTimeout(0)

    assert(tx.getStatus() == usb1.TRANSFER_COMPLETED)
    assert(tx.getActualLength() == 63)
    assert(rx.getStatus() == usb1.TRANSFER_COMPLETED)
    assert(rx.getActualLength() == 64)
```

![Small packets 2](small_pkts2.png "Small packets 2")

Much better! 1.2MB/s of useful data (excluding padding) and 4600 requests per second. Seventy times faster.

TO BE CONTINUED...

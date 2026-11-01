# Ghost in the Wire

**Category:** Network forensics  
**Artifact:** `ghost_traffic.pcap`  
**Main technique:** Recovering data carried in ICMP echo-request payloads.

## 1. Extract and identify the capture

```bash
wget https://hicon26-files-b19d71.nbg1.your-objectstorage.com/ghost-in-the-wire/93f74ff6d42fa2b1/ghost-in-the-wire.zip
unzip ghost-in-the-wire.zip
file ghost_traffic.pcap
```

The extracted file was a PCAP capture.

## 2. Summarise the capture

```bash
capinfos ghost_traffic.pcap
```

The capture contained 30 packets, lasted approximately 52 seconds, and used Ethernet encapsulation.

## 3. Filter ICMP traffic

```bash
tshark -r ghost_traffic.pcap -Y icmp
```

The ICMP listing showed a sequence of echo requests from `10.13.37.1` to `10.13.37.2`. Many of these packets carried very small payloads.

## 4. Extract packet payloads

```bash
tshark -r ghost_traffic.pcap -Y icmp \
  -T fields \
  -e frame.number \
  -e ip.src \
  -e ip.dst \
  -e icmp.type \
  -e icmp.seq \
  -e data.data
```

The payloads from the `10.13.37.1` source were individual hexadecimal bytes. Their sequence formed readable ASCII when concatenated and decoded.

## 5. Reconstruct the message

```bash
tshark -r ghost_traffic.pcap \
  -Y "icmp && ip.src == 10.13.37.1" \
  -T fields -e data.data | xxd -r -p
```

This reconstructed the hidden text from the ICMP payload stream. The public write-up intentionally omits the recovered flag.

**Flag:** `hiCTF{REDACTED}`

## 6. Takeaway

Packet captures should not be judged only by protocol metadata. Application data can be carried in fields that look like ordinary diagnostic traffic. Filtering on protocol and source, then reconstructing payload bytes in packet order, exposed the concealed message.

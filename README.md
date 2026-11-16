# Wireless Network Packet Transfer and Analysis Using Python Scapy and Wireshark

A computer communication and network security project. Two Kali Linux virtual machines exchange a text message and files (text, JPEG, WebP) inside **ICMP echo (ping) packets** built with Python **Scapy**. The traffic is captured with **Wireshark** on both sides, and a Python script rebuilds the files from the captures and checks them with MD5 hashes.

> **For learning and lab use only.** Run this only on networks and machines you own or have permission to test.

## Contents

- [How it works](#how-it-works)
- [Packet format](#packet-format)
- [Repository layout](#repository-layout)
- [Requirements](#requirements)
- [Quick start](#quick-start)
- [Analysing the captures](#analysing-the-captures)
- [Results](#results)
- [Security findings](#security-findings)
- [Known limitations](#known-limitations)
- [Future work](#future-work)
- [Author](#author)

## How it works

```
Sender (Kali VM)                                        Receiver (Kali VM)
10.46.216.204                                           10.46.216.99
     |   ICMP echo request: chunk 1/N (4-byte header + data)    |
     | -------------------------------------------------------> |
     |   ICMP echo reply (kernel copies the payload back)       |
     | <------------------------------------------------------- |
     |   ... one chunk about every 28 ms ...                    |
     |   ICMP echo request: chunk N/N                           |
     | -------------------------------------------------------> |   all chunks stored: write file
```

1. The sender cuts a file into chunks of up to 1000 bytes.
2. Each chunk gets a 4-byte header and goes out in one ICMP echo request.
3. The receiver sniffs the requests, stores each chunk under its number, and writes the file when all chunks have arrived.
4. The operating system answers every request with an echo reply. The script itself does not send replies.
5. Wireshark records both ends. `pcapng` captures are then analysed offline.

## Packet format

| Layer | Size (bytes) | Content |
| --- | --- | --- |
| Ethernet II | 14 | MAC addresses |
| IPv4 | 20 | 10.46.216.204 to 10.46.216.99, TTL 64 |
| ICMP echo | 8 | type 8, id 0, sequence 0 |
| Chunk header | 4 | chunk number (2 bytes) and total chunks (2 bytes), big-endian (`struct` format `!HH`) |
| File data | up to 1000 | one slice of the file |

A full frame is 14 + 20 + 8 + 4 + 1000 = **1046 bytes**. The last chunk of a file is shorter. The text-message test sends raw text with no header.

## Repository layout

The names below are the ones used in the project. Rename them to match your own repository.

| File | Purpose |
| --- | --- |
| `sender_file.py` | Sender: splits a file into chunks and sends them as ICMP echo requests |
| `receiver_file.py` | Receiver for JPEG images (the text and PNG receivers are the same script with a different output name) |
| `message_sender.py` / `message_receiver.py` | Send and print a short text message in four pings |
| `analyse_pcapng.py` | Reads every `.pcapng` in a folder, rebuilds the transferred data and prints MD5 hashes (name is a suggestion) |
| `receiver_improved.py` | Improved receiver: resets state, times out, reports missing chunks, optional SHA-256 check |
| `sender_reference.py` | Reference sender that produces the captured packet format |
| `detect_icmp_tunnel.py` | Flags captures with large, constant-id ICMP echo payloads |
| `captures/` | The ten `.pcapng` files (five sender side, five receiver side) |
| `report/` | The written project report |

## Requirements

- Python 3.8 or newer
- [Scapy](https://scapy.net): `pip install scapy`
- [Wireshark](https://www.wireshark.org) for viewing the captures
- Root privileges on the VMs (`sudo`), because Scapy needs raw packet access
- Two machines on the same IPv4 subnet (the project used two Kali Linux VMs)

## Quick start

1. Edit the IP addresses in the scripts to match your two machines. The project used `10.46.216.204` (sender) and `10.46.216.99` (receiver).
2. Check connectivity in both directions:

   ```bash
   ping -c 4 10.46.216.99
   ```

3. On the **receiver**, start Wireshark on `eth0` with the display filter `icmp`, then start the receiver:

   ```bash
   sudo python3 receiver_file.py
   ```

4. On the **sender**, send the file:

   ```bash
   sudo python3 sender_file.py
   ```

5. The receiver prints `Received chunk n/N` and then reconstructs the file. Stop the capture and save it as `.pcapng`.

To use the improved receiver and reference sender instead:

```bash
# receiver
sudo python3 receiver_improved.py --out received_photo.jpg --sender 10.46.216.204 --me 10.46.216.99

# sender
sudo python3 sender_reference.py photo.jpg --dst 10.46.216.99
```

`receiver_improved.py` stops by itself when the file is complete or after `--timeout` seconds (default 120), and accepts `--sha256 <value>` to check the finished file.

## Analysing the captures

Useful Wireshark display filters:

| Filter | Shows |
| --- | --- |
| `icmp` | The whole transfer |
| `icmp.type == 8` | Echo requests (file chunks) |
| `icmp.type == 0` | Echo replies |
| `ip.addr == 10.46.216.204 && ip.addr == 10.46.216.99 && icmp` | Only the two test hosts |
| `frame.len >= 800 && icmp` | Large echo packets that a normal ping does not produce |

To rebuild the files from the captures and print their hashes, set `INPUT_FOLDER` and `OUTPUT_FOLDER` at the top of `analyse_pcapng.py`, then run:

```bash
python analyse_pcapng.py
```

To check the captures for ICMP tunnelling patterns:

```bash
python3 detect_icmp_tunnel.py captures/*.pcapng
```

> The two text-message captures show `INCOMPLETE TRANSFER` in the analysis output. This is expected: the message has no chunk header, so the script reads the first four characters (`Hell`) as a header (chunk 18,533 of 27,756). The message itself arrived whole in all four pings.

## Results

All transfers completed with no missing chunks, every request had a matching reply, and the sender-side and receiver-side captures give the same MD5 hash.

| Transfer | Size (bytes) | Chunks | MD5 of rebuilt data |
| --- | --- | --- | --- |
| Text message | 33 | 4 pings, no header | not applicable |
| Text file | 51 | 1 | `858000e240a5e0342d22de83fd7866ad` |
| JPEG (204 x 192) | 10,678 | 11 | `86721a0b5bd6507d3f092f3c868cc788` |
| WebP (474 x 474) | 10,844 | 11 | `649ca1a94faf1bad27808626135ec011` |
| JPEG logo (662 x 463) | 37,813 | 38 | `cd8c9951f5abe337167f0c6176221cfc` |

Speed was about 35 KB/s (roughly 0.3 Mbit/s) for the longer transfers, with about 28 to 31 ms between chunks. The round trip was about 8 ms as seen at the sender.

The file called `png_file` is actually a JPEG, and `wireshark_image1` is a WebP image.

## Security findings

- **No confidentiality.** Data is sent in clear text. Every file was rebuilt from the capture files alone.
- **No integrity or authentication.** There is no checksum, and the receiver trusts any packet with the sender's source address, which can be spoofed.
- **Easy to detect.** Echo payloads are 1004 bytes (a default Linux `ping` uses 56), ICMP id and sequence stay at 0, the data has high entropy (about 7.7 bits per byte), and about 38 packets arrive in one second.
- **Mitigations.** Limit the size and rate of ICMP echo at the firewall, add intrusion-detection rules for large echo payloads, and, if such a channel were wanted, add a SHA-256 check and encrypt and authenticate the data (for example AES-GCM).

## Known limitations

- The original receivers never reset their state and never stop, so each run ends with Ctrl+C and a second file needs a restart.
- No timeout, retransmission or checksum: a lost chunk leaves the receiver waiting forever.
- The echo reply comes from the kernel, so it shows the packet arrived, not that the script processed it.
- `txt_receiver_file.py` saves `received.txt` but prints "Saved as received_txt.txt".
- `message_receiver.py` filters on `dst host 10.46.216.204`, which matches its own outgoing replies. `dst host 10.46.216.99` is the correct filter.

## Future work

- Merge the receivers into one script that takes the output name as an argument.
- Add acknowledgements and retransmission at the application level.
- Encrypt and authenticate the chunks and compare the capture with the clear-text version.
- Repeat the test with UDP and TCP, and measure speed over a real Wi-Fi link with 802.11 monitor-mode captures.

## Author

Irfaan

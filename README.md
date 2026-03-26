# Lab 2: TCP vs. UDP File Transfer

## Overview

In this lab, you will implement a file transfer suite in Java — a **client** that sends files to a **server** — and use it to benchmark the performance differences between **TCP** and **UDP**. You will measure transfer speed and data accuracy, then reflect on the fundamental trade-offs each protocol makes.

> **Note:** This lab will be done in pairs of 2 (you'll work on one repository with 2 separate branches and will be on the same orion machine will give details later in this README). The only operation supported is **PUT** (sending a file to the server). There is no GET/retrieval.

---

## Learning Objectives

By the end of this lab, you should be able to:

- Implement a TCP client–server application using Java's `Socket` and `ServerSocket`.
- Implement a UDP client–server application using Java's `DatagramSocket` and `DatagramPacket`.
- Design a simple binary protocol for exchanging structured metadata alongside raw file data.
- Measure and compare transfer throughput and data integrity between TCP and UDP.
- Explain the core trade-offs between reliability and performance at the transport layer.

---

## Quick thing to know

### MD5 Checksum
An MD5 hash produces a 128-bit (16-byte) digest of arbitrary data. If even a single byte changes, the hash changes. By computing MD5 on both ends and comparing, you can verify that the received file is bit-for-bit identical to the original. A mismatch is evidence of data corruption or loss — expect to see this with UDP.


---

## Usage

### Build

```bash
javac Server.java Client.java
```
or 
```bash
javac *.java
```

### Run the Server

```bash
java Server <port> [download-dir]

# Example: listen on 9898, save files to ./received/
java Server 9898 ./received/
```

The `[download-dir]` should be optional. If omitted, files are saved to the current directory. The server listens on **both TCP and UDP** on the same port number simultaneously using **threads**.

### Run the Client

```bash
java Client <host:port> <tcp|udp> <file-path>

# Send via TCP
java Client localhost:9898 tcp /path/to/photo.jpg

# Send same file via UDP
java Client localhost:9898 udp /path/to/photo.jpg

# Send to a remote host
java Client orion12:9898 tcp /tmp/bigfile.bin
```

Since we'll be testing using large files you'll be accessing the school's orion servers 
Here is how you can get access to those servers
1. ssh into stargate as you would normally

2. 
```bash
# the # means any orion server number, just know you and your partner needs to be on the same server for this lab to work
ssh orion##
```

3. The files you will be using to send will be located in ```orion##:/bigdata/datasets``` 
```bash
# you can access it by after getting on orion server do 
cd /bigdata/datasets
```

## MAKE SURE TO **NOT** PUSH THE TEXT FILE ONTO YOUR GITHUB IT SHOULD **ONLY** CONTAIN .java AND .md FILES THERE WILL BE A PENALTY FOR EACH INVALID FILE OTHER THAN IDE SETTINGS

---

## Implementation Guide

Both you and your partner must implement **two files**: `Client.java` and `Server.java`.

### Server

The server should:

1. Parse command-line arguments (`port`, optional `download-dir`).
2. Create the download directory if it does not exist.
3. Spawn two concurrent listener threads — one for TCP, one for UDP — both on the same port.
4. For each incoming **TCP** connection, spawn a thread to handle it concurrently.

**TCP client handler:**
1. Read the file name (prefixed with its length as a 4-byte int) and file size (8-byte long).
2. Send a 1-byte acknowledgment (`true`) to allow the client to begin streaming.
3. Receive exactly `fileSize` bytes, writing them to disk and feeding them into an MD5 digest simultaneously.
4. Read the 16-byte MD5 checksum sent by the client.
5. Compare it to the locally computed digest.
6. Send back: checksum match result, bytes received, throughput, and elapsed time.

**UDP transfer handler:**
1. Wait for a **META** datagram (type byte `0`): contains name length, name, file size, and total chunk count.
2. Send a 1-byte ACK.
3. Receive **DATA** datagrams (type `1`) keyed by sequence number. Store each chunk in a `HashMap<Integer, byte[]>`.
4. Receive a **FIN** datagram (type `2`) containing the MD5 checksum.
5. Reassemble the file in sequence-number order. For any **missing** chunk, zero-fill the corresponding region (this makes checksum mismatches observable).
6. Send back: checksum match, chunks received, total chunks, throughput, elapsed time.

### Client

The client should:

1. Parse command-line arguments (`host:port`, `tcp|udp`, `file-path`).
2. Verify the file exists.
3. Dispatch to the appropriate send function.

**`sendTCP()`:**
1. Open a `Socket`, send metadata (name length + name + file size), flush.
2. Wait for the server's ACK.
3. Stream the file in a loop; feed each chunk into the MD5 digest while sending.
4. Send the 16-byte digest.
5. Read the server's result and print a formatted summary.

**`sendUDP()`:**
1. Open a `DatagramSocket`.
2. Construct and send a META packet.
3. Wait for the ACK (use a timeout; error if server is unreachable).
4. Read the file in `4096`-byte chunks. For each chunk: update the MD5 digest, build a DATA packet (`type=1`, sequence number, data length, data bytes), send it.
5. Wait for the server's result datagram (use a timeout).
6. Print a formatted summary including **packet loss rate** and checksum result.

### Recommended Extras (not required, but encouraged)

- Check whether the destination file already exists before overwriting.
- Check available disk space before accepting a transfer.
- Add a `--delay` flag to the client to artificially slow UDP sending and stress-test loss behavior. (If you want to play around with UDP vs TCP)

---

## Deliverables
The project structure will look like 

.
├── Client.java 
├── README.md
└── Server.java

---

## Rubric 
This lab will be out of 10 points in total 

- 2 points for Client code (make sure to include javadoc!)
- 2 points for Server code (make sure to include javadoc!)
- 2 points for proper github branching from both you and your partner (you can work on the lab together, the written code needs to be done individually though) 
- 2 points | Screenshot of both you and your partner's server and client communication (basically showing that code is indeed working and properly sending / receiving files)
- 2 points | Writing analysis in a separate markdown file showing what you have observed after running the 3 text files 
    - the 3 text files are "large-log.txt" "mega-super-log.txt" "venti-frappuchino-log.txt"
    - It is recommended that you track how much bytes / small packets the Server received and the time it took to transfer the data (track it in Client side and have Server sends its information to the client) 
    - You can print it out from your Client.java and copy paste it onto a markdown file 

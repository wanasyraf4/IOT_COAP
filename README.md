
# Pico + SIM800L → CoAP → PostgreSQL Pipeline

This repository demonstrates how to stream sensor data from a Raspberry Pi Pico over GPRS using a SIM800L module and CoAP (UDP), then insert it into a PostgreSQL database via a Python CoAP server.

## Architecture Overview

![image](https://github.com/user-attachments/assets/26c633a0-ec4f-47c4-af1c-f2eefe9f47bc)


## Contents

- `pico_coap_client.c` – Pico firmware in C using UART/AT to:
  1. Bring up GPRS  
  2. Open a UDP socket  
  3. Send a CoAP POST with JSON payload  
  4. Close the socket & sleep  

- `coap_server.py` – Python CoAP server using **aiocoap** to:
  1. Listen on UDP port 5683  
  2. Parse incoming JSON  
  3. Insert into PostgreSQL via **psycopg2**  

## Getting Started

1. **Pico Firmware**  
   - Install the Pico SDK and toolchain.  
   - Edit `pico_coap_client.c` with your APN and CoAP server IP.  
   - Build and flash to your Pico.

2. **CoAP Server**  
   - Create a PostgreSQL database and user.  
   - Install Python dependencies:  
     ```bash
     pip install aiocoap psycopg2
     ```  
   - Edit `coap_server.py` with your DB credentials.  
   - Run the server:  
     ```bash
     python coap_server.py
     ```

3. **Run & Test**  
   - Power your Pico + SIM800L with battery.  
   - Observe inserted records in PostgreSQL:
     ```sql
     SELECT * FROM sensor_data ORDER BY ts DESC;
     ```

4. **Savings per message**

Time-on-air reduction: (0.19 – 0.028)/0.19 ≈ 85 % <br/>

Energy reduction: (0.35 – 0.052)/0.35 ≈ 85 % <br/>

What this means for your battery <br/>
– If you send one reading per minute, you save ~0.30 J per message → ~432 J per day. <br/>
– A 2500 mAh Li-ion cell (≈33 000 J usable) gets ~1.3 % more life per day. <br/>
– At higher rates (e.g. every 10 s), savings scale linearly, so you could double or triple run-time. <br/>

| Protocol      | On-air bytes | On-air time (s)           | Energy per message (J)            |
| ------------- | ------------ | ------------------------- | --------------------------------- |
| HTTP/TCP POST | \~470 B¹     | 470 × 8 / 20 000 ≈ 0.19 s | 0.5 A × 3.7 V × 0.19 s ≈ 0.35 J   |
| CoAP/UDP POST | \~ 70 B²     | 70 × 8 / 20 000 ≈ 0.028 s | 0.5 A × 3.7 V × 0.028 s ≈ 0.052 J |



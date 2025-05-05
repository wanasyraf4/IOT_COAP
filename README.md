
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




# Step1: Part 1: Pico firmware in C (CoAP/UDP via SIM800L)
``` cpp
// pico_coap_client.c
#include <stdio.h>
#include <string.h>
#include "pico/stdlib.h"
#include "hardware/uart.h"
#include "hardware/gpio.h"

// UART configuration
#define UART_ID        uart0
#define BAUD_RATE      115200
#define UART_TX_PIN    0
#define UART_RX_PIN    1

// Your SIM800L and server settings
const char *APN        = "your.apn.here";
const char *SERVER_IP  = "203.0.113.10"; // CoAP server IPv4
const uint16_t SERVER_PORT = 5683;

// Simple AT-command helper
void at_cmd(const char *cmd, uint32_t wait_ms) {
    uart_puts(UART_ID, cmd);
    uart_puts(UART_ID, "\r\n");
    sleep_ms(wait_ms);
    // (Optionally read+print response here)
}

int main() {
    stdio_init_all();
    // init UART
    uart_init(UART_ID, BAUD_RATE);
    gpio_set_function(UART_TX_PIN, GPIO_FUNC_UART);
    gpio_set_function(UART_RX_PIN, GPIO_FUNC_UART);

    // 1. Basic AT check
    at_cmd("AT", 1000);

    // 2. Bring up GPRS
    at_cmd("AT+SAPBR=3,1,\"Contype\",\"GPRS\"", 500);
    char buf[128];
    sprintf(buf, "AT+SAPBR=3,1,\"APN\",\"%s\"", APN);
    at_cmd(buf, 500);
    at_cmd("AT+SAPBR=1,1", 3000);
    at_cmd("AT+SAPBR=2,1", 1000);

    while (true) {
        // 3. Open UDP socket
        sprintf(buf, "AT+CIPSTART=\"UDP\",\"%s\",\"%u\"", SERVER_IP, SERVER_PORT);
        at_cmd(buf, 3000);

        // 4. Build a minimal CoAP POST to /data with JSON payload
        const uint8_t coap_msg[] = {
            0x40,       // ver=1, type=CON, tkl=0
            0x02,       // code=POST
            0x12,0x34,  // message ID
            // option: Uri-Path "data" (number=11, delta=11, len=4)
            0xB4, 'd','a','t','a',
            0xFF,       // payload marker
            // JSON payload:
            '{','"','s','e','n','s','o','r','"',':','"','t','e','m','p','"',',',
            '"','v','a','l','u','e','"',':','2','3','.','5','}'
        };
        int msglen = sizeof(coap_msg);

        // 5. Tell SIM800L how many bytes we’ll send
        sprintf(buf, "AT+CIPSEND=%d", msglen);
        at_cmd(buf, 500);

        // 6. Send raw CoAP bytes
        uart_write_blocking(UART_ID, coap_msg, msglen);
        sleep_ms(1000);  // give it time

        // 7. Close UDP
        at_cmd("AT+CIPCLOSE", 500);

        // 8. Deep sleep or wait before next send
        sleep_ms(60000);  // e.g. once per minute
    }

    return 0;
}
```
Compile with the Pico SDK and flash to your board. Fill in your APN and server IP. This will: <br/>

1. Bring up GPRS. <br/>

2. Open a UDP “socket” to your CoAP server. <br/>

3. Send a confirmable CoAP POST to /data carrying a JSON body. <br/>

4. Close the connection and wait. <br/>


# Part 2: CoAP server + PostgreSQL insertion (Python)

```python
# coap_server.py
import asyncio
import json
import psycopg2
from aiocoap import resource, Context, Message
from aiocoap.numbers.codes import Code

# 1. Set up your database connection
conn = psycopg2.connect(
    dbname="yourdb",
    user="dbuser",
    password="dbpass",
    host="localhost",
    port=5432
)
cur = conn.cursor()
cur.execute("""
CREATE TABLE IF NOT EXISTS sensor_data (
    id SERIAL PRIMARY KEY,
    ts TIMESTAMPTZ DEFAULT now(),
    sensor_type TEXT,
    value REAL
)
""")
conn.commit()

# 2. Define a CoAP resource at /data
class DataResource(resource.Resource):
    async def render_post(self, request):
        # parse JSON payload
        payload = request.payload.decode('utf-8')
        data = json.loads(payload)
        sensor = data.get('sensor')
        value  = data.get('value')

        # insert into Postgres
        cur.execute(
            "INSERT INTO sensor_data(sensor_type, value) VALUES (%s, %s)",
            (sensor, value)
        )
        conn.commit()

        # reply with 2.04 Changed
        return Message(code=Code.CHANGED, payload=b"OK")

# 3. Run the CoAP server
async def main():
    root = resource.Site()
    root.add_resource(['data'], DataResource())
    # bind on all interfaces, port 5683
    await Context.create_server_context(root, bind=('0.0.0.0', 5683))
    # keep running
    await asyncio.get_running_loop().create_future()

if __name__ == "__main__":
    asyncio.run(main())
```

How it works: <br/>

The CoAP server listens on UDP port 5683. <br/>

When it gets a POST to /data, it parses the JSON. <br/>

It inserts the reading into sensor_data with a timestamp. <br/>

It returns a CoAP 2.04 (Changed) response. <br/>

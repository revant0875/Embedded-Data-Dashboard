/* ================= main.c ================= */
#include "webserver.h"
#include "serial_reader.h"
#include "json_formatter.h"

int main() {
    serial_init("/dev/ttyUSB0", 9600);
    run_webserver();
    return 0;
}

/* ============== webserver.h ============== */
#ifndef WEBSERVER_H
#define WEBSERVER_H

void run_webserver(void);

#endif // WEBSERVER_H

/* ============== webserver.c ============== */
#include "webserver.h"
#include "mongoose.h"
#include "json_formatter.h"
#include <stdio.h>

static const char *s_listen_on = "http://0.0.0.0:8000";
static const char *s_web_root = "web_root";

static void handle_request(struct mg_connection *c, int ev, void *ev_data, void *fn_data) {
    if (ev == MG_EV_HTTP_MSG) {
        struct mg_http_message *hm = (struct mg_http_message *) ev_data;

        if (mg_http_match_uri(hm, "/api/data")) {
            char json[256];
            get_serial_data_as_json(json, sizeof(json));
            mg_http_reply(c, 200, "Content-Type: application/json\r\n", "%s", json);
        } else {
            mg_http_serve_dir(c, hm, s_web_root);
        }
    }
    (void) fn_data;
}

void run_webserver(void) {
    struct mg_mgr mgr;
    mg_mgr_init(&mgr);
    mg_http_listen(&mgr, s_listen_on, handle_request, &mgr);

    printf("Starting web server on %s\n", s_listen_on);
    for (;;) {
        mg_mgr_poll(&mgr, 1000);
    }
    mg_mgr_free(&mgr);
}

/* ============ serial_reader.h ============ */
#ifndef SERIAL_READER_H
#define SERIAL_READER_H

#include <stddef.h>

void serial_init(const char *port, int baudrate);
int read_serial_data(char *buffer, size_t size);

#endif // SERIAL_READER_H

/* ============ serial_reader.c ============ */
#include "serial_reader.h"
#include <fcntl.h>
#include <termios.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

static int serial_fd = -1;

void serial_init(const char *port, int baudrate) {
    struct termios tty;
    serial_fd = open(port, O_RDWR | O_NOCTTY | O_SYNC);
    if (serial_fd < 0) {
        perror("serial open");
        return;
    }

    memset(&tty, 0, sizeof tty);
    if (tcgetattr(serial_fd, &tty) != 0) {
        perror("tcgetattr");
        return;
    }

    cfsetospeed(&tty, B9600);
    cfsetispeed(&tty, B9600);

    tty.c_cflag = (tty.c_cflag & ~CSIZE) | CS8;
    tty.c_iflag &= ~IGNBRK;
    tty.c_lflag = 0;
    tty.c_oflag = 0;
    tty.c_cc[VMIN]  = 0;
    tty.c_cc[VTIME] = 5;

    tty.c_iflag &= ~(IXON | IXOFF | IXANY);
    tty.c_cflag |= (CLOCAL | CREAD);
    tty.c_cflag &= ~(PARENB | PARODD);
    tty.c_cflag &= ~CSTOPB;
    tty.c_cflag &= ~CRTSCTS;

    if (tcsetattr(serial_fd, TCSANOW, &tty) != 0)
        perror("tcsetattr");
}

int read_serial_data(char *buffer, size_t size) {
    if (serial_fd < 0) return -1;
    return read(serial_fd, buffer, size);
}

/* ========== json_formatter.h ========== */
#ifndef JSON_FORMATTER_H
#define JSON_FORMATTER_H

void get_serial_data_as_json(char *json, int max_len);

#endif // JSON_FORMATTER_H

/* ========== json_formatter.c ========== */
#include "json_formatter.h"
#include "serial_reader.h"
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

void get_serial_data_as_json(char *json, int max_len) {
    char buffer[128] = {0};
    int n = read_serial_data(buffer, sizeof(buffer)-1);

    float temp = 0.0, hum = 0.0;
    if (n > 0) {
        sscanf(buffer, "TEMP:%f,HUM:%f", &temp, &hum);
    }
    snprintf(json, max_len, "{\"temperature\": %.2f, \"humidity\": %.2f}", temp, hum);
}

/* ============== Makefile ============== */
CC = gcc
CFLAGS = -Wall -Wextra -I.
SRC = main.c webserver.c serial_reader.c json_formatter.c mongoose.c
BIN = dashboard

all:
	$(CC) $(CFLAGS) $(SRC) -o $(BIN)

clean:
	rm -f $(BIN)

/* ============ web_root/index.html ============ */
<!DOCTYPE html>
<html>
<head>
    <title>Embedded Dashboard</title>
    <meta charset="utf-8">
    <style>
        body { font-family: Arial; margin: 20px; }
    </style>
</head>
<body>
    <h1>Device Data</h1>
    <div id="data"></div>
    <script src="script.js"></script>
</body>
</html>

/* ============ web_root/script.js ============ */
function fetchData() {
    fetch('/api/data')
        .then(res => res.json())
        .then(data => {
            document.getElementById('data').innerHTML =
                `<pre>${JSON.stringify(data, null, 2)}</pre>`;
        });
}

setInterval(fetchData, 1000);
fetchData();

/* ============ README.md ============ */
# Embedded Dashboard in C (Full C Project)

### Features
- Reads data from serial port `/dev/ttyUSB0`
- Parses format like `TEMP:25.5,HUM:60`
- Serves JSON API and static HTML/JS from embedded C web server (Mongoose)

### Build & Run
```bash
# Download Mongoose
curl -O https://raw.githubusercontent.com/cesanta/mongoose/master/mongoose.c
curl -O https://raw.githubusercontent.com/cesanta/mongoose/master/mongoose.h

make
./dashboard
```

### Access Dashboard
Open [http://localhost:8000](http://localhost:8000)

### Output Format
```json
{
  "temperature": 25.5,
  "humidity": 60.0
}
```

---

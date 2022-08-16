---
layout: post
title: Creating a web server on Raspberry Pi Pico W using Pico SDK and lwIP
subtitle: It's super fast!
date: 2022-08-15 14:21 +0200
cover-img: /assets/img/2022-08-15-creating-a-web-server-on-raspberry-pi-pico-w-using-pico-sdk-and-lwip/cover.jpg
gh-repo: krzmaz/pico-w-webserver-example
gh-badge: [follow, star, fork]
tags: [pico, pico_w, cmake, lwip]

---
*If you want to skip the boilerplate jump [here](#no_boilerplate)*
## In this post
{:.no_toc}
1. TOC
{:toc}
## Background  
I never really looked into the first Raspberry Pi Pico. I get that the RP2040 microcontroller is a nice alternative to some members of the STM32 family, but I haven't been using those as well.  
Any development board that doesn't come with wireless connectivity features is not likely to pique my interest. Most of my projects are about automating some part of my life and for the most part that needs a wireless data exchange. Some projects can even be done with a dev board alone, without connecting anything to it - one example being [LinakDeskEsp32Controller](https://github.com/krzmaz/LinakDeskEsp32Controller)

I ended up using ESP8266 and ESP32 in all my recent projects. They are great because of their high availability, the big community and in the ESP32's case - the use of industry standard: FreeRTOS.  
However the downside is their unique Xtensa architecture which makes dealing with the toolchain a bit of a pain. 
## New and shiny ESP32 alternative from Raspberry Pi!
Now, with the Pico W having a wireless connectivity module on the dev board, finally I can have another alternative to consider. Unfortunately as of right now there seems to be no way of using Bluetooth with the wireless module. We have to wait for the drivers to be developed.  
The Raspberry Pi foundation is promoting MicroPython as the easy way to develop on Pi Pico. It's great for quick prototyping or introducing people to embedded world, but when I saw in [Jeff Geerling's video](https://www.youtube.com/watch?v=VEWdxvIphnI) that a simple static web page request takes 230ms, I wanted to find out how much quicker will the [Pico C SDK](https://github.com/raspberrypi/pico-sdk) be.

## Technologies used
### CMake
CMake is a very powerful build system. Using it for a tiny "hello world" project can feel like not using the right tool for the job. But it's the industry standard and getting used to it will pay off more likely than not. Also, this is going to make your project more scalable and portable than putting all the compiler and linker invocations in a shell script (been there, done that 😞)  
### lwIP
Pico SDK creators have wisely chosen not to implement their own TCP stack, but rather use a known open-source solution.  
As per [Wikipedia](https://en.wikipedia.org/wiki/LwIP):  
>lwIP (lightweight IP) is a widely used open-source TCP/IP stack designed for embedded systems. (...) The focus of the lwIP network stack implementation is to reduce resource usage while still having a full-scale TCP stack. This makes lwIP suitable for use in embedded systems with tens of kilobytes of free RAM and room for around 40 kilobytes of code ROM.

## Project overview
{: #no_boilerplate}
The [web server implementation](https://github.com/krzmaz/pico-w-webserver-example) consists of a couple of main components:
<!-- poor man's TOC as kramdown doesn't allow generating two -->
1. [CMake logic](#CMake)
1. [HTML files to be served from the Pico W](#HTML)
1. [Code for configuring and running the lwIP http server](#Code)

### CMake logic
{: #CMake}
First of all, we need to download the `pico_sdk_import.cmake` script that will help us setup the SDK. To do that, there is a `get_pico_sdk_import_cmake` function in `cmake/utils.cmake`. The script download location is added to `.gitignore` file so that it isn't added to the repo.  
After getting the script, we need to `include()` it before calling `project()` and then `pico_sdk_init()` in the top level `CMakeLists.txt`.

Another external dependency is the `makefsdata` perl script. Its role will be described in the next section. As for CMake logic, we need to `file(DOWNLOAD` it and run it via `execute_process()`

### HTML files to be served from the Pico W
{: #HTML}
lwIP supports providing HTML files in a filesystem structure that we can use in URLs when connecting to the created server. Those files can be found in `src/fs` folder. For now, there is only the most basic `index.html` file that will be the root of the server, and `ssi.shtml` that shows the ability to serve dynamic content using [Server Side Includes](https://en.wikipedia.org/wiki/Server_Side_Includes).  
The "filesystem" data is provided in `fsdata.c` file by default, but storing HTTP data embedded into C makes editing it a pain. For quickly generating the C file, lwIP provides the `makefsdata` perl script which, when pointed to a directory, creates the `fsdata` file containing the whole directory structure. Since I didn't want to store two versions of the same data, I'm keeping the HTML files in git and generating the C file during build time. The script creates `fsdata.c` which I'm renaming to `my_fsdata.c` to match the define of `HTTPD_FSDATA_FILE` in `lwipopts.h`. Leaving the name as is would require forcing the build system to ignore the default [fsdata.c](https://github.com/lwip-tcpip/lwip/blob/master/src/apps/http/fsdata.c) file in the lwIP sources.

Additionally I noticed that due to limited file extension matching in the `makefsdata` script, my `ssi.shtml` page was generated with `text/plain` content type instead of `text/html` making it not being rendered by the browser. I created a fork with a patch for that, used it in CMake of the example, and created [lwip#15](https://github.com/lwip-tcpip/lwip/pull/15).

### Code for configuring and running the lwIP http server
{: #Code}
I don't want to go into too much detail. If you want too much detail, you can explore [lwIP examples](https://github.com/lwip-tcpip/lwip/tree/master/contrib/examples/httpd).  
For the simplest case of hosting a static HTTP file using [my example](https://github.com/krzmaz/pico-w-webserver-example), you need to add your file to the `fs` folder, and run the `build.sh` script.

Here's the simplest `main.c` file:
```c
// for httpd_init
#include "lwip/apps/httpd.h"
// for cy43_* functions
#include "pico/cyw43_arch.h"
// for #define's that configure lwIP stack (enable what we need and not more)
#include "lwipopts.h"


int main() {
    // needed to get the RP2040 chip talking with the wireless module
    if (cyw43_arch_init()) {
        return 1;
    }
    // we'll be connecting to an access point, not creating one
    cyw43_arch_enable_sta_mode();
    // WiFi credentials are taken from cmake/credentials.cmake
    // create it based on cmake/credentials.cmake.example if you haven't already!
    if (cyw43_arch_wifi_connect_timeout_ms(WIFI_SSID, WIFI_PASSWORD, CYW43_AUTH_WPA2_AES_PSK, 30000)) {
        return 1;
    }
    // let lwIP do it's magic
    httpd_init();
    // loop forever
    for (;;) {}
}
```
Nice, huh? What was it, 20 lines of code? That's shorter than the MicroPython version!*  
**If you don't count all the CMake code of course*

#### SSI
For using SSI to dynamically change the page content we need to define three things:
1. SSI tags - identifiers that you can use in the html code to note where your dynamic data will go:
    - In your SHTML file:

    ```html
    <body> <h1>Pico W</h1>
        <p><!--#Hello--> Times <!--#counter-->!</p>
        <p>GPIO26 voltage is: <!--#GPIO-->!</p>
    </body>
    ```

    - In code:

    ```c
    const char * ssi_example_tags[] = {
    "Hello",
    "counter",
    "GPIO"
    };
    ```
2. SSI handler - a piece of code that knows what data to inject for each tag
```c
u16_t ssi_handler(int iIndex, char *pcInsert, int iInsertLen) {
    size_t printed;
    switch (iIndex) {
        case 0: /* "Hello" */
            printed = snprintf(pcInsert, iInsertLen, "Hello user number %d!", rand());
            break;
        // more cases here, see src/ssi.h
        default: /* unknown tag */
            printed = 0;
            break;
    }
    LWIP_ASSERT("sane length", printed <= 0xFFFF);
    return (u16_t)printed;
}
```
3. `http_set_ssi_handler` function call that registers the tags and the handler
```c
const size_t tags_number = LWIP_ARRAYSIZE(ssi_example_tags);
http_set_ssi_handler(ssi_handler, ssi_example_tags, tags_number);
```

And that's it! Now you have a server that's serving dynamic content!

#### Code placement
You might've noticed these things in `src/ssi.h`:
```c
__not_in_flash("httpd")
__time_critical_func(ssi_handler)
```
Those are perfect examples of premature optimizations - section attribute macros for placement not in flash (i.e in RAM) to avoid possible flash latency.  
You can read more about it in [SDK doxygen documentation](https://raspberrypi.github.io/pico-sdk-doxygen/group__pico__platform.html).


## Performance of the server
It's fast. Requests take usually below 15ms in my local network, which considering the ping times means that handling the response takes around 10ms!  
I wanted to compare the fully static page with the dynamic one that has a counter and an analog voltage read. The differences are hard to measure at these speeds (especially considering the fluctuation of ping times due to varying network conditions), but the dynamic page was usually about 1-10% slower.  
Below are the test results using ApacheBench, command used
```bash
ab -n 1000 -c 1 <URL>
```
<details><summary markdown="span">Static page results</summary>
```
Server Software:        lwIP/pre-0.6
Server Hostname:        192.168.0.204
Server Port:            80

Document Path:          /
Document Length:        137 bytes

Concurrency Level:      1
Time taken for tests:   12.802 seconds
Complete requests:      1000
Failed requests:        0
Total transferred:      236000 bytes
HTML transferred:       137000 bytes
Requests per second:    78.11 [#/sec] (mean)
Time per request:       12.802 [ms] (mean)
Time per request:       12.802 [ms] (mean, across all concurrent requests)
Transfer rate:          18.00 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        4    7   3.1      6      47
Processing:     3    6   3.1      5      38
Waiting:        3    6   3.1      5      38
Total:          8   13   4.5     12      54

Percentage of the requests served within a certain time (ms)
  50%     12
  66%     13
  75%     14
  80%     14
  90%     16
  95%     18
  98%     28
  99%     35
 100%     54 (longest request)
```
</details>

<details><summary markdown="span">SSI page results</summary>
```
Server Software:        lwIP/pre-0.6
Server Hostname:        192.168.0.204
Server Port:            80

Document Path:          /ssi.shtml
Document Length:        207 bytes

Concurrency Level:      1
Time taken for tests:   12.952 seconds
Complete requests:      1000
Failed requests:        990
   (Connect: 0, Receive: 0, Length: 990, Exceptions: 0)
Total transferred:      308411 bytes
HTML transferred:       209411 bytes
Requests per second:    77.21 [#/sec] (mean)
Time per request:       12.952 [ms] (mean)
Time per request:       12.952 [ms] (mean, across all concurrent requests)
Transfer rate:          23.25 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        4    7   3.3      6      50
Processing:     4    6   2.4      5      39
Waiting:        4    6   2.4      5      39
Total:          9   13   4.5     12      71

Percentage of the requests served within a certain time (ms)
  50%     12
  66%     13
  75%     14
  80%     14
  90%     16
  95%     18
  98%     23
  99%     37
 100%     71 (longest request)
```
</details>
Ping results
```
--- 192.168.0.204 ping statistics ---
1000 packets transmitted, 1000 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 3.232/5.366/40.522/3.208 ms
```

## Next steps
Example from this post is using bare metal (`NO_SYS`) lwIP. I'd like to see how it compares to a more real-life scenario of using lwIP with FreeRTOS.  
Another thing to explore is running [Arduino core](https://github.com/earlephilhower/arduino-pico) on the Pico W. This would ease the usage of many existing Arduino libraries, but probably at the cost of a performance hit.
Lastly, I want to compare the performance against the alternatives - ESP8266 and ESP32

I'll try to update this post with links to new posts once I do the experiments:  
- Comparing performance of bare metal lwIP vs FreeRTOS on Pico W
- Checking the performance cost of using Arduino cor on Pico W
- Comparing all the different variants of Pico W web server with alternatives - ESP8266 and ESP32

## Useful links

- Code of the discussed example project can be found on github:  
    <https://github.com/krzmaz/pico-w-webserver-example>  
- lwIP documentation and examples:
    - <https://www.nongnu.org/lwip/2_1_x/group__httpd.html>
    - <https://github.com/lwip-tcpip/lwip/tree/master/contrib/examples/httpd>



---
*The cover photo shows a young stray cat that my friends in Germany have just took in when I was visiting them*
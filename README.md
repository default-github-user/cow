# COW (Climb Over the Wall) proxy

COW is an HTTP proxy server that simplifies walking through walls. It can automatically detect walled sites and use a secondary proxy only for those sites.

[English README] (README-en.md).

Current version: 0.9.8 [CHANGELOG] (CHANGELOG)
[! [Build Status] (https://travis-ci.org/cyfdecyf/cow.png?branch=master)] (https://travis-ci.org/cyfdecyf/cow)

** Welcome to develop and send pull request in develop branch:) **

## Features

The design goal of COW is automation. Ideally, users do not need to worry about which websites are unreachable, and can directly connect to the website without slowing down the access speed by using a secondary proxy.

-As an HTTP proxy, it can be provided to mobile devices; if deployed on a domestic server, it can be used as an APN proxy
-Support HTTP, SOCKS5, [shadowsocks] (https://github.com/clowwindy/shadowsocks/wiki/Shadowsocks-%E4%BD%BF%E7%94%A8%E8%AF%B4%E6%98%8E ) And cow itself as secondary agents
  -Multiple secondary agents can be used to support simple load balancing
-Automatically detect whether the website is walled or not, only use secondary agents for walled websites
-Automatically generate PACs with directly connected websites, bypassing COW when visiting those websites
  -Built-in [commonly accessible websites] (site_direct.go), such as domestic social networking, video, banking, e-commerce websites (can be added manually)

# Quick start

installation:

-** OS X, Linux (x86, ARM): ** execute the following command (also available for update)

        curl -L git.io/cow | bash

  -The environment variable `COW_INSTALLDIR` can specify the installation path. If the environment variable is not a directory, ask the user
  -All binaries are compiled on OS X. If the ARM version may not work, please download [Go ARM] (https://storage.googleapis.com/golang/go1.6.2.linux-amd64.tar.gz) from Source installation
-** Windows: ** Download from [release page] (https://github.com/cyfdecyf/cow/releases)
-Go users can install `go get github.com / cyfdecyf / cow` from source

Edit `~ / .cow / rc` (Linux) or` rc.txt` (Windows). A simple configuration example is as follows:

    Lines beginning with # are comments and will be ignored
    # Local HTTP proxy address
    # Please fill in this address when configuring HTTP and HTTPS proxy
    # If there is an option to use the proxy for all protocols when configuring the proxy, and you do not know the meaning of this option, please check
    # Or fill in http://127.0.0.1:7777/pac in automatic proxy configuration
    listen = http://127.0.0.1:7777

    # SOCKS5 Secondary Agent
    proxy = socks5: //127.0.0.1: 1080
    # HTTP secondary proxy
    proxy = http://127.0.0.1:8080
    proxy = http: // user: password@127.0.0.1: 8080
    # shadowsocks secondary proxy
    proxy = ss: // aes-128-cfb: password@1.2.3.4: 8388
    # cow secondary agent
    proxy = cow: // aes-128-cfb: password@1.2.3.4: 8388

The secondary agent using the cow protocol needs to install COW on a foreign server and use the following configuration:

    listen = cow: // aes-128-cfb: password@0.0.0.0: 8388

After the configuration is complete, start COW and configure the agent to use it.

# Detailed instructions

The configuration file is `~ / .cow / rc` on Unix systems, and the` rc.txt 'file in the directory where COW is located on Windows. ** [Sample Configuration] (doc / sample-config / rc) contains all options and detailed instructions **, it is recommended to download and modify.

Start COW:

-Unix system executes `cow &` on the command line (if COW is not in the directory where `PATH` is located, execute` ./cow & `)
  -[Linux startup script] (doc / init.d / cow), please refer to the comments on how to use it (Debian test passed, other Linux distributions should also work)
-Windows
  -Double-click `cow-taskbar.exe`, hide to tray execution
  -Double-click `cow-hide.exe` to hide execution as background program
  -Both of them will start `cow.exe`

The PAC url is `http: // <listen address> / pac`. You can also set your browser's HTTP / HTTPS proxy to` listen address` to make all websites accessible through COW.

** Use PAC for better performance, but if a website in PAC changes from direct connection to blocked, the browser will still try direct connection. In this case, you can temporarily not use PAC but always use HTTP proxy to let COW learn the new blocked website. **

Command line options can override options in some configuration files, turn on debug / request / reply logging, and execute `cow -h` for more information.

## Manually specify walls and direct sites

** Under normal circumstances, there is no need to manually specify the wall and directly connected websites. This function is only for special situations and performance optimization. **

The `blocked` and` direct` in the directory where the configuration file is located can specify walled and directly connected websites (the host in `direct` will be added to the PAC).
The file names under Windows are `blocked.txt` and` direct.txt`.

-One domain name or host name per line (COW will check if the host name is in the list before checking the domain name)
  -Second-level domain names such as `google.com` are equivalent to` * .google.com`
  -Third-level domain names under `com.hk`,` edu.cn` and other second-level domain names are treated as second-level domain names. For example, `google.com.hk` is equivalent to` * .google.com.hk`
  -Exact match for other third-level domain names / hostnames, such as `plus.google.com`

# technical details

## Visit Website History

COW records the number of frequently accessed websites being accessed by walls and direct connections in a `stat` json file in the directory where the configuration file is located.

-** For unknown websites, try to connect directly first, retry the request with a secondary proxy after failure, and try directly after 2 minutes **
  -Built-in [common wall website] (site_blocked.go), reducing the time required to detect wall (can be added manually)
-The corresponding host will be added to the PAC after a certain number of direct access successes
-The host will be accessed directly with a secondary proxy after being walled a certain number of times
  -To avoid misjudgment, direct access will be tried again with a certain probability
-The host will be automatically deleted if there is no access for a period of time (to avoid infinite growth of the `stat` file)
-Built-in website list and user-specified websites will not appear in the statistics file

## COW How to detect walled sites

COW considers the following errors to be wall-failing:

-Server connection is reset (connection reset)
-Create connection timeout
-Server read operation timed out

Whether it is a normal HTTP GET request or a CONNECT request, the COW will automatically retry the request after a failure. (If there is already content sent back to the client, it will not be retried but will be disconnected directly.)

It is generally more reliable to judge the wall by resetting the connection, and it is not reliable to time out. COW will try to estimate the appropriate timeout interval every half a minute to avoid treating directly connected websites as wall because of timeouts.
After COW is detected in the default configuration, trying to connect directly again in two minutes is also to avoid misjudgment.

If the timeout auto-retry caused you problems, please refer to the `readTimeout`,` dialTimeout` options in [Sample Configuration] (doc / sample-config / rc) advanced options.

## restrictions

-No cache
-Does not support HTTP pipeline (Chrome and Firefox have no pipeline enabled by default. Supporting this feature can easily increase problems and the benefits are not obvious)

# Acknowledgements

Contribution code:

-@fzerorubigd: various bug fixes and feature implementation
-@tevino: http parent proxy basic authentication
-@xupefei: provide cow-hide.exe to execute cow.exe in the background on windows
-@sunteya: Improved startup and installation scripts

Bug reporter:

-GitHub users: glacjay, trawor, Blaskyy, lucifer9, zellux, xream, hieixu, fantasticfears, perrywky, JayXon, graminc, WingGao, polong, dallascao, luosheng
-Twitter users: Special thanks to @ shao222 for helping test the new version and reporting many bugs, @xixitalk

@ glacjay's suggestion to make COW version 0.3 more automated made me reconsider the design goals of COW and improve how it works after version 0.5.

<h1 align="center">tun2socks</h1>
<h3 align="center">A tun2socks implementation written in Go.</h3>

[![GitHub Workflow](https://img.shields.io/github/workflow/status/xjasonlyu/tun2socks/Go/master?style=flat-square)](https://github.com/xjasonlyu/tun2socks/actions)
[![Docker Pulls](https://img.shields.io/docker/pulls/xjasonlyu/tun2socks?style=flat-square)](https://hub.docker.com/r/xjasonlyu/tun2socks)
[![Docker Image Size (tag)](https://img.shields.io/docker/image-size/xjasonlyu/tun2socks/latest?style=flat-square)](https://hub.docker.com/r/xjasonlyu/tun2socks)
[![Go version](https://img.shields.io/github/go-mod/go-version/xjasonlyu/tun2socks?style=flat-square)](https://img.shields.io/github/go-mod/go-version/xjasonlyu/tun2socks)
[![Go report](https://goreportcard.com/badge/github.com/xjasonlyu/tun2socks?style=flat-square)](https://goreportcard.com/badge/github.com/xjasonlyu/tun2socks)
[![GitHub License](https://img.shields.io/github/license/xjasonlyu/tun2socks?style=flat-square)](https://github.com/xjasonlyu/tun2socks/blob/master/LICENSE)
[![Lines of code](https://img.shields.io/tokei/lines/github/xjasonlyu/tun2socks?style=flat-square)](https://img.shields.io/tokei/lines/github/xjasonlyu/tun2socks)
[![Release](https://img.shields.io/github/v/release/xjasonlyu/tun2socks?include_prereleases&style=flat-square)](https://github.com/xjasonlyu/tun2socks/releases)
[![FOSSA Status](https://app.fossa.com/api/projects/git%2Bgithub.com%2Fxjasonlyu%2Ftun2socks.svg?type=shield)](https://app.fossa.com/projects/git%2Bgithub.com%2Fxjasonlyu%2Ftun2socks?ref=badge_shield)

## Features

- ICMP echoing
- IPv6 support
- Optimized UDP transmission for game acceleration
- Pure Go implementation, no CGO required
- Socks5, Shadowsocks protocol support for remote connections
- TCP/IP stack powered by [gVisor](https://github.com/google/gvisor)
- Up to 2.5Gbps throughput (10x faster than [v1](https://github.com/xjasonlyu/tun2socks/tree/v1))

## Requirements

| Target | Minimum | Recommended |
| :----- | :-----: | :---------: |
| System | linux darwin | linux |
| Memory | >20MB | >128MB |
| CPU | amd64 arm64 | amd64 |

## Performance

> iPerf3 tested on Debian 10 with i5-10500, 8G RAM

![iperf3 test](assets/iperf3.png)

## QuickStart

Download from precompiled [Releases](https://github.com/xjasonlyu/tun2socks/releases).

<details>
  <summary><b>With Docker</b></summary>

> Since Go 1.12, the runtime now uses MADV_FREE to release unused memory on **linux**. This is more efficient but may result in higher reported RSS. The kernel will reclaim the unused data when it is needed. To revert to the Go 1.11 behavior (MADV_DONTNEED), set the environment variable GODEBUG=madvdontneed=1.

create docker network (macvlan mode)

```shell script
docker network create -d macvlan \
  --subnet=172.20.1.0/25 \
  --gateway=172.20.1.1 \
  -o parent=eth0 \
  switch
```

pull `tun2socks` docker image

```shell script
docker pull xjasonlyu/tun2socks:latest
```

run as gateway (DNS configuration required)

```shell script
docker run -d \
  --network switch \
  --name tun2socks \
  --ip 172.20.1.2 \
  --privileged \
  --restart always \
  --sysctl net.ipv4.ip_forward=1 \
  -e PROXY=socks5://server:port \
  -e KEY=VALUE... \
  xjasonlyu/tun2socks:latest
```

or use docker-compose (recommended)

```yaml
version: '2.4'

services:
  tun2socks:
    image: xjasonlyu/tun2socks:latest
    cap_add:
      - NET_ADMIN
    devices:
        - '/dev/net/tun:/dev/net/tun'
    environment:
      # - GODEBUG=madvdontneed=1
      - PROXY=socks5://server:port
      - LOGLEVEL=warning
      - API=:8080
      - DNS=:53
      - HOSTS=
      - EXCLUDED=
      - EXTRACMD=
    networks:
      switch:
        ipv4_address: 172.20.1.2
    restart: always
    container_name: tun2socks

networks:
  switch:
    name: switch
    ipam:
      driver: default
      config:
        - subnet: '172.20.1.0/25'
          gateway: 172.20.1.1
    driver: macvlan
    driver_opts:
      parent: eth0
```
</details>

<details>
  <summary><b>With Linux</b></summary>

create tun

```shell script
ip tuntap add mode tun dev tun0
ip addr add 198.18.0.1/15 dev tun0
ip link set dev tun0 up
```

add route table

```shell script
ip route del default
ip route add default via 198.18.0.1 dev tun0
```

run

```shell script
./tun2socks --loglevel WARN --device tun://tun0 --proxy socks5://server:port --interface eth0
```
</details>

<details>
  <summary><b>With Script</b></summary>

```shell script
PROXY=socks5://server:port LOGLEVEL=WARN sh ./scripts/entrypoint.sh
```
</details>

## How to Build

### build from source code

Go compiler version >= 1.15 is required

```text
$ git clone https://github.com/xjasonlyu/tun2socks.git
$ cd tun2socks
$ make
```

### build docker image

```text
$ docker build -t tun2socks .
```

## Usage

<details>
  <summary><b>Help Text</b></summary>

```text
NAME:
   tun2socks - A tun2socks implementation written in Go.

USAGE:
   tun2socks [global options] [arguments...]

GLOBAL OPTIONS:
   --api value                  URL of external API to listen
   --device value, -d value     URL of device to open
   --dns value                  URL of fake DNS to listen
   --hosts value                Extra hosts mapping
   --interface value, -i value  Bind interface to dial
   --loglevel value, -l value   Set logging level (default: "INFO")
   --proxy value, -p value      URL of proxy to dial
   --version, -v                Print current version (default: false)
   --help, -h                   show help (default: false)
```
</details>

## Known Issues

Due to the implementation of pure Go, the memory usage is higher than the previous version.
If you are sensitive to memory, please go back to [v1](https://github.com/xjasonlyu/tun2socks/tree/v1).

## TODO

- [ ] Windows support

## Credits

- [Dreamacro/clash](https://github.com/Dreamacro/clash)
- [google/gvisor](https://github.com/google/gvisor)
- [majek/slirpnetstack](https://github.com/majek/slirpnetstack)


## License
[![FOSSA Status](https://app.fossa.com/api/projects/git%2Bgithub.com%2Fxjasonlyu%2Ftun2socks.svg?type=large)](https://app.fossa.com/projects/git%2Bgithub.com%2Fxjasonlyu%2Ftun2socks?ref=badge_large)
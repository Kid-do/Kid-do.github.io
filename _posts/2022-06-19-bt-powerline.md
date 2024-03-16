---
title: "Low Cost BT Mini Connector V2 Powerline Adapter Performance"
header:
  teaser: /assets/images/2022-06-19-bt-powerline/adapters.jpg
categories:
  - blog
---

tl;dr: See table for speed and latency tests.

![picture of the BT Mini Connector V2 Powerline Adapters](/assets/images/2022-06-19-bt-powerline/adapters.jpg)

Powerline adapters seem to divide opinion online, however, in some situations they can be a good option. I was looking at purchasing some 'gigabit' capable adapters and I stumbled across the BT Mini Connector V2 Powerline Adapters on eBay for less than £10 each and decided to give them a go. I couldn't find any details of performance online, so I'm writing my findings here in the hope that it is useful to others.

The adapters appear to be based on the [BCM60350](https://www.bt.com/content/dam/bt/help/powerline-adapters/BTDoC20035-Broadband-Extender-Flex-1000-Kit-BT-Mini-Connector-V2.pdf) SoC, which [several well known manufacturers also use](https://wikidevi.wi-cat.ru/index.php/Special:Ask?title=Special%3AAsk&q=%3Cq%3E%5B%5BCPU1+model::~BCM60350*%5D%5D+OR+%5B%5BCPU2+model::~BCM60350*%5D%5D%3C%2Fq%3E&po=%3FEmbedded+system+type=Type%0D%0A%3FFCC+ID%0D%0A%3FManuf%0D%0A%3FManuf+product+model=Manuf.+mdl%0D%0A%3FCPU1+model=CPU1%0D%0A%3FCPU1+clock+speed%0D%0A%3FCPU2+model=CPU2%0D%0A%3FCPU2+clock+speed=CPU2+clock+speed%0D%0A%3FFLA1+amount=FLA1%0D%0A%3FRAM1+amount=RAM1%0D%0A%3FWI1+chip1+model=WI1+chip1%0D%0A%3FWI1+MIMO+config=WI1+MIMO%0D%0A%3FWI2+chip1+model=WI2+chip1%0D%0A%3FWI2+MIMO+config=WI2+MIMO%0D%0A%3FSupported+802dot11+protocols=PHY+modes%0D%0A%3FOUI%0D%0A%3FOUI+(ethernet)=OUI+(Eth)%0D%0A%3FEstimated+year+of+release=Est.+year&eq=yes&p%5Bformat%5D=broadtable&order%5B0%5D=ASC&sort_num=&order_num=ASC&p%5Blimit%5D=500&p%5Boffset%5D=&p%5Blink%5D=all&p%5Bsort%5D=&p%5Bheaders%5D=show&p%5Bmainlabel%5D=&p%5Bintro%5D=&p%5Boutro%5D=&p%5Bsearchlabel%5D=%E2%80%A6+further+results&p%5Bdefault%5D=&p%5Bclass%5D=sortable+wikitable+smwtable). The adapters have two gigabit Ethernet ports and use the HomePlug AV2 spec which allows them to reach ['gigabit-class'](https://en.wikipedia.org/wiki/HomePlug#HomePlug_AV2) speeds.

Once they arrived, I reset them by pressing the '+Add' button for 15 seconds, after which they started working.

## Performance

Speed measured using iperf3. Latency measured using the ping command. Tests performed in the UK (230V), with results averaged over multiple runs. These are only rough figures. Powerline speeds can vary depending on many factors, so YMMV.

| Type      | Location                          | Speed (Mbps) | Latency (avg ms) |
| --------- | --------------------------------- | ------------ | ---------------- |
| Wired*    | Gigabit CAT5e 30m cable           | 930          | <1               |
| WiFi*     | 1m from AP                        | 450          | ~4.5             |
| WiFi*     | Different room, around 8m from AP | 280-320      | ~4.5 (max 115)   |
| Powerline | Same socket                       | 320          | ~3.5             |
| Powerline | Same room                         | 230-240      | ~3.5             |
| Powerline | Different room                    | 160-170      | ~3.5             |

*Baseline tests demonstrating the rough speeds and latency over a wired gigabit connection and a AC1350 WiFi connection from a TP-Link EAP225.

## Power Consumption

| State                                | Power Consumption |
| ------------------------------------ | ----------------- |
| Connected, Idle                      | 2.2W each         |
| Connected, running iperf test        | 2.2W each         |
| Standby, no network device connected | 0.3W each         |

## Bonus: TP-Link Powerline Utility

Surprisingly, the [tpPLC utility](https://www.tp-link.com/uk/support/download/tl-pa4010p-kit/v1/) provided with my previous TP-Link powerline adapters recognises these adapters, albeit with reduced functionality. The Secure button appears to work, allowing me to generate a new encryption key for the network. I have not yet tested any of the other functionality.

![tpPLC utility screenshot](/assets/images/2022-06-19-bt-powerline/tpplc.jpg)

## Conclusion

The speed is unsurprisingly nowhere near the gigabit speeds the adapters are advertised at. Initially, I was impressed with the 320 Mbps on the same socket, but the speed drops by half when one adapter is moved to another room (on a different circuit), which is how these adapters will likely be used in the real world.

AC WiFi will at close range still be faster, however, the benefit of powerline adapters is the lower and more consistent latency. It is also worth noting these speeds are still much faster than the [average download speed in the UK](https://labs2.thinkbroadband.com/local/uk) which at the time of writing is 62.5 Mbps.

Thanks for reading and I hope this was useful.

Feedback or corrections? Leave a comment below.

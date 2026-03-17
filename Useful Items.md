Jibo has many different interesting things that sometimes you can get confused! This document contains things that are nice to know for tinkering or developing for Jibo Revival.
# Useful Ports
> [!INFORMATION]
> You might not be able to access some ports if you haven't unblocked them on Jibo's firewall.
### 8779
This port has the skills manager. From here you can do a semi-reboot by stopping and starting the "@be/Be" skill.
### 15150
This port has logs. This port is only available on v3+ of the revival project (can be checked on the information section in settings).
### 10004
Error service, allows you to simulate errors in realtime. Shows names of third parties by accident. It seems in recent versions of Jibo errors relating to him not being able to connect to Jibo Inc Servers don't create a pop up L2, L7, Q1 (oddly WIFI4 and WIFI4a still give pop up), Q4. N1-N12 don't have pop ups either. OTA11 and R1 also makes an error inside an error saying "NOT HANDLED BY ERROR SKILL".
# Useful Files
### /opt/jibo/Jibo/Skills/@be/be/resources/JiboSplash.png
This file allows you to edit the splash screen. One thing to clarify is that it is only the splash screen for the Be skill meaning it will only edit the splash seen when you restart "@be/Be" or in the 'second boot stage' when Jibo spins and shows the splash
### /usr/local/bin/
Has a lot of random assets, cool thing to dig around and tinker with.
### /opt/jibo/Jibo/Skills/@be/be/node_modules/jibo-anim-db-animations/audio/
Lots of audio assets, the surrounding folders also contain other assets

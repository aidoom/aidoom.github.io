---
layout: post
title:  "Headless execution of IB Gateway on Ubuntu Server"
date:   2014-07-10 13:25:08
categories: linux
---

I have a headless machine running [Ubuntu Server](http://www.ubuntu.com/server) that is up 24/7 and would like to use it to run our algo trading system. If you're reading this, then you're probably aware that running the IB Gateway is not possible in Linux without an X server to handle the GUI. A possible workaround is to use X forwarding over SSH but if you're network connection is interrupted it will kill IB Gateway and have to be restarted manually. So, what we need is a way to run IB Gateway without the GUI, the ability to start/stop it as a service and a method to monitor it's status.

After some googling, I found the [IBController](https://github.com/ib-controller/ib-controller) project on github, which provides hands-free operation of IB Gateway. It completes the login dialog automatically with credentials from a `.ini` file and allows IB Gateway to be launched from a script. The [IBController userguide](https://github.com/ib-controller/ib-controller/blob/master/userguide.md) suggests that [headless execution](https://github.com/ib-controller/ib-controller/blob/master/userguide.md#headless-execution-unix) can be achieved by sending the display to a virtual framebuffer. For Arch Linux, a [package](https://aur.archlinux.org/packages/ib-controller/) has been created using IBController that manages headless IB Gateway instances. The source for this package is availbe in the [IBController AUR](https://github.com/benalexau/ibcontroller-aur) repository. It also shows how to use [monit](http://mmonit.com/) to monitor the status of the IB Gateway instances.

Following the methodology of the [IBController AUR](https://github.com/benalexau/ibcontroller-aur) project, I created an [IBController init script](https://gist.github.com/aidoom/72972af41470eebca743) for Ubuntu to start/stop instances of the IB Gateway. You will need to install the virtual frame buffer, xvfb, which is available via apt. I couldn't find a nice solution for capturing the PID of the java process that's started using `xvfb-run` so I resorted to this hacky solution for stopping the service:
{% highlight bash %}
pgrep -f "(${JAR}.*${INI}.ini)" | xargs kill -9
{% endhighlight %}

You will also need a `.ini` file in `/etc/ibcontroller` for the login credentials. You can use the demo credentials to test everything is working. After adding your real credentials, ensure that the `.ini` is only readable by `root`.
{% highlight ini %}
# Options that should be unique to each instantiated service
IbLoginId=edemo
IbPassword=demouser
ForceSocketPort=4003
IbControllerPort=7464
IbDir=/raid/ib/gateway/IBJts

# Options that rarely require changes
LogToConsole=no
FIX=no
PasswordEncrypted=no
FIXLoginId=
FIXPassword=
FIXPasswordEncrypted=yes
StoreSettingsOnServer=no
MinimizeMainWindow=yes
ExistingSessionDetectedAction=primary
AcceptIncomingConnectionAction=accept
ShowAllTrades=no
IbAutoClosedown=no
ClosedownAt=
AllowBlindTrading=yes
DismissPasswordExpiryWarning=yes
DismissNSEComplianceNotice=yes
IbControlFrom=
IbBindAddress=127.0.0.1
CommandPrompt=
SuppressInfoMessages=yes
{% endhighlight %}

The next step is to setup monitoring of the IB Gateway instances using monit. Firstly, install monit via the apt package and add the following to `/etc/monitrc`

{% highlight text %}
set daemon 60
set mailserver mail.someisp.com
set alert you@you.com not on { instance, action }

check process ib-api-edemo matching 'xvfb.*java.*edemo.ini'
  start program = "/usr/sbin/systemctl start ibcontroller@fdemo.service"
  stop program = "/usr/sbin/systemctl stop ibcontroller@fdemo.service"
  if failed host 127.0.0.1 port 4003
    send "63\0x0071\0x001\0x005556\0x00" # clientVer\startAPI\startApiVer\5556
    expect "[0-9]{2,}"                   # serverVer reply
    send "49\0x001\0x00"                 # reqCurrTime\reqCurrTimeVer
    expect "49"                          # serverTime reply
  then restart
{% endhighlight %}

Now, restart monit and point your browser to `localhost:2812` and you should see the `ib-api-edemo` process.

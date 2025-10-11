# Shelly WakeOnLan fence agent docs
fence_shelly_wol is a Power Fencing agent which can be used with Shelly Switches supporting the gen 2+ API to fence attached hardware. Additionally, this agent can send a Wake-on-LAN (WOL) packet to the device when powering on.
## Features
### Shelly
The shelly device can be used to powermanage a node by cutting the cutting power and making sure node gets a full reset.
The agent communicates with the shelly through HTTP reqests, meaning the shelly must be connected to the internet either to WiFi or LAN network. Following the [SHELLY API docs](https://shelly-api-docs.shelly.cloud/gen2/Devices/Gen2/ShellyPro1/) the agent uses a digest authentication mechanism when connecting to the device, so shelly password authentication must be set up and --password option must be provided for the agent. 
### WakeOnLan
After switching on power is enabled trough the shelly but in many cases this isn't enough for a node to start, we use WakeOnLan magic packet sent trough local network to wake the node. We use "wakeonlan" python module, if the program doesn't find it a warning is logged, the node can't be waked by this agent if the module is not downloaded. WakeOnLan is not mandatory for the agent to work.
The magic packet is sent after the shelly switch is set to "on" through HTTP request.
### Ping
Startup is approved via ping checks. We use "pythonping" module, if the program doesn't find it a warning is logged, the node's bootup is not precise unless the module is downloaded.
When checking on the node we first look at the shelly device's state if it returns off the node is taken as turned off, else we send 3 ping packets if all of them timeout the device is taken as off and lastly the device is on if the shelly returned on and at least 1 ping packet returned. The ping's timeout is 1 by default but can be set accoriding to the network's speed.
## Agent options:
### Required
- "password": the shelly device's password
- "ip": ipv4 code of the shelly device
- "plug": switch number of the shelly device (0 or 1)
- "host_ip": ip of the node the agent targets, ping check uses this ip (optional but recommended)
- "wol_mac": mac address of the node the agent targets, wakeonlan uses this address (optional but recommended)
### Optional
- "ping_timeout": time to wait for ping packet to return, default 1
- "wol_ip": WakeOnLan via ip
- "wol_port": port to use when WakeOnLan via ip, default 9
- "wol_interface": when multiple network interfaces are available set which one the magic packet should be sent trough
- "power_wait": OS-s especially servers often take some time to completly boot up, after on command is sent (including wol magic packet) wait this amount of time for the hardware to boot
## Agent setup
This tutorial walks trough how to install fence_shelly_wol fencing agent onto a working pacemaker cluster.
```
# Clone github repository
git clone https://github.com/ClusterLabs/fence-agents
cd fence-agents
# run autogen.sh and configure files to set up enviroment, you must have the following installed on the nodes for ./configure to finish: 
./autogen.sh && ./configure

# run make files to create executables
make xml-upload
make xml-check

# copy fence_shelly_wol into sbin and mark it executable
cp /root/fence-agents/agents/shelly_wol/fence_shelly_wol /sbin && chmod +x /sbin/fence_shelly_wol
```
Next check if pcs can see the fencing agent.
```
pcs stonith describe fence_shelly_wol
```
If it doesn't show any errors we can procede to creating the fencing agent. Here the name of the fencing agent is 'myshelly' and the following options are required or strongly recommended to use.
```
pcs stonith create myshelly fence_shelly_wol ip="shelly device ip address" plug=0 pcmk_host_list="fencing agent target node name in cluster" pcmk_host_check="static-list" password="YOURPASS" wol-mac="NODE MAC" host_ip="fencing agent target node ip"

```
Another recommended option to use is 'power_wait' since it takes time for the server to boot up propely we don't need to check if it's started until then. If you use 'power_wait' make sure the 'stonith-timeout' pacemaker property is greater the the power_wait other wise the agent will exit before finishing. You can do this with the following command:
```
pcs property set stonith-timeout=120
```
After creating the agent we can check it's configuration with:
```
pcs stonith config myshelly
```
With this you have succesfully created a fence_shelly_wol fencing agent. Testing can be done manually and then by creating an error where pacemaker decides to fence the targeted node. Manual testing can be done by using the `pcs fence targeted.node.name`. To check logs you can use `cat /var/logs/messages` and to enable debug logs inreace verbose level in the agent configuration `pcs stonith update myshelly verbose_level=1`.
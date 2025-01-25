# xpra — viewer for remote, persistent X applications

Sometimes you need to run a graphical application from a remote machine, but don't want to mess with VNC or RDP configuration. In such cases you can use xpra, which requires no configuration, works even without X11 and XWayland, and runs fine from a container. No changes to the sshd config are required.

We will use:
Client: distrobox with Arch Linux.
Server: A machine with Fedora Server

On the client (distrobox with Arch Linux):
```bash
sudo pacman -S xpra.
```
On the server (Fedora Server):
```bash
sudo dnf install xpra
```

1. Start xpra on the server with the following command:
```bash
xpra start :100
```

2. Start the graphical application by specifying the DISPLAY variable:
```bash
DISPLAY=:100 firefox
```
 You can run multiple applications at once in the same way.


To display the windows of running applications on the client, run the following command in distrobox:
```bash
xpra attach ssh://username@server_ip/100 --compress=0
```

The --compress=0 parameter is suitable for LAN operation to avoid compression artifacts.
When connecting to a remote machine, it is best not to use this parameter.


To stop an xpra session on the server, execute:
```bash
xpra stop :100
```

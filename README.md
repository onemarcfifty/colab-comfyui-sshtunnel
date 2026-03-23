# colab-comfyui-sshtunnel
run ComfyUI in Google Colab and access it using a reverse ssh tunnel over a bastion VPS

## The Goal

I want to run generative AI in Google CoLab. Why? Because CoLab is less expensive than other GPU platforms (you get a T4 for free but may pay for more). Also, it has one big advantage towards all other platforms: You can mount Google Drive as a local drive and run code from it.

## The Challenge(s)

There is no free lunch. Every concept has challenges. Google CoLab is not an "always on" Server platform. It is great for testing and playing around with code. But there are limitations:

1. When resources are low, you may be disconnected (you do get a warning 30m before though)
2. comfyUI models need A LOT of disk space
3. There is no VPN or tunneling mechanism, i.e. you can't reach your platform from the outside. This is because the CoLab Workbook sits behind a double NAT (similar to Carrier Grade NAT / CGNAT) inside a Docker Container. Also, the container has no NetAdmin Capabilities, i.e. you can't VPN out. There are no TUN/TAP devices neither.

## The Solution

- Challenge #1 can only be solved by either throwing money at it or living with it.
- Challenge #2 can easily be solved with a 2 TB subscription to Google Drive
- Challenge #3 can be worked around as follows:

In order to be able to connect to CoLab code from the outside world, people usually use Tunneling providers. Popular ones are

- CloudFlare / Argo
- Ngrok
- LocalTunnel
- Instatunnel

These services are great. But they can be slow sometimes. Especially when you are not connecting from and to the same Geo (e.g. connecting from Europe to the US you geet a lot of latency). Also, I am quite "old school" in Networking, i.e. I prefer to run things on my own infrastructure rather than relying on services. I do like Platform services (PaaS), because they can run stuff cheaper and better than I can, but I love to run my own services. Last but not least, there is no authentication - anyone who has the link can connect. So here is my solution to the problem: **Reverse ssh Tunnel** .

## How it works

As we can't connect in, we will connect out. To any host in the internet as long as we control it and as long as we can reach it with ssh. I run a couple of cheap (1$-2$ per month)  virtual servers, mainly as bastion hosts or VPN hosts. So we connect out to that host with ssh. Usually ssh is used to open a terminal on a remote host. But it can also do something quite exciting: Reverse Tunneling. This lets a remote server expose a local service by forwarding ports over a secure SSH connection. It’s a lightweight, dependency-free alternative to the aforementioned services. 

So we ssh to a Server (VPS) and create a socket there which forwards all traffic back to CoLab. Something like this:

```
ssh -N -R 8100:localhost:8188 user@vps.example.org
```

This listens to port 8100 on the VPS and forwards it to port 8188 in our CoLab instance. On the VPS we run nginx that proxy passes traffic to the socket on the local host such as:

```
# ################################################
# colab (localhost ssh reverse tunnel from colab)
# ################################################

server
{
  listen  443 ssl;
  listen [::]:443 ssl;
  server_name colab.example.org;
  # include /etc/nginx/snippets/authelia-location.conf;

  location /
  {
    client_max_body_size 10G;
    # include /etc/nginx/snippets/proxy.conf;
    # include /etc/nginx/snippets/authrequest.conf;
    proxy_pass  http://127.0.0.1:8100/;
  }

  ssl_certificate /etc/letsencrypt/live/colab.example.org/fullchain.pem; # managed by Certbot
  ssl_certificate_key /etc/letsencrypt/live/colab.example.org/privkey.pem; # managed by Certbot
}
```

The complete flow is as follows:

```
+----------------------+
|   COLAB instance     |
|----------------------|
|  comfyUI exposed on  |
|  Port 8188           |
+----------------------+
          |
outgoing  |  ssh Tunnel
          v
+--------------------------+
|         VPS              |<--------------------+
|--------------------------|  colab.example.org  |
|  NGINX exposing Port 443 |                     | 
|  ssh Tunnel listening    |                     |
|  to Port 8100            |                     |
+--------------------------|                     |
                                                 |
                                           https |
                                        +----------------+
                                        |      User      |
                                        |  Web-Browser   |
                                        +----------------+
```

## Using the Notebooks.

Things need to be done in the following order:

1. Connect to a CoLab Instance
2. Mount the Google Drive in CoLab
3. (optional, needs to be done once) Install comfyUI
4. (optional) Install models into comfyUI
5. (optional) install venv
6. Start comfyUI
7. (optional) query the public IP of the CoLab instance and generate an ephemeral ssh key for the vps
8. (optional) modify your firewall and ssh keys on the vps
9. establish the ssh tunnel to the vps
10. use and enjoy comfyUI over the vps
11. (optional) revert firewall changes and ssh keys on the vps

- Step 1 is done either with a browser or with VS Code using the CoLab Extension.
- Steps 2,3,5,6 can be done in the ```ComfyUI.ipynb``` Jupyter Notebook
- Step 4 can either be done using code samples in the ```tools``` subdir or using something like https://github.com/romandev-codex/ComfyUI-Downloader.git 
- Steps 7,8,9 can be done using the ```ssh-tunnel.ipynb``` Jupyter Notebook
- Steps 8,10,11 can/need to be done outside of this environment


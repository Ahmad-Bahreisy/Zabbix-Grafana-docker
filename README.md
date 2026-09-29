# zabbix-grafana-docker

My docker compose setup for Zabbix 6.4 + Grafana. Works for me on Ubuntu 22.04 and 24.04.

## Setup

```
git clone https://github.com/YOUR-USERNAME/zabbix-grafana-docker.git
cd zabbix-grafana-docker
cp .env.example .env
```

Open `.env` and change the passwords to your own, then start it:

```
docker compose up -d
```

First start is slow because MySQL needs to set itself up, so just wait a couple of minutes.

Zabbix runs on port 8080 (login is `Admin` / `zabbix`) and Grafana on 3000. If those ports are already used by something else, change them in `.env`.

## Grafana

Go to Administration > Plugins and enable the Zabbix plugin. After that add a new Zabbix data source and use this as the URL:

```
http://zabbix-web:8080/api_jsonrpc.php
```

Put your Zabbix username and password in the Zabbix Connection section.

## Adding a server

On the machine you want to monitor, install the agent. If it's Ubuntu 22.04, swap `24.04` for `22.04` in the commands below.

```
wget https://repo.zabbix.com/zabbix/6.4/ubuntu/pool/main/z/zabbix-release/zabbix-release_6.4-1+ubuntu24.04_all.deb
sudo dpkg -i zabbix-release_6.4-1+ubuntu24.04_all.deb
sudo apt update
sudo apt install zabbix-agent2
```

Edit `/etc/zabbix/zabbix_agent2.conf` and set `Server`, `ServerActive` and `Hostname`, then restart the agent:

```
sudo systemctl restart zabbix-agent2
```

Last step, add the host in the Zabbix web UI. Use the exact same hostname as in the config, and attach the "Linux by Zabbix agent" template. Also make sure port 10050 is open from the Zabbix server, otherwise it won't connect.

## Proxy

`docker-compose-proxy.yml` is for servers sitting on a private network. Create an active proxy in Zabbix first, fill in the two variables in the file, and run it on a machine inside that network.

## Notes

- Grafana showing "No data" after a restart? Run `docker compose restart grafana` and it usually fixes itself.
- `.env` is in `.gitignore`. Don't remove it, you don't want your passwords on GitHub.

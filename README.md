# zabbix-grafana-docker

Zabbix 6.4 and Grafana on docker compose. Tested on Ubuntu 22.04 and 24.04.

## Setup

```
git clone https://github.com/YOUR-USERNAME/zabbix-grafana-docker.git
cd zabbix-grafana-docker
cp .env.example .env
```

Edit `.env` and set your own passwords, then:

```
docker compose up -d
```

Give it a minute or two, MySQL has to initialize first.

Zabbix is on port 8080 (default login `Admin` / `zabbix`), Grafana is on 3000. Ports can be changed in `.env` if they clash with something else.

## Grafana

Enable the Zabbix plugin under Administration > Plugins, then add a Zabbix data source with this URL:

```
http://zabbix-web:8080/api_jsonrpc.php
```

Use your Zabbix username and password in the Zabbix Connection section.

## Adding a server

Install the agent on the machine you want to monitor (change 24.04 to 22.04 if needed):

```
wget https://repo.zabbix.com/zabbix/6.4/ubuntu/pool/main/z/zabbix-release/zabbix-release_6.4-1+ubuntu24.04_all.deb
sudo dpkg -i zabbix-release_6.4-1+ubuntu24.04_all.deb
sudo apt update
sudo apt install zabbix-agent2
```

Set `Server`, `ServerActive` and `Hostname` in `/etc/zabbix/zabbix_agent2.conf`, then restart it:

```
sudo systemctl restart zabbix-agent2
```

Then create the host in Zabbix with the same hostname and the "Linux by Zabbix agent" template. Port 10050 must be open from the Zabbix server.

## Proxy

`docker-compose-proxy.yml` is for machines on a private network. Create an active proxy in Zabbix first, fill in the two variables in the file, and run it on a box inside that network.

## Notes

- If Grafana shows "No data" after a restart, run `docker compose restart grafana`.
- `.env` is in `.gitignore`, keep it that way.

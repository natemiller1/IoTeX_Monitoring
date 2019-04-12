# IoTeX_Monitoring

This will guide you to setting up prometheus, node exporter, and grafana, to monitor your nodes. This will take about 30 minutes, if you already know what you're doing and just want the grafana dashboards, head to:

https://grafana.com/dashboards/8919 for our Chinese speaking friends

or

https://grafana.com/dashboards/10028 Which is the same dashboard that I translated to English

![alt text](https://github.com/natemiller1/IoTeX_Monitoring/blob/master/Screenshot%20from%202019-04-10%2015-43-42.png)

Initial considerations:
K8s will require a slightly different set up.

Architecture: You can run prometheus/node exporter/grafana on the same server you are running iotex OR you can run prometheus and grafana on another server but you will still need to install node exporter on the server you want to monitor. I prefer the second option but whatever option you chose, be very careful with the ports you open and the ip addresses you open those ports to!

***Step 1 — Creating Service Users***
For security purposes, create two new user accounts, and use the --no-create-home and --shell /bin/false options so that these users can't log into the server.

```
sudo useradd --no-create-home --shell /bin/false prometheus
sudo useradd --no-create-home --shell /bin/false node_exporter
```
Create necessary directories:
```
sudo mkdir /etc/prometheus
sudo mkdir /var/lib/prometheus
```
Change directory permissions:
```
sudo chown prometheus:prometheus /etc/prometheus
sudo chown prometheus:prometheus /var/lib/prometheus
```

***Step 2 — Downloading Prometheus***
First, download and unpack the current stable version of Prometheus into your home directory. You can find the latest binaries along with their checksums on the Prometheus download page. https://prometheus.io/download/
```
cd ~
curl -LO https://github.com/prometheus/prometheus/releases/download/v2.8.1/prometheus-2.8.1.linux-amd64.tar.gz
tar xvf prometheus-2.8.1.linux-amd64.tar.gz
```

Copy the two binaries to the /usr/local/bin directory.
```
sudo cp prometheus-2.8.1.linux-amd64/prometheus /usr/local/bin/
sudo cp prometheus-2.8.1.linux-amd64/promtool /usr/local/bin/
```
Set the user and group ownership on the binaries to the prometheus user created in Step 1.
```
sudo chown prometheus:prometheus /usr/local/bin/prometheus
sudo chown prometheus:prometheus /usr/local/bin/promtool
```
Copy the consoles and console_libraries directories to /etc/prometheus.
```
sudo cp -r prometheus-2.8.1.linux-amd64/consoles /etc/prometheus
sudo cp -r prometheus-2.8.1.linux-amd64/console_libraries /etc/prometheus
```
Set the user and group ownership on the directories to the prometheus user.
```
sudo chown -R prometheus:prometheus /etc/prometheus/consoles
sudo chown -R prometheus:prometheus /etc/prometheus/console_libraries
```
Lastly, remove the leftover files from your home directory as they are no longer needed.
```
rm -rf prometheus-2.8.1.linux-amd64.tar.gz prometheus-2.8.1.linux-amd64
```

***Step 3 — Configuring Prometheus***
```
sudo nano /etc/prometheus/prometheus.yml
```

In the global settings, define the default interval for scraping metrics. Note that Prometheus will apply these settings to every exporter unless an individual exporter's own settings override the globals.

Prometheus config file - /etc/prometheus/prometheus.yml
```
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'iotex'
    scrape_interval: 5s
    static_configs:
      - targets: ['localhost:9090']
```
Save the file and exit your text editor.

Now, set the user and group ownership on the configuration file to the prometheus user created in Step 1.
```
sudo chown prometheus:prometheus /etc/prometheus/prometheus.yml
```

***Step 4 — Running Prometheus***

```
sudo -u prometheus /usr/local/bin/prometheus \
    --config.file /etc/prometheus/prometheus.yml \
    --storage.tsdb.path /var/lib/prometheus/ \
    --web.console.templates=/etc/prometheus/consoles \
    --web.console.libraries=/etc/prometheus/console_libraries
```

If you get an error message, double-check that you've used YAML syntax in your configuration file and then follow the on-screen instructions to resolve the problem.

Now, halt Prometheus by pressing CTRL+C, and then open a new systemd service file.
```
sudo nano /etc/systemd/system/prometheus.service
```

Copy the following content into the file:
```
Prometheus service file - /etc/systemd/system/prometheus.service
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
    --config.file /etc/prometheus/prometheus.yml \
    --storage.tsdb.path /var/lib/prometheus/ \
    --web.console.templates=/etc/prometheus/consoles \
    --web.console.libraries=/etc/prometheus/console_libraries

[Install]
WantedBy=multi-user.target
```

Save the file and close your text editor.

To use the newly created service, reload systemd.
```
sudo systemctl daemon-reload
```
You can now start Prometheus using the following command:
```
sudo systemctl start prometheus
```
To make sure Prometheus is running, check the service's status.
```
sudo systemctl status prometheus
```
Lastly, enable the service to start on boot.
```
sudo systemctl enable prometheus
```
Now that Prometheus is up and running, we can install an additional exporter to generate metrics about our server's resources.

***Step 5 — Downloading Node Exporter***
To expand Prometheus beyond metrics about itself only, we'll install an additional exporter called Node Exporter. Node Exporter provides detailed information about the system, including CPU, disk, and memory usage. You'll need to install node exporter on each node you want to monitor

First, download the current stable version of Node Exporter into your home directory. You can find the latest binaries along with their checksums on Prometheus' download page.
```
cd ~
curl -LO https://github.com/prometheus/node_exporter/releases/download/v0.15.1/node_exporter-0.17.0.linux-amd64.tar.gz
tar xvf node_exporter-0.17.0.linux-amd64.tar.gz
```
Copy the binary to the /usr/local/bin directory and set the user and group ownership to the node_exporter user that you created in Step 1.
```
sudo cp node_exporter-0.15.1.linux-amd64/node_exporter /usr/local/bin
sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter
```
Remove the leftover files from your home directory.
```
rm -rf node_exporter-0.17.0.linux-amd64.tar.gz node_exporter-0.17.0.linux-amd64
```

***Step 6 — Running Node Exporter***
The steps for running Node Exporter are similar to those for running Prometheus itself. Start by creating the Systemd service file for Node Exporter.
```
sudo nano /etc/systemd/system/node_exporter.service
```

Copy the following content into the service file:
```
Node Exporter service file - /etc/systemd/system/node_exporter.service
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
```

Save the file and close your text editor.

Get systemd ready to use the newly created service.
```
sudo systemctl daemon-reload

sudo systemctl start node_exporter

sudo systemctl status node_exporter

sudo systemctl enable node_exporter
```
With Node Exporter fully configured and running as expected, we'll tell Prometheus to start scraping the new metrics.

***Step 7 — Configuring Prometheus to Scrape Node Exporter***
Because Prometheus only scrapes exporters which are defined in the scrape_configs portion of its configuration file, we'll need to add an entry for Node Exporter.

Open the configuration file.
```
sudo nano /etc/prometheus/prometheus.yml
```
At the end of the scrape_configs block, add a new entry called node_exporter. You can add multiple entries for multiple nodes you want to monitor

Your whole configuration file should look like this:

Prometheus config file - /etc/prometheus/prometheus.yml
```
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    scrape_interval: 5s
    static_configs:
      - targets: ['localhost:9090']
  - job_name: 'iotex_node'
    scrape_interval: 5s
    static_configs:
      - targets: ['localhost:9100']   
```      
**Remember that for node_exporter, localhost, after the targets tag, could also be the ip address of the node you're monitoring, you can have multiple node_explorer jobs in one prometheus config file**
Save the file and exit your text editor when you're ready to continue.

Restart Prometheus to put the changes into effect.
```
sudo systemctl restart prometheus
```
Verify that everything is running correctly with the status command.
```
sudo systemctl status prometheus
```

***Step 8 - Grafana***


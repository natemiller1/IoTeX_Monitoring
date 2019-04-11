# IoTeX_Monitoring

This will guide you to setting up prometheus, node exporter, and grafana, to monitor your nodes

Initial considerations:
This is written for a single node, k8s require a slightly different set up.

Architecture: You can run prometheus/node exporter/grafana on the same server you are running iotex OR you can run prometheus and grafana on another server but you will still need to install node exporter on the server you want to monitor. I prefer the second option but whatever option you chose, be very careful with the ports you open and the ip addresses you open those ports to!


Step 1 — Creating Service Users
For security purposes, we'll begin by creating two new user accounts, prometheus and node_exporter. We'll use these accounts throughout the tutorial to isolate the ownership on Prometheus' core files and directories.

Create these two users, and use the --no-create-home and --shell /bin/false options so that these users can't log into the server.

sudo useradd --no-create-home --shell /bin/false prometheus
sudo useradd --no-create-home --shell /bin/false node_exporter
Before we download the Prometheus binaries, create the necessary directories for storing Prometheus' files and data. Following standard Linux conventions, we'll create a directory in /etc for Prometheus' configuration files and a directory in /var/lib for its data.

sudo mkdir /etc/prometheus
sudo mkdir /var/lib/prometheus
Now, set the user and group ownership on the new directories to the prometheus user.

sudo chown prometheus:prometheus /etc/prometheus
sudo chown prometheus:prometheus /var/lib/prometheus
With our users and directories in place, we can now download Prometheus and then create the minimal configuration file to run Prometheus for the first time.

Step 2 — Downloading Prometheus
First, download and unpack the current stable version of Prometheus into your home directory. You can find the latest binaries along with their checksums on the Prometheus download page.

cd ~
curl -LO https://github.com/prometheus/prometheus/releases/download/v2.0.0/prometheus-2.0.0.linux-amd64.tar.gz
Next, use the sha256sum command to generate a checksum of the downloaded file:

sha256sum prometheus-2.0.0.linux-amd64.tar.gz
Compare the output from this command with the checksum on the Prometheus download page to ensure that your file is both genuine and not corrupted.

Output
e12917b25b32980daee0e9cf879d9ec197e2893924bd1574604eb0f550034d46  prometheus-2.0.0.linux-amd64.tar.gz
If the checksums don't match, remove the downloaded file and repeat the preceding steps to re-download the file.

Now, unpack the downloaded archive.

tar xvf prometheus-2.0.0.linux-amd64.tar.gz
This will create a directory called prometheus-2.0.0.linux-amd64 containing two binary files (prometheus and promtool), consoles and console_libraries directories containing the web interface files, a license, a notice, and several example files.

Copy the two binaries to the /usr/local/bin directory.

sudo cp prometheus-2.0.0.linux-amd64/prometheus /usr/local/bin/
sudo cp prometheus-2.0.0.linux-amd64/promtool /usr/local/bin/
Set the user and group ownership on the binaries to the prometheus user created in Step 1.

sudo chown prometheus:prometheus /usr/local/bin/prometheus
sudo chown prometheus:prometheus /usr/local/bin/promtool
Copy the consoles and console_libraries directories to /etc/prometheus.

sudo cp -r prometheus-2.0.0.linux-amd64/consoles /etc/prometheus
sudo cp -r prometheus-2.0.0.linux-amd64/console_libraries /etc/prometheus
Set the user and group ownership on the directories to the prometheus user. Using the -R flag will ensure that ownership is set on the files inside the directory as well.

sudo chown -R prometheus:prometheus /etc/prometheus/consoles
sudo chown -R prometheus:prometheus /etc/prometheus/console_libraries
Lastly, remove the leftover files from your home directory as they are no longer needed.

rm -rf prometheus-2.0.0.linux-amd64.tar.gz prometheus-2.0.0.linux-amd64
Now that Prometheus is installed, we'll create its configuration and service files in preparation of its first run.

Step 3 — Configuring Prometheus
In the /etc/prometheus directory, use nano or your favorite text editor to create a configuration file named prometheus.yml. For now, this file will contain just enough information to run Prometheus for the first time.

sudo nano /etc/prometheus/prometheus.yml
Warning: Prometheus' configuration file uses the YAML format, which strictly forbids tabs and requires two spaces for indentation. Prometheus will fail to start if the configuration file is incorrectly formatted.

In the global settings, define the default interval for scraping metrics. Note that Prometheus will apply these settings to every exporter unless an individual exporter's own settings override the globals.

Prometheus config file part 1 - /etc/prometheus/prometheus.yml
global:
  scrape_interval: 15s
This scrape_interval value tells Prometheus to collect metrics from its exporters every 15 seconds, which is long enough for most exporters.

Now, add Prometheus itself to the list of exporters to scrape from with the following scrape_configs directive:

Prometheus config file part 2 - /etc/prometheus/prometheus.yml
...
scrape_configs:
  - job_name: 'prometheus'
    scrape_interval: 5s
    static_configs:
      - targets: ['localhost:9090']
Prometheus uses the job_name to label exporters in queries and on graphs, so be sure to pick something descriptive here.

And, as Prometheus exports important data about itself that you can use for monitoring performance and debugging, we've overridden the global scrape_interval directive from 15 seconds to 5 seconds for more frequent updates.

Lastly, Prometheus uses the static_configs and targets directives to determine where exporters are running. Since this particular exporter is running on the same server as Prometheus itself, we can use localhost instead of an IP address along with the default port, 9090.

Your configuration file should now look like this:

Prometheus config file - /etc/prometheus/prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    scrape_interval: 5s
    static_configs:
      - targets: ['localhost:9090']
Save the file and exit your text editor.

Now, set the user and group ownership on the configuration file to the prometheus user created in Step 1.

sudo chown prometheus:prometheus /etc/prometheus/prometheus.yml
With the configuration complete, we're ready to test Prometheus by running it for the first time.

Step 4 — Running Prometheus
Start up Prometheus as the prometheus user, providing the path to both the configuration file and the data directory.

sudo -u prometheus /usr/local/bin/prometheus \
    --config.file /etc/prometheus/prometheus.yml \
    --storage.tsdb.path /var/lib/prometheus/ \
    --web.console.templates=/etc/prometheus/consoles \
    --web.console.libraries=/etc/prometheus/console_libraries
The output contains information about Prometheus' loading progress, configuration file, and related services. It also confirms that Prometheus is listening on port 9090.

Output
level=info ts=2017-11-17T18:37:27.474530094Z caller=main.go:215 msg="Starting Prometheus" version="(version=2.0.0, branch=HEAD, re
vision=0a74f98628a0463dddc90528220c94de5032d1a0)"
level=info ts=2017-11-17T18:37:27.474758404Z caller=main.go:216 build_context="(go=go1.9.2, user=root@615b82cb36b6, date=20171108-
07:11:59)"
level=info ts=2017-11-17T18:37:27.474883982Z caller=main.go:217 host_details="(Linux 4.4.0-98-generic #121-Ubuntu SMP Tue Oct 10 1
4:24:03 UTC 2017 x86_64 prometheus-update (none))"
level=info ts=2017-11-17T18:37:27.483661837Z caller=web.go:380 component=web msg="Start listening for connections" address=0.0.0.0
:9090
level=info ts=2017-11-17T18:37:27.489730138Z caller=main.go:314 msg="Starting TSDB"
level=info ts=2017-11-17T18:37:27.516050288Z caller=targetmanager.go:71 component="target manager" msg="Starting target manager...
"
level=info ts=2017-11-17T18:37:27.537629169Z caller=main.go:326 msg="TSDB started"
level=info ts=2017-11-17T18:37:27.537896721Z caller=main.go:394 msg="Loading configuration file" filename=/etc/prometheus/promethe
us.yml
level=info ts=2017-11-17T18:37:27.53890004Z caller=main.go:371 msg="Server is ready to receive requests."
If you get an error message, double-check that you've used YAML syntax in your configuration file and then follow the on-screen instructions to resolve the problem.

Now, halt Prometheus by pressing CTRL+C, and then open a new systemd service file.

sudo nano /etc/systemd/system/prometheus.service
The service file tells systemd to run Prometheus as the prometheus user, with the configuration file located in the /etc/prometheus/prometheus.yml directory and to store its data in the /var/lib/prometheus directory. (The details of systemd service files are beyond the scope of this tutorial, but you can learn more at Understanding Systemd Units and Unit Files.)

Copy the following content into the file:

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
Finally, save the file and close your text editor.

To use the newly created service, reload systemd.

sudo systemctl daemon-reload
You can now start Prometheus using the following command:

sudo systemctl start prometheus
To make sure Prometheus is running, check the service's status.

sudo systemctl status prometheus
The output tells you Prometheus' status, main process identifier (PID), memory use, and more.

If the service's status isn't active, follow the on-screen instructions and re-trace the preceding steps to resolve the problem before continuing the tutorial.

Output
● prometheus.service - Prometheus
   Loaded: loaded (/etc/systemd/system/prometheus.service; disabled; vendor preset: enabled)
   Active: active (running) since Fri 2017-07-21 11:40:40 UTC; 3s ago
 Main PID: 2104 (prometheus)
    Tasks: 7
   Memory: 13.8M
      CPU: 470ms
   CGroup: /system.slice/prometheus.service
...
When you're ready to move on, press Q to quit the status command.

Lastly, enable the service to start on boot.

sudo systemctl enable prometheus
Now that Prometheus is up and running, we can install an additional exporter to generate metrics about our server's resources.

Step 5 — Downloading Node Exporter
To expand Prometheus beyond metrics about itself only, we'll install an additional exporter called Node Exporter. Node Exporter provides detailed information about the system, including CPU, disk, and memory usage.

First, download the current stable version of Node Exporter into your home directory. You can find the latest binaries along with their checksums on Prometheus' download page.

cd ~
curl -LO https://github.com/prometheus/node_exporter/releases/download/v0.15.1/node_exporter-0.15.1.linux-amd64.tar.gz
Use the sha256sum command to generate a checksum of the downloaded file:

sha256sum node_exporter-0.15.1.linux-amd64.tar.gz
Verify the downloaded file's integrity by comparing its checksum with the one on the download page.

Output
7ffb3773abb71dd2b2119c5f6a7a0dbca0cff34b24b2ced9e01d9897df61a127  node_exporter-0.15.1.linux-amd64.tar.gz
If the checksums don't match, remove the downloaded file and repeat the preceding steps.

Now, unpack the downloaded archive.

tar xvf node_exporter-0.15.1.linux-amd64.tar.gz
This will create a directory called node_exporter-0.15.1.linux-amd64 containing a binary file named node_exporter, a license, and a notice.

Copy the binary to the /usr/local/bin directory and set the user and group ownership to the node_exporter user that you created in Step 1.

sudo cp node_exporter-0.15.1.linux-amd64/node_exporter /usr/local/bin
sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter
Lastly, remove the leftover files from your home directory as they are no longer needed.

rm -rf node_exporter-0.15.1.linux-amd64.tar.gz node_exporter-0.15.1.linux-amd64
Now that you've installed Node Exporter, let's test it out by running it before creating a service file for it so that it starts on boot.

Step 6 — Running Node Exporter
The steps for running Node Exporter are similar to those for running Prometheus itself. Start by creating the Systemd service file for Node Exporter.

sudo nano /etc/systemd/system/node_exporter.service
This service file tells your system to run Node Exporter as the node_exporter user with the default set of collectors enabled.

Copy the following content into the service file:

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
Collectors define which metrics Node Exporter will generate. You can see Node Exporter's complete list of collectors — including which are enabled by default and which are deprecated — in the Node Exporter README file.

If you ever need to override the default list of collectors, you can use the --collectors.enabled flag, like:

Node Exporter service file part - /etc/systemd/system/node_exporter.service
...
ExecStart=/usr/local/bin/node_exporter --collectors.enabled meminfo,loadavg,filesystem
...
The preceding example would tell Node Exporter to generate metrics using only the meminfo, loadavg, and filesystem collectors. You can limit the collectors to however few or many you need, but note that there are no blank spaces before or after the commas.

Save the file and close your text editor.

Finally, reload systemd to use the newly created service.

sudo systemctl daemon-reload
You can now run Node Exporter using the following command:

sudo systemctl start node_exporter
Verify that Node Exporter's running correctly with the status command.

sudo systemctl status node_exporter
Like before, this output tells you Node Exporter's status, main process identifier (PID), memory usage, and more.

If the service's status isn't active, follow the on-screen messages and re-trace the preceding steps to resolve the problem before continuing.

Output
● node_exporter.service - Node Exporter
   Loaded: loaded (/etc/systemd/system/node_exporter.service; disabled; vendor preset: enabled)
   Active: active (running) since Fri 2017-07-21 11:44:46 UTC; 5s ago
 Main PID: 2161 (node_exporter)
    Tasks: 3
   Memory: 1.4M
      CPU: 11ms
   CGroup: /system.slice/node_exporter.service
Lastly, enable Node Exporter to start on boot.

sudo systemctl enable node_exporter
With Node Exporter fully configured and running as expected, we'll tell Prometheus to start scraping the new metrics.

Step 7 — Configuring Prometheus to Scrape Node Exporter
Because Prometheus only scrapes exporters which are defined in the scrape_configs portion of its configuration file, we'll need to add an entry for Node Exporter, just like we did for Prometheus itself.

Open the configuration file.

sudo nano /etc/prometheus/prometheus.yml
At the end of the scrape_configs block, add a new entry called node_exporter.

Prometheus config file part 1 - /etc/prometheus/prometheus.yml
...
  - job_name: 'node_exporter'
    scrape_interval: 5s
    static_configs:
      - targets: ['localhost:9100']
Because this exporter is also running on the same server as Prometheus itself, we can use localhost instead of an IP address again along with Node Exporter's default port, 9100.

Your whole configuration file should look like this:

Prometheus config file - /etc/prometheus/prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    scrape_interval: 5s
    static_configs:
      - targets: ['localhost:9090']
  - job_name: 'node_exporter'
    scrape_interval: 5s
    static_configs:
      - targets: ['localhost:9100']       
Save the file and exit your text editor when you're ready to continue.

Finally, restart Prometheus to put the changes into effect.

sudo systemctl restart prometheus
Once again, verify that everything is running correctly with the status command.

sudo systemctl status prometheus
If the service's status isn't set to active, follow the on screen instructions and re-trace your previous steps before moving on.

Output
● prometheus.service - Prometheus
   Loaded: loaded (/etc/systemd/system/prometheus.service; disabled; vendor preset: enabled)
   Active: active (running) since Fri 2017-07-21 11:46:39 UTC; 6s ago
 Main PID: 2219 (prometheus)
    Tasks: 6
   Memory: 19.9M
      CPU: 433ms
   CGroup: /system.slice/prometheus.service

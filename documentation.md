# Network Management Project

# Summary

For this project the team used Grafana, Prometheus, Node Exporter, and Windows Exporter. Grafana was used to visualize the log data and send alerts to Discord. Prometheus ingested all the logs collected from the endpoints and passed the data to Grafana. Node Exporter and WMI Exporter were used to collect system information on Linux and Windows systems respectively. 

# Prometheus Setup

## Download

Setting up Prometheus was the first thing our group did. To begin the Prometheus setup you will want to create a new user for the service. To do this run the following command `useradd -m -s /bin/bash prometheus`. This command will create a new user; the m flag creates the users home directory; the s flag specifies what shell the user will use, in this case it is bash. Now that the user has been created we can switch to that user account with `su - prometheus`. On the Prometheus user's account use `wget` to download the latest version of Prometheus, the latest version can be found [here]([https://github.com/prometheus/prometheus/releases/](https://github.com/prometheus/prometheus/releases/)). At the time of writing this v2.24.1 was the latest release and we used that. If you were to use the same version the `wget` command would go as follows `wget [https://github.com/prometheus/prometheus/releases/download/v2.24.1/prometheus-2.24.1.linux-amd64.tar.gz](https://github.com/prometheus/prometheus/releases/download/v2.24.1/prometheus-2.24.1.linux-amd64.tar.gz)`. This will download a file onto the system titled "prometheus-2.24.1.linux-amd64.tar.gz". You can use tar to uncompress the archive. The following command would uncompress the archive assuming you have the same version: `tar -xzvf prometheus-2.24.1.linux-amd64.tar.gz`. Now we can rename the directory to remove the version name from the path. Run `mv prometheus-2.24.1.linux-amd64/ prometheus/` to change the directory name. 

## Service Configuration

Now that Prometheus is downloaded we will need to configure the service. We will create a new service file in the `/etc/systemd/system/` directory. There are a few ways to do this but we will use a text editor to create the file. Running `nano /etc/systemd/system/prometheus.service` will create and open the file. Now we will want to configure the service in this file. Below is how we configured the `promethues.service` file. 

```bash
[Unit]
Description=Prometheus Server
Documentation=https://prometheus.io/docs/introduction/overview/
After=network-online.target

[Service]
User=prometheus
Restart=on-failure

ExecStart=/home/prometheus/prometheus/prometheus \
  --config.file=/home/prometheus/prometheus/prometheus.yml \
  --storage.tsdb.path=/home/prometheus/prometheus/data

[Install]
WantedBy=multi-user.target
```

Now that the service is configured we need to enable it on start up so it automatically runs. Before we start the service we will want to run `systemctl daemon-reload` this will do a reload of the services. To enable the service run `systemctl enable prometheus`. Now the service will start on boot. To start the service without rebooting the system run `systemctl start prometheus`. To ensure the service is running run `systemctl status prometheus`. 

## Firewall Configuration

By default the CentOS firewall will not allow the service to be exposed. We will need to allow the traffic through the firewall. To accomplish this run `firewall-cmd --add-port=9090/tcp --permanent`. If you leave off the permanent flag then the firewall rule will be deleted after reboot. Now that the rule has been added we will need to reload the firewall using `firewall-cmd --reload` for changes to take effect. You can use `firewall-cmd --list -all` to see all your firewall rules and `firewall-cmd --state` to see the current status of the firewall. 

## Nginx Install and Configuration

We will use nginx to proxy traffic to port 9090 and put basic authentication on the interface. Run the following commands to install and start the nginx service: `yum install epel-release`, `yum install nginx`, `systemctl enable nginx`, `systemctl start nginx`. Now we will need to create a new firewall rule for the port we will run the nginx redirect on, for this lab we are using port 3030 so the command is `firewall-cmd --add-port=3030/tcp --permanent` and then `firewall-cmd --reload` to reload and apply the changes. Before we continue we need to adjust a selinux policy to allow port binding. To do this we will run the following three commands `yum install policycoreutils -y`, `yum install policycoreutils-python -y`, and `setsebool https_can_network_connect 1 -P`. Now we can begin to configure the nginx configuration for the proxy. Run `nano /etc/nginx/conf.d/prometheus.conf` to create and open the config for this service. In there we want to configure it as show below. 

```bash
server {
    listen 3030;

    location / {
      proxy_pass           http://localhost:9090/;
    }
}
```

Now we will restart nginx to load the new configuration with this command, `systemctl restart nginx` if there are any errors use `journalctl -f -u nginx.service` to check what's wrong. Now we need to update the Prometheus service file. Open the file, like so `nano /etc/systemd/system/prometheus.service`, and adjust the configuration to reflect the one show here.

```bash
[Unit]
Description=Prometheus Server
Documentation=https://prometheus.io/docs/introduction/overview/
After=network-online.target

[Service]
User=prometheus
Restart=on-failure

ExecStart=/home/prometheus/prometheus/prometheus \
  --config.file=/home/prometheus/prometheus/prometheus.yml \
  --storage.tsdb.path=/home/prometheus/prometheus/data
 --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries \
  --web.listen-address=0.0.0.0:9090 \
  --web.enable-admin-api \
  --web.external-url=https://localhost:3030

[Install]
WantedBy=multi-user.target
```

Now we need to reload the daemon using `systemctl daemon-reload` and restart the Prometheus service with `systemctl restart prometheus`. At this point you should make sure you can access Prometheus on port 3030. Next we will add a little bit of security to our setup. First lets configure a password to access the webserver on port 3030. Run the following commands to protect the server with a password, `yum install httpd-tools`, `cd /etc/prometheus`, and `htpasswd -c .credentials admin`. Next we will want to configure SSL. First we need to install some tools for this using `yum install gnutls-utils`. Next we will create and enter a directory using `mkdir /etc/ssl/prometheus && cd /etc/ssl/prometheus`. From here we will run the following two commands to create and setup our certificates, `certtool --generate-privkey --outfile prometheus-private-key.pem` and `sudo certtool --generate-self-signed --load-privkey prometheus-private-key.pem --outfile prometheus-cert.pem`. We configured our certs with the following options: expiration 3650 days, belongs to authority yes, will be used to sign other certs yes, and will be used to sign CRLs yes. Now we need to update the nginx configuration using `nano /etc/nginx/conf.d/prometheus.conf`. Update the configuration to match the one shown below. 

```bash
server {
    listen 3030 ssl;
    ssl_certificate /etc/ssl/prometheus/prometheus-cert.pem;
    ssl_certificate_key /etc/ssl/prometheus/prometheus-private-key.pem;

    location / {
      auth_basic           "Prometheus";
      auth_basic_user_file /etc/prometheus/.credentials;
      proxy_pass           http://localhost:9090/;
    }
}
```

Finally lets restart nginx using `systemctl restart nginx`. You should be able to access Prometheus on port 3030 and be prompted to log in use the username admin and the password you entered when you ran the .htaccess command. 

# Grafana Setup

Now that Prometheus is installed and configured we will configure Grafana. First we will create a repo file using the following command, `vim /etc/yum.repos.d/grafana.repo`. Adjust the file to have the contents show below.

```bash
[grafana]
name=grafana
baseurl=https://packages.grafana.com/enterprise/rpm
repo_gpgcheck=1
enabled=1
gpgcheck=1
gpgkey=https://packages.grafana.com/gpg.key
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
```

Now you need to install Grafana using `yum install grafana-enterprise -y` then enable and start the Grafana service using `systemctl enable grafana-server.service` and `systemctl start grafana-server.service`. Next we will need to allow Grafana through the firewall using `firewall-cmd --add-port=3000/tcp --permanent`. Then reload the firewall with `firewall-cmd --reload`. You should now be able to access Grafana on port 3000 with the default username admin and the default password admin. Change the default credentials form admin admin. Next go to configuration and then data services. You will need to adjust the settings to add Prometheus, make sure to add the nginx login credentials. 

![Untitled](https://user-images.githubusercontent.com/36739730/108148106-22726c80-709e-11eb-9704-15ffbab018b6.png)

# Node Exporter Setup

Node Exporter is what we will be using to share our data with Prometheus. Every Linux device you want to monitor will need to have Node Exporter setup. Like setting up Prometheus we will want to create a Prometheus user account. Run `useradd -m -s /bin/bash prometheus` to add the user the do `su prometheus` to switch to that user. Now make sure wget is installed with `yum install wget`. Now we will want to download the latest version of Node Exporter from [here]([https://github.com/prometheus/node_exporter/releases/](https://github.com/prometheus/node_exporter/releases/)). At the time of writing this we used 1.1.0 as it was the latest version [EDIT: v1.1.1 is the latest]. Use wget to download the file `wget [https://github.com/prometheus/node_exporter/releases/download/v1.1.0/node_exporter-1.1.0.linux-amd64.tar.gz](https://github.com/prometheus/node_exporter/releases/download/v1.1.0/node_exporter-1.1.0.linux-amd64.tar.gz)`. Again we will use tar to uncompress the archive `tar -xzvf node_exporter-1.1.0.linux-amd64.tar.gz` then rename the directory to just `node_exporter`. Next we will need to configure the service for node_exporter. To do this create the service file using `nano /etc/systemd/system/node_exporter.service` and put the following information into the file. 

```bash
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
ExecStart=/home/prometheus/node_exporter/node_exporter

[Install]
WantedBy=default.target
```

Now we need to run the following commands to get the service setup and enabled. `systemctl daemon-reload`, `systemctl enable node_exporter`, and `systemctl start node_exporter.service`. We will also need to allow the service through the firewall using `firewall-cmd --add-port=9100/tcp --permanent` and `firewall-cmd --reload`. **Now we will need to adjust the Prometheus configuration on the Prometheus host** **NOT THE NODE CLIENT**. One the Prometheus server adjust the `/home/prometheus/prometheus/prometheus.yml` to make it look like the below config. 

```bash
- job_name: 'node_exporter'
    static_configs:
      - targets: ['localhost:9100']
```

To add more clients change `['localhost:9100']` to `['localhost:9100','IP:91000']`. Restart the Prometheus service. Now restart the Prometheus service using `systemctl restart prometheus`. Now you will be able to see the client in the web interface under status, targets. 

# Windows Exporter

Windows Exporter works the same as Node Exporter except its designed for Windows instead of Linux. You will need to download the latest release from [here]([https://github.com/prometheus-community/windows_exporter/releases](https://github.com/prometheus-community/windows_exporter/releases)). Run the installer and then ensure the service is running and set to automatic start up in `services.msc`. You can navigate to the system on port 9182 to see if the service is working correctly. **Now we will need to adjust the Prometheus configuration on the Prometheus host** **NOT THE NODE CLIENT**. One the Prometheus server adjust the `/home/prometheus/prometheus/prometheus.yml` to make it look like the below config.

```bash
- job_name: 'windows_exporter'
    static_configs:
      - targets: ['iphere:9182']
```

Adding additional endpoints is the same as Node Export. Run `systemctl restart prometheus` to restart the service on the Prometheus server. Now you will be able to see the client in the web interface under status, targets. 

![Untitled 1](https://user-images.githubusercontent.com/36739730/108148107-22726c80-709e-11eb-823b-7fc981b22939.png)

# Grafana Graph Configuration

Now that we are shipping data to Prometheus and in turn Grafana we can start to setup chats and graphs. Log into Grafana and select the plus sign (+) then select import. 

![Untitled 2](https://user-images.githubusercontent.com/36739730/108148089-20a8a900-709e-11eb-9ca4-38f87bbd3ac2.png)

## Windows Dashboard

We will be making a graph for our Windows systems first. In the import via [grafana.com](http://grafana.com) box enter 13466. The code 13466 corresponds with a template on the Grafana website. If you want to see the dashboard information you can go [here]([https://grafana.com/grafana/dashboards/13466](https://grafana.com/grafana/dashboards/13466)). Ensure that you select Prometheus as the data source.

![Untitled 3](https://user-images.githubusercontent.com/36739730/108148091-21413f80-709e-11eb-975f-826433bfaa57.png)

To see the dashboard you just imported select windows_exporter under the job dropdown menu. Once you select the job you can select a specified host. Below is an example of what the windows_exporter dashboard looks like. 

!![Untitled 4](https://user-images.githubusercontent.com/36739730/108148092-21413f80-709e-11eb-95bc-d22553b31059.png)

## Linux Dashboard

The process to creating a Linux dashboard is almost identical to a Windows dashboard. Select the plus button and then hit import. From there enter 1860 into the import via [grafana.com](http://grafana.com) box. This will load the dashboard found [here]([https://grafana.com/grafana/dashboards/1860](https://grafana.com/grafana/dashboards/1860)). Make sure to select Prometheus as the data source. To view the graph you select node_exporter under the job dropdown menu and then select the host. 

![Untitled 5](https://user-images.githubusercontent.com/36739730/108148093-21413f80-709e-11eb-9b92-da0b1c9c93bd.png)

## Alerting with Discord

Having Graphs is nice and provides you with a good overview of information however you only get the notifications when you open up the dashboard. Now we will configure alerts so you can get notifications of important events without having to keep Grafana open 24/7. We will be configuring Grafana to send alerts through a Discord web hook. First we will need to create a new dashboard for alerts. Click the plus icon and create a new dashboard. Now add a new panel to the dashboard. This first panel will alert when a system goes down. To do this we will monitor RAM and if it hits 0% the system has gone down. We need to do these separate for Windows and Linux and will begin with Windows.

![Untitled 6](https://user-images.githubusercontent.com/36739730/108148096-21413f80-709e-11eb-8f66-3b192d76a142.png)

You will want to add the following to the query field. Adjust the where it says IP and put your Windows system's IP address.

```bash
100 - (windows_os_physical_memory_free_bytes{job=~"windows_exporter",instance=~"IP:9182"} / windows_cs_physical_memory_bytes{job=~"windows_exporter",instance=~"IP:9182"})*100
```

Now select the axes section and configure it with the following options; show: yes, unit: percent (0-100), scale: linear, y-min: 0, y-max: 100, decimals: auto

![Untitled 7](https://user-images.githubusercontent.com/36739730/108148098-21d9d600-709e-11eb-8be3-a7e1bc5f87f5.png)

Now under the field panel select percent (0-100) under the unit dropdown menu. When you are done it should look similar to this.

![Untitled 8](https://user-images.githubusercontent.com/36739730/108148099-21d9d600-709e-11eb-85f7-d72de8473dbc.png)

Now that the panel is setup we can begin to configure the alert. Select the alert tab and configure it as shown in the screenshot below. 

![Untitled 9](https://user-images.githubusercontent.com/36739730/108148100-21d9d600-709e-11eb-83fd-47edf5a2ede1.png)

Now save the dashboard and then go to the alerting page (bell icon). From here you will want to select notification channels and then add channel. From the dropdown link select discord and then past your web hook URL.


![Untitled 10](https://user-images.githubusercontent.com/36739730/108148101-22726c80-709e-11eb-932f-7c04969d1369.png)

Select the test button to ensure the web hook is functioning correctly and then save the channel. Configuring the Linux web hook would be almost identical to this except the query text would be this:

```bash
100 - ((node_memory_MemAvailable_bytes{instance="IP:9100",job="node_exporter"} * 100) / node_memory_MemTotal_bytes{instance="IP:9100",job="node_exporter"})
```

Again make sure to put your system's IP where it says IP. You may want to include an image of the graph with your alert. To do this we will need to install a plugin for Grafana. To install the plugin run `grafana-cli plugins install grafana-image-renderer`. You may need to install the dependencies. If you are unsure run the following to install all the dependencies at once. 

```bash
sudo yum install libXcomposite libXdamage libXtst cups libXScrnSaver pango atk adwaita-cursor-theme adwaita-icon-theme at at-spi2-atk at-spi2-core cairo-gobject colord-libs dconf desktop-file-utils ed emacs-filesystem gdk-pixbuf2 glib-networking gnutls gsettings-desktop-schemas gtk-update-icon-cache gtk3 hicolor-icon-theme jasper-libs json-glib libappindicator-gtk3 libdbusmenu libdbusmenu-gtk3 libepoxy liberation-fonts liberation-narrow-fonts liberation-sans-fonts liberation-serif-fonts libgusb libindicator-gtk3 libmodman libproxy libsoup libwayland-cursor libwayland-egl libxkbcommon m4 mailx nettle patch psmisc redhat-lsb-core redhat-lsb-submod-security rest spax time trousers xdg-utils xkeyboard-config -y
```

Now you will need to restart Grafana with `systemctl restart grafana` for the changes to take effect. Now your alerts will look like the one shown below.


![Untitled 11](https://user-images.githubusercontent.com/36739730/108148102-22726c80-709e-11eb-845f-6dc1736b4b77.png)

The final alert that we are going to setup is for Windows disk usage. Again this is the same as configuring the RAM alert. The query text for this alert will be the following.

```bash
100 - (windows_logical_disk_free_bytes{job=~"windows_exporter",instance=~"IP:9182",volume="C:"} / windows_logical_disk_size_bytes{job=~"windows_exporter",instance=~"IP:9182",volume="C:"})*100
```

For this alert we set it to notify when disk usage goes over 40% but you can configure as you please. The disk usage alert will look something like the following screenshot. 

![Untitled 12](https://user-images.githubusercontent.com/36739730/108148105-22726c80-709e-11eb-8aa3-ff85e1e088c5.png)

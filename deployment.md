# Deploying and configuring Grafana to monitor an Elys node

## 1. Installation and Configuration - Server Side

The following will let you deploy Grafana Enterprise (free edition) on a standalone server running Ubuntu. This software isn't resource-intensive and can run on a small server, e.g. a **CAX11** (2 vCPUs, 4G RAM) on Hetzner -- while Hetzner has an anti-crypto policy and it is not recommended to run a node on one of their machines, a monitoring tool does not fall under this restriction.<br>
It can also absolutely be deployed on a Raspberry Pi at home; with the appropriate `arm64` version.<br>
The steps described below allow you to set up a basic instance; there are more sophisticated methods (e.g. using Docker, Alert Manager, etc.), dashboards, visualizations and alert rules, but once you get the hang of it you will be able to explore the options on your own.

You can refer to the [node deployment tutorial](https://github.com/elys-network/cosmovisor/blob/main/tutorial_validator_setup.md) to set up the server -- as always, it is best to create a limited account and disable the root login (log in with that account and acquire elevated privileges).

Let's install the necessary packages (you can check the full documentation [here](https://grafana.com/docs/grafana/latest/setup-grafana/installation/debian/)):

**As root**:

```
apt update && apt upgrade -y && apt install -y apt-transport-https software-properties-common wget vim
wget -q -O /usr/share/keyrings/grafana.key https://apt.grafana.com/gpg.key
echo "deb [signed-by=/usr/share/keyrings/grafana.key] https://apt.grafana.com stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
echo "deb [signed-by=/usr/share/keyrings/grafana.key] https://apt.grafana.com beta main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
apt update
apt install grafana-enterprise
```

The above will add the Grafana repository to the `apt` sources and install Grafana Enterprise, including the corresponding `systemd` service.<br>

We should now configure a few items. The configuration file when using Debian/Ubuntu is `/etc/grafana/grafana.ini`. While Grafana should work out of the box, you can tweak a few items.<br>
Open it with your favorite text editor.

***Note***: a semi-colon `;` at the beginning of a line means that it is commented out. Remove it to uncomment.

By default, the Grafana interface will run on port 3000, in `http` -- once your firewall is configured to open port 3000, you can access it at `http://your_ip:3000`.<br>
However, it is best to run it in `https` so that the data is encrypted. Nevertheless, this requires a valid SSL certificate for this server, with a domain name.
<br>
A simpler option (with a couple downsides) is to use a **self-signed certificate**. This will allow the connection from your browser to be encrypted, although with a security warning since the certificate won't be signed by an official authority.

If you decide to go with this option, exit the file and follow the steps below:<br>
First off, let us create this certificate by running:

```
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/grafana/selfsigned.key -out /etc/grafana/selfsigned.crt
```


(`--days 365` means that the certificate will be valid for one year, adjust as you see fit)

Then make sure these files are readable by the `grafana` user that was automatically created during the installation:

```chown grafana:grafana /etc/grafana/selfsigned.*```

Edit `/etc/grafana/grafana.ini` again and update the following:<br>

In the `[server]` section:
```
protocol = https
```
If you want the interface to be accessible on a custom port:
```
http_port = port number of your choice
```
Don't forget to update the firewall accordingly: inbound connections to this port must be allowed.
<br>

```
cert_file = /etc/grafana/selfsigned.crt
cert_key = /etc/grafana/selfsigned.key
```
If you have a valid SSL certificate and the server IP is associated to your domain, you can also update the following line, else leave it commented out.
```
root_url = https://your_domain_name:your_custom_port
```

Next, edit the following:

In the section `[unified_alerting]`:
```
enabled = true
```
In the section `[alerting]`:
```
enabled = false
```

If you would like to log into the interface using your Google or Github account, you can configure that in the relevant sections further in the file.<br>

Else, save and exit, then restart the service with `systemctl restart grafana-server`.

The Grafana interface should now be reachable at `https://your_ip:your_custom_port/`, with the warning regarding the SSL certificate that was mentioned earlier. You can ignore this warning and proceed (on Chrome or Firefox, click on **Advanced** then **Continue**).

![image](https://github.com/elys-network/grafana/assets/65887967/4f5fa112-bad3-43f4-9553-c839565a00cc)


Log in using the default credentials: `username = admin, password = admin`.<br>
The interface will immediately prompt you to set up a new password, and once done, you will land on the main page.<br>

You can click on the menu (the usual "hamburger" icon on the top left) and update this admin user details, create others, etc. Nothing is really important at this point.<br>
Perhaps you can improve the overall readability of the interface by switching the colorscheme: *Default Preferences* --> *UI Theme* --> set to *Light* and hit the Save button.

![image](https://github.com/elys-network/grafana/assets/65887967/dc283933-38d8-42a7-acd7-0a32516cb4c3)

For now, we are done here!

## 2. Configure of the node and server

For Grafana to connect to the node and retrieve useful data, a few configuration items and packages are required.<br>
Connect to the node server, and:

- Edit `/home/elys/.elys/config/config.toml` and at the bottom of the file in section `[instrumentation]`, set `prometheus = true`.<br>Keep the default `prometheus_listen_addr` as it is or change the port as you see fit.<br>Save and exit the file, then restart the node.
 
- If you run `curl localhost:26660` now, it will output a lot of text: these are the metrics exposed by the node, and that Grafana will retrieve and display.

- Now we need to install two packages: `Prometheus` and `node_exporter`. The latter will allow us to feed Grafana with the system metrics: CPU usage, memory, disk space...
- Let's run the following commands to install and configure these packages, starting with Prometheus:
```
curl -s https://api.github.com/repos/prometheus/prometheus/releases/latest | grep browser_download_url | grep linux-amd64 | cut -d '"' -f 4 | wget -qi -
tar xf prometheus*.tar.gz
cd prometheus*/
```
We need to edit `prometheus.yml` and update the line at the bottom so that it becomes `- targets: ["localhost:9100","localhost:26660"]`.

Either do it manually or run the below command:
```
sed -i 's/"localhost:9090"/"localhost:9100","localhost:26660"/g' prometheus.yml
```
(Changing default port 9090 to 9100 is to avoid a conflict with node_exporter's default port).

Follow with:

```
groupadd --system prometheus
useradd -s /sbin/nologin --system -g prometheus prometheus

mkdir /var/lib/prometheus
for i in rules rules.d files_sd; do sudo mkdir -p /etc/prometheus/${i}; done

mv prometheus promtool /usr/local/bin/
mv prometheus.yml /etc/prometheus/prometheus.yml
mv consoles/ console_libraries/ /etc/prometheus/

for i in rules rules.d files_sd; do sudo chown -R prometheus:prometheus /etc/prometheus/${i}; done
for i in rules rules.d files_sd; do sudo chmod -R 775 /etc/prometheus/${i}; done
chown -R prometheus:prometheus /var/lib/prometheus/
```

Finally, create the systemd service. Edit `/etc/systemd/system/prometheus.service` and type in the following:

```
[Unit]
Description=Prometheus
Documentation=https://prometheus.io/docs/introduction/overview/
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
User=prometheus
Group=prometheus
ExecReload=/bin/kill -HUP \$MAINPID
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries \
  --web.listen-address=0.0.0.0:9090 \
  --web.external-url=

SyslogIdentifier=prometheus
Restart=always

[Install]
WantedBy=multi-user.target
```

Save and exit, and let's do the same with Node Exporter:

```curl -s https://api.github.com/repos/prometheus/node_exporter/releases/latest | grep browser_download_url | grep linux-amd64 | cut -d '"' -f 4 | wget -qi -
tar xf node_exporter*.tar.gz
mv node_exporter*/node_exporter /etc/prometheus
```

The systemd service again: edit `/etc/systemd/system/node_exporter.service` and fill it out:

```
[Unit]
Description=Prometheus - Node Exporter
Documentation=https://prometheus.io/docs/introduction/overview/
Wants=network-online.target
After=network-online.target
Before=prometheus.service

[Service]
Type=simple
User=prometheus
Group=prometheus
ExecReload=/bin/kill -HUP $MAINPID
ExecStart=/etc/prometheus/node_exporter

SyslogIdentifier=prometheus
Restart=always

[Install]
WantedBy=multi-user.target
```

And now we can activate everything:

```
systemctl daemon-reload
systemctl enable prometheus
systemctl start prometheus
systemctl enable node_exporter
systemctl start node_exporter
```

Finally, we can clean up the files that we won't need any further:

```
cd .. 
rm -r prometheus-*
```

**You *MUST* configure the firewall so that connections from the Grafana server to port 9090 on the node's machine are allowed**

**Do *NOT* leave this port open to all IPs, otherwise everyone case see everything about your node**

Verify that it is working by running the following from the Grafana server: `curl node_server_ip:9090`.<br>
This should display `<a href="/graph">Found</a>.`. If it does not and just hangs, verify the firewall on the node server.

## 3. Retrieve the metrics in Grafana

- Back on the Grafana interface, open the Menu and go to **Connections** --> **Data sources**, then click on **Add data source**.
- Select the first item, **Prometheus**.
- Here you can set a name that suits you, e.g. **Prometheus - Elys**, and fill out the second line (*Prometheus server URL*) with `http://node_server_ip:9090`.
- Now scroll down and hit **Save & test**, which should display a success message. If not: verify the firewall, the prometheus and node_exporter services on the node server.

## 4. Create a dashboard

- We provide a dashboard template to help you kickstart your monitoring by analyzing a few important metrics: download it [here](https://github.com/elys-network/grafana/blob/main/grafana_template.json)
- Menu --> Dashboards --> Import --> Select the downloaded .json file.
- Alternatively, you can try some [community templates](https://grafana.com/grafana/dashboards/?search=cosmos&dataSource=prometheus) -- open the one you want and copy its **ID**, then paste it in the `Import via grafana.com` line and click on **Load**.
- You can also build your own dashboard from scratch.

- Once you imported the dashboard, and if your Prometheus instance is running on the default port 26660, you should immediately see some data in the different panels. If you have set up a different port, you'll need to edit the panels. We'll get to that below.
- The **Disk Space | Volume** panels may display no data, if you have no additional volume on the server or its mountpoint isn't `/dev/sdb`.
- If so:  
  - Click on the 3 vertical dots at the upper right of the panel, then on **Edit**.
  - In the **Metrics browser** line where it currently says `node_filesystem_avail_bytes{mountpoint="/dev/sdb"}/1073741824`, update the path to match the correct one and click on **Run queries** to verify that it works.
  - Click Apply or Save in the upper right, else just hit Escape.
- If you are running your node on another port than 26660, edit the concerned panels and adjust the port accordingly. 
- Once done, click on the small **Save** icon on the upper right (which resembles an old 3"5 floppy disk) and confirm. Do not omit this step or your changes since the last save will be lost. 

## 5. Alerting

The next and hopefully final step is to configure Grafana so that you receive an alert in case of an issue.

We'll show how to set up Discord alerts -- Grafana supports a wide range of communication channels and this is only one example.<br>
You need to have an administrator access to a Discord channel. Best is to have your own server and set up a private channel, as you don't want your alerts to be public: right click on the channel --> **Edit Channel** --> **Integrations** --> **New webhook** --> copy the address.

- We'll start with creating an alert template: **Menu** --> **Alerting** --> **Contact points** --> **Add template**
  - Give a name to your template, e.g. *Discord*
  - In the **Content** field, paste the following:

`````
{{ define "discord.default.description" }}
{{ "ALERTING" }}
{{ range . }}
{{ .Annotations.description }}
{{ with .ValueString }}
{{- . | reReplaceAll `\[\s` "" | reReplaceAll `\],\s` ""  | reReplaceAll `\]` "" | reReplaceAll `labels=.*}` "" | reReplaceAll `value=([0-9\.]+)` "**$1**" }} 
        {{ end }}
{{ end }}
{{ end }}
{{ define "discord.resolved.description" }}
{{ "RESOLVED" }}
{{ range . }}
{{ .Annotations.description }}
{{ with .ValueString }}
{{- . | reReplaceAll `\[\s` "" | reReplaceAll `\],\s` ""  | reReplaceAll `\]` "" | reReplaceAll `labels=.*}` "" | reReplaceAll `value=([0-9\.]+)` "**$1**" }} 
        {{ end }}
{{ end }}
{{ end }}
{{ define "discord" }}
        {{ if gt (len .Alerts.Firing) 0 }}{{ template "discord.default.description" .Alerts.Firing }}{{ end }}
        {{ if gt (len .Alerts.Resolved) 0 }}{{ template "discord.resolved.description" .Alerts.Resolved }}{{ end }}
{{ end }}
`````

- This uses the Go templating language: check the [documentation](https://grafana.com/docs/grafana/latest/alerting/manage-notifications/template-notifications/) to learn how to customize the alerts.

- Save the template, then

<br>

- Click on **Add contact point**
  - Name it as you wish, and in the `Integration` list select `Discord`, then paste your webhook address in the relevant field.
  - You can already click on the **Test** button, and you should see a default message land in that channel.
  - You can leave the **Title** field as it is or customize it.
  - Next up, you can customize a bit, for example you can ensure that specific users are tagged in the messages: in the **Message Content** field, define the message followed by the user id, e.g.:<br>
`{{ template "discord.message" . }}  <@675319125258595896>  <@951375404471185732>`
    - ⚠ Make sure to set the template name properly (the one we create above). 
    - You can get your user id from the Discord app by clicking on your username at the bottom of the window, then clicking on **Copy User ID**.
    - If you click on **Test** again, you will see that the users specified are now tagged.
  - You can also disable the *Resolved* message when the metrics return to normal by clicking on the next checkbox. This may be relevant for some types of alerts, such as missed blocks or a change in voting power.<br> In other cases you want to know if the situation is back to normal (e.g. node is down), so you can set up **2 different contact points** that will be identical except for that parameter, called *Default* and *No_resolved* for example.
  - Save the contact points.

<br>

Now we'll create the alert rules for some of the dashboard's panels. **Note that the rules and algorithms we'll propose here are extremely basic and serve as examples. We recommend that you check the capabilities of the software and test variations and combinations of the functions -- you can build very precise rules once you understand how it works.**

- Starting with the **Up** panel:
  - Click on the **Alert** tab, then on the button **Create alert rule from this panel**.
  - In *Section 1* you can set a custom name for this alert.
  - In *Section 2*, edit the **time range** data, by default *now-12h to now*, which means that the rule evaluates the metrics on the last 12h which is not ideal.
  - Set the **From** value to `now-10s`, which means that the rule will check the metric value over the last 10 seconds, then click on **Apply time range**.
  - Below, next to **B**, click on *Reduce* and select *Classic condition*
   - Set the rule as follows: `WHEN avg() OF A IS BELOW 1`
   - Delete the condition **C** on the right
  - In *Section 3*, click on the drop-down list below **Folder** and either click out immediately to use **General**, or define a new folder e.g. Elys.
  - Evaluation group: click on the drop-down list and write **Up** (or whatever name you want to give it), then Enter.
  - Evaluate every: **10s**
  - For: **30s** (if the node is down for more than 30s, an alert will be sent. You can set that to 0s if you want to be alerted immediately, but sometimes there's simply a network disturbance and it would cause a false alert)
  - **Configure no data and error handling**: set **Alerting** for both. --> This way, you'll be alerted if the server goes down.
  - Next, you may want to add some text to the alert message: click on **Add annotation** --> **Description** --> write the text of your choice, e.g. "Elys is down"

   ![image](https://github.com/elys-network/grafana/assets/65887967/0de82ec7-dc05-43a2-a844-c199f0eaaab4)

  - At the top of the page, click on **Save rule and exit**

<br>

- **Missed Blocks** panel:
  - Time range: **now-5m**
  - Condition: *Classic condition* -->  `WHEN diff_abs() OF A IS ABOVE 5` --> alert only if the node missed more than 5 blocks in the last 5 minutes (change the value as you like)
  - Evaluate every: **5m**
  - For: **10s**
  - **Configure no data and error handling**: set **Alerting** for both.
  - Set a custom description
  - In *Section 6*: this is an alert where it is best to not receive a *Resolved* notification:
   - In the *Choose key* field, select **Contact**, and in *Choose value*, select *No_resolved*
  - Save & exit.
 
<br>

- **Block Height**: this rule will alert if the node stops minting new blocks.
  - Time range: **now-1m**
  - Condition: *Classic condition* -->  `WHEN diff() OF A IS BELOW 1` --> will alert if no new block was minted in the past minute.
  - Evaluate every: **1m**
  - For: **0s**
  - **Configure no data and error handling**: set **Alerting** for both.
  - Set a custom description
  - Save & exit.    

<br>

- **Signed Blocks**: this rule will alert if the node stops signing blocks, indicating that it is either jailed or fell out of the active set.
  - Time range: **now-30**
  - Condition: *Classic condition* -->  `WHEN diff() OF A IS BELOW 1`.
  - Evaluate every: **10s**
  - For: **1m**
  - **Configure no data and error handling**: set **Alerting** for both.
  - Set a custom description
  - Save & exit.

<br>

 - **Total Voting Power**: this rule will alert when the voting power has increased or decreased by more than 5% (adjust the value as you like).
   - Time range: **now-2m**
   - Condition: *Classic condition* -->  `WHEN percent_diff() OF A IS OUTSIDE RANGE -5 to 5`.
   - Evaluate every: **1m**
   - For: **0s**
   - **Configure no data and error handling**: set **Alerting** for both.
   - Set a custom description
   - In *Section 6*: in the *Choose key* field, select **Contact**, and in *Choose value*, select *No_resolved* 
   - Save & exit.

<br>

 - **Disk Space** panel(s): alert when the remaining available space falls below 5G (or more, or less: it has to give you enough time to resolve the issue before the disk is full, else the node will stop).
   - Time range: **now-30m**
   - Condition: *Classic condition* -->  `WHEN avg() OF A IS BELOW 5`.
   - Evaluate every: **10m**
   - For: **10m**
   - **Configure no data and error handling**: set **Alerting** for both.
   - Set a custom description
   - Save & exit
  
 <br>
 You can then add other rules on the remaining panels, to alert you when the CPU load or memory usage exceeds a threshold for example.


 Your monitoring solution is now operational and you will be alerted whenever a problem arises with your node or server!
 
    

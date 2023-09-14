# RaspberryPi-IDS-Suricata-ELK-Setup

<br><br>

## About


While configuring Suricata on a Raspberry Pi to forward logs to a SIEM, I encountered a lack of  guides tailored to the Raspberry Pi OS. As a result, this guide was created.

Within this guide, I will outline the steps for setting up Suricata on a Raspberry Pi. Subsequently, we will employ Ansible to streamline the installation of the ELK (Elasticsearch, Logstash, Kibana) stack for in-depth log analysis.

Lastly, we will incorporate Filebeat to facilitate the efficient transmission of Suricata logs to the ELK stack, enhancing visualization and analytical capabilities.
<br><br>

### Goals
-------------

- Headless Rasberry Pi OS setup.

- Suricata - installation & rules 

- ELK stack - OS = Ubuntu | Install = Ansible | Collector = Filebeat

<br><br>



### Outline
-------------
- Rasberry PI
  - OS installation
  - ssh setup

- Suricata
  - Install
  - Configure
  - Rules
    - Enable 
    - Add 
    - Update

- Elasticsearch, Logstash, Kibana (ELK)
  - CentOS & Ubuntu set up
  - Ansible ELK playbook
  - Filebeat 
    - compile for ARM64 (docker)
    - suricata module
  - Kibana



<br><br>
### Rasberry Pi
-------------

#### Rasbperry Pi OS
 
1.  Raspberry Pi imager:
 
	- https://www.raspberrypi.org/downloads/

 ![alt text](https://github.com/Ab-Mhd/rbp_wkt_img/blob/main/ex/16.jpg)

<br><br>

2. Enter details *(settings button - bottom right)*
	- ssh key (your .pub file)
	- wifi (SSID & password)

![alt text](https://github.com/Ab-Mhd/rbp_wkt_img/blob/main/ex/25.jpg)

<br><br>
4. Flash image
<br><br>
3. ssh to Rasberry Pi

`ssh -i id.pub ras.p`

*(Make an entry to /etc/hosts file to avoid having to reenter ip. eg.  192.168.2.2  ras.p)*
<br><br>


#### Suricata installation
-------------
Due to raspbianOS being built on Debian we can use apt package manager for installataion.
Paste the following into terminal to update system and install surticata.

`sudo apt-get update && sudo apt-get upgrade -y`<br> 
`sudo apt-get install suricata`
<br><br>
Upon completion, any of these command to modify or check the service.

`sudo systemctl start|stop|restart|status suricata`

<br>

After starting the service a status check should output the following:


![alt text](https://github.com/Ab-Mhd/rbp_wkt_img/blob/main/ex/18.jpg)

<br><br>
### Configuring Suricata
-------------
Before continuning we need to find the the wifi interface and set it to promoicious mode. This allows the interface to "listin" to traffic on the local network.

<br>
Use the `ipconfig` command to find your wifi interface.
<br><br>

Turn on promisicous mode:

`sudo ifconfig *interface* promisc`
<br>
Check interface:

![alt text](https://github.com/Ab-Mhd/rbp_wkt_img/blob/main/ex/19.jpg)

<br>

#### Variables to edit:
<br>
Configuration options are located in /etc/suricata.yml.
Open suricata.yml and alter the following variables to match your enviroment.

*Note: Always a good idea to a make a backup of the original incase any errors occur while editing* 
<br><br>
- HOME_NET: "[local  network/CIDR]"

	  #HOME_NET: "[192.168.0.0/16,10.0.0.0/8,172.16.0.0/12]"
	  HOME_NET:  "[192.168.2.0/24]"
<br><br>
 
- Interface: *(default is eth0)*

  		af-packet:
		- interface: wlan0
<br><br>

- Default rule set:

		default-rule-path: /var/lib/suricata/rules

		rule-files:`<br>
 		 - suricata.rules`

<br>
After editing configuration file, we need to check that none of the edits made damaged the yml file. *Remember the backup suggestion?*

check configuration validity:

	sudo suricata -T -c /etc/suricata/suricata.yaml -v
<br>
Expected output:

![alt text](https://github.com/Ab-Mhd/rbp_wkt_img/blob/main/ex/20.jpg)


<br>
Restart suricata:

	sudo systemctl restart suricata

<br>
### Configuring rules

<br>

Running `sudo suricata-update` will add the ET open ruleset.

To add and enable different rulesets:
<br>


Enable free rulesets<br>


	 suricata-update enable-source oisf/trafficid
	 suricata-update enable-source sslbl/ssl-fp-blacklist
	 suricata-update enable-source tgreen/hunting
	 suricata-update enable-source tgreen/network_intel
	 suricata-update enable-source nsm2019/scan
<br>
List enabled sources:
		
  	suricata-update list-enabled-sources


<br>

### Logs
-------------

Logs are located at 

	/var/log/suricata/*

To test suricata well use a curl command to simulate an attack and then grep the alert from our logs.
<br><br>
Simulate attack:

	curl http://testmynids.org/uid/index.html
<br>
Search logs for alert:

	grep 2100498 /var/log/suricata/fast.log
<br>
Expected output:

	[**] [1:2100498:7] GPL ATTACK_RESPONSE id check returned root [**] [Classification: Potentially Bad Traffic] [Priority: 2] {TCP} 2600:9000:2000:4400:0018:30b3:e400:93a1:80 -> 2001:DB8::1:34628

<br>

### Ansible
-------------

Using an ansible play book we will automate the installation of the ELK stack.

For this we'll need two VM's.
 - Ubuntu 18.x (ELK)
 - CentOs 7 (Ansible)

For this portion there is a well set out guide that we can follow.

https://github.com/lmakonem/ELK-SIEM-Ansible-Playbook

<br><br>

### Filebeat
-------------
Due to the raspbbery Pi using an ARM64 build, convential methods of installing file beat wont work.

Solution: Using docker and RT-elastic repo to build and configure filebeat.


#### Download source code
	
 	wget https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-7.6.0-linux-x86.tar.gz
<br>

#### Add docker-ce repo
	
 	sudo apt install apt-transport-https ca-certificates curl gnupg2 software-properties-common
	wget -q -O - https://download.docker.com/linux/debian/gpg | sudo apt-key add -
	sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/debian stretch stable
	apt update
 <br>

#### Install docker-ce

	sudo apt install docker-ce
<br>

#### Debian go container

	docker run -it --rm -v `pwd`:/build golang:1.14 /bin/bash
<br>

 #### Create Filebeat ARM modules
	(execute in go container)

  	go get github.com/elastic/beats
	cd /go/src/github.com/elastic/beats/filebeat/
	git checkout v7.6.0
	GOARCH=arm go build
	cp filebeat /build/filebeat-arm
	GOARCH=arm64 go build
	cp filebeat /build/filebeat-arm64
	exit
<br>

In order to automate the installation we can use the following repository

	git clone https://github.com/rothirschtec/RT-Blog-elastic.git
<br>

### filebeat.yml

Find the configuration folder at ../filebeat-latest/filebeat.yml

Now we copy it to the home directory

(Note: Don't change any of the names or the script wont work)

	cp ../filebeat-latest/filebeat.yml my-filebeat.yml
<br>

Now use the text editor of your choice to change the variables to suit your enviroment.

(Make sure to change "localhost" to the the IP address of your Ubuntu machine.)
<br>

#### Suricata module

Next we can use the prebundeled suricata module to ease integration.

	cp filebeat-latest/modules.d/suricata.yml.disabled my-suricata.yml


Update file contents to:

	- module: suricata
	  # All logs
	  eve:
	    enabled: true
	
	    # Set custom paths for the log files. If left empty,
	    # Filebeat will choose the paths depending on your OS.
	    var.paths: ["/var/log/suricata/eve.json"]

Now navigate to the RT-elastic directory.

To run script:

	sudo ./install
 
*Note: If you encounter any errors, try manually setting the {id} variable to location of filebeat-latest.*

<br>

Check filebeat installation:
	
 	sudo systemctl restart filebeat
	sudo systemctl status filebeat
<br>

Expected output:

![](https://github.com/Ab-Mhd/rbp_wkt_img/blob/main/ex/21.jpg)


Check module

Visit this link and scroll to the bottom and press the Check data button.

	http://your_elkstack_ip:5601/app/kibana#/home/tutorial/suricataLog 

If everything has been set up properly you should see a success message:

![](https://github.com/Ab-Mhd/rbp_wkt_img/blob/main/ex/22.jpg)
<br>
<br>


### Kibana
-------------

Time to visualize our data!

Click the "Suricata logs dashboard" button at the bottom of the page we used to check our modules connection.

Type in suricata into the search bar, youll be greeted by the following dashboards, you can pick either.


![](https://github.com/Ab-Mhd/rbp_wkt_img/blob/main/ex/23.jpg)


Dashboard:


![](https://github.com/Ab-Mhd/rbp_wkt_img/blob/main/ex/24.jpg)











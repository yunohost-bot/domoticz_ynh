<!--
N.B.: This README was automatically generated by https://github.com/YunoHost/apps/tree/master/tools/README-generator
It shall NOT be edited by hand.
-->

# Domoticz for YunoHost

[![Integration level](https://dash.yunohost.org/integration/domoticz.svg)](https://dash.yunohost.org/appci/app/domoticz) ![Working status](https://ci-apps.yunohost.org/ci/badges/domoticz.status.svg) ![Maintenance status](https://ci-apps.yunohost.org/ci/badges/domoticz.maintain.svg)  
[![Install Domoticz with YunoHost](https://install-app.yunohost.org/install-with-yunohost.svg)](https://install-app.yunohost.org/?app=domoticz)

*[Lire ce readme en français.](./README_fr.md)*

> *This package allows you to install Domoticz quickly and simply on a YunoHost server.
If you don't have YunoHost, please consult [the guide](https://yunohost.org/#/install) to learn how to install it.*

## Overview

Domoticz is a Home Automation system design to control various devices and receive input from various sensors.
For example this system can be used with: 

* Light switches
* Door sensors
* Doorbells
* Security devices
* Weather sensors like: UV/Rain/Wind Meters
* Temperature Sensors
* Pulse Meters
* Voltage / AD Meters
* And more ...


**Shipped version:** Always the last stable one. The last compiled version is retrieved from [this directory](https://releases.domoticz.com/releases/?dir=./release) during install.
Once installed, **updates from the uptream app are managed from within the app**. Yunohost upgrade script will only upgrade the Yunohost package. 

The MQTT broker mosquitto is integrated into the package. It requires its own domain or subdomain. It's an optional setting: during install if you set the same domaine as your main app domain, it won't be installed.

## Configuration

### Broker Mosquitto

During installation, a [MQTT](https://en.wikipedia.org/wiki/MQTT) broker, [Mosquitto](https://mosquitto.org/), is installed at the same time as Domoticz. The installed version is the one from the official project repo and not from Debian ones.
This broker requires a dedicated domain or subdomain to work (ex : mqtt.your.domain.tld) : creating this domain prior installation is a prerequisite

#### Adding in domoticz

To use mosquitto, you need to customize the communication between domoticz and the broker by following the [domoticz documentation](https://www.domoticz.com/wiki/MQTT#Installing_Mosquitto), part *Add hardware "MQTT Client Gateway"*.
User and password are automatically generated during installation, you may retrieve them with
````
sudo yunohost app setting domoticz mqtt_user
sudo yunohost app setting domoticz mqtt_pwd
````

#### Publish/Subscribe

By default, mosquitto will listen on 2 ports:
- 1883 on localhost using mqtt protocol
- 8883 using websocket protocol. Nginx redirect external port 443 to this internal port.

Hence, To publish/subscribe on a topic from the outside, you have to use a software supporting websocket protocol (ex : paho python library).

#### Mosquitto_pub et mosquitto_sub

These 2 tools do not support websocket protocol, only direct mqtt: base settings will not allow communication from an outside device.
If you're using them directly from your server, this kind of syntax should work:
````
mosquitto_pub -u *user* -P *password* -h mqtt.your.domain.tld -p 1883 -t 'domoticz/in' -m '{ "idx" : 1, "nvalue" : 0, "svalue" : "25.0" }'
````
In the same way:
````
mosquitto_sub -u *user* -P *password* -h mqtt.your.domain.tld -p 1883 -t 'domoticz/out'
````

If you wish to open direct mqtt protocol from an outside device, you'll need to:
- open port 1883 on Yunohost firewall (**Attention, security risk**)
- Allows IP addresses in mosquitto configuration for this listener
- Set the tls setting in mosquitto configuration by giving access to crt.pem and key.pem from your mqtt domain by setting respective certfile et keyfile variables. **This is mandatory to ensure a secure connection.**

#### Upgrade from version without mosquitto
If you have package ynh3 or below, mosquitto is not installed by default.
If you have chosen to not set a domain during initial installation also.
So, if you need to activate mosquitto in retrospect, do following actions:
1. Create a domain or a subdomain (for example : 'mqtt.your.domain.tld')
2. Connect to your server in command line 
3. Type following command : `yunohost app setting domoticz mqtt_domain -v mqtt.your.domain.tld`
4. Upgrade domoticz to last package.
If you're already on the last package version, use the following command : `yunohost app upgrade domoticz --force`

## Configuration

### Sensors, language and this kind of stuff
Main configuration of the app take place inside the app itself.

### Zwave management
If you're using zwave devices, install mosquitto along domoticz and give a try to [zwave-JS-UI package](https://github.com/YunoHost-Apps/zwave-js-ui_ynh).
Once installed, just follow instructions from the [wiki](https://www.domoticz.com/wiki/Zwave-JS-UI)

### Access and API
By default, access for the [JSON API](https://www.domoticz.com/wiki/Domoticz_API/JSON_URL's) is allowed on following path `/yourdomain.tld/api_/domoticzpath`.
So if you access domoticz via https://mydomainname.tld/domoticz, use the following webpath for the api : `/mydomainname.tld/api_/domoticz/json.htm?yourapicommand`

By default, only sensor updates and switch toogle are authorized. To authorized a new command, you have (for now) to manually update the nginx config file :
````
sudo nano /etc/nginx/conf.d/yourdomain.tld.d/domoticz.conf
````
Then edit the following block by adding the regex of the command you want to allow:
````
  #set the list of authorized json command here in regex format
  #you may retrieve the command from https://www.domoticz.com/wiki/Domoticz_API/JSON_URL's
  #By default, sensors updates and toggle switch are authorized
  if ( $args ~* type=command&param=udevice&idx=[0-9]*&nvalue=[0-9]*&svalue=.*$|type=command&param=switchlight&idx=[0-9]*&switchcmd=Toggle$) {
    set $api "1";
    }
````
For example, to add the json command to retrieve the status of a device (/json.htm?type=devices&rid=IDX),modify the line as this:
````
  #set the list of authorized json command here in regex format
  #you may retrieve the command from https://www.domoticz.com/wiki/Domoticz_API/JSON_URL's
  #By default, sensors updates and toggle switch are authorized
  if ( $args ~* type=command&param=udevice&idx=[0-9]*&nvalue=[0-9]*&svalue=.*$|type=command&param=switchlight&idx=[0-9]*&switchcmd=Toggle$|type=devices&rid=[0-9]* ) {
    set $api "1";
    }
````

All IPv4 addresses within the local network (192.168.0.0/24) and *all IPv6* addresses are authorized as API.
As far as I know, there is no way to filter for IPv6 address on local network : You may remove the authorization by removing or commenting this line in `/etc/nginx/conf.d/yourdomain.tld.d/domoticz.conf`:
````
allow ::/1;
````
This will authorized only IPv4 within local network to access your domoticz API.
You may add individual IPv6 address in the same way.

**Shipped version:** 2020.2~ynh7
## Disclaimers / important information


## Limitations

* No user management nor LDAP integration This function is [not planned to be implemented into the app](https://github.com/domoticz/domoticz/issues/838), hence it's not planned into the package neither.
* Backup cannot be restored on a different machine type (arm, x86...) as compiled sources are different

## Security consideration

Although you may activate a login page on the application (either from the *Setup/Settings/System/Website protection* menu or from the *Setup/More Options/Edit Users* menu), it doesn't seems to be very reliable and secure so far (version 2022.2 at the time of writing). Work is ongoing to strengthen the security ([see here](https://www.domoticz.com/wiki/Security)) in future version but is not yet released.

### recommandation

It seems advisable to not make the app publicly available outside of the yunohost sso (public = yes at install or setting the domoticz permission to 'visitors' in the admin panel). If for any reason you need to, I recommend the following:
 - Activate the website protection/user management (with login page instead of Basic-auth)
 - In *Setup/Settings/System/Local Networks (no username/password)* enter the address of the nginx proxy (should be "::1;127.0.0.1" in any standard Yunohost installation) so that the Fail2ban settings is active (see last lines of [this wiki](https://www.domoticz.com/wiki/WebServer_Proxy)

## Documentation and resources

* Official app website: <https://domoticz.com/>
* Official user documentation: <https://www.domoticz.com/DomoticzManual.pdf>
* Official admin documentation: <https://www.domoticz.com/wiki/Main_Page>
* Upstream app code repository: <https://github.com/domoticz/domoticz>
* YunoHost documentation for this app: <https://yunohost.org/app_domoticz>
* Report a bug: <https://github.com/YunoHost-Apps/domoticz_ynh/issues>

## Developer info

Please send your pull request to the [testing branch](https://github.com/YunoHost-Apps/domoticz_ynh/tree/testing).

To try the testing branch, please proceed like that.

``` bash
sudo yunohost app install https://github.com/YunoHost-Apps/domoticz_ynh/tree/testing --debug
or
sudo yunohost app upgrade domoticz -u https://github.com/YunoHost-Apps/domoticz_ynh/tree/testing --debug
```

**More info regarding app packaging:** <https://yunohost.org/packaging_apps>

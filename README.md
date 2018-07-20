# nagios_whatsapp
nagios talking with whatsapp

## How to configure Nagios to send WhatsApp alerts on any linux distro.

### Overview

One of the most powerfull feature in nagios is the system alert that's use email to send alerts, but if you can't reach a SMTP server this can be a problem, here we'll combine nagios and WhatsApp message service to send alerts to your cellphone.

### Requirements:

* Nagios server;
* yowsup (python WhatsApp library);
* SIM card with a cellphone number that's has never been used before with WhatsApp service. (Usually a new SIM card, data plan is optional, this will be used to receive an SMS confirmation message throughout the setup process and will be the nagios "cellphone number" which send the messages).


### Step 1: Nagios server
First you need a working nagios server running, if you don't have one, follow this guide: [Configure Nagios on Ubuntu](https://www.vultr.com/docs/configure-nagios-on-ubuntu-part-1-nagios-server).

### Step 2: yowsup (python WhatsApp library)
[Yowsup](https://github.com/tgalal/yowsup) requires python2.6+ or python3.0+.

#### Install
Install using pip to pull all Python dependencies:

    pip install yowsup2

#### Setup

Put the "new" SIM card on any phone.

In your home dir create a new file "whatsapp.conf" 

    vim whatsapp.conf

Add the follow content to whatsapp.conf (see explanation below)

    cc=1
    phone=16475555555
    MCC=302
    MNC=920
    id=
    password=

cc = country code without preceding "+" or "00";

MCC = Mobile Country Code. Check your mcc [here](https://en.wikipedia.org/wiki/Mobile_country_code);

MNC = Mobile network Code. Check your mcc [here](https://en.wikipedia.org/wiki/Mobile_country_code);

phone= Your full phone number including the country code you defined in "cc", without preceding "+" or "00";

id= leave blank;

password= leave blank.


Start the registration with this command:

    yowsup-cli -r sms -c whatsapp.conf

The reply will be something like this:

    INFO:yowsup.common.http.warequest:{"status":"sent","length":6,"method":"sms",   "retry_after":1805}
    status: sent
    retry_after: 1805
    length: 6
    method: sms

And you'll receive an SMS with a registration code, run the register command:

    yowsup-cli -R XXXXXX -c whatsapp.conf

XXXXXX = registration code received by SMS

The reply will be something like this:

    INFO:yowsup.common.http.warequest:{"status":"ok","login":"xxxxxxxxxxxxxxx", "pw":"0pJj5cLaGSk6pDTa6rJR/5bDiR0=","type":"new","expiration":1471273284,"kind":"free"}
    status: ok
    kind: free
    pw: 0pJj5cLaGSk6pDTa6rJR/5bDiR0=
    login: xxxxxxxxxxxxxx
    type: new

Edit your "whatsapp.conf"

    vim whatsapp.conf

Remove the MMC and MNC lines and add a password received in register command, (pw line):

    cc=1
    phone=16475555555
    id=
    password=0pJj5cLaGSk6pDTa6rJR/5bDiR0=

That's it, now you can pull off the SIM card from the phone, you can use this number as a normal phone number but remember **do not use this SIM card with whatsapp on another device, if you do that nagios stop send messages**.

It's time to test, send a message to you:

    yowsup-cli -s XXXXXXXXXX  "Hello from nagios" -c whatsapp.conf

XXXXXXXX = whatsapp destination full phone number

### Step 3: Nagios configuration

#### Nagios commands
Now we'll add two new commands to nagios "command.cfg" to send messages by whatsapp, one for monitored hosts and another for monitored services, to do that edit the file /usr/local/nagios/etc/objects/commands.cfg

    vim /usr/local/nagios/etc/objects/commands.cfg

search for this block of text on file:

    ################################################################################
    #
    # SAMPLE NOTIFICATION COMMANDS
    #
    # These are some example notification commands.  They may or may not work on
    # your system without modification.  As an example, some systems will require 
    # you to use "/usr/bin/mailx" instead of "/usr/bin/mail" in the commands below.
    #
    ################################################################################

below that block add those lines and save the file:

    # "notify-host-by-whatsapp" command definition
    define command{
        command_name    notify-host-by-whatsapp
        command_line    /usr/local/bin/yowsup-cli -c /home/user/whatsapp.conf -s $_CONTACTWHATSAPP$ "$NOTIFICATIONTYPE$ Host : $HOSTNAME$ - Service : $SERVICEDESC$ is $SERVICESTATE$ @ $LONGDATETIME$"
        }

    # "notify-service-by-whatsapp" command definition
    define command{
        command_name    notify-service-by-whatsapp
        command_line    /usr/local/bin/yowsup-cli -c /home/user/whatsapp.conf -s $_CONTACTWHATSAPP$ “$NOTIFICATIONTYPE$ Host : $HOSTNAME$ is $HOSTSTATE$ @ $LONGDATETIME$”
        }

**Important**: At lines "command_line" change the path of "yowsup-cli" and "whatsapp.conf" to match with your system, you can find the "yowsup-cli" path with this command:

    which yowsup-cli

#### Nagios contacts
Now you have two options:

1. Change your contact to send whatsapp message
2. Add new contact to send whatsapp message

**Change your contact**:

Edit the file /usr/local/nagios/etc/objects/contacts.cfg

    vim /usr/local/nagios/etc/objects/contacts.cfg

search for those lines in your contact define block:

    service_notification_commands   notify-service-by-email
    host_notification_commands  notify-host-by-email

change to:

    service_notification_commands   notify-service-by-email notify-service-by-whatsapp
    host_notification_commands    notify-host-by-email notify-host-by-whatsapp

finally add this line inside your contact define block and save the file:

    _whatsapp XXXXXXXXXX

XXXXXXXXXX = your whatsapp full phone number.

**Or add new contact**:

Add new define block to your "contact.cfg"

    vim /usr/local/nagios/etc/objects/contacts.cfg

add this block and save the file:

    define contact{
        contact_name                    admin
        use                             generic-contact
        alias                           Admin 
        email                           admin@example.com
        _whatsapp                       XXXXXXXXXX
        service_notification_commands   notify-service-by-email notify-service-by-whatsapp
        host_notification_commands      notify-host-by-email notify-host-by-whatsapp
    }

XXXXXXXXXX = your whatsapp full phone number.

Check for errors in nagios config files:

    /usr/local/nagios/bin/nagios -v /usr/local/nagios/etc/nagios.cfg

Restart nagios as root:

    systemctl restart nagios

### Conclusion

This guide is very useful for those can't reach an SMTP server to send emails and need a tool to send alerts for monitored services.

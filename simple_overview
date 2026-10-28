This gonna become a guide on how to step by step perform Kasperky integration with the wazuh manager. 
- Installed stunnel4 and openssl on the Wazuh server.
- Created a 5-year certificate and key for the client Domain, stored in /etc/stunnel/certs/.
- Set up stunnel in /etc/stunnel/kaspersky-syslog.conf to listen on TCP 6514 and forward to 127.0.0.1:5514.
- Added a local Wazuh listener for port 5514 in ossec.conf, accepting only 127.0.0.1.
- Set up public access: DNS for xxx-xxx-xxx.com and a port forward for TCP 6514 to the SIEM.
- Configured Export to SIEM in the Kaspersky Next console, and confirmed the status shows Connected.
- Turned on logall, so every received event is saved in archives.log.
- Created the decoder (kaspersky-ksc) in local_decoder.xml, extracting device, message, component, user, path, process name and error.
- Created 25 rules (IDs 100011 to 100035) in local_rules.xml, giving each event a severity level.
- Tested with wazuh-logtest, test events and real events, and confirmed the alerts in the dashboard.
- Left open: restricting TCP 6514 on the firewall to Kaspersky’s source IP.

Decoder and Rules are done and uploaded 
One thing remainining is the Whole Guide gonna be here soon.
*If possible understand how to create out the rules and decoder and create over a guide on that aswell*
**Improve over the About and the Readme File**

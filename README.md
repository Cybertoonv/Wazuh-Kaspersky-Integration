# Wazuh + Kaspersky Next integration (syslog over TLS via stunnel)

## How it works

```
Kaspersky Next --TLS TCP 6514--> stunnel (decrypts) --TCP--> 127.0.0.1:5514 --> Wazuh remoted --> archives / rules / alerts
```

## Prerequisites

- *root on the Wazuh manager*
- *a DNS name pointing at it (`siem.example.com` below)*
- *port 6514 reachable from the internet*
- *admin access to the Kaspersky Next console*

Kaspersky Next can export events to a SIEM, but it only speaks syslog over TLS, and Wazuh's syslog listener only speaks plain UDP or TCP. stunnel sits between them: it terminates the TLS connection and hands the decrypted syslog to Wazuh on localhost.

This is written from a real deployment where the console is the cloud-hosted version of Kaspersky Next, so the connection comes **from Kaspersky's infrastructure to your server**. Your SIEM needs a public address and port 6514 reachable from the internet, which is why we link it to a domain such as `siem.example.com`.

**Why 5514 and not 514:** Wazuh can't run two TCP listeners on the same port, and 514 is usually already in use by other syslog senders.

---

## Part 1: Add a syslog listener to Wazuh

**1.1 Create a backup**

```bash
cp /var/ossec/etc/ossec.conf /var/ossec/etc/ossec.conf.bak_kaspersky
```

**1.2 Add the listener by editing ossec.conf**

```bash
nano /var/ossec/etc/ossec.conf
```

Add this **inside** `<ossec_config>`, next to the existing `<remote>` blocks (not before the opening tag):

```xml
<remote>
  <connection>syslog</connection>
  <port>5514</port>
  <protocol>tcp</protocol>
  <local_ip>127.0.0.1</local_ip>
  <allowed-ips>127.0.0.1</allowed-ips>
</remote>
```

`allowed-ips` is `127.0.0.1` because every Kaspersky message reaches Wazuh from stunnel. stunnel lives locally: it decrypts the received Kaspersky logs and passes them to Wazuh on the same machine.

**1.3 Validate and restart**

```bash
/var/ossec/bin/wazuh-analysisd -t     # no output = config is valid
/var/ossec/bin/wazuh-remoted -t
systemctl restart wazuh-manager
```

**1.4 Check it's listening**

```bash
ss -tlnp | grep ':5514'     # the new Kaspersky listener
ss -tlnp | grep ':514 '     # the existing TCP syslog listener, still there
ss -ulnp | grep ':514 '     # the existing UDP syslog listener, still there
```

Expected: `wazuh-remoted` on `127.0.0.1:5514`. If it's missing, check `grep -i remoted /var/ossec/logs/ossec.log | tail -20`.

Later, once stunnel is running, we run it again and look for all three ports:

```bash
ss -tlnp | grep -E ':514|:5514|:6514'
```

---

## Part 2: Install stunnel

```bash
apt install -y stunnel4 openssl
```

---

## Part 3: Create the certificate

If the Kaspersky team supplied a certificate and key, copy them to `/etc/stunnel/certs/` and skip to Part 4. Otherwise create a self-signed one (5 years here; we normally go for 2):

```bash
mkdir -p /etc/stunnel/certs

openssl req -x509 -newkey rsa:4096 -sha256 -days 1825 -nodes \
  -keyout /etc/stunnel/certs/siem.example.com.key \
  -out /etc/stunnel/certs/siem.example.com.crt \
  -subj "/CN=siem.example.com" \
  -addext "subjectAltName=DNS:siem.example.com"

chmod 600 /etc/stunnel/certs/siem.example.com.key
```

- `.key` is the private key. Never share it.
- `.crt` is the certificate you give to Kaspersky.
- The name must match exactly what Kaspersky connects to, i.e. the DNS name pointing at your Wazuh manager's public address, or the TLS check fails.
- Note the expiry date: `openssl x509 -in /etc/stunnel/certs/siem.example.com.crt -noout -enddate`

---

## Part 4: Configure stunnel

**4.1 Create the config**

```bash
nano /etc/stunnel/kaspersky-syslog.conf
```

Paste:

```
pid = /run/stunnel4/kaspersky-syslog.pid
debug = info

[kaspersky-syslog]
accept  = 0.0.0.0:6514
connect = 127.0.0.1:5514
cert    = /etc/stunnel/certs/siem.example.com.crt
key     = /etc/stunnel/certs/siem.example.com.key
sslVersionMin = TLSv1.2
```

With `debug = info` enabled in the stunnel configuration, stunnel logs a message for each connection showing how many bytes were sent over TLS and how many were forwarded to the underlying socket:

```
Connection closed: N byte(s) sent to TLS, M byte(s) sent to socket
```

If the socket byte count (`M`) is greater than zero, it indicates that data was forwarded to Wazuh.

**Optional: log to a file and ingest it into Wazuh.** This is the only place the real sender IP is recorded. Add `output = /var/log/stunnel4/stunnel-kaspersky.log` above the service section, like this:

```
pid = /run/stunnel4/kaspersky-syslog.pid
output = /var/log/stunnel4/stunnel-kaspersky.log
debug = info

[kaspersky-syslog]
accept  = 0.0.0.0:6514
connect = 127.0.0.1:5514
cert    = /etc/stunnel/certs/siem.example.com.crt
key     = /etc/stunnel/certs/siem.example.com.key
sslVersionMin = TLSv1.2
```

Once you set `output`, the connection lines go to that file and stop appearing in `journalctl -u stunnel4`, so use `tail -f` on the file in Part 8. To have Wazuh read the file, add this inside `<ossec_config>` in `ossec.conf` and restart the manager:

```xml
<localfile>
  <log_format>syslog</log_format>
  <location>/var/log/stunnel4/stunnel-kaspersky.log</location>
</localfile>
```

This saves all the stunnel logs to `stunnel-kaspersky.log`.

Save and exit.

**4.2 Start it**

```bash
systemctl enable stunnel4
systemctl restart stunnel4
systemctl restart wazuh-manager     # only needed if you added the localfile block
systemctl status stunnel4 --no-pager
ss -tlnp | grep 6514
```

Expected: `active (running)` and a line showing `0.0.0.0:6514` owned by stunnel. If it fails, run `journalctl -u stunnel4 -n 30 --no-pager`.

---

## Part 5: Test it yourself before involving Kaspersky

**5.1 Check the certificate is served**

```bash
openssl s_client -connect 127.0.0.1:6514 -servername siem.example.com </dev/null 2>/dev/null | openssl x509 -noout -subject -dates
```

Expected: `subject=CN = siem.example.com` and the dates.

**5.2 Make sure archiving is on for the test**

```bash
grep -n "logall" /var/ossec/etc/ossec.conf
```

**5.3 Send a test log through the full chain**

```bash
echo "<14>Oct 12 12:00:00 testhost app: plain-test" | nc -q1 127.0.0.1 5514
echo "<14>Oct 12 12:00:00 testhost app: tls-test" | openssl s_client -connect 127.0.0.1:6514 -quiet
grep -E "plain-test|tls-test" /var/ossec/logs/archives/archives.log
```

The second command may sit there waiting. Give it a couple of seconds, then press `Ctrl+C`.

Expected: both lines appear with `->127.0.0.1` in them. That works because `logall` is on. If you see them, the receiving side is done and working.

---

## Part 6: Check reachability from outside

From a Windows machine outside the SIEM's network:

```powershell
Test-NetConnection siem.example.com -Port 6514
```

*From Linux*: `nc -vz siem.example.com 6514`. If it fails, check DNS, NAT or port forwarding, and the firewall in front of the SIEM.

---

## Part 7: Configure Kaspersky

Menu names differ between versions, so find the matching ones in your console. The settings to enter are:

1. Open Kaspersky Next and click the *Go to workspace* button.
2. In the left menu, open **Administration Server**.

   ![Kaspersky Next left menu with Administration Server highlighted](images/01-administration-server-menu.png)

3. Open the **Administration Server** properties and wait for the **General** tab to load.
    - If it doesn't load, press `Ctrl+Shift+R` to reload, or move to **Event configuration** and back to **General**. This should work.
4. In the list on the left, open **Export to SIEM**, then click **Settings**.

   ![Export to SIEM page before it is configured](images/02-export-to-siem-settings.png)

   This is what the page looks like before you configure it: the address shows `:0` and **Automatically export events to SIEM system database** is **Disabled**. You'll switch that on in Part 8.

Enter these settings:

| Setting | Value |
| --- | --- |
| Server | `siem.example.com` |
| Port | `6514` |
| Protocol | TCP with TLS |
| Format | syslog |
| Certificate | Upload `siem.example.com.crt` as trusted |

To get the certificate file off the server, run `cat /etc/stunnel/certs/siem.example.com.crt` and copy the text into a file ending in `.crt`.

Then click **Check connection**. If it shows **Failure**, check that stunnel is running, that the certificate name matches the server name, and that port 6514 is reachable (Parts 4 to 6).

![Connection to SIEM system showing Failure](images/04-connection-check-failure.png)

---

## Part 8: Confirm real logs arrive and find the sender IP

```bash
journalctl -u stunnel4 -f | grep -E "accepted|Connection closed"    # connections as they happen, the real sender IP shows here
tcpdump -i any -nn 'tcp port 6514'
grep -- "->127.0.0.1" /var/ossec/logs/archives/archives.log | tail -5    # stunnel hands logs over locally, so they show as ->127.0.0.1
tail -f /var/log/stunnel4/stunnel-kaspersky.log    # if you used the optional log file
tail -f /var/ossec/logs/archives/archives.log | grep "event@"    # every Kaspersky log contains event@
```

In another window:

```bash
journalctl -u stunnel4 -f
```

Stop with `Ctrl+C`. If Kaspersky connects but you see TLS or certificate errors, the usual causes are a name mismatch (it must be exactly `siem.example.com`) or the certificate not being uploaded as trusted.

**Choose which events to export:**

- Go to **Event configuration**
- From the options Critical, Functional failure, Warning, Info, check the ones you prefer
- Click **Mark for export to SIEM system by using Syslog**, then hit **Save**
- Turn on **Automatically export events to SIEM system database**. Don't forget to turn it on.

![Event configuration tab with event types marked for Syslog](images/03-event-configuration.png)

The **Syslog** column shows a tick for every event type that is marked for export.

---

## Part 9: Parsing (decoder and rules)

Wazuh has no built-in Kaspersky decoder, so logs may arrive in `archives.log` without producing alerts. Events are RFC 5424 syslog with a structured-data block:

```
1 <timestamp> <host> <app> - <EVENT_TYPE> [event@<id> ... et="..." etdn="..." hdn="..." hip="..." gn="..." kscfqdn="..."] <readable message>
```

You can find the decoder and rules in the GitHub repository: https://github.com/Cybertoonv/Kaspersky_Integration

Copy them into these files:

```
/var/ossec/etc/decoders/local_decoder.xml
/var/ossec/etc/rules/local_rules.xml
```

For the rules, make sure no rule ID repeats.

Once done, check there are no errors by running:

```bash
/var/ossec/bin/wazuh-analysisd -t
```

```bash
/var/ossec/bin/wazuh-logtest
```

and paste a log line from Kaspersky (everything after `->127.0.0.1 ` in `archives.log`).

If phase 2 in the output shows the decoder name, the log is matching the decoder.

Once the logtest looks right, restart the manager so the live system loads the new decoder and rules:

```bash
systemctl restart wazuh-manager
```

---

## Part 10: See the alerts

- **Dashboard:** in Discover, filter `rule.groups: kaspersky`. Add columns such as `rule.description`, `rule.level` and the decoded fields (they appear under `data.*`), then click **Save** and name it "Kaspersky alerts".
- **Command line:** `grep '"kaspersky"' /var/ossec/logs/alerts/alerts.json | tail`

---

## Done

Kaspersky and the Wazuh dashboard are now in sync: you should be able to see the Kaspersky logs as events and as alerts in the Wazuh dashboard.

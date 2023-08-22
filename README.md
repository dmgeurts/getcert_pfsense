# getcert_pfsense
Automate pfSense FreeIPA certificate renewal using ipa-getcert. pfSense does not have an API, thus ssh/scp are utilised.

Inspired by: https://github.com/tylerhoadley/letsencrypt-automation

# FreeIPA Certificates for pfSense

While free LetsEncrypt certificates are great, however, when you have an internal CA this opens the door for using your own certificates. Ideally, a firewall management interface should not be publicly exposed, so using an internal CA stands to reason.

Not using LetsEncrypt avoids problems with permitting ingress HTTP-01 traffic or setting up DNS-01 verification, both have their uses and while I use both, FreeIPA PKI is a useful local solution for a local problem. It also avoids exposing internal management dns records externally, all Let's Encrypt-issued certificate domains are publicly listed.

The following will detail how to automate the ipa-getcert certificate process for pfSense devices via ssh/scp.

# Prerequisites

Image TBD

### Before getting started, the following are required:

+ A FreeIPA server with Dogtag CA.
+ A linux-based container, VM, or host with reachability to the pfSense device(s) and the following packages installed:
  + ipa-getcert - part of ipa-client-install on a FreeIPA enrolled host.
+ A Service Principal for the web interface that needs a certificate:
  + Manually create the host with the FQDN of the pfSense device.
    + Edit the host to add the enrolled host under 'managed by'.
  + Create a Service Principal for pfSense: HTTP/fqdn@DOMAIN.COM
    + Edit the Service Principal to add the enrolled host under 'managed by'.
+ pfSense credentials:
  + A dedicated protected private/public key pair: This script relies on ssh passwordless logins (as root)
    + Add the public key to the admin user in the GUI, to make it persistent when upgrading the firewall.
      + pfSense Webui >> System >> User Management >> Admin >> Authorized SSH Keys

# The Scripts

There are two scripts:

"pfs_getcert" --> https://github.com/dmgeurts/getcert_pfsense/edit/master/fps_getcert
Obtains a certificate from FreeIPA and sets the post-save to use pfs_instcert for automatic installation of a renewed certificate.
"pfs_instcert" --> https://github.com/dmgeurts/getcert_pfsense/edit/master/pfs_instcert
Installs the certificate on a pfSense appliance.

# How It Works

### pfs_getcert performs the following actions:
1. Uses the privileges set in FreeIPA (managed by) to call ipa-getcert and request a certificate from FreeIPA.
2. ipa-getcert will automatically renew a certificate when it's due, as long as the FQDN DNS record resolves, and the host and Service Principal still exist in FreeIPA.
3. Sets the post-save command to fps_instcert with the same parameters as issued to fps_getcert, for automated installation of renewed certificates.
    - Post-save will run on the first certificate save, using fps_instcert for certificate installation.

### pfs_instcert performs the following actions:
1. Uploads the management certificate in json format using the REST API.
2. Logs all output to `/var/log/pfs_instcert.log`.

### Script options
Run pfs_getcert without sudo, doing so may block access to your keytab and then ipa-getcert can't authenticate to IPA. It will test for and run ipa-getcert with sudo privileges.

pfs_getcert uses the following options:
```
Usage: pfs_getcert [-hv] -c CERT_CN [-a CRED_FILE] [OPTIONS] [FQDN]
This script requests a certificate from FreeIPA using ipa-getcert and calls a
partner script to deploy the certificate to a pfSense appliance via ssh/scp.

    FQDN              Fully qualified name of the pfSense appliance web interface.
                      Must be reachable from this host on port TCP/22. Defaults
                      to CERT_CN if omitted.
    -c CERT_CN        REQUIRED. Common Name (Subject) of the certificate (must be a
                      FQDN). Will also present in the certificate as a SAN.
    -a CRED_FILE      File with the pfSense admin/root privte key.
                      Defaults to /etc/ipa/.pfsrc
OPTIONS:
    -i                Resolve CERT_CN to an IP address, for inclusion in the SAN.
    -I IP4            IPv4 address of the pfSense server, for inclusion in the SAN.
    -S SERVICE        Service of the Service Principal. Default: HTTP.
    -T CERT_PROFILE   Ask IPA to process the request using the named profile or
                      template.
    -G TYPE           Type of key to be generated if one is not already in place.
                      IPA CA uses RSA by default. (RSA, DSA, EC or ECDSA)
    -b BITS           In case a new key pair needs to be generated, this option
                      specifies the size of the key. Default: 2048 (RSA/DSA).
                      EC: 256, 384 or 512. See certmonger.conf for the default.

    -h                Display this help and exit.
    -v                Verbose mode.
```

At a minimum pfs_getcert can be run as: `pfs_getcert -c firewall.domain.com` This will look for API credentials in `/etc/api/.pfsrc` and will assume that the pfSense firewall can be reached via the common name given for the certificate. If you want to include the IPv4 address in the SAN add the `-i` option to have the common name resolved to the IPv4 address.

**Note:** FreeIPA only supports RSA keys. Hence the -G option is in preparation of future support of other keys. [More info](https://www.reddit.com/r/FreeIPA/comments/134puyw/freeipa_ca_pki_ecdsa_support/).

pfs_instcert uses the same options, minus the FreeIPA specifics:
```
Usage: pfs_instcert [-hv] -c CERT_CN -a CRED_FILE FQDN
This script uploads a certificate issued by ipa-getcert to a pfSense appliance
via ssh/scp.

    FQDN          Fully qualified name of the pfSense appliance mgmt interface.
                  Must be reachable from this host on port TCP/22.
    -c CERT_CN    REQUIRED. Common Name (Subject) of the certificate, to find
                  the certificate and key files.
    -a CRED_FILE  File with the pfSense admin/root private key.
                  Defaults to /etc/ipa/.pfsrc

    -h            Display this help and exit.
    -v            Verbose mode.
```

# Automated Renewal and Installation
ipa-getcert requests the certificate with the post-save option, thus no cronjob is needed to install renewed certificates and pfs_instcert does not need to be manually called.

The post-save process will upload the renewed certificates, the assumed object name for the certificate is the common name. **You must manually upload the first certificate, pfSense does not have an API to make adding certificates easier, this script only updates an existing certificate!**

Check the configured post-save command with:
```
sudo ipa-getcert list [-i Request_ID | -f Path_to_cert_file]
```

# Prepare the Script for Execution
+ In an environment with centralised credentials it's best to run code under a user that will never be removed. Install the code to a common location: `/usr/local/bin` is assumed. When using a different location, change the variable INST_CERT in pfs_getcert accordingly.
  + For example: `sudo ssh-keygen -t ecdsa -b 521 -C "pfSense admin" -f /etc/ipa/.pfsrc`
  + Add the public key to the admin user on the firewall.
    + Copy the output of `sudo cat /etc/ipa/.pfsrc.pub` to System >> User Manager >> admin (edit) >> Keys (Authorized SSH Keys)
    + Ensure certificate authentication is enabled. System >> Advanced >> Admin Access >> Secure Shell >> SSHd Key Only. Set it to either 'Password or Public Key' or 'Public Key Only'.
  + Test access with: `sudo ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -i /etc/ipa/.pfsrc root@firewall.domain.com 'ls -l'`
+ Install and ensure pfs_getcert and pfs_instcert are executable:
  ```
  sudo cp getcert_pfsense/pfs_{get,inst}cert /usr/local/bin/
  sudo chmod +x /usr/local/bin/pfs_{get,inst}cert
  ```

# Expected CLI output
When requesting and deploying a certificate:
```
user@host:~$ pfs_getcert -v -c firewall.domain.local -i
Certificate Common Name: firewall.domain.local
Private key found in: /etc/ipa/.pfsrc
  verbose=1
  CERT_CN: firewall.domain.local
  IP4: 192.168.6.2
  PFS_FQDN: firewall.domain.local
IPv4 will be included in the SAN.
New signing request "20230822132800" added.
Certificate requested for: firewall.domain.local
  Certificate install and commit by the post-save process on: firewall.domain.local took 1 seconds.
FINISHED: Check the pfSense appliance to see if the certificate is in use.
```

# Crontab
Not required; ipa-getcert certificate monitoring takes care of running post-save after renewing certificates.


# Validate Automated Renewals
pfs_instcert will log to `/var/log/pfs_instcert.log`.

pfs_getcert doesn't log as it's expected to be run once (manually).
```
user@host:~$ sudo /usr/local/bin/pfs_instcert -v -c firewall.domain.local firewall.domain.local
START of pfs_instcert.
Certificate Common Name: firewall.domain.local
Private key found in: /etc/ipa/.pfsrc
PFS_MGMT: firewall.domain.local
Certmonger certificate state MONITORING, stuck: no.
Result of SCP upload of certificate: 0
Restarting webConfigurator... done.

Config update and UI restart output: 0
Finished upload and activation of certificate for: firewall.domain.local
END - Finished certificate installation to: firewall.domain.local
```

If successful, you should see something similar to the following in the log file:
```
[2023-08-22 16:20:56+02:00]: START of pfs_instcert.
[2023-08-22 16:20:56+02:00]: Certificate Common Name: firewall.domain.local
[2023-08-22 16:20:58+02:00]: Finished upload and activation of certificate for: firewall.domain.local
[2023-08-22 16:20:58+02:00]: END - Finished certificate installation to: firewall.domain.local
```

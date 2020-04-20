# How to configure Duo Directory Sync with FreeIPA / RedHat|CentOS Identity Manager.

After struggling with getting the directory sync to work, and a few Google searches, then I finally found a old and useful thread from 2018 called [Directory Sync with idM](https://community.duo.com/t/directory-sync-with-idm/2171) on the Duo Community forums that put me on the right track. My setup consists of CentOS 8.1, IPA version 4.8.0 and Duo Authentication Proxy version 3.2.4, if you're using a older version of IPA then this guide might need some modification on your part to work. 

For this guide I'm assuming that you already have a functional install of IPA / idM, Duo Authentication Proxy and are using Duo for 2FA.


### Duo Admin Portal

1. Go to Users -> Directory Sync and fill out the required fields.

Option|Value
------|-----
Directory server(s)|IPA FQDN or IP (internal)
Base DN|cn=accounts,dc=example,dc=com
Authentication Type|Plain
Transport Type|LDAPS
SSL CA Certs|Content of /etc/ipa/ca.crt
Username|uid
Username alias 1|mail
Full name|cn
Email|mail

2. Still on the Directory Sync page, under Selected Groups, then select the security group you will use to sync your users from, then click on the Save Directory button.

---

### Duo Authentication Proxy

Here's my configuration for the Duo proxy, I'm using three IPA servers, if you have less than that then you can just remove the host_2 and host_3 lines. In the cloud section, you can retrieve the values you need there from Users -> Directory Sync -> Authenticaion Proxy in the Duo Admin Portal. Remember to restart the service if you modify the configuration file.

**authproxy.cfg**
```
[main]
debuge=false

[ad_client]
host=<IPA Server #1 FQDN / IP>
host_2=<IPA Server #2 FQDN / IP>
host_3=<IPA Server #3 FWDN / IP>
service_account_username=<username>
service_account_password=<password>
bind_dn=uid=<username>,cn=users,cn=accounts,dc=example,dc=com
search_dn=cn=accounts,dc=example,dc=com
username_attribute=uid
auth_type=plain
transport=ldaps
ssl_ca_certs_file=/etc/ipa/ca.crt
ssl_verify_hostname=true

[cloud]
ikey=<ikey>
skey=<skey>
api_host=api-<xxxxxxxx>.duosecurity.com
service_account_username=uid=<username>,cn=users,cn=accounts,dc=example,dc=com
service_account_password=<password>

[ldap_server_auto]
client=ad_client
ikey=<ikey>
skey=<skey>
api_host=api-<xxxxxxxx>.duosecurity.com
failmode=safe
ssl_key_path=ldap.key
ssl_cert_path=ldap.crt
exempt_primary_bind=true
```

---

### IPA / idM

1. First we need to modify the LDAP schema by editing the file /etc/dirsrv/slapd-EXAMPLE-COM/schema/60basev2.ldif

*Find the following line*
```
attributeTypes: (2.16.840.1.113730.3.8.3.1 NAME 'ipaUniqueID' DESC 'Unique identifier' EQUALITY caseIgnoreMatch ORDERING caseIgnoreOrderingMatch SYNTAX 1.3.6.1.4.1.1466.115.121.1.15 X-ORIGIN 'IPA v2' )
```

*Change to*
```
attributeTypes: (2.16.840.1.113730.3.8.3.1 NAME ('ipaUniqueID' 'entryUUID') DESC 'Unique identifier' EQUALITY caseIgnoreMatch ORDERING caseIgnoreOrderingMatch SYNTAX 1.3.6.1.4.1.1466.115.121.1.15 X-ORIGIN 'IPA v2' )
```

Then we need to apply the changes with: ipa-ldap-updater -u --schema-file=/etc/dirsrv/slapd-EXAMPLE-COM/schema/60basev2.ldif


2. Log into your IPA web UI as admin, go to IPA Server, Role-Based Access Control -> Permissions.

Find "System: Read Groups", click Add under Target / Effective attributes and add entrydn.

Find "System: Read User IPA Attributes", click Add under Target / Effective attributes and add entrydn.

---

When all of the above is done then you should be able to sync groups and users from IPA/idM to your Duo portal.


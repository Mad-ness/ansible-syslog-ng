# README.md
# Ansible Role: syslog-ng 0.1

The role configures Syslog-Ng daemon to send logs over an SSL encrypted channel to Syslog-Ng servers.


## Requirements

These requirements should be satisfied before running the role.

- Target systems: Gentoo Linux, FreeBSD.
- Installed package: Syslog-Ng.
- SSL certificates, including private keys for Syslog-ng servers must be created in advance.


## Role Variables

List variables and their default values find out in the section.


### Common variables

Path where to store log files

	syslog_logdir: /var/log		

The variable specifies a root directory as for local messages and for messages that will be received from remote clients. Example:

	Client1:
		syslog_dir: /var/log
	Local log files will be saved in files: /var/log/auth.log, /var/log/kernel.log, etc.

	Client2: 
		syslog_dir: /data/logs	
 	Local log files will be saved in files: /data/logs/auth.log, /data/logs/kernel.log, etc.

	SyslogServer:
		syslog_dir: /var/log
	Local log files: /var/log/auth.log, /var/log/kernel.log, etc.
	Remote logs: /var/log/remote-logs/client1/auth.log, /var/log/remote-logs/client2/kernel.log, etc.


Version of syslog-ng server, it gives warnings if syslog-ng version differs from version mentioned in a configuration file

	syslog_ng_version: 3.7

Directory where Syslog-Ng will looking for certificates on startup

	syslog_certs_dir: /etc/syslog-ng/certs

It says if send logs to a remote Syslog-Ng server

	syslog_send_logs_to_central_server: true


### Customized variables

A profile that will be activated for each of a client Syslog-Ng instance

	syslog_client_profile: airlan

A profile that will be activated for each of a server Syslog-Ng instances

	syslog_server_profile: <empty>


If a current instance should log messages to remote server it should have **syslog_client_profile** filled. And otherwise it should have configured **syslog_server_profile** to configure syslog for accepting incoming log messages. Same host may have both profiles set, in these case it runs a server part and sends logs to other syslog servers.

If both profiles are blank then no remote logging will be configured and log messages will store in local log files.


In case of a **syslog_server_profile** configured and a profile have **transport** variable set (see below), there must exist a private SSL key (see below).


The variable **syslog_profiles** defines a list of profiles both of clients and servers.

    syslog_profiles:
    	airlan:
        	servers:
        		- { ip: '192.168.168.103', port: '25214', transport: 'tls' }
      	serv03:
        	servers: [ { ip: '10.6.0.1', port: '56101', transport: 'udp' } ] 
      	madness:
        	servers: [ { ip: log.example.com, port: 25214, transport: 'tls' } ] 

As shown above, profile is a set of variables that should be filled properly.

- ip - may contain FQDN, hostname, of IP address
- port - a port of where log messages to be sent
- transport - this may be either 'udp', or 'tls'. TCP supporting is not tested enough.

If a profile is a client Syslog-Ng instance it will sent log messages to transport:ip:port. If a profile is a server Syslog-Ng instance it will configured to bind on this transport:ip:port. So a profile may contain a lot of records, such way it will sent log messages to so many servers as configured. If a profile describes a server, the profile should have valid ip:address-es for a server.

Note. If 'udp' transport is chosen then SSL certificates are not required and encryption is not involved.


### Definition of the standard parsing rules of messages

There are a predefined set of rules that will be applied at all Syslog-Ng instances. This is described by variable **syslog_standard_services**.


Below is a complete list of such services. The structure is.

	id: name				# Some identificator is expected to be unique in the list.
	    source: local		# Only this value is supported
	  	filepath: out.log	# Name of a file where log messages will be saved.
							# It appends to {{ syslog_logdir }} variable.
		filter: rule		# A rule that acceptable by Syslog-Ng filter() function
		final: true			# A boolean variable, it says to stop (true) or keep going (false)
							# parsing messages at this rule.

    syslog_standard_services:
      - id: auth_err
        source: local
        filepath: auth.err
        filter: facility(auth) and level(err)
        final: true
      - id: sshd
        source: local
        filepath: sshd.log
        filter: program(sshd)
        final: true
      - id: mail_err
        source: local
        filepath: mail.err
        final: true
        filter: facility(mail) and level(err)
      - id: mail_info
        source: local
        filepath: mail.info
        final: true
        filter: facility(mail) and level(info)
      - id: mail_warn
        source: local
        filepath: mail.warn
        final: true
        filter: facility(mail) and level(warn)
      - id: mail
        source: local
        filepath: mail.log
        filter: facility(mail)
        final: true
        df_owner: root
        df_group: root
        df_perm: '0644'
      - id: kernel
        source: local
        filepath: kernel.log
        filter: facility(kern)
        final: true
      - id: auth
        source: local
        filepath: auth.log
        filter: facility(auth)
        final: true
      - id: crond
        source: local
        filepath: cron.log
        filter: facility(cron)
        final: true
      - id: ntpd
        source: local
        filepath: ntpd.log
        filter: program(ntpd)
        final: true
      - id: warn
        source: local
        filepath: warn.log
        filter: level(warn..emerg) and not facility(kern)
        final: true
      - id: debug
        source: local
        filepath: debug.log
        filter: level(debug)
        final: true
      - id: openvpn
        source: local
        filter: program(openvpn)
        filepath: openvpn.log
        final: true
      - id: sudo_err
        source: local
        filter: program(sudo) and level(err)
        filepath: sudo.err
        final: true
      - id: sudo
        source: local
        filter: program(sudo)
        filepath: sudo.log
        final: true
      - id: messages
        source: local
        filter: message('.*')
        filepath: messages
        final: true


### Definition of the extra parsing rules of messages

Besides the standard rules there is also possible to write personal rules for every host. Use **syslog_extra_services** variable to define personal rules. The format is exact same as for standard rules. The extra rules will be applied before the standard rules. For instances, if you say `final: true` in personal rules for kernel's messages *facility(kern))* these messages will not be delivered to standard rules.


### Other variables

Because the role designed for Gentoo and FreeBSD systems, they have different paths in system. There a group of such variables:

	syslog_root_path:			# defines a root directory of Syslog-Ng configuration files
		Gentoo 	- /etc/syslog-ng
		FreeBSD - /usr/local/etc/syslog-ng
	syslog_certs_dir:			# where Syslog-Ng should looking for certificates
		Gentoo	- /etc/syslog-ng/certs
		FreeBSD	- /usr/local/etc/syslog-ng/certs
	syslog_create_dirs_group: 	# who should own the directories created for remote clients
		Gentoo	- root
		FreeBSD	- wheel

### SSL related variables

These variables specifies where to get SSL certificates and what names are used for.


All certificates placed at this path

	sslcert_certs_location: ../../../files/certs


The target directory should have the structure:
	
	../../../files/certs/ca.cert.pem
						/certs
							/clients
							/servers
								{{ inventory_hostname }}.cert.pem	
								...
						/private
								{{ inventory_hostname }}.cert.pem
								...


This name of a root CA certificate, finding in ../../../files/certs directory
	
	sslcert_cert_caname: ca.cert.pem


This specifies what name postfix should certificates have

	sslcert_cert_ext: '.cert.pem'

## Dependencies 

None.


## SSL certificates

There is no an automatic way to create certificates and private keys. This should be considered separately.


## Example Playbooks

Example 1:

	- hosts: syslog_service
	  roles: 
			- { role: syslog-ng }

Example 2:

	- hosts: syslog_service_freebsd
	  roles: 
			- { role: syslog-ng, ansible_os_family: FreeBSD }



## License

MIT


# Phase 1 – Reconnaissance

## Objective

The objective of this phase was to identify exposed services, enumerate accessible applications, fingerprint underlying technologies, and map the overall attack surface of the target system before attempting any exploitation.

Target:

```text
192.168.96.134
```

---

# Network Discovery and Service Enumeration

## Nmap Service Enumeration

Initial reconnaissance began with a comprehensive Nmap scan.

Command:

```bash
nmap -sC -sV -A 192.168.96.134
```

### Purpose

The scan was designed to:

* Discover open ports
* Identify running services
* Determine software versions
* Execute default NSE scripts
* Perform operating system fingerprinting

### Results

| Port | Service                   | Version              |
| ---- | ------------------------- | -------------------- |
| 22   | SSH                       | OpenSSH 5.3p1        |
| 80   | HTTP                      | Apache 2.2.14        |
| 139  | NetBIOS                   | Samba                |
| 143  | IMAP                      | Courier IMAP         |
| 443  | HTTPS                     | Apache SSL           |
| 445  | SMB                       | Samba                |
| 5001 | Java Object Serialization | Unknown Java Service |
| 8080 | Tomcat                    | Apache-Coyote 1.1    |
| 8081 | Jetty                     | Jetty 6.1.25         |

### Assessment

Several exposed services were identified running outdated software versions.

Particularly notable were:

* Apache 2.2.14
* PHP 5.3.2
* OpenSSL 0.9.8k
* Jetty 6.1.25
* OpenSSH 5.3

These versions are significantly outdated and may contain publicly documented vulnerabilities.

### Security Impact

Legacy software often contains:

* Known security flaws
* Unsupported components
* Missing security patches
* Increased attack surface

### Risk Rating

Medium

---

# Web Technology Fingerprinting

## WhatWeb Enumeration

Command:

```bash
whatweb -a 3 http://192.168.96.134
```

### Purpose

Identify web technologies, frameworks, plugins, libraries, and exposed metadata.

### Results

Technologies discovered:

* Apache 2.2.14
* PHP 5.3.2
* OpenSSL 0.9.8k
* Phusion Passenger 4.0.38
* mod_mono
* mod_python
* mod_perl
* JQuery 1.3.2

Email addresses exposed:

* [admin@metacorp.com](mailto:admin@metacorp.com)
* [admin@owaspbwa.org](mailto:admin@owaspbwa.org)
* [jack@metacorp.com](mailto:jack@metacorp.com)
* [test@thebodgeitstore.com](mailto:test@thebodgeitstore.com)

### Assessment

The application disclosed extensive technology stack information through HTTP responses.

Email addresses and software versions provide valuable intelligence to an attacker during reconnaissance.

### Security Impact

Information disclosure can facilitate:

* Targeted attacks
* Social engineering
* Version-specific exploitation
* User enumeration

### Risk Rating

Low

---

# HTTP Header Analysis

Command:

```bash
curl -I http://192.168.96.134
```

### Purpose

Review HTTP response headers for server information disclosure.

### Results

```http
Server: Apache/2.2.14 (Ubuntu)
PHP/5.3.2
OpenSSL/0.9.8k
```

### Assessment

The server openly discloses detailed software versions.

This allows attackers to identify potential vulnerabilities before authentication.

### Security Impact

Version disclosure significantly reduces attacker effort during target profiling.

### Risk Rating

Low

---

# SMB Enumeration

## Share Discovery

Command:

```bash
smbclient -L //192.168.96.134 -N
```

### Results

Discovered shares:

```text
apache
tomcat
var
etc
usr
owaspbwa
```

### Assessment

The SMB server exposed internal share names without authentication.

Although access was denied to the shares themselves, their names reveal server structure and application deployment locations.

Examples:

```text
apache
tomcat
etc
usr
```

These names provide insight into:

* Web server layout
* Application hosting locations
* Operating system structure

### Security Impact

This information assists attackers in:

* Building attack paths
* Identifying likely configuration locations
* Understanding deployment architecture

### Risk Rating

Low

---

# SMB Security Configuration

Nmap reported:

```text
message_signing: disabled
```

### Assessment

SMB signing was not enforced.

### Security Impact

In real-world environments, disabled SMB signing may increase susceptibility to relay attacks and man-in-the-middle scenarios.

### Risk Rating

Medium

---

# Application Discovery

Several applications were identified through manual enumeration.

## phpMyAdmin

Command:

```bash
curl -I http://192.168.96.134/phpmyadmin/
```

Response:

```http
HTTP/1.1 200 OK
```

Evidence:

```html
<title>phpMyAdmin</title>
```

### Assessment

A database administration interface is directly accessible.

### Security Impact

Administrative interfaces are high-value targets because successful compromise may lead to database access, credential disclosure, and application compromise.

### Risk Rating

Medium

---

## DVWA

Command:

```bash
curl -I http://192.168.96.134/dvwa/
```

Response:

```http
Location: login.php
Set-Cookie: security=low
```

### Assessment

DVWA was operational and configured for intentionally vulnerable training purposes.

### Security Impact

Not considered a production vulnerability but confirms an intentionally vulnerable application is present.

---

## bWAPP

Command:

```bash
curl -I http://192.168.96.134/bWAPP/
```

Response:

```http
Location: portal.php
```

### Assessment

The application is active and accessible.

---

## WordPress

Command:

```bash
curl -s http://192.168.96.134/wordpress/ | grep -i title
```

Evidence:

```html
<title>Broken WordPress</title>
```

Additional indicators:

```text
xmlrpc.php
RSS Feed
RSD endpoint
```

### Assessment

WordPress was confirmed as operational.

XML-RPC functionality was enabled and publicly exposed.

### Security Impact

WordPress installations increase attack surface through:

* Authentication endpoints
* Plugins
* Themes
* XML-RPC functionality

### Risk Rating

Low

---

## Joomla

Command:

```bash
curl -s http://192.168.96.134/joomla/ | grep -i title
```

Evidence:

```html
<title>Broken Joomla</title>
```

### Assessment

A Joomla content management system was operational and publicly accessible.

### Security Impact

CMS platforms expand the attack surface through:

* Administrative panels
* Extensions
* Components
* Authentication systems

### Risk Rating

Low

---

# Information Disclosure – Exposed Shell History

## Discovery

Command:

```bash
curl -i http://192.168.96.134/.bash_history
```

Response:

```text
/usr/local/rvm/bin/rvm user gemsets
/usr/local/rvm/wrappers/ruby-1.9.3-p484@railsgoat/ruby
...
```

### Assessment

A shell history file was publicly accessible through the web server.

The file disclosed:

* Internal filesystem paths
* Deployment commands
* Ruby environment details
* RailsGoat application references
* Passenger configuration information

### Security Impact

Exposed administrative artifacts can provide attackers with:

* Internal infrastructure knowledge
* Technology stack details
* Deployment procedures
* Additional attack vectors

### Risk Rating

Medium

---

# Jetty Service Enumeration

## Jetty Discovery

Command:

```bash
curl http://192.168.96.134:8081
```

### Results

Discovered application paths:

```text
/vulnerable
/securish
```

### Assessment

The Jetty service exposed separate application environments that may contain different security configurations.

### Security Impact

Multiple application environments increase the attack surface and may expose inconsistent security controls.

### Risk Rating

Low

---

# Overall Reconnaissance Assessment

The target presents a large and diverse attack surface consisting of:

* Legacy web technologies
* Multiple web application frameworks
* Administrative interfaces
* CMS platforms
* Exposed development artifacts
* SMB services
* Java application servers

The most significant finding during reconnaissance was the exposure of the `.bash_history` file, which disclosed internal deployment information and application details.

No exploitation was performed during this phase. The focus remained on attack surface mapping, service identification, technology fingerprinting, and information gathering to support later vulnerability assessment activities.

---
title: "Ubuntu Server Setup and API Server Development"
date: 2026-06-21 00:00:00 +0900
categories: [Backend, Server]
tags: [ubuntu, nodejs, mysql, rest-api, ssh]
---

## Overview

A record of my journey from setting up Ubuntu 26.04 LTS on VMware
to building a REST API server with Node.js, Express, and MySQL.

## 1. Ubuntu Server Installation

Installed Ubuntu 26.04 LTS Server on VMware Workstation.
During installation, OpenSSH server was also installed to enable remote access.

## 2. Static IP Configuration

To prevent the IP address from changing on every reboot,
a static IP was configured using netplan.

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

## 3. SSH Remote Access

Connected to the server via SSH from a laptop.
Also set up VMware port forwarding to enable access from a mobile device.

```bash
ssh hchwang@192.168.13.250
```

## 4. Node.js + Express API Server

Installed Node.js and Express, then built a simple REST API server.

```bash
npm install express
node app.js
```

## 5. MySQL Integration

Installed MySQL, created a users table, and connected it to the API.
Sensitive information was separated into a .env file.

```bash
sudo apt install mysql-server -y
npm install mysql2 dotenv
```

## 6. CRUD API Complete

Implemented all HTTP methods to build a complete REST API.
GET    /users        # Get all users

POST   /users        # Add a user

PUT    /users/:id    # Update a user

DELETE /users/:id    # Delete a user

GET    /users/:id    # Get a single user

## 7. GitHub Integration

Generated an SSH key, registered it on GitHub, and pushed the code.
Used Claude Code to automate code writing and commits.

## Takeaways

Managed to go from server installation to a fully working API server in a single day.
By finding and fixing small mistakes like YAML indentation errors and bracket mismatches,
I was able to understand the overall flow of backend development.

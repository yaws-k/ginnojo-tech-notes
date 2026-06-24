---
---
# Ginnojo Tech Notes

A collection of technical notes and guides on setting up a multi-purpose server (web, database, mail, and various applications) using Debian GNU/Linux.

Everything is set up on the same server for simplicity and ease of management. For heavy-duty production environments, consider using separate servers for each service.

## Debian version

The guides are based on Debian 13 (Trixie), released in August 2025.

## Overview

An overview of the filesystem, OS, and software on the server.  
(Click the image to enlarge it.)

{% include figure popup=true image_path="/assets/images/overview.drawio.svg" alt="Overview" %}

- The `XFS` partition is used for MongoDB data storage.
- `certbot` runs as a snap package.
- `vouch-proxy` runs as a Docker container.

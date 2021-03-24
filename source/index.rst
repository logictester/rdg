.. SafeNet Trusted Access documentation master file, created by
   sphinx-quickstart on Wed Mar 24 16:12:19 2021.
   You can adapt this file completely to your liking, but it should at least
   contain the root `toctree` directive.

=========================================================================
Protecting Enterprise Remote Desktop Services with SafeNet Trusted Access
=========================================================================

.. toctree::
   :maxdepth: 3
   :hidden:

   index


Overview
========

This guide primarily documents the procedure for protecting Remote Desktop Services (RDS) through native enforcement in the Remote Desktop Gateway (RDGW) extending Network Policy Server (NPS) RADIUS to SafeNet Trusted Access (STA) and authenticating the requesting user with Push Authentication to MobilePASS+.
Secondarily, this guide also documents an alternative architecture where instead of RADIUS, a SafeNet agent is deployed co-located with NPS to facilitate agent-based authentication with SafeNet Trusted Access over encrypted tunnel (TLS 1.2 over 443). Related to this, a proposed High Availability (HA) architecture is documented also.
Lastly, the guide briefly expands on troubleshooting tools and procedures related to deployment.

.. note::

   This guide documents alternative deployment architectures. These do not replace SafeNet official deployment architectures using RDWeb and RDGW agents (see Support Portal of downloads and documentation of said agents).


Prerequisites
=============

  - RDGW is configured and published with valid public certificates
  - RDGW Resource Authorization Policy (RD RAP) is configured to allow access to required resources
  - NPS role is installed for RDGW Connection Authorization Policy (RD CAP)
  - Auth Node for the RDGW server is configured in STA
  - A user with the MobilePASS+ authenticator is enrolled


Solution Architecture
=====================

The high level target architecture is shown below. The user requests a remote desktop service and inputs his/her username and domain password. The access-request is intercepted by RDGW policy and forwarded to SafeNet Trusted Access (STA) over RADIUS (UDP 1812) via Microsoft NPS. STA then sends an OTP Push notification to the requesting user’s mobile device. The user approves the push request and is authorized to the remote desktop service.

.. thumbnail:: _images/pic.png

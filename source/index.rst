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
Secondarily, this guide also documents an alternative architecture where instead of RADIUS, a SafeNet agent is deployed co-located with NPS to facilitate agent-based authentication with SafeNet Trusted Access over encrypted tunnel (TLS 1.2 over 443). Related to this, a proposed High Availability (HA) architecture is documented as well.
Lastly, the guide briefly expands on troubleshooting tools and procedures related to deployment.

.. note::

   This guide documents alternative deployment architectures. These do not replace SafeNet official deployment architectures using RDWeb and RDGW agents (see Support Portal of downloads and documentation of said agents).


Prerequisites
=============

  - RDGW is configured and published with valid public certificates
  - RDGW Resource Authorization Policy (RD RAP) is configured to allow access to required resources
  - NPS role is installed for RDGW Connection Authorization Policy (RD CAP)
  - Auth Node for the RDGW server is configured in STA
  - A user with a MobilePASS+ authenticator is enrolled


Solution Architecture
=====================

The high level target architecture is shown below. The user requests a remote desktop service and inputs their username and domain password. The access-request is intercepted by RDGW policy and forwarded to SafeNet Trusted Access (STA) over RADIUS (UDP 1812) via Microsoft NPS. STA then sends an OTP Push notification to the requesting user’s mobile device. The user approves the push request and is authorized to the remote desktop service.

.. thumbnail:: _images/pic1.png

.. note::

   Please see Appendix A for additional deployment options, using an on-premise NPS server with STA NPS Agent installed




Step 1: Configure STA Auth Node for RDGW
========================================

To be able to use RDGW with STA RADIUS, an **Auth Node** has to be created with the Public IP of the RDG server. To do so:

1. Open STA (MFA Management Console)

2. Navigate to **Comms** tab

3. Scroll down to **Auth Nodes** and click on :guilabel:`Auth Nodes`

4. Click :guilabel:`Add` to add a new **Auth Node**:

.. _step:

.. thumbnail:: _images/pic5.png

5. In **Auth Node Name**, type in any name for this Auth Node, for example *RD Gateway*

6. In **Low IP Address in Range**, type in the Public IP of your RDG server

7. In **Configure FreeRADIUS Synchronization – Shared Secret**, type in a Shared Secret or click :guilabel:`Generate` to generate a random one (copy and save the shared secret to be used in :ref:`step 2.9 <c>`)

8. Click :guilabel:`Save` to save the newly created *Auth Node*



Step 2: Configure RD CAP policy
===============================

RD CAP configuration will create all the required policies in the local NPS server to allow forwarding of the RADIUS requests to the STA Cloud RADIUS. To do so:

1. Open Server Manager and click on :guilabel:`Remote Desktop Services`

2. In the Remote Desktop Services, click on :guilabel:`Servers`

3. Right click your server and select **RD Gateway Manager**:

.. _a:

.. thumbnail:: _images/pic2.png


4. In the RD Gateway Manager, right-click on the server name and expand **Policies**

5. Click on :guilabel:`Central Network Policy Servers`

6. Click on :guilabel:`Configure Central RD CAP`, server properties opens, click on :guilabel:`RD CAP Store`:

.. _b:

.. thumbnail:: _images/pic3.png

7. Select **Central server running NPS**

8. Type in the RADIUS server FQDN or IP (see reference below) in the **Enter a name or IP address for the server running NPS:** field and click :guilabel:`Add`

.. note:: Depending on your service zone select the appropriate FQDN or IP address (FQDN is preferred):

+-------------------------------------+-----------------------------+-------------------+
| SafeNet Trusted Access Service Zone |           FQDN              | IP Address        |
+=====================================+=============================+===================+
|                                     | ::                          | ::                |
|                                     |                             |                   |
|                                     |    radius1.eu.safenetid.com |    52.47.114.37   |
| EU Service Zone                     |                             |                   |
|                                     | ::                          | ::                |
|                                     |                             |                   |
|                                     |    radius2.eu.safenetid.com |    52.47.42.111   |
+-------------------------------------+-----------------------------+-------------------+
|                                     | ::                          | ::                |
| US Service Zone                     |                             |                   |
|                                     |    radius1.us.safenetid.com |    35.196.203.247 |
+-------------------------------------+-----------------------------+-------------------+
|                                     | ::                          | ::                |
|                                     |                             |                   |
|                                     |    radius1.safenet-inc.com  |    109.73.120.148 |
| Classic Service Zone                |                             |                   |
|                                     | ::                          | ::                |
|                                     |                             |                   |
|                                     |    radius2.safenet-inc.com  |    69.20.230.201  |
+-------------------------------------+-----------------------------+-------------------+

9. Type in your Shared Secret (Same Shared Secret configured in STA Auth Node in :ref:`step 1.7 <step>`) in the **Enter a new shared secret:** field and click :guilabel:`OK`

.. _c:

.. thumbnail:: _images/pic4.png

10. Click :guilabel:`Apply` and :guilabel:`OK` to save the RD CAP settings

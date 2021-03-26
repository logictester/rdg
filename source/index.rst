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

.. _architecture:

.. thumbnail:: _images/architecture.png
   :title: Figure: Architecture diagram.
   :show_caption: true

|

.. note::

   Please see :ref:`Appendix A <appendix-a>` for additional deployment options, using an on-premise NPS server with STA NPS Agent installed




Step 1: Configure STA Auth Node for RDGW
========================================

To be able to use RDGW with STA RADIUS, an **Auth Node** has to be created with the Public IP of the RDG server. To do so:

1. Open STA (MFA Management Console)

2. Navigate to **Comms** tab

3. Scroll down to **Auth Nodes** and click on :guilabel:`Auth Nodes`

4. Click :guilabel:`Add` to add a new **Auth Node**:

   .. thumbnail:: _images/addingAuthNode.png
      :width: 80%
      :align: center
      :title: Figure: Adding an Auth Node.

.. _step:

5. In **Auth Node Name**, type in any name for this Auth Node, for example *RD Gateway*

6. In **Low IP Address in Range**, type in the Public IP of your RDG server

7. In **Configure FreeRADIUS Synchronization – Shared Secret**, type in a Shared Secret or click :guilabel:`Generate` to generate a random one (copy and save the shared secret to be used in :ref:`step 2.9 <secret>`)

8. Click :guilabel:`Save` to save the newly created *Auth Node*



Step 2: Configure RD CAP policy
===============================

RD CAP configuration will create all the required policies in the local NPS server to allow forwarding of the RADIUS requests to the STA Cloud RADIUS. To do so:

1. Open Server Manager and click on :guilabel:`Remote Desktop Services`

2. In the Remote Desktop Services, click on :guilabel:`Servers`

3. Right-click your server and select **RD Gateway Manager**:

.. _rdg:

   .. thumbnail:: _images/selectRDGWManager.png
      :width: 80%
      :align: center
      :title: Figure: Right-click to select RD Gateway Manager.

4. In the RD Gateway Manager, right-click on the server name and expand **Policies**

5. Click on :guilabel:`Central Network Policy Servers`

6. Click on :guilabel:`Configure Central RD CAP`, server properties opens, click on :guilabel:`RD CAP Store`:

.. _cap:

   .. thumbnail:: _images/configureRDCAPStore.png
      :width: 80%
      :align: center
      :title: Figure: Configure RD CAP Store.

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

.. _secret:

9. Type in your **Shared Secret** (same **Shared Secret** configured in STA Auth Node in :ref:`step 1.7 <step>`) in the **Enter a new shared secret:** field and click :guilabel:`OK`

   .. thumbnail:: _images/setSharedSecret.png
      :width: 80%
      :align: center
      :title: Figure: Set the RADIUS shared secret.

|

10. Click :guilabel:`Apply` and :guilabel:`OK` to save the RD CAP settings


Step 3: Adjust NPS Policies and Settings
========================================

The configuration of the RD CAP Policy creates the necessary policies and settings in NPS, those policies and settings need to be adjusted to allow Push Authentication to work correctly. To do so:

1.	Open NPS and expand **RADIUS Clients and Servers**

2.	Click on :guilabel:`Remote RADIUS Server`

3.	A policy named **TS GATEWAY SERVER GROUP** will be available (created by RD CAP)

4.	Double-click to open the policy

5.	Select the RADIUS server (created by RD CAP) and click on :guilabel:`Edit`, the **Edit RADIUS Server** window opens

6.	Click on :guilabel:`Load Balancing`

7.	Adjust the value of: **Number of seconds without response before request is considered dropped:** and **Number of seconds between requests when the server is identified as unavailable:** to **60** (to allow enough time for the Push Authentication to complete):

.. _nps:

   .. thumbnail:: _images/setRequestTimeout.png
      :width: 80%
      :align: center
      :title: Figure: Set request timeout to accomodate Push OTP.

8.	Click :guilabel:`OK` to save the timeout settings and :guilabel:`Apply` and :guilabel:`OK` to save the RADIUS Server settings

9.	Expand **Policies** and click on :guilabel:`Connection Request Policies`

10.	A policy named **TS GATEWAY AUTHORIZATION POLICY** will be available (created by RD CAP)

11.	Double click to open the policy

12.	Click on :guilabel:`Conditions` and adjust the conditions based on the company needs (for example Group Membership or Date and Time restrictions)

13.	Click on :guilabel:`Settings` and click on :guilabel:`Authentication`, verify **Forward requests to the following remote RADIUS server group for authentication: TS GATEWAY SERVER GROUP** is selected:

.. _tsgroup:

   .. thumbnail:: _images/setGroup.png
      :width: 80%
      :align: center
      :title: Figure: Verify/set correct group.

.. note:: By default, RDG will send the username in DOMAIN\User format, if STA user doesn’t have a username or an alias in the same format, the authentication will fail. To overcome this, UPN stripping is required, please see the steps below to configure attribute manipulation rule to adjust the username

14.	Click on :guilabel:`Attribute`

15.	Change the Attribute to be adjusted to **User-Name**

16.	Click :guilabel:`Add` to add a new **Attribute Manipulation Rule**

17.	In **Find:** filed type: ::

                                (.*)\\(.*)

    and in **Replace:** field type ::

                                      $2

18.	Click :guilabel:`OK` to save the Attribute settings and :guilabel:`Apply` and :guilabel:`OK` to save the Connection Policy:

.. _policy:

   .. thumbnail:: _images/setAttribute.png
      :width: 80%
      :align: center
      :title: Figure: Adding attribute(s).

19.	Click on :guilabel:`Network Policies`

20.	Review and adjust the policy as required and then close NPS.

The RDG server side configurations are complete.


Step 4: Test the configuration
==============================

To test the configuration, adjust the RDP connection to use RD Gateway. The connection flow will first establish a connection to the RDG which will prompt the user to authenticate using domain username and password (if the credentials haven’t been saved yet) and send a Push Authentication to the user’s MobilePASS+ token, after successful Push Authentication, the user will be connected to the remote desktop. Depending on the RDP configuration, the user might be prompted for the credentials for the Remote Desktop or SSO can be used so the same credentials provided during the connection to RDG will be used for the Remote Desktop as well (Configured in Remote Desktop Services – Deployment Properties).

.. _mstsc:

   .. thumbnail:: _images/setDeploymentParameters.png
      :width: 80%
      :align: center
      :title: Figure: Setting deployment parameters prior test.

1.	Open RDP (mstsc)

2.	Click on :guilabel:`Advanced`

3.	Under Connect from Anywhere click on :guilabel:`Settings`

4.	Click on :guilabel:`Use these RD Gateway server settings:`

5.	In **Server Name**, type in the FQDN of your RDG server

6.	Adjust the **Log-on method** and **Bypass RD Gateway server for local addresses** as required:

.. _mstsc_server:

   .. thumbnail:: _images/adjustingLoginMethod.png
      :width: 80%
      :align: center
      :title: Figure: Adjusting RPP login.

7.	Click :guilabel:`OK` to save the settings

8.	Click :guilabel:`Connect` to initiate the RDP connection through RD Gateway

9.	The user is asked to provide the domain credentials (if the credentials haven’t been previously saved)

.. _mstsc_connection:

   .. thumbnail:: _images/windowsPassword.png
      :width: 80%
      :align: center
      :title: Figure: Providing Widows password as first factor.

10.	After providing the credentials, the connection initiates:

.. _mstsc_connect:


   .. thumbnail:: _images/connectingRDP.png
      :width: 80%
      :align: center
      :title: Figure: RDP connection being established.

11. The user receives and approves the Push Authentication request:

.. _push:

   .. thumbnail:: _images/approvingPush.png
      :width: 60%
      :align: center
      :title: Figure: Approving Push OTP on MobilePASS+.

12. After approving the Push Authentication request, the connection is established:

.. _connection:

   .. thumbnail:: _images/connectingRDPCont.png
      :width: 80%
      :align: center
      :title: Figure: Connection is finalized / established.

.. _appendix-a:

Appendix A: Deployment Options
==============================

The deployment demonstrated in this guide is based on RADIUS communication between Microsoft Network Policy Server (NPS) and SafeNet Trusted Access (STA). Moreover, the suggested model uses a single NPS instance.
It is also possible to deploy this solution using an additional NPS instance together with the SafeNet NPS Agent. In this deployment model, the RDGW NPS communicates locally to an additional NPS using RADIUS and acting as a client, and the STA NPS Agent then communicates to STA using HTTPS (443).

.. note:: The approach may be favorable when the first NPS instance is not allowed internet access or when RADIUS over the internet is not desired.

A high-level architecture of this alternative deployment model is shown below:

.. _2nps:

.. thumbnail:: _images/alternativeArchitecture.png
   :title: Figure: alternative deployment architecture.
   :show_caption: true

|

.. note:: Depending on your service zone select the appropriate FQDN or IP address to configure the STA NPS Agent (FQDN is preferred):


+-------------------------------------+---------------------------+------+
| SafeNet Trusted Access Service Zone |           FQDN            | Port |
+-------------------------------------+---------------------------+------+
|                                     | ::                        |      |
|                                     |                           |      |
| EU Service Zone                     |    cloud.eu.safenetid.com | 443  |
+-------------------------------------+---------------------------+------+
|                                     | ::                        |      |
|                                     |                           |      |
| US Service Zone                     |    cloud.us.safenetid.com | 443  |
+-------------------------------------+---------------------------+------+
|                                     | ::                        |      |
|                                     |                           |      |
|                                     |    agent1.safenet-inc.com | 443  |
| Classic Service Zone                |                           |      |
|                                     | ::                        |      |
|                                     |                           +------+
|                                     |                           |      |
|                                     |    agent2.safenet-inc.com | 443  |
+-------------------------------------+---------------------------+------+


Appendix B: High Availability
=============================

Expanding on the deployment architecture shown in :ref:`Appendix A <appendix-a>`, the below diagram shows a high-level architecture aimed at achieving redundancy or high availability in the solution. Here, components are mirrored to a second datacenter or zone and configured to communicate to two (2) RDG, NPS and so on. If one service is not available, another service is requested.
High availability in SafeNet Trusted Access (STA) is ensured by the service provider (Thales) as long as the customer configures FQDN’s instead of IP address in directing traffic to the STA virtual server.

.. _ha:


.. thumbnail:: _images/alternativeArchitectureHA.png
   :title: Figure: alternative deployment architecture with High Availability.
   :show_caption: true

|


Appendix C: Troubleshooting
===========================

Test and Troubleshooting Tools
******************************

If possible, download the following tools to the servers involved in your RDGW and STA architecture:

.. _ntradping:

**NTRadPing** - You can download NTRadPing :download:`here <_downloads/ntradping.zip>`

.. _wireshark:

**Wireshark** - You can download Wireshark `here <https://www.wireshark.org/download.html>`_

Command Reference
*****************

The following commands are used from command line (CMD) to more rapidly recycle the NPS service :

+----------------------------------+---------------------+-------------------------------------------------------------+
| Command                          | Description         | Message                                                     |
+----------------------------------+---------------------+-------------------------------------------------------------+
| ::                               |                     |                                                             |
|                                  |                     | The Network Policy Server service is stopping...            |
|    net stop ias                  | Stop NPS Service    |                                                             |
|                                  |                     | The Network Policy Server service was stopped successfully. |
+----------------------------------+---------------------+-------------------------------------------------------------+
| ::                               |                     |                                                             |
|                                  |                     | The Network Policy Server service is starting.              |
|    net start ias                 | Start NPS Service   |                                                             |
|                                  |                     | The Network Policy Server service was started successfully. |
+----------------------------------+---------------------+-------------------------------------------------------------+
| ::                               |                     | The Network Policy Server service is stopping...            |
|                                  |                     |                                                             |
|    net stop ias && net start ias | Restart NPS Service | The Network Policy Server service was stopped successfully. |
|                                  |                     |                                                             |
|                                  |                     | The Network Policy Server service is starting.              |
|                                  |                     |                                                             |
|                                  |                     | The Network Policy Server service was started successfully. |
+----------------------------------+---------------------+-------------------------------------------------------------+

Common Issues
*************

Missing or wrongly defined Auth Node
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Whether you configure NPS to communicate directly to SafeNet Trusted Access (STA) over RADIUS (UDP 1812) or you employ the SafeNet Agent for NPS to communicate to STA over TLS (443) you must define an Auth Node in in the virtual server corresponding to the external IP address of your server. Verify that IP using an online tool such as https://www.whatsmyip.org/ or by executing the following command in command line (CMD)

  ::

     curl ifconfig.me


Proxy impedes communication
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Open a command line window (CMD) and execute the following command to learn if a proxy is present:

  ::

     netsh winhttp show proxy

Set requests to inherit any proxy configuration from Internet Explorer settings:

  ::

     netsh winhttp import proxy source = ie

Firewall impedes communication
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Install Wireshark (:ref:`see above <wireshark>`) and run a capture when attempting a connection.

To simplify: run the test using NTRadPing (:ref:`see above <ntradping>`) first and if it is successful only then move on to include actual components. A successful connection will appear as below. An unsuccessful connection will likely fail following resolving the STA FQDNs to IP addresses (follow TCP stream and it will show reconnection attempts):

Succesful attmpt:

.. _noerror:


.. thumbnail:: _images/wiresharkConnectionSuccess.png
   :title: Figure: Successful connection shown in Wireshark.
   :show_caption: true

Failed attempt:

.. _error:


.. thumbnail:: _images/wiresharkConnectionFail.png
   :title: Figure: Unsuccessful connection shown in Wireshark.
   :show_caption: true

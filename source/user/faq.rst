==========================
Frequently Asked Questions
==========================

.. _faq-nat-ice-stun-turn:

About NAT, ICE, STUN, TURN
==========================

These are very important concepts that developers must understand well to work with WebRTC. Here is a collection of all Kurento material talking about these acronyms:

* Glossary:

  - :term:`What is NAT <NAT>`?
  - :term:`What is NAT Traversal <NAT Traversal>`?
  - :term:`What is ICE <ICE>`?
  - :term:`What is STUN <STUN>`?
  - :term:`What is TURN <TURN>`?

* Installing and configuring a STUN/TURN server:

  - :ref:`faq-coturn-install`
  - :ref:`faq-stun-test`
  - :ref:`faq-stun-configure`

* Troubleshooting :ref:`troubleshooting-webrtc`
* Advanced knowledge: :doc:`/knowledge/nat`



.. _faq-stun-needed:

When are STUN and TURN needed?
------------------------------

:term:`STUN` (and possibly :term:`TURN`) is needed **for every WebRTC participant behind a NAT**. All peers that try to connect from behind a :term:`NAT` will need to "*open*" their own ports, a process that is known as :term:`NAT Traversal`. This is achieved by using a STUN server that is deployed **outside of the NAT**.

The STUN server is configured to use a range of UDP & TCP ports. All those ports should also be opened to all traffic, in the server's network configuration or security group.

If you are installing Kurento in a NAT environment (e.g. if your server is behind a NAT firewall), you also need to configure an external STUN server, in */etc/kurento/modules/kurento/WebRtcEndpoint.conf.ini*. Similarly, all browser clients that are behind a NAT need to configure the STUN server details with the ``iceServers`` field of the `RTCPeerConnection constructor <https://developer.mozilla.org/en-US/docs/Web/API/RTCPeerConnection/RTCPeerConnection>`__.

**Example:**

Kurento Media Server and its Application Server are running in a cloud machine **without any NAT** or port restriction on incoming connections, while a browser client runs from a possibly restricted :term:`NAT` network that forbids incoming connections on any port that hasn't been "opened" in advance

The browser client may communicate with the Application Server for signaling purposes, but at the end of the day the bulk of the audio/video communication is done between the WebRTC engines of the browser and KMS.

.. figure:: /images/faq-stun-1.png
   :align:  center
   :alt:    NAT client without STUN

In scenarios like this, the client is able to send data to KMS because its NAT will allow outgoing packets. However, KMS will *not* be able to send data to the client, because the client's NAT is closed for incoming packets. This is solved by configuring the client to use a STUN server; this server will be used by the client's browser to open the appropriate ports in the NAT. After this operation, the client is now able to receive audio/video streams from KMS:

.. figure:: /images/faq-stun-2.png
   :align:  center
   :alt:    NAT client with STUN

This procedure is done by the :term:`ICE` implementation of the client's browser.

Note that you *can* also deploy KMS behind a NAT firewall, as long as KMS itself is also configured to use a STUN server.

Further reading:

* `WebRTC - How many STUN/TURN servers do I need to specify? <https://stackoverflow.com/questions/23292520/webrtc-how-many-stun-turn-servers-do-i-need-to-specify/23307588#23307588>`__.
* `What are STUN, TURN, and ICE? <https://www.twilio.com/docs/stun-turn/faq#faq-what-is-nat>`__ (`archive <https://web.archive.org/web/20181009181338/https://www.twilio.com/docs/stun-turn/faq>`__).



.. _faq-coturn-install:

How to install Coturn?
----------------------

Coturn is a :term:`STUN` server and :term:`TURN` relay, supporting all features required for the :term:`ICE` protocol and allowing to establish WebRTC connections from behind a :term:`NAT`.

Coturn can be installed directly from the Ubuntu package repositories:

.. code-block:: bash

   sudo apt-get update && sudo apt-get install --no-install-recommends --yes \
       coturn

To configure it for WebRTC, follow these steps:

1. Edit */etc/turnserver.conf*.

   This example configuration is a good first step; it will work for using Coturn with Kurento Media Server for WebRTC streams. However, you may want to change it according to your needs:

   .. code-block:: text

      # This server's external/public address, if Coturn is behind a NAT.
      # It must be an IP address, not a domain name.
      external-ip=<CoturnIp>

      # STUN listener port for UDP and TCP.
      # Default: 3478.
      #listening-port=<CoturnPort>

      # TURN lower and upper bounds of the UDP relay ports.
      # Default: 49152, 65535.
      #min-port=49152
      #max-port=65535

      # Uncomment to run server in 'normal' 'moderate' verbose mode.
      # Default: verbose mode OFF.
      #verbose

      # Use fingerprints in the TURN messages.
      fingerprint

      # Use long-term credential mechanism.
      lt-cred-mech

      # Realm used for the long-term credentials mechanism.
      realm=kurento.org

      # 'Static' user accounts for long-term credentials mechanism.
      user=<TurnUser>:<TurnPassword>

      # Set the log file name.
      # The log file can be reset sending a SIGHUP signal to the turnserver process.
      log-file=/var/log/turnserver/turnserver.log

      # Disable log file rollover and use log file name as-is.
      simple-log

   Notes:

   - The *external-ip* is necessary in cloud providers which use internal NATs, such as **Amazon EC2** (AWS). Write your server's **public** IP address, like ``198.51.100.1``, in the ``<CoturnIp>`` parameter. **It must be an IP address, not a domain name**.

   - The options *fingerprint*, *lt-cred-mech*, and *realm* are needed for WebRTC.

   - The *user* parameter is the most basic form of authorization to use the :term:`TURN` relay capabilities. Write your desired user name and password in the fields ``<TurnUser>`` and ``<TurnPassword>``.

   - Other parameters can be tuned as needed. For more information, check the Coturn help pages:

     - https://github.com/coturn/coturn/wiki/turnserver
     - https://github.com/coturn/coturn/wiki/CoturnConfig
     - A fully commented example configuration file: https://raw.githubusercontent.com/coturn/coturn/master/examples/etc/turnserver.conf

2. Edit the file */etc/default/coturn* and set

   .. code-block:: text

      TURNSERVER_ENABLED=1

   so the server starts automatically as a system service daemon.

3. Follow with the next sections to test that Coturn is working, and then set it up as your STUN/TURN server in Kurento Media Server.



.. _faq-stun-test:

How to test my STUN/TURN server?
--------------------------------

To test if your :term:`STUN`/:term:`TURN` server is functioning properly, open the `Trickle ICE test page <https://webrtc.github.io/samples/src/content/peerconnection/trickle-ice/>`__. In that page, follow these steps:

1. Remove any server that might be filled in already by default.

2. Fill in your STUN/TURN server details.

   - To only test STUN server (TURN relay will not be tested):

     .. code-block:: text

        stun:<StunServerIp>:<StunServerPort>

   - To test both STUN server and TURN relay:

     .. code-block:: text

        turn:<TurnServerIp>:<TurnServerPort>

     ... and also fill in the *TURN username* and *TURN password*.

3. Click on *Add Server*. You should have only **one entry** in the list, with your server details.

4. Click on *Gather candidates*. **Verify** that you get candidates of type ``srflx`` if you are testing STUN. Likewise, you should get candidates of type ``srflx`` *and* type ``relay`` if you are testing TURN.

   If you are missing any of the expected candidate types, *your STUN/TURN server is not working well* and WebRTC will fail. Check your server configuration, and your cloud provider's network settings.



.. _faq-stun-configure:

How to configure STUN/TURN?
---------------------------

To configure a :term:`STUN` server or :term:`TURN` relay with Kurento Media Server, you may use either of two methods:

A. Write the parameters into the file */etc/kurento/modules/kurento/WebRtcEndpoint.conf.ini*. Do this if your settings are static and you know them beforehand.

   To only use STUN server (TURN relay will not be used):

   .. code-block:: text

      stunServerAddress=<StunServerIp>
      stunServerPort=<StunServerPort>

   ``<StunServerIp>`` should be the public IP address of the STUN server. **It must be an IP address, not a domain name**. For example:

   .. code-block:: text

      stunServerAddress=198.51.100.1
      stunServerPort=3478

   To use both STUN server and TURN relay:

   .. code-block:: text

      turnURL=<TurnUser>:<TurnPassword>@<TurnServerIp>:<TurnServerPort>

   ``<TurnServerIp>`` should be the public IP address of the TURN relay. **It must be an IP address, not a domain name**. For example:

   .. code-block:: text

      turnURL=myuser:mypassword@198.51.100.1:3478

B. Use the API methods to set the parameters dynamically. Do this if your STUN server details are not known beforehand, or if your TURN credentials are generated on runtime:

   To only use STUN server (TURN relay will not be used):

   .. code-block:: text

      setStunServerAddress("<StunServerIp>");
      setStunServerPort(<StunServerPort>);

   Kurento Client API docs: `Java <https://doc-kurento.readthedocs.io/en/latest/_static/client-javadoc/org/kurento/client/WebRtcEndpoint.html#setStunServerAddress-java.lang.String->`__, `JavaScript <https://doc-kurento.readthedocs.io/en/latest/_static/client-jsdoc/module-elements.WebRtcEndpoint.html#setStunServerAddress>`__.

   To use both STUN server and TURN relay:

   .. code-block:: text

      setTurnUrl("<TurnUser>:<TurnPassword>@<TurnServerIp>:<TurnServerPort>");

   Kurento Client API docs: `Java <https://doc-kurento.readthedocs.io/en/latest/_static/client-javadoc/org/kurento/client/WebRtcEndpoint.html#setTurnUrl-java.lang.String->`__, `JavaScript <https://doc-kurento.readthedocs.io/en/latest/_static/client-jsdoc/module-elements.WebRtcEndpoint.html#setTurnUrl>`__.

.. note::

   **You don't need to set a STUN server up if you have already configured a TURN relay**, because TURN is just an extension of STUN.

The following ports should be open in the firewall or your cloud machine's *Security Groups*:

- **<CoturnPort>** (Default: 3478) UDP & TCP.
- **49152-65535** UDP & TCP: As per :rfc:`5766`, these are the ports that the TURN relay will use to exchange media. These ports can be changed using Coturn's ``min-port`` and ``max-port`` parameters.

.. note::

   **Port ranges must match between Coturn and Kurento Media Server**. Check the files */etc/turnserver.conf* and */etc/kurento/modules/kurento/BaseRtpEndpoint.conf.ini*, to verify that both will be using the same set of ports.

When you are done, (re)start both Coturn and Kurento servers:

.. code-block:: bash

   sudo service coturn restart
   sudo service kurento-media-server restart



How many Media Pipelines do I need for my Application?
======================================================

A Pipeline is a top-level container that handles every resource that should be able to achieve any kind of interaction with each other. Media Elements can only communicate when they are part of the same Pipeline. Different Pipelines in the server are independent, so they do not share audio, video, data or events.

99% times, this translates to using 1 Pipeline object for each "room"-like videoconference. It doesn't matter if there is 1 single presenter and N viewers ("one-to-many"), or if there are N participants Skype-style ("many-to-many"), all of them are managed by the same Pipeline. So, most actual real-world applications would only ever create 1 Pipeline, because that's good enough for most needs.

A good heuristic is that you will need one Pipeline per each set of communicating partners in a channel, and one Endpoint in this Pipeline per audio/video streams exchanged with a participant.



How many Endpoints do I need?
=============================

Your application will need to create at least one Endpoint for each media stream flowing to (or from) each participant. You might actually need more, if the streams are to be recorded or if streams are being duplicated for other purposes.



Which participant corresponds to which Endpoint?
================================================

The Kurento API offers no way to get application-level semantic attributes stored in a Media Element. However, the application developer can maintain a HashMap or equivalent data structure, storing the Endpoint identifiers (which are plain strings) to whatever application information is desired, such as the names of the participants.

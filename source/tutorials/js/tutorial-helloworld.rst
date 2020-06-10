%%%%%%%%%%%%%%%%%%%%%%%%
JavaScript - Hello world
%%%%%%%%%%%%%%%%%%%%%%%%

This web application has been designed to introduce the principles of
programming with Kurento for JavaScript developers. It consists of a
`WebRTC`:term: video communication in mirror (*loopback*). This tutorial
assumes you have basic knowledge of JavaScript, HTML and WebRTC. We also
recommend reading the :doc:`Introducing Kurento </user/about>`
section before starting this tutorial.



Running this example
====================

First of all, install Kurento Media Server: :doc:`/user/installation`. Start the media server and leave it running in the background.

Install :term:`Node.js`, :term:`Bower`, and a web server in your system:

.. code-block:: bash

   curl -sL https://deb.nodesource.com/setup_8.x | sudo -E bash -
   sudo apt-get install -y nodejs
   sudo npm install -g bower
   sudo npm install -g http-server

Here, we suggest using the simple Node.js ``http-server``, but you could use any other web server.

You also need the source code of this tutorial. Clone it from GitHub, then start the web server:

.. code-block:: bash

    git clone https://github.com/Kurento/kurento-tutorial-js.git
    cd kurento-tutorial-js/kurento-hello-world/
    git checkout |VERSION_TUTORIAL_JS|
    bower install
    http-server -p 8443 --ssl --cert keys/server.crt --key keys/server.key

Note that HTTPS is required by browsers to enable WebRTC, so the web server must use SSL and a certificate file. For instructions, check :ref:`features-security-js-https`. For convenience, this tutorial already provides dummy self-signed certificates (which will cause a security warning in the browser).

When your web server is up and running, use a WebRTC compatible browser (Firefox, Chrome) to open the tutorial page:

* If KMS is running in your local machine:

  .. code-block:: text

     https://localhost:8443/

* If KMS is running in a remote machine:

  .. code-block:: text

     https://localhost:8443/index.html?ws_uri=ws://{KMS_HOST}:8888/kurento

.. note::

   By default, this tutorial works out of the box by using non-secure WebSocket (``ws://``) to establish a client connection between the browser and KMS. This only works for ``localhost``. *It will fail if the web server is remote*.

If you want to run this tutorial from a **remote web server**, then you have to do 3 things:

1. Configure **Secure WebSocket** in KMS. For instructions, check :ref:`features-security-kms-wss`.

2. In *index.js*, change the ``ws_uri`` to use Secure WebSocket (``wss://`` instead of ``ws://``) and the correct KMS port (8433 instead of 8888).

3. As explained in the link from step 1, if you configured KMS to use Secure WebSocket with a self-signed certificate you now have to browse to ``https://{KMS_HOST}:8433/kurento`` and click to accept the untrusted certificate.



Understanding this example
==========================

Kurento provides developers a **Kurento JavaScript Client** to control
**Kurento Media Server**.  This client library can be used in any kind of
JavaScript application including desktop and mobile browsers.

This *hello world* demo is one of the simplest web applications you can create
with Kurento. The following picture shows an screenshot of this demo running:

.. figure:: ../../images/kurento-java-tutorial-1-helloworld-screenshot.png
   :align:   center
   :alt:     Kurento Hello World Screenshot: WebRTC in loopback

   *Kurento Hello World Screenshot: WebRTC in loopback*

The interface of the application (an HTML web page) is composed by two HTML5
video tags: one showing the local stream (as captured by the device webcam) and
the other showing the remote stream sent by the media server back to the client.

The logic of the application is quite simple: the local stream is sent to the
Kurento Media Server, which sends it back to the client without modifications.
To implement this behavior, we need to create a `Media Pipeline`:term: composed
by a single `Media Element`:term:, i.e. a **WebRtcEndpoint**, which holds the
capability of exchanging full-duplex (bidirectional) WebRTC media flows. This
media element is connected to itself,, so that the media it receives (from
browser) is send back (to browser). This media pipeline is illustrated in the
following picture:

.. figure:: ../../images/kurento-java-tutorial-1-helloworld-pipeline.png
   :align:   center
   :alt:     Kurento Hello World Media Pipeline in context

   *Kurento Hello World Media Pipeline in context*

This is a web application, and therefore it follows a client-server
architecture. Nevertheless, due to the fact that we are using the Kurento
JavaScript client, there is not need to use an application server since all the
application logic is held by the browser. The Kurento JavaScript Client is used
directly to control Kurento Media Server by means of a WebSocket bidirectional
connection:

.. figure:: ../../images/kurento-js-tutorial-1-helloworld-signaling.png
   :align:   center
   :alt:     Complete sequence diagram of Kurento Hello World (WebRTC in loopbak) demo

   *Complete sequence diagram of Kurento Hello World (WebRTC in loopbak) demo*

The following sections analyze in deep the client-side (JavaScript) code of this
application, the dependencies, and how to run the demo. The complete source
code can be found in
`GitHub <https://github.com/Kurento/kurento-tutorial-js/tree/master/kurento-hello-world>`_.

JavaScript Logic
================

The Kurento *hello-world* demo follows a *Single Page Application* architecture
(`SPA`:term:). The interface is the following HTML page:
`index.html <https://github.com/Kurento/kurento-tutorial-js/blob/master/kurento-hello-world/index.html>`_.
This web page links two Kurento JavaScript libraries:

* **kurento-client.js** : Implementation of the Kurento JavaScript Client.

* **kurento-utils.js** : Kurento utility library aimed to simplify the WebRTC
  management in the browser.

In addition, these two JavaScript libraries are also required:

* **Bootstrap** : Web framework for developing responsive web sites.

* **jquery.js** : Cross-platform JavaScript library designed to simplify the
  client-side scripting of HTML.

* **adapter.js** : WebRTC JavaScript utility library maintained by Google that
  abstracts away browser differences.

* **ekko-lightbox** : Module for Bootstrap to open modal images, videos, and
  galleries.

* **demo-console** : Custom JavaScript console.


The specific logic of the *Hello World* JavaScript demo is coded in the
following JavaScript file:
`index.js <https://github.com/Kurento/kurento-tutorial-js/blob/master/kurento-hello-world/js/index.js>`_.
In this file, there is a function which is called when the green button labeled
as *Start* in the GUI is clicked.

.. sourcecode:: js

   var startButton = document.getElementById("start");

   startButton.addEventListener("click", function() {
      var options = {
        localVideo: videoInput,
        remoteVideo: videoOutput
      };

      webRtcPeer = kurentoUtils.WebRtcPeer.WebRtcPeerSendrecv(options, function(error) {
         if(error) return onError(error)
         this.generateOffer(onOffer)
      });

      [...]
   }

The function *WebRtcPeer.WebRtcPeerSendrecv* abstracts the WebRTC internal
details (i.e. PeerConnection and getUserStream) and makes possible to start a
full-duplex WebRTC communication, using the HTML video tag with id *videoInput*
to show the video camera (local stream) and the video tag *videoOutput* to show
the remote stream provided by the Kurento Media Server.

Inside this function, a call to *generateOffer* is performed. This function
accepts a callback in which the SDP offer is received. In this callback we
create an instance of the *KurentoClient* class that will manage communications
with the Kurento Media Server. So, we need to provide the URI of its WebSocket
endpoint. In this example, we assume it's listening in port 8888 at the same
host than the HTTP serving the application.

.. sourcecode:: js

   [...]

   var args = getopts(location.search,
   {
     default:
     {
       ws_uri: 'ws://' + location.hostname + ':8888/kurento',
       ice_servers: undefined
     }
   });

   [...]

   kurentoClient(args.ws_uri, function(error, client){
     [...]
   };

Once we have an instance of ``kurentoClient``, we need to create a
*Media Pipeline*, as follows:

.. sourcecode:: js

   client.create("MediaPipeline", function(error, _pipeline){
      [...]
   });

If everything works correctly, we will have an instance of a media pipeline
(variable ``_pipeline`` in this example). With it, we are able to create
*Media Elements*. In this example we just need a single *WebRtcEndpoint*.

In WebRTC, :term:`SDP` is used for negotiating media exchanges between
applications. Such negotiation happens based on the SDP offer and answer
exchange mechanism by gathering the :term:`ICE` candidates as follows:

.. sourcecode:: js

   pipeline = _pipeline;

   pipeline.create("WebRtcEndpoint", function(error, webRtc){
      if(error) return onError(error);

      setIceCandidateCallbacks(webRtcPeer, webRtc, onError)

      webRtc.processOffer(sdpOffer, function(error, sdpAnswer){
        if(error) return onError(error);

        webRtcPeer.processAnswer(sdpAnswer, onError);
      });
      webRtc.gatherCandidates(onError);

      [...]
   });

Finally, the *WebRtcEndpoint* is connected to itself (i.e., in loopback):

.. sourcecode:: js

   webRtc.connect(webRtc, function(error){
      if(error) return onError(error);

      console.log("Loopback established");
   });

.. note::

   The :term:`TURN` and :term:`STUN` servers to be used can be configured simple adding
   the parameter ``ice_servers`` to the application URL, as follows:

   .. sourcecode:: bash

      https://localhost:8443/index.html?ice_servers=[{"urls":"stun:stun1.example.net"},{"urls":"stun:stun2.example.net"}]
      https://localhost:8443/index.html?ice_servers=[{"urls":"turn:turn.example.org","username":"user","credential":"myPassword"}]

Dependencies
============

All dependencies of this demo can to be obtained using `Bower`:term:. The list
of these dependencies are defined in the
`bower.json <https://github.com/Kurento/kurento-tutorial-js/blob/master/kurento-hello-world/bower.json>`_
file, as follows:

.. sourcecode:: js

   "dependencies": {
      "kurento-client": "|VERSION_CLIENT_JS|",
      "kurento-utils": "|VERSION_UTILS_JS|"
   }

To get these dependencies, just run the following shell command:

.. sourcecode:: bash

   bower install

.. note::

   We are in active development. You can find the latest version of
   Kurento JavaScript Client at `Bower <https://bower.io/search/?q=kurento-client>`_.

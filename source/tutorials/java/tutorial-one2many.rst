%%%%%%%%%%%%%%%%%%%%%%%%%%%%%
Java - One to many video call
%%%%%%%%%%%%%%%%%%%%%%%%%%%%%

This web application consists of a one-to-many video call using `WebRTC`:term:
technology. In other words, it is an implementation of a video broadcasting web
application.

.. note::

   This tutorial has been configured to use https. Follow the `instructions </features/security.html#configure-java-applications-to-use-https>`_
   to secure your application.

For the impatient: running this example
=======================================

First of all, you should install Kurento Media Server to run this demo. Please
visit the :doc:`installation guide </user/installation>` for further
information.

To launch the application, you need to clone the GitHub project where this demo
is hosted, and then run the main class:


.. sourcecode:: bash

    git clone https://github.com/Kurento/kurento-tutorial-java.git
    cd kurento-tutorial-java/kurento-one2many-call
    git checkout |VERSION_TUTORIAL_JAVA|
    mvn -U clean spring-boot:run

The web application starts on port 8443 in the localhost by default. Therefore,
open the URL https://localhost:8443/ in a WebRTC compliant browser (Chrome,
Firefox).

.. note::

   These instructions work only if Kurento Media Server is up and running in the same machine
   as the tutorial. However, it is possible to connect to a remote KMS in other machine, simply adding
   the flag ``kms.url`` to the JVM executing the demo. As we'll be using maven, you should execute
   the following command

   .. sourcecode:: bash

      mvn -U clean spring-boot:run \
          -Dspring-boot.run.jvmArguments="-Dkms.url=ws://{KMS_HOST}:8888/kurento"


Understanding this example
==========================

There will be two types of users in this application: 1 peer sending media
(let's call it *Presenter*) and N peers receiving the media from the
*Presenter* (let's call them *Viewers*). Thus, the Media Pipeline is composed
by 1+N interconnected *WebRtcEndpoints*. The following picture shows an
screenshot of the Presenter's web GUI:

.. figure:: ../../images/kurento-java-tutorial-3-one2many-screenshot.png
   :align:   center
   :alt:     One to many video call screenshot

   *One to many video call screenshot*

To implement this behavior we have to create a `Media Pipeline`:term: composed
by 1+N **WebRtcEndpoints**. The *Presenter* peer sends its stream to the rest
of the *Viewers*. *Viewers* are configured in receive-only mode. The
implemented media pipeline is illustrated in the following picture:

.. figure:: ../../images/kurento-java-tutorial-3-one2many-pipeline.png
   :align:   center
   :alt:     One to many video call Media Pipeline

   *One to many video call Media Pipeline*

This is a web application, and therefore it follows a client-server
architecture. At the client-side, the logic is implemented in **JavaScript**.
At the server-side, we use a Spring-Boot based application server consuming the
**Kurento Java Client** API, to control **Kurento Media Server** capabilities.
All in all, the high level architecture of this demo is three-tier. To
communicate these entities two WebSockets are used. First, a WebSocket is
created between client and server-side to implement a custom signaling
protocol. Second, another WebSocket is used to perform the communication
between the Kurento Java Client and the Kurento Media Server. This
communication is implemented by the **Kurento Protocol**. For further
information, please see this :doc:`page </features/kurento_protocol>`.

Client and application server communicate using a signaling protocol based on
`JSON`:term: messages over `WebSocket`:term: 's. The normal sequence between
client and server is as follows:

1. A *Presenter* enters in the system. There must be one and only one
*Presenter* at any time. For that, if a *Presenter* has already present, an
error message is sent if another user tries to become *Presenter*.

2. N *Viewers* connect to the presenter. If no *Presenter* is present, then an
error is sent to the corresponding *Viewer*.

3. *Viewers* can leave the communication at any time.

4. When the *Presenter* finishes the session each connected *Viewer* receives an
*stopCommunication* message and also terminates its session.


We can draw the following sequence diagram with detailed messages between
clients and server:

.. figure:: ../../images/kurento-java-tutorial-3-one2many-signaling.png
   :align:   center
   :alt:     One to many video call signaling protocol

   *One to many video call signaling protocol*

As you can see in the diagram, `SDP`:term: and :term:`ICE` candidates need to be
exchanged between client and server to establish the `WebRTC`:term: connection
between the Kurento client and server. Specifically, the SDP negotiation
connects the WebRtcPeer in the browser with the WebRtcEndpoint in the server.
The complete source code of this demo can be found in
`GitHub <https://github.com/Kurento/kurento-tutorial-java/tree/master/kurento-one2many-call>`_.

Application Server Logic
========================

This demo has been developed using **Java** in the server-side, based on the
`Spring Boot`:term: framework, which embeds a Tomcat web server within the
generated maven artifact, and thus simplifies the development and deployment
process.

.. note::

   You can use whatever Java server side technology you prefer to build web
   applications with Kurento. For example, a pure Java EE application, SIP
   Servlets, Play, Vert.x, etc. We chose Spring Boot for convenience.

In the following, figure you can see a class diagram of the server side code:

.. figure:: ../../images/digraphs/One2Many.png
   :align: center
   :alt:   Server-side class diagram of the One2Many app

   *Server-side class diagram of the One2Many app*

..
 digraph:: One2Many
   :caption: Server-side class diagram of the One2Many app

   size="12,8"; fontname = "Bitstream Vera Sans" fontsize = 8

   node [
        fontname = "Bitstream Vera Sans" fontsize = 8 shape = "record"
         style=filled
        fillcolor = "#E7F2FA"
   ]

   edge [
        fontname = "Bitstream Vera Sans" fontsize = 8 arrowhead = "vee"
   ]

   One2ManyCallApp -> CallHandler; One2ManyCallApp -> KurentoClient;
   CallHandler -> UserSession; CallHandler -> KurentoClient [constraint = false]

The main class of this demo is named
`One2ManyCallApp <https://github.com/Kurento/kurento-tutorial-java/blob/master/kurento-one2many-call/src/main/java/org/kurento/tutorial/one2manycall/One2ManyCallApp.java>`_.
As you can see, the *KurentoClient* is instantiated in this class as a Spring
Bean. This bean is used to create **Kurento Media Pipelines**, which are used
to add media capabilities to your applications. In this instantiation we see
that a WebSocket is used to connect with Kurento Media Server, by default in
the *localhost* and listening in the port 8888.

.. sourcecode:: java

   @EnableWebSocket
   @SpringBootApplication
   public class One2ManyCallApp implements WebSocketConfigurer {

      @Bean
      public CallHandler callHandler() {
         return new CallHandler();
      }

      @Bean
      public KurentoClient kurentoClient() {
         return KurentoClient.create();
      }

      public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
         registry.addHandler(callHandler(), "/call");
      }

      public static void main(String[] args) throws Exception {
         new SpringApplication(One2ManyCallApp.class).run(args);
      }

   }

This web application follows a *Single Page Application* architecture
(`SPA`:term:), and uses a `WebSocket`:term: to communicate client with server
by means of requests and responses. Specifically, the main app class implements
the interface ``WebSocketConfigurer`` to register a ``WebSocketHandler`` to
process WebSocket requests in the path ``/call``.

`CallHandler <https://github.com/Kurento/kurento-tutorial-java/blob/master/kurento-one2many-call/src/main/java/org/kurento/tutorial/one2manycall/CallHandler.java>`_
class implements ``TextWebSocketHandler`` to handle text WebSocket requests.
The central piece of this class is the method ``handleTextMessage``. This
method implements the actions for requests, returning responses through the
WebSocket. In other words, it implements the server part of the signaling
protocol depicted in the previous sequence diagram.

In the designed protocol there are three different kind of incoming messages to
the *Server* : ``presenter``, ``viewer``,  ``stop``, and ``onIceCandidate``.
These messages are treated in the *switch* clause, taking the proper steps in
each case.

.. sourcecode:: java

   public class CallHandler extends TextWebSocketHandler {

      private static final Logger log = LoggerFactory.getLogger(CallHandler.class);
      private static final Gson gson = new GsonBuilder().create();

      private final ConcurrentHashMap<String, UserSession> viewers = new ConcurrentHashMap<String, UserSession>();

      @Autowired
      private KurentoClient kurento;

      private MediaPipeline pipeline;
      private UserSession presenterUserSession;

      @Override
      public void handleTextMessage(WebSocketSession session, TextMessage message) throws Exception {
         JsonObject jsonMessage = gson.fromJson(message.getPayload(), JsonObject.class);
         log.debug("Incoming message from session '{}': {}", session.getId(), jsonMessage);

         switch (jsonMessage.get("id").getAsString()) {
         case "presenter":
            try {
               presenter(session, jsonMessage);
            } catch (Throwable t) {
               handleErrorResponse(t, session, "presenterResponse");
            }
            break;
         case "viewer":
            try {
               viewer(session, jsonMessage);
            } catch (Throwable t) {
               handleErrorResponse(t, session, "viewerResponse");
            }
            break;
         case "onIceCandidate": {
            JsonObject candidate = jsonMessage.get("candidate").getAsJsonObject();

            UserSession user = null;
            if (presenterUserSession != null) {
               if (presenterUserSession.getSession() == session) {
                  user = presenterUserSession;
               } else {
                  user = viewers.get(session.getId());
               }
            }
            if (user != null) {
               IceCandidate cand = new IceCandidate(candidate.get("candidate").getAsString(),
                     candidate.get("sdpMid").getAsString(), candidate.get("sdpMLineIndex").getAsInt());
               user.addCandidate(cand);
            }
            break;
         }
         case "stop":
            stop(session);
            break;
         default:
            break;
         }
      }

      private void handleErrorResponse(Throwable t, WebSocketSession session,
            String responseId) throws IOException {
         stop(session);
         log.error(t.getMessage(), t);
         JsonObject response = new JsonObject();
         response.addProperty("id", responseId);
         response.addProperty("response", "rejected");
         response.addProperty("message", t.getMessage());
         session.sendMessage(new TextMessage(response.toString()));
      }

      private synchronized void presenter(final WebSocketSession session, JsonObject jsonMessage) throws IOException {
         ...
      }

      private synchronized void viewer(final WebSocketSession session, JsonObject jsonMessage) throws IOException {
         ...
      }

      private synchronized void stop(WebSocketSession session) throws IOException {
         ...
      }

      @Override
      public void afterConnectionClosed(WebSocketSession session, CloseStatus status) throws Exception {
         stop(session);
      }

   }

In the following snippet, we can see the ``presenter`` method. It creates a
Media Pipeline and the ``WebRtcEndpoint`` for ``presenter``:

.. sourcecode:: java

   private synchronized void presenter(final WebSocketSession session, JsonObject jsonMessage) throws IOException {
      if (presenterUserSession == null) {
         presenterUserSession = new UserSession(session);

         pipeline = kurento.createMediaPipeline();
         presenterUserSession.setWebRtcEndpoint(new WebRtcEndpoint.Builder(pipeline).build());

         WebRtcEndpoint presenterWebRtc = presenterUserSession.getWebRtcEndpoint();

         presenterWebRtc.addIceCandidateFoundListener(new EventListener<IceCandidateFoundEvent>() {

            @Override
            public void onEvent(IceCandidateFoundEvent event) {
               JsonObject response = new JsonObject();
               response.addProperty("id", "iceCandidate");
               response.add("candidate", JsonUtils.toJsonObject(event.getCandidate()));
               try {
                  synchronized (session) {
                     session.sendMessage(new TextMessage(response.toString()));
                  }
               } catch (IOException e) {
                  log.debug(e.getMessage());
               }
            }
         });

         String sdpOffer = jsonMessage.getAsJsonPrimitive("sdpOffer").getAsString();
         String sdpAnswer = presenterWebRtc.processOffer(sdpOffer);

         JsonObject response = new JsonObject();
         response.addProperty("id", "presenterResponse");
         response.addProperty("response", "accepted");
         response.addProperty("sdpAnswer", sdpAnswer);

         synchronized (session) {
            presenterUserSession.sendMessage(response);
         }
         presenterWebRtc.gatherCandidates();

      } else {
         JsonObject response = new JsonObject();
         response.addProperty("id", "presenterResponse");
         response.addProperty("response", "rejected");
         response.addProperty("message", "Another user is currently acting as sender. Try again later ...");
         session.sendMessage(new TextMessage(response.toString()));
      }
   }

The ``viewer`` method is similar, but not he *Presenter* WebRtcEndpoint is
connected to each of the viewers WebRtcEndpoints, otherwise an error is sent
back to the client.

.. sourcecode:: java

   private synchronized void viewer(final WebSocketSession session, JsonObject jsonMessage) throws IOException {
      if (presenterUserSession == null || presenterUserSession.getWebRtcEndpoint() == null) {
         JsonObject response = new JsonObject();
         response.addProperty("id", "viewerResponse");
         response.addProperty("response", "rejected");
         response.addProperty("message", "No active sender now. Become sender or . Try again later ...");
         session.sendMessage(new TextMessage(response.toString()));
      } else {
         if (viewers.containsKey(session.getId())) {
            JsonObject response = new JsonObject();
            response.addProperty("id", "viewerResponse");
            response.addProperty("response", "rejected");
            response.addProperty("message",
                  "You are already viewing in this session. Use a different browser to add additional viewers.");
            session.sendMessage(new TextMessage(response.toString()));
            return;
         }
         UserSession viewer = new UserSession(session);
         viewers.put(session.getId(), viewer);

         String sdpOffer = jsonMessage.getAsJsonPrimitive("sdpOffer").getAsString();

         WebRtcEndpoint nextWebRtc = new WebRtcEndpoint.Builder(pipeline).build();

         nextWebRtc.addIceCandidateFoundListener(new EventListener<IceCandidateFoundEvent>() {

            @Override
            public void onEvent(IceCandidateFoundEvent event) {
               JsonObject response = new JsonObject();
               response.addProperty("id", "iceCandidate");
               response.add("candidate", JsonUtils.toJsonObject(event.getCandidate()));
               try {
                  synchronized (session) {
                     session.sendMessage(new TextMessage(response.toString()));
                  }
               } catch (IOException e) {
                  log.debug(e.getMessage());
               }
            }
         });

         viewer.setWebRtcEndpoint(nextWebRtc);
         presenterUserSession.getWebRtcEndpoint().connect(nextWebRtc);
         String sdpAnswer = nextWebRtc.processOffer(sdpOffer);

         JsonObject response = new JsonObject();
         response.addProperty("id", "viewerResponse");
         response.addProperty("response", "accepted");
         response.addProperty("sdpAnswer", sdpAnswer);

         synchronized (session) {
            viewer.sendMessage(response);
         }
         nextWebRtc.gatherCandidates();
      }
   }

Finally, the ``stop`` message finishes the communication. If this message is
sent by the *Presenter*, a ``stopCommunication`` message is sent to each
connected *Viewer*:

.. sourcecode:: java

   private synchronized void stop(WebSocketSession session) throws IOException {
      String sessionId = session.getId();
      if (presenterUserSession != null && presenterUserSession.getSession().getId().equals(sessionId)) {
         for (UserSession viewer : viewers.values()) {
            JsonObject response = new JsonObject();
            response.addProperty("id", "stopCommunication");
            viewer.sendMessage(response);
         }

         log.info("Releasing media pipeline");
         if (pipeline != null) {
            pipeline.release();
         }
         pipeline = null;
         presenterUserSession = null;
      } else if (viewers.containsKey(sessionId)) {
         if (viewers.get(sessionId).getWebRtcEndpoint() != null) {
            viewers.get(sessionId).getWebRtcEndpoint().release();
         }
         viewers.remove(sessionId);
      }
   }

Client-Side
===========

Let's move now to the client-side of the application. To call the previously
created WebSocket service in the server-side, we use the JavaScript class
``WebSocket``. We use a specific Kurento JavaScript library called
**kurento-utils.js** to simplify the WebRTC interaction with the server. This
library depends on **adapter.js**, which is a JavaScript WebRTC utility
maintained by Google that abstracts away browser differences. Finally
**jquery.js** is also needed in this application.

These libraries are linked in the
`index.html <https://github.com/Kurento/kurento-tutorial-java/blob/master/kurento-one2many-call/src/main/resources/static/index.html>`_
web page, and are used in the
`index.js <https://github.com/Kurento/kurento-tutorial-java/blob/master/kurento-one2many-call/src/main/resources/static/js/index.js>`_.
In the following snippet we can see the creation of the WebSocket (variable
``ws``) in the path ``/call``. Then, the ``onmessage`` listener of the
WebSocket is used to implement the JSON signaling protocol in the client-side.
Notice that there are four incoming messages to client: ``presenterResponse``,
``viewerResponse``, ``iceCandidate``, and ``stopCommunication``. Convenient
actions are taken to implement each step in the communication. For example, in
the function ``presenter`` the function ``WebRtcPeer.WebRtcPeerSendonly`` of
*kurento-utils.js* is used to start a WebRTC communication. Then,
``WebRtcPeer.WebRtcPeerRecvonly`` is used in the ``viewer`` function.

.. sourcecode:: javascript

   var ws = new WebSocket('ws://' + location.host + '/call');

   ws.onmessage = function(message) {
      var parsedMessage = JSON.parse(message.data);
      console.info('Received message: ' + message.data);

      switch (parsedMessage.id) {
      case 'presenterResponse':
         presenterResponse(parsedMessage);
         break;
      case 'viewerResponse':
         viewerResponse(parsedMessage);
         break;
      case 'iceCandidate':
          webRtcPeer.addIceCandidate(parsedMessage.candidate, function (error) {
           if (!error) return;
            console.error("Error adding candidate: " + error);
          });
          break;
      case 'stopCommunication':
         dispose();
         break;
      default:
         console.error('Unrecognized message', parsedMessage);
      }
   }

   function presenter() {
      if (!webRtcPeer) {
         showSpinner(video);

         var options = {
                  localVideo: video,
                  onicecandidate: onIceCandidate
                }
         webRtcPeer = new kurentoUtils.WebRtcPeer.WebRtcPeerSendonly(options,
            function (error) {
              if(error) {
                 return console.error(error);
              }
              webRtcPeer.generateOffer(onOfferPresenter);
         });
      }
   }

   function viewer() {
      if (!webRtcPeer) {
         showSpinner(video);

         var options = {
                  remoteVideo: video,
                  onicecandidate: onIceCandidate
                }
         webRtcPeer = new kurentoUtils.WebRtcPeer.WebRtcPeerRecvonly(options,
            function (error) {
              if(error) {
                 return console.error(error);
              }
             this.generateOffer(onOfferViewer);
         });
      }
   }

Dependencies
============

This Java Spring application is implemented using `Maven`:term:. The relevant
part of the
`pom.xml <https://github.com/Kurento/kurento-tutorial-java/blob/master/kurento-show-data-channel/pom.xml>`_
is where Kurento dependencies are declared. As the following snippet shows, we
need two dependencies: the Kurento Client Java dependency (*kurento-client*)
and the JavaScript Kurento utility library (*kurento-utils*) for the
client-side. Other client libraries are managed with
`webjars <https://www.webjars.org/>`_:

.. sourcecode:: xml

   <dependencies>
      <dependency>
         <groupId>org.kurento</groupId>
         <artifactId>kurento-client</artifactId>
      </dependency>
      <dependency>
         <groupId>org.kurento</groupId>
         <artifactId>kurento-utils-js</artifactId>
      </dependency>
      <dependency>
         <groupId>org.webjars</groupId>
         <artifactId>webjars-locator</artifactId>
      </dependency>
      <dependency>
         <groupId>org.webjars.bower</groupId>
         <artifactId>bootstrap</artifactId>
      </dependency>
      <dependency>
         <groupId>org.webjars.bower</groupId>
         <artifactId>demo-console</artifactId>
      </dependency>
      <dependency>
         <groupId>org.webjars.bower</groupId>
         <artifactId>adapter.js</artifactId>
      </dependency>
      <dependency>
         <groupId>org.webjars.bower</groupId>
         <artifactId>jquery</artifactId>
      </dependency>
      <dependency>
         <groupId>org.webjars.bower</groupId>
         <artifactId>ekko-lightbox</artifactId>
      </dependency>
   </dependencies>

.. note::

   We are in active development. You can find the latest version of
   Kurento Java Client at `Maven Central <https://search.maven.org/#search%7Cga%7C1%7Ckurento-client>`_.

Kurento Java Client has a minimum requirement of **Java 7**. Hence, you need to
include the following properties in your pom:

.. sourcecode:: xml

   <maven.compiler.target>1.7</maven.compiler.target>
   <maven.compiler.source>1.7</maven.compiler.source>

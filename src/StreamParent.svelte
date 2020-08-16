<script>
  import  { onMount } from "svelte";
  import adapter from "webrtc-adapter";
  import { WebRtcPeer } from "kurento-utils";
  const KurentoClient = require("kurento-client");

  let webRtcPeerInstance;
  let pipeline;


  const args = {
    // Non-secure WebSocket
    // Only valid for localhost access! Browsers won't allow using this for
    // URLs that are not localhost. Also, this matches the default KMS config:
    ws_uri: "ws://127.0.0.1:8888/kurento",

    // Secure WebSocket
    // Valid for localhost and remote access. To use this, you have to edit the
    // KMS settings file "kurento.conf.json", and configure the section
    // "mediaServer.net.websocket.secure". Check the docs:
    // https://doc-kurento.readthedocs.io/en/latest/features/security.html#features-security-kms-wss
    //ws_uri: "wss://" + location.hostname + ":8433/kurento",

    ice_servers: undefined
  };

  function setIceCandidateCallbacks(webRtcPeer, webRtcEp, onerror)
{
  webRtcPeer.on('icecandidate', function(candidate) {
    console.log("Local candidate:",candidate);

    candidate = KurentoClient.getComplexType('IceCandidate')(candidate);

    webRtcEp.addIceCandidate(candidate, onerror)
  });

  webRtcEp.on('IceCandidateFound', function(event) {
    var candidate = event.candidate;

    console.log("Remote candidate:",candidate);

    webRtcPeer.addIceCandidate(candidate, onerror);
  });
}


  function startHandle(e) {
  let videoInput = document.getElementById("videoInput");
  let videoOutput = document.getElementById("videoOutput");
  console.log(videoInput);
    const options = {
      localVideo: videoInput,
      remoteVideo: videoOutput,
      configuration: {
        iceServers: undefined,
      },
    };

    webRtcPeerInstance = WebRtcPeer.WebRtcPeerSendrecv(options, function (
      error
    ) {
      if (error) return onError(error);

      this.generateOffer(onOffer);
    });
  }

  function onOffer(error, sdpOffer) {
    if (error) return onError(error);

    KurentoClient(args.ws_uri, function (error, client) {
      if (error) return onError(error);

      client.create("MediaPipeline", function (error, _pipeline) {
        if (error) return onError(error);

        pipeline = _pipeline;

        pipeline.create("WebRtcEndpoint", function (error, webRtc) {
          if (error) return onError(error);

          setIceCandidateCallbacks(webRtcPeerInstance, webRtc, onError);

          webRtc.processOffer(sdpOffer, function (error, sdpAnswer) {
            if (error) return onError(error);

            webRtcPeerInstance.processAnswer(sdpAnswer, onError);
          });
          webRtc.gatherCandidates(onError);

          webRtc.connect(webRtc, function (error) {
            if (error) return onError(error);

            console.log("Loopback established");
          });
        });
      });
    });
  }

  function stopHandle() {
    if (webRtcPeerInstance) {
      webRtcPeerInstance.dispose();
      webRtcPeerInstance = null;
    }

    if (pipeline) {
      pipeline.release();
      pipeline = null;
    }
  }

  function onError(error) {
    if (error) {
      console.error(error);
      stop();
    }
  }
</script>

<div class="col-md-5">
  <h3>Local stream</h3>
  <video id="videoInput" autoplay width="480px" height="360px" />
</div>

<div class="col-md-5">
  <h3>Remote stream</h3>
  <video id="videoOutput" autoplay width="480px" height="360px" />
</div>

<div class="col-md-2">
  <button on:click={startHandle}>Start</button>
  <button on:click={stopHandle}>Stop</button>
</div>

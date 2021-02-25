<template>
  <div id="app">
    <div id="room-selection-container" class="centered">
      <h1>WebRTC video conference</h1>
      <label>Enter the number of the room you want to connect</label>
      <input id="room-input" type="text"/>
      <button id="connect-button" v-on:click="this.callUser">CONNECT</button>
    </div>

    <div id="video-chat-container" class="video-position">
      <video id="local-video" playsinline autoplay></video>
      <video id="remote-video" playsinline autoplay></video>
    </div>
  </div>
</template>

<script>

const token = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJlbWFpbCI6InNhcmFnZTkyQG1haWwucnUiLCJpZCI6IjVlOGVjNTdmNTg3NTY1MmI2MGQ5NmVkMSIsImlhdCI6MTYwNzA2MDgwNCwiZXhwIjoxNjA3MDY4MDA0LCJpc3MiOiJETUFTVEVSUyJ9.t5DIFzO7hCSIEd4qBKuCdLnrHUvZnbC1VBSUEjtj8-U";

// eslint-disable-next-line no-undef
window.io = io("http://localhost:9090/videochat", {query: {token}});

export default {
  name: "app",
  components: {},
  async created() {
  	this.PeerConnectionClass = window.RTCPeerConnection ||
    window.mozRTCPeerConnection ||
    window.webkitRTCPeerConnection ||
    window.msRTCPeerConnection;

    this.SessionDescriptionClass = window.RTCSessionDescription ||
    window.mozRTCSessionDescription ||
    window.webkitRTCSessionDescription ||
    window.msRTCSessionDescription;

    navigator.getUserMedia  = navigator.getUserMedia ||
    navigator.webkitGetUserMedia ||
    navigator.mozGetUserMedia ||
    navigator.msGetUserMedia;

  	this.subSubscriptEvents()
  },
  data() {
    return {
      PeerConnectionClass: null,
      SessionDescriptionClass: null,
      localStream: null,
      mediaConstraints: {
        audio: true,
        video: {width: 800, height: 720},
      },
      remoteStream: null,
      rtcPeerConnection: null,
      remotePeerConnection: null,
      iceServers: [
					{
				      'urls': 'stun:stun.l.google.com:19302'
				    },
				    {
				      'urls': 'turn:192.158.29.39:3478?transport=udp',
				      'credential': 'JZEOEt2V3Qb0y27GRntt2u2PAYA=',
				      'username': '28224511:1379330808'
				    },
				    {
				      'urls': 'turn:192.158.29.39:3478?transport=tcp',
				      'credential': 'JZEOEt2V3Qb0y27GRntt2u2PAYA=',
				      'username': '28224511:1379330808'
				    }
				]
	}	
  },
  methods: {
  	async onIceMethod(){
  		console.log('On ice methods')
  		await this.setLocalStream(this.mediaConstraints)
  		this.localStream.getTracks().forEach(track => this.rtcPeerConnection.addTrack(track, this.localStream));
  	},
   	subSubscriptEvents(){
   		window.io.on('connect',()=>{
   			console.log('Current socket id', window.io.id)
   			this.rtcPeerConnection = new this.PeerConnectionClass({
   				iceServers: this.iceServers
   			})
   		});

        window.io.on('call-made', async (data) => {
          if (data.from !== window.io.id) {
          	 console.log('Call from ', data)
          	await this.onIceMethod()
			await this.rtcPeerConnection.setRemoteDescription(new this.SessionDescriptionClass(data.offer));
			const answer = await this.rtcPeerConnection.createAnswer();
			await this.rtcPeerConnection.setLocalDescription(answer);
			window.io.emit('call-response',{
				from: window.io.id,
				to: data.from,
				sdp: answer
			})
			this.rtcPeerConnection.ontrack = (event) =>  {
				const remoteVideo = document.getElementById("remote-video");
				if (remoteVideo.srcObject) {
					console.log('srcObject',remoteVideo.srcObject)
					return
				};
				remoteVideo.srcObject  = event.streams[0]
			};
          }
        });

        window.io.on('call-response', async (data) => {
			if(data.to == window.io.id.toString()){
					console.log('call-response', data)
					await this.rtcPeerConnection.setRemoteDescription(new this.SessionDescriptionClass(data.sdp));
					window.io.emit('call-response-full', data);
					console.log('RTC', this.rtcPeerConnection)
					this.rtcPeerConnection.ontrack = (event) =>  {
				const remoteVideo = document.getElementById("remote-video");
				if (remoteVideo.srcObject) {
					console.log('srcObject',remoteVideo.srcObject)
					return
				};
				remoteVideo.srcObject  = event.streams[0]
			};
			}
        });

        window.io.on('call-response-full', (data) => {
        	console.log('RTC', this.rtcPeerConnection)
        })
  	},
  	async callUser(event){
  		if(!this.rtcPeerConnection){
  			this.rtcPeerConnection = new this.PeerConnectionClass({
   				iceServers: this.iceServers
   			})
  		}
  		await this.onIceMethod()

        const offer = await this.rtcPeerConnection.createOffer();
 		await this.rtcPeerConnection.setLocalDescription(offer);
        window.io.emit('call-made', {
          from: window.io.id,
          offer
        })
  	},
    async init() {
      window.io.on('connect', async () => {
        console.log('My socket id ', window.io.id)
      })
    },
    async setLocalStream(mediaConstraints) {
      let stream
      try {
        stream = await window.navigator.mediaDevices.getUserMedia(mediaConstraints)
      } catch (error) {
        console.error('Could not get user media', error)
      }

      this.localStream = stream
      const localVideoComponent = document.getElementById('local-video')
      localVideoComponent.srcObject = stream
      this.localStream = stream
      console.log('local stream', stream)
    },
  }
};
</script>

<style>
#app {
  font-family: "Avenir", Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
.video-chat-container{
	display: block;
}
</style>

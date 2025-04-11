<template>
    <div>
        <h1>Join Video Call</h1>
        <div>
            <input v-model="roomId" placeholder="Enter Room ID" />
            <button @click="startCall">Join</button>
        </div>
        <div v-if="inRoom">
            <video ref="myVideo" autoplay playsinline muted></video>
            <video ref="remoteVideo" autoplay playsinline></video>
        </div>
    </div>
</template>

<script setup>
import { ref } from "vue";
import { io } from "socket.io-client";

const roomId = ref("");
const inRoom = ref(false);

const myVideo = ref(null);
const remoteVideo = ref(null);

const socket = io(import.meta.env.VITE_SIGNALING_SERVER);
let localStream;
let peerConnection;

const config = {
    iceServers: [{ urls: "stun:stun.l.google.com:19302" }]
};

const startCall = async () => {
    if (!roomId.value) return alert("Room ID required");

    await initStream();
    createPeerConnection();
    socket.emit("join", roomId.value);
    inRoom.value = true;
};

async function initStream() {
    localStream = await navigator.mediaDevices.getUserMedia({ video: true, audio: true });
    myVideo.value.srcObject = localStream;
}

function createPeerConnection() {
    peerConnection = new RTCPeerConnection(config);

    localStream.getTracks().forEach(track => {
        peerConnection.addTrack(track, localStream);
    });

    peerConnection.ontrack = event => {
        remoteVideo.value.srcObject = event.streams[0];
    };

    peerConnection.onicecandidate = event => {
        if (event.candidate) {
            socket.emit("ice-candidate", { room: roomId.value, candidate: event.candidate });
        }
    };
}

socket.on("user-joined", async () => {
  const offer = await peerConnection.createOffer();
  await peerConnection.setLocalDescription(offer);
  socket.emit("offer", { room: roomId.value, offer });
});

socket.on("offer", async (offer) => {
  await peerConnection.setRemoteDescription(new RTCSessionDescription(offer));
  const answer = await peerConnection.createAnswer();
  await peerConnection.setLocalDescription(answer);
  socket.emit("answer", { room: roomId.value, answer });
});

socket.on("answer", async (answer) => {
  await peerConnection.setRemoteDescription(new RTCSessionDescription(answer));
});

socket.on("ice-candidate", async ({ candidate }) => {
  try {
    await peerConnection.addIceCandidate(new RTCIceCandidate(candidate));
  } catch (e) {
    console.error("Error adding ICE candidate:", e);
  }
});
</script>

<style scoped>
video {
  width: 300px;
  height: 200px;
  margin: 10px;
  background: black;
}
</style>
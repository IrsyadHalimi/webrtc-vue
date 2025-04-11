<template>
    <div>
        <video ref="localVideo" autoplay muted playsinline></video>
        <video ref="remoteVideo" autoplay playsinline></video>
        <input v-model="room" placeholder="Enter Room ID" />
        <button @click="startCall">Start</button>
        <button @click="joinCall">Join</button>
    </div>
</template>

<script setup>
import { ref } from "vue";
import { io } from "socket.io-client";

const localVideo = ref(null);
const remoteVideo = ref(null);
const room = ref("");
let localStream;
let peerConnection;
const socket = io(import.meta.env.VITE_SIGNALING_SERVER);

const config = {
    iceServers: [{ urls: "stun:stun.l.google.com:19302" }]
};

socket.on("user-joined", async () => {
    const offer = await peerConnection.createOffer();
    await peerConnection.setLocalDescription(offer);
    socket.emit("offer", { room: room.value, offer });
});

socket.on("offer", async (offer) => {
    await peerConnection.setRemoteDescription(new RTCSessionDescription(offer));
    const answer = await peerConnection.createAnswer();
    await peerConnection.setLocalDescription(answer);
    socket.emit("answer", { room: room.value, answer });
});

socket.on("answer", async (answer) => {
    await peerConnection.setRemoteDescription(new RTCSessionDescription(answer));
});

socket.on("ice-candidate", async (candidate) => {
    try {
        await peerConnection.addIceCandidate(new RTCIceCandidate(candidate));
    } catch (e) {
        console.error("Error adding ICE candidate: ", e);
    }
});

async function startCall() {
    await initStream();
    socket.emit("join", room.value);
    createPeerConnection();
}

async function joinCall() {
    await initStream();
    socket.emit("join", room.value);
    createPeerConnection();
}

async function initStream() {
    localStream = await navigator.mediaDevices.getUserMedia({ video: true, audio: true });
    localVideo.value.srcObject = localStream;
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
            socket.emit("ice-candidate", { room: room.value, candidate: event.candidate });
        }
    };
}
</script>

<style scoped>
video {
    width: 300px;
    height: 200px;
    margin: 10px;
    background: black;
}
</style>
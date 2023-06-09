<template>
    <el-form :model="form" label-width="120px">
        <el-form-item label="你的名字：">
            <el-input v-model="form.name" />
        </el-form-item>
        <el-form-item>
            <el-button type="primary" @click="register" :disabled=!registerButtonEnable>注册</el-button>
        </el-form-item>
        <el-form-item label="呼叫对方名字：">
            <el-input v-model="form.peerName" id="peerName" />
        </el-form-item>
        <el-form-item>
            <el-button type="success" :disabled=!startButtonEnable @click="call">开始</el-button>
            <el-button type="danger" :disabled=!stopButtonEnable @click="stop">结束</el-button>
        </el-form-item>
    </el-form>

    <el-row :gutter="20">
        <el-col :span="12">
            <video id="videoInput" autoplay width="640" height="480" poster="@/assets/webrtc.png"></video>
        </el-col>
        <el-col :span="12">
            <video id="videoOutput" autoplay width="640" height="480" poster="@/assets/webrtc.png"></video>
        </el-col>
    </el-row>
</template>
<script setup lang="ts">
import { ref, computed, reactive } from 'vue';
import { ElMessageBox } from 'element-plus';
import WebRtcPeer from 'kurento-utils';
const form = reactive({
    name: '',
    peerName: '',
});
let videoInput: HTMLVideoElement;
let videoOutput: HTMLVideoElement;
let webRtcPeer: any;
let from: string;
//===============================页面加载或关闭自动执行逻辑================================

window.onload = function () {
    console.log("page loaded");
    videoInput = document.getElementById("videoInput") as HTMLVideoElement;
    videoOutput = document.getElementById("videoOutput") as HTMLVideoElement;
    curentPlayState.value = PlayState.PLAY_STATE_IDLE;
}
window.onbeforeunload = function () {
    ws.close();
}
//===============================播放状态控制================================
enum PlayState {
    PLAY_STATE_IDLE,
    PLAY_STATE_STARTING,
    PLAY_STATE_STARTED,
}
const curentPlayState = ref(PlayState.PLAY_STATE_IDLE);

let startButtonEnable = computed(() => {
    return curentPlayState.value == PlayState.PLAY_STATE_IDLE;
})
let stopButtonEnable = computed(() => {
    return curentPlayState.value == PlayState.PLAY_STATE_STARTED
        || curentPlayState.value == PlayState.PLAY_STATE_STARTING;
})
function call() {
    if (form.peerName == '') {
        ElMessageBox.alert('请输入对方的名字', '注意', {
            confirmButtonText: 'OK'
        })
        return;
    }
    curentPlayState.value = PlayState.PLAY_STATE_STARTING;
    showSpinner();

    var options = {
        localVideo: videoInput,
        remoteVideo: videoOutput,
        onicecandidate: onIceCandidate,
        onerror: () => curentPlayState.value = PlayState.PLAY_STATE_IDLE,
    }
    webRtcPeer = WebRtcPeer.WebRtcPeer.WebRtcPeerSendrecv(options, handleRtcResult);
}

function handleRtcResult(error: string | undefined) {
    if (error) {
        console.error("open webrtc handle error : ", error);
        return console.error(error);
    }
    webRtcPeer.generateOffer(onOfferCall);
}

function onIceCandidate(candidate: any) {
    console.log("Local candidate" + JSON.stringify(candidate));

    var message = {
        id: 'onIceCandidate',
        candidate: candidate
    };
    sendMessage(message);
}

function onOfferCall(error: string | undefined, sdp: string) {
    if (error) {
        return console.error('Error generating the offer');
    }
    console.log('Invoking SDP offer callback function : ', sdp);
    var message = {
        id: 'call',
        from: form.name,
        to: form.peerName,
        sdpOffer: sdp
    };
    sendMessage(message);
}

function stop(send: boolean | undefined) {
    curentPlayState.value = PlayState.PLAY_STATE_IDLE;
    if (webRtcPeer) {
        webRtcPeer.dispose();
        webRtcPeer = null;

        if (!send) {
            sendMessage({ id: 'stop' });
        }
    }
    hideSpinner();
}
function showSpinner() {
    videoInput.poster = "@/assets/transparent-1px.png";
    videoInput.style.background = 'center transparent url("@/assets/spinner.gif") no-repeat';
    videoOutput.poster = "@/assets/transparent-1px.png";
    videoOutput.style.background = 'center transparent url("@/assets/spinner.gif") no-repeat';
}

function hideSpinner() {
    videoInput.poster = "@/assets/webrtc.png";
    videoInput.style.background = '';
    videoInput.src = '';
    videoOutput.poster = "@/assets/webrtc.png";
    videoOutput.style.background = '';
    videoOutput.src = '';
}
//===============================会话注册控制================================
enum RegisterState {
    NOT_REGISTERED,
    REGISTERING,
    REGISTERED,
}
const registerState = ref(RegisterState.NOT_REGISTERED);

let registerButtonEnable = computed(() => {
    return registerState.value == RegisterState.NOT_REGISTERED;
})

function register() {
    if (form.name == '') {
        ElMessageBox.alert('请输入你的名字', '注意', {
            confirmButtonText: 'OK'
        })
        return;
    }
    registerState.value = RegisterState.REGISTERING;

    sendMessage({ id: "register", name: form.name });

    document.getElementById("peerName")?.focus();
}
//===============================与服务端通信（websocket)================================
const ws = new WebSocket("wss://192.168.1.10:8011/ws");
ws.onmessage = function (message) {
    console.log("Front page received message : " + message.data)
    let data = JSON.parse(message.data);
    switch (data.id) {
        case "registerResponse": {
            registerResponse(data);
            break;
        }
        case "callResponse": {
            callResponse(data);
            break;
        }
        case "incomingCall": {
            incomingCall(data);
            break;
        }
        case "startCommunication": {
            startCommunication(data);
            break;
        }
        case "stopCommunication": {
            console.info('Communication ended by remote peer');
            stop(true);
            break;
        }
        case "iceCandidate": {
            webRtcPeer.addIceCandidate(data.candidate, function (error: string) {
                if (error)
                    return console.error('Error adding candidate: ' + error);
            });
            break;
        }
        default:
            console.error("Front page unrecognized message", data);
    }

}
ws.onopen = function () {
    console.log("websocket open complete");
}
function registerResponse(data: any) {
    if (data.response == "accepted") {
        registerState.value = RegisterState.REGISTERED;
    } else {
        registerState.value = RegisterState.NOT_REGISTERED;
        var errorMessage = data.message ? data.message
            : 'Unknown reason for register rejection.';
        console.log(errorMessage);
        alert('Error registering user. See console for further information.');
    }
}
function callResponse(data: any) {
    if (data.response != 'accepted') {
        console.info('Call not accepted by peer. Closing call');
        var errorMessage = data.message ? data.message
            : 'Unknown reason for call rejection.';
        console.log(errorMessage);
        stop(false);
    } else {
        curentPlayState.value = PlayState.PLAY_STATE_STARTED;

        webRtcPeer.processAnswer(data.sdpAnswer, function (error: string) {
            if (error)
                return console.error(error);
        });
    }
}
function incomingCall(data: any) {
    if (curentPlayState.value != PlayState.PLAY_STATE_IDLE) {
        let response = {
            id: "incomingCallResponse",
            from: data.from,
            callResponse: "reject",
            message: "bussy"
        };
        sendMessage(response);
        return;
    }
    curentPlayState.value = PlayState.PLAY_STATE_STARTING;
    if (confirm("User " + data.from + " is calling you. Do you accept the call?")) {
        showSpinner();
        from = data.from;
        var options = {
            localVideo: videoInput,
            remoteVideo: videoOutput,
            onicecandidate: onIceCandidate,
            onerror: onError
        }
        webRtcPeer = WebRtcPeer.WebRtcPeer.WebRtcPeerSendrecv(options,
            function (error) {
                if (error) {
                    return console.error(error);
                }
                webRtcPeer.generateOffer(onOfferIncomingCall);
            });
    } else {
        var response = {
            id: 'incomingCallResponse',
            from: data.from,
            callResponse: 'reject',
            message: 'user declined'
        };
        sendMessage(response);
        stop(false);
    }
}
function onOfferIncomingCall(error: string, offerSdp: string) {
    if (error)
        return console.error("Error generating the offer");
    var response = {
        id: 'incomingCallResponse',
        from: from,
        callResponse: 'accept',
        sdpOffer: offerSdp
    };
    sendMessage(response);
}
function onError() {
    curentPlayState.value = PlayState.PLAY_STATE_IDLE;
}
function sendMessage(message: any) {
    let jsonMessage = JSON.stringify(message);
    console.log("Front page send message : " + jsonMessage);
    ws.send(jsonMessage);
}

function startCommunication(data: any) {
    curentPlayState.value = PlayState.PLAY_STATE_STARTED;
    webRtcPeer.processAnswer(data.sdpAnswer, function (error: string) {
        if (error)
            return console.error(error);
    });
}

//===============================播放器================================
let videoFrame;
let palyState = null;




</script>
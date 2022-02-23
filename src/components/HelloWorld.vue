<script setup lang="ts">
import {  onMounted, reactive, ref } from "vue";
import {
	createLocalTracks,
	connect,
    Room,
    createLocalVideoTrack,
    LocalTrack,
    CreateLocalTrackOptions,
    LocalAudioTrack,
    LocalVideoTrack,
} from "twilio-video";

const localVideoRef = ref<HTMLDivElement>();
const localTrackState = reactive({
    audio: true,
    video: true,
    isTalking: false,
});
const localVideoTrackOpts: CreateLocalTrackOptions = {
    width: 480,
};

const publishLocalVideoTrack: () => Promise<LocalVideoTrack> = async (
    opts: CreateLocalTrackOptions = localVideoTrackOpts
) => await createLocalVideoTrack(opts);

const localTracks: LocalTrack[] = await createLocalTracks({
    audio: true,
    video: localVideoTrackOpts,
});
const room: Room = await connect(
    "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCIsImN0eSI6InR3aWxpby1mcGE7dj0xIn0.eyJqdGkiOiJTSzkzOGUwYjhlM2ZlNTllMWZlZjA3Y2U0NTE2NjE0NDcxLTE2NDU1MTMzMDUiLCJncmFudHMiOnsiaWRlbnRpdHkiOiJub3N0cmEtdHdpbGlvIiwidmlkZW8iOnt9fSwiaWF0IjoxNjQ1NTEzMzA1LCJleHAiOjE2NDU1MTY5MDUsImlzcyI6IlNLOTM4ZTBiOGUzZmU1OWUxZmVmMDdjZTQ1MTY2MTQ0NzEiLCJzdWIiOiJBQzg2ZTg4YWE2YWFlMGQ4NzVkZDVjYTAxZmIzZjYxMTQ4In0.SgW3RtsVCF9bGJu83WvbR9h9mIydPEi6_rYhTDtzWSk",
    {
        name: "test-room",
        tracks: localTracks,
    }
);

const remoteVideoRef = ref<HTMLDivElement>();
room.on("participantConnected", participant => {
    console.log(`participant joined : ${participant}`);
    participant.tracks.forEach(publication => {
        if (publication.track?.kind === "video") {
            if (remoteVideoRef.value)
                remoteVideoRef.value.appendChild(publication.track.attach());
        }
    });
});

const onToggleMute = () => {
    if (!room) return;

    localTrackState.audio = !localTrackState.audio;

    room.localParticipant.audioTracks.forEach(t => {
        if (localTrackState.audio) t.track.enable();
        else t.track.disable();
    });
};

const onToggleCam = async () => {
    if (!room) return;

    localTrackState.video = !localTrackState.video;

    if (!localTrackState.video) {
        room.localParticipant.videoTracks.forEach(async t => {
            t.track.stop();
            t.unpublish();
        });
        if (localVideoRef.value) {
            const existingChild = localVideoRef.value.querySelector("video");
            if (existingChild) localVideoRef.value.removeChild(existingChild);
        }
    } else {
        const newVidTrack = await publishLocalVideoTrack();
        room.localParticipant.publishTrack(newVidTrack);
        localVideoRef.value?.appendChild(newVidTrack.attach());
    }
};

const AudioContext = window.AudioContext;
const audioContext = AudioContext ? new AudioContext() : null;

function rootMeanSquare(samples: Uint8Array) {
    const sumSq = samples.reduce((sumSq, sample) => sumSq + sample * sample, 0);
    return Math.sqrt(sumSq / samples.length);
}

async function pollAudioLevel(
    track: LocalAudioTrack,
    onLevelChanged: (level: number) => void
) {
    if (!audioContext) {
        return;
    }

    // Due to browsers' autoplay policy, the AudioContext is only active after
    // the user has interacted with your app, after which the Promise returned
    // here is resolved.
    await audioContext.resume();

    // Create an analyser to access the raw audio samples from the microphone.
    const analyser = audioContext.createAnalyser();
    analyser.fftSize = 1024;
    analyser.smoothingTimeConstant = 0.5;

    // Connect the LocalAudioTrack's media source to the analyser.
    const stream = new MediaStream([track.mediaStreamTrack]);
    const source = audioContext.createMediaStreamSource(stream);
    source.connect(analyser);

    const samples = new Uint8Array(analyser.frequencyBinCount);
    let level = 0;

    // Periodically calculate the audio level from the captured samples,
    // and if changed, call the callback with the new audio level.
    requestAnimationFrame(function checkLevel() {
        analyser.getByteFrequencyData(samples);
        const rms = rootMeanSquare(samples);
        const log2Rms = rms && Math.log2(rms);

        // Audio level ranges from 0 (silence) to 10 (loudest).
        const newLevel = Math.ceil((10 * log2Rms) / 8);
        if (level !== newLevel) {
            level = newLevel;
            onLevelChanged(level);
        }

        // Continue calculating the level only if the audio track is live.
        if (track.mediaStreamTrack.readyState === "live") {
            requestAnimationFrame(checkLevel);
        } else {
            requestAnimationFrame(() => onLevelChanged(0));
        }
    });
}
pollAudioLevel(
    localTracks.find(t => t.kind === "audio") as LocalAudioTrack,
    level => {
        localTrackState.isTalking =
            localTrackState.audio && level >= 5 ? true : false;
    }
);
onMounted(() => {
    if (localVideoRef.value) {
        localTracks.forEach(t => {
            if (t.kind === "video") {
                localVideoRef.value?.appendChild(t.attach());
            }
        });
    }
});
</script>

<template>
    <div>
        <div>
            <div class="local" ref="localVideoRef"></div>
            <div>Is muted : {{ !localTrackState.audio }}</div>
            <div>Is cam off : {{ !localTrackState.video }}</div>
            <p>
                {{
                    localTrackState.isTalking
                        ? "Me is talking..."
                        : "not talking"
                }}
            </p>
            <button @click="onToggleMute">toggle mic</button>
            <button @click="onToggleCam">toggle cam</button>
        </div>
        <div class="remote" ref="remoteVideoRef"></div>
    </div>
</template>

<style scoped></style>

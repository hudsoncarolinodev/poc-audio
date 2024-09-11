<template>
  <div>
    <button @click="startRecording">Iniciar Gravação</button>
    <button @click="stopRecording" :disabled="!isRecording">Parar Gravação</button>
    <a v-if="mp3Url" :href="mp3Url" download="audio.mp3">Baixar MP3</a>
    <audio :src="mp3Url" controls></audio>
    <audio :src="webmUrl" controls></audio>
  </div>
</template>

<script>
import * as Lame from '@breezystack/lamejs';
export default {
  data() {
    return {
      mediaRecorder: null,
      audioChunks: [],
      isRecording: false,
      mp3Url: null,
      mp3Encoder: null,
      webmUrl: null,
      sampleRate: 44100,
      channels: 1,
      kbps: 128,
    };
  },
  methods: {
    async startRecording() {
      try {
        const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
        this.mediaRecorder = new MediaRecorder(stream);

        this.mediaRecorder.ondataavailable = (event) => {
          this.audioChunks.push(event.data);
        };

        this.mediaRecorder.onstop = async () => {
          const audioBlob = new Blob(this.audioChunks, { type: 'audio/webm' });
          this.webmUrl  = URL.createObjectURL(audioBlob);
          await this.convertToMP3(audioBlob);
        };

        this.mediaRecorder.start();
        this.isRecording = true;
      } catch (err) {
        console.error('Error accessing media devices.', err);
      }
    },
    stopRecording() {
      if (this.mediaRecorder) {
        this.mediaRecorder.stop();
        this.isRecording = false;
      }
    },
    async convertToMP3(audioBlob) {
      const arrayBuffer = await audioBlob.arrayBuffer();
      const audioContext = new (window.AudioContext || window.webkitAudioContext)();
      const audioBuffer = await audioContext.decodeAudioData(arrayBuffer);
      const mp3Encoder = new Lame.Mp3Encoder(this.channels, this.sampleRate, this.kbps);
      const mp3Data = [];

      const samples = audioBuffer.getChannelData(0);
      const sampleBlockSize = 1152; // Multiple of 576

      for (let i = 0; i < samples.length; i += sampleBlockSize) {
        const sampleChunk = samples.subarray(i, i + sampleBlockSize);
        // Convert float32 to int16
        const int16Samples = new Int16Array(sampleChunk.length);
        for (let j = 0; j < sampleChunk.length; j++) {
          int16Samples[j] = Math.max(-32768, Math.min(32767, sampleChunk[j] * 32767));
        }
        const mp3buf = mp3Encoder.encodeBuffer(int16Samples);
        if (mp3buf.length > 0) {
          mp3Data.push(mp3buf);
        }
      }
      const mp3buf = mp3Encoder.flush(); // Finish writing MP3
      if (mp3buf.length > 0) {
        mp3Data.push(new Uint8Array(mp3buf));
      }

      const mp3Blob = new Blob(mp3Data, { type: 'audio/mp3' });
      this.mp3Url = URL.createObjectURL(mp3Blob);
    },
  },
};
</script>

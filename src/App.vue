<script setup>
import { computed, ref } from 'vue'

const stream = ref(null)
const microphoneStream = ref(null)
const recorder = ref(null)
const mixedStream = ref(null)
const chunks = ref([])
const currentMimeType = ref('')

const includeSystemAudio = ref(true)
const includeMicrophone = ref(true)
const isRecording = ref(false)
const recordingReady = ref(false)
const recordedVideoUrl = ref('')
const currentTime = ref(0)
const duration = ref(0)

const downloadFilename = computed(() => {
  return currentMimeType.value === 'video/mp4' ? 'recording.mp4' : 'recording.webm'
})

const videoFeedback = ref(null)
const recordedVideo = ref(null)

function getMimeType() {
  if (MediaRecorder.isTypeSupported('video/mp4')) {
    return 'video/mp4'
  }
  if (MediaRecorder.isTypeSupported('video/webm;codecs=vp9')) {
    return 'video/webm;codecs=vp9'
  }
  if (MediaRecorder.isTypeSupported('video/webm;codecs=vp8')) {
    return 'video/webm;codecs=vp8'
  }
  if (MediaRecorder.isTypeSupported('video/webm')) {
    return 'video/webm'
  }
  return ''
}

async function setupStream() {
  try {
    const displayConstraints = {
      video: true,
      audio: includeSystemAudio.value ? { echoCancellation: false } : false,
    }

    stream.value = await navigator.mediaDevices.getDisplayMedia(displayConstraints)

    if (includeMicrophone.value) {
      microphoneStream.value = await navigator.mediaDevices.getUserMedia({
        audio: {
          echoCancellation: true,
          noiseSuppression: true,
          sampleRate: 44100,
        },
      })
    } else {
      microphoneStream.value = null
    }

    if (videoFeedback.value && stream.value) {
      videoFeedback.value.srcObject = stream.value
      await videoFeedback.value.play().catch(() => {
        /* autoplay may be blocked */
      })
    }
  } catch (error) {
    console.error('Error accessing media devices.', error)
  }
}

async function startRecording() {
  await setupStream()

  if (!stream.value) {
    console.error('No stream available')
    return
  }

  const displayTracks = stream.value.getTracks()
  const audioTracks = []

  if (includeSystemAudio.value) {
    audioTracks.push(...stream.value.getAudioTracks())
  }

  if (includeMicrophone.value && microphoneStream.value) {
    audioTracks.push(...microphoneStream.value.getAudioTracks())
  }

  mixedStream.value = new MediaStream([...displayTracks, ...audioTracks])

  const mimeType = getMimeType() || 'video/webm'
  currentMimeType.value = mimeType
  recorder.value = new MediaRecorder(mixedStream.value, { mimeType })
  recorder.value.ondataavailable = handleDataAvailable
  recorder.value.onstop = handleStop

  chunks.value = []
  recorder.value.start(200)
  isRecording.value = true
  recordingReady.value = false
  console.log('Recording started')
}

function handleDataAvailable(event) {
  if (event.data && event.data.size > 0) {
    chunks.value.push(event.data)
  }
}

function stopRecording() {
  if (!recorder.value) {
    return
  }

  recorder.value.stop()
  isRecording.value = false
  console.log('Recording stopped')
}

function formatTime(seconds) {
  if (!Number.isFinite(seconds) || seconds <= 0) {
    return '00:00'
  }
  const minutes = Math.floor(seconds / 60)
  const secs = Math.floor(seconds % 60)
  return `${String(minutes).padStart(2, '0')}:${String(secs).padStart(2, '0')}`
}

function onLoadedMetadata() {
  if (!recordedVideo.value) return
  duration.value = recordedVideo.value.duration || 0
  currentTime.value = recordedVideo.value.currentTime || 0
}

function onTimeUpdate() {
  if (!recordedVideo.value) return
  currentTime.value = recordedVideo.value.currentTime
}

function seekTo(event) {
  if (!recordedVideo.value) return
  const newTime = parseFloat(event.target.value)
  recordedVideo.value.currentTime = newTime
  currentTime.value = newTime
}

function handleStop() {
  const mimeType = currentMimeType.value || getMimeType() || 'video/webm'
  const blob = new Blob(chunks.value, { type: mimeType })
  recordedVideoUrl.value = URL.createObjectURL(blob)
  recordingReady.value = true

  if (recordedVideo.value) {
    recordedVideo.value.load()
    recordedVideo.value.onloadeddata = () => {
      recordedVideo.value.play().catch(() => {
        /* ignore play errors */
      })
    }
  }

  if (stream.value) {
    stream.value.getTracks().forEach((track) => track.stop())
  }

  if (microphoneStream.value) {
    microphoneStream.value.getTracks().forEach((track) => track.stop())
  }
}

function seekVideo(seconds) {
  if (!recordedVideo.value || !recordingReady.value) return

  const newTime = recordedVideo.value.currentTime + seconds
  recordedVideo.value.currentTime = Math.min(
    Math.max(newTime, 0),
    recordedVideo.value.duration || newTime,
  )
}
</script>

<template>
  <div class="bg-gray-800 text-white min-h-screen">
    <header class="bg-gradient-to-r from-fuchsia-600 via-pink-600 to-orange-500">
      <div class="container mx-auto flex justify-center items-center py-4">
        <h1 class="text-2xl font-bold uppercase text-white">Screen Recorder</h1>
      </div>
    </header>

    <main class="overflow-hidden">
      <div class="container mx-auto flex flex-col justify-center items-center py-8 px-4">
        <h2 class="text-xl text-gray-500 uppercase font-light mb-4">Video Recorder</h2>

        <div class="w-full mb-6">
          <label class="flex items-center gap-2 mb-3">
            <input type="checkbox" v-model="includeSystemAudio" />
            Record system audio from screen share
          </label>
          <label class="flex items-center gap-2">
            <input type="checkbox" v-model="includeMicrophone" />
            Record microphone audio
          </label>
        </div>

        <video
          ref="videoFeedback"
          class="video-feedback bg-black w-full h-100 mb-2"
          autoplay
          playsinline
          muted
        ></video>

        <div class="flex flex-col md:flex-row justify-center items-center w-full gap-4 mb-6">
          <button
            type="button"
            class="mx-4 flex-1 bg-gradient-to-br from-fuchsia-500 via-pink-500 to-violet-600 text-white rounded-2xl p-4 uppercase text-lg font-bold transition duration-300 hover:scale-[1.01] hover:opacity-95 disabled:opacity-50 disabled:cursor-not-allowed shadow-lg shadow-fuchsia-500/20"
            :disabled="isRecording"
            @click="startRecording"
          >
            Start Recording
          </button>

          <button
            type="button"
            class="mx-4 flex-1 bg-gradient-to-br from-fuchsia-500 via-pink-500 to-violet-600 text-white rounded-2xl p-4 uppercase text-lg font-bold transition duration-300 hover:scale-[1.01] hover:opacity-95 disabled:opacity-50 disabled:cursor-not-allowed shadow-lg shadow-fuchsia-500/20"
            :disabled="!isRecording"
            @click="stopRecording"
          >
            Stop Recording
          </button>
        </div>

        <section v-if="recordingReady" class="recorded-video-wrap w-full">
          <h2 class="text-xl text-gray-500 uppercase font-light mb-4">Download Your Video</h2>

          <video
            ref="recordedVideo"
            class="recorded-video bg-black w-full h-auto mb-4"
            controls
            playsinline
            preload="metadata"
            :src="recordedVideoUrl"
            @loadedmetadata="onLoadedMetadata"
            @timeupdate="onTimeUpdate"
          ></video>

          <div class="w-full mb-4">
            <input
              type="range"
              class="w-full"
              min="0"
              :max="duration"
              step="0.1"
              :value="currentTime"
              @input="seekTo"
              :disabled="!recordingReady || duration === 0"
            />
            <div class="flex justify-between text-sm text-gray-300 mt-1">
              <span>{{ formatTime(currentTime) }}</span>
              <span>{{ formatTime(duration) }}</span>
            </div>
          </div>

          <div class="flex flex-col md:flex-row justify-center items-center gap-4 mb-4">
            <button
              type="button"
              class="mx-4 flex-1 bg-gradient-to-br from-fuchsia-500 via-pink-500 to-violet-600 text-white rounded-2xl p-4 uppercase text-lg font-bold transition duration-300 hover:scale-[1.01] hover:opacity-95 disabled:opacity-50 disabled:cursor-not-allowed shadow-lg shadow-fuchsia-500/20"
              :disabled="!recordingReady"
              @click="seekVideo(-10)"
            >
              Rewind 10s
            </button>
            <button
              type="button"
              class="mx-4 flex-1 bg-gradient-to-br from-fuchsia-500 via-pink-500 to-violet-600 text-white rounded-2xl p-4 uppercase text-lg font-bold transition duration-300 hover:scale-[1.01] hover:opacity-95 disabled:opacity-50 disabled:cursor-not-allowed shadow-lg shadow-fuchsia-500/20"
              :disabled="!recordingReady"
              @click="seekVideo(10)"
            >
              Forward 10s
            </button>
          </div>

          <div class="flex justify-center items-center -mx-4">
            <a
              class="mx-4 flex-1 bg-gradient-to-br from-fuchsia-500 via-pink-500 to-violet-600 text-white rounded-2xl p-4 uppercase text-lg font-bold text-center transition duration-300 hover:scale-[1.01] hover:opacity-95 shadow-lg shadow-fuchsia-500/20"
              :href="recordedVideoUrl"
              :download="downloadFilename"
            >
              Download Video
            </a>
          </div>
        </section>
      </div>
    </main>
  </div>
</template>

<style scoped>
.video-feedback {
  min-height: 260px;
}
.recorded-video {
  min-height: 260px;
}
</style>

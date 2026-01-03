<template>
  <div class="mp3-splitter">
    <h1>MP3 Splitter</h1>
    <p class="subtitle">Upload an MP3 file and split it into multiple segments</p>

    <!-- File Upload Section -->
    <div class="upload-section">
      <div
        class="drop-zone"
        :class="{ 'drag-over': isDragOver, 'has-file': uploadedFile }"
        @dragover.prevent="isDragOver = true"
        @dragleave.prevent="isDragOver = false"
        @drop.prevent="handleDrop"
        @click="triggerFileInput"
      >
        <input
          ref="fileInput"
          type="file"
          accept=".mp3,audio/mpeg"
          @change="handleFileSelect"
          hidden
        />
        <div v-if="!uploadedFile" class="drop-content">
          <svg class="upload-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor">
            <path
              stroke-linecap="round"
              stroke-linejoin="round"
              stroke-width="2"
              d="M7 16a4 4 0 01-.88-7.903A5 5 0 1115.9 6L16 6a5 5 0 011 9.9M15 13l-3-3m0 0l-3 3m3-3v12"
            />
          </svg>
          <p>Drag & drop MP3 file here</p>
          <p class="or-text">or click to browse</p>
        </div>
        <div v-else class="file-info">
          <svg class="music-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor">
            <path
              stroke-linecap="round"
              stroke-linejoin="round"
              stroke-width="2"
              d="M9 19V6l12-3v13M9 19c0 1.105-1.343 2-3 2s-3-.895-3-2 1.343-2 3-2 3 .895 3 2zm12-3c0 1.105-1.343 2-3 2s-3-.895-3-2 1.343-2 3-2 3 .895 3 2zM9 10l12-3"
            />
          </svg>
          <p class="file-name">{{ uploadedFile.name }}</p>
          <p class="file-size">{{ formatFileSize(uploadedFile.size) }}</p>
          <button class="remove-file-btn" @click.stop="removeFile">Remove</button>
        </div>
      </div>
      <p v-if="errorMessage" class="error-message">{{ errorMessage }}</p>
    </div>

    <!-- Split Points Section (MP3 Player Style) -->
    <div v-if="uploadedFile" class="split-points-section">
      <h3>Split Points</h3>
      
      <!-- Hidden Audio Element -->
      <audio
        ref="audioPlayer"
        :src="audioUrl"
        @loadedmetadata="onAudioLoaded"
        @timeupdate="onTimeUpdate"
        @ended="onAudioEnded"
      />
      
      <!-- Audio Player Controls -->
      <div class="audio-player-controls">
        <button
          class="play-pause-btn"
          @click="togglePlayPause"
          :disabled="isProcessing"
          @keydown.space.prevent="togglePlayPause"
        >
          <svg v-if="!isPlaying" class="play-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M14.752 11.168l-3.197-2.132A1 1 0 0010 9.87v4.263a1 1 0 001.555.832l3.197-2.132a1 1 0 000-1.664z" />
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M21 12a9 9 0 11-18 0 9 9 0 0118 0z" />
          </svg>
          <svg v-else class="pause-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M10 9v6m4-6v6m7-3a9 9 0 11-18 0 9 9 0 0118 0z" />
          </svg>
        </button>
        
        <div class="time-display">
          <span class="current-time">{{ formatTime(currentTime) }}</span>
          <span class="separator">/</span>
          <span class="total-time">{{ formatTime(audioDuration) }}</span>
        </div>
        
        <button
          class="add-split-btn"
          @click="addSplitPointAtCurrent"
          :disabled="isProcessing"
          title="Add split point at current position (Press Enter)"
          @keydown.enter.prevent="addSplitPointAtCurrent"
        >
          <svg class="plus-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 4v16m8-8H4" />
          </svg>
          Add Split Point
        </button>
      </div>
      
      <!-- Progress Bar with Split Markers -->
      <div class="progress-container">
        <div
          class="progress-bar-track"
          ref="progressBar"
          @click="seekToPosition"
        >
          <div
            class="progress-fill"
            :style="{ width: (currentTime / audioDuration * 100) + '%' }"
          ></div>
          
          <!-- Split Markers -->
          <div
            v-for="(point, index) in sortedSplitPoints"
            :key="index"
            class="split-marker"
            :style="{ left: (point / audioDuration * 100) + '%' }"
            @mousedown.stop="startDragMarker(index, $event)"
            @touchstart.stop="startDragMarker(index, $event)"
          >
            <div class="marker-dot"></div>
            <div class="marker-label">{{ formatTime(point) }}</div>
            <button class="remove-marker-btn" @click.stop="removeSplitPoint(index)">
              <svg viewBox="0 0 24 24" fill="none" stroke="currentColor">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12" />
              </svg>
            </button>
          </div>
        </div>
      </div>
      
      <!-- Keyboard Shortcuts Hint -->
      <div class="keyboard-hint">
        <span>␣ Space: Play/Pause</span>
        <span>↵ Enter: Add Split Point</span>
      </div>
      
      <!-- Segments Preview -->
      <div class="segments-preview">
        <div
          v-for="(segment, index) in segmentsPreview"
          :key="index"
          class="segment-preview-item"
          :class="{ playing: isPlayingSegment === index }"
        >
          <div class="segment-info">
            <span class="segment-name">Part {{ index + 1 }}: {{ formatTime(segment.start) }} - {{ formatTime(segment.end) }}</span>
            <span class="segment-duration">{{ formatDuration(segment.end - segment.start) }}</span>
          </div>
          <button class="play-preview-btn" @click="playSegment(index)" :disabled="isProcessing">
            <svg v-if="isPlayingSegment !== index" class="play-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor">
              <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M14.752 11.168l-3.197-2.132A1 1 0 0010 9.87v4.263a1 1 0 001.555.832l3.197-2.132a1 1 0 000-1.664z" />
              <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M21 12a9 9 0 11-18 0 9 9 0 0118 0z" />
            </svg>
            <svg v-else class="pause-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor">
              <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M10 9v6m4-6v6m7-3a9 9 0 11-18 0 9 9 0 0118 0z" />
            </svg>
          </button>
        </div>
      </div>
    </div>

    <!-- Split Button -->
    <button
      class="split-btn"
      :disabled="!uploadedFile || isProcessing || !audioDuration || splitPoints.length === 0"
      @click="splitMP3"
    >
      <span v-if="!isProcessing">Split MP3 into {{ splitPoints.length + 1 }} Parts</span>
      <span v-else>Processing...</span>
    </button>

    <!-- Progress Section -->
    <div v-if="statusMessage || isProcessing" class="progress-section">
      <div class="status-message">{{ statusMessage }}</div>
      <div v-if="isProcessing" class="progress-bar">
        <div class="progress-fill" :style="{ width: progress + '%' }"></div>
      </div>
    </div>

    <!-- Segments List -->
    <div v-if="segments.length > 0" class="segments-section">
      <h2>Generated Segments ({{ segments.length }})</h2>
      <div class="segments-list">
        <div v-for="segment in segments" :key="segment.id" class="segment-item">
          <div class="segment-info">
            <span class="segment-name">{{ segment.name }}</span>
            <span class="segment-size">{{ formatFileSize(segment.size) }}</span>
          </div>
          <div class="segment-actions">
            <button class="play-segment-btn" @click="playResultSegment(segment)">
              <svg class="play-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M14.752 11.168l-3.197-2.132A1 1 0 0010 9.87v4.263a1 1 0 001.555.832l3.197-2.132a1 1 0 000-1.664z" />
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M21 12a9 9 0 11-18 0 9 9 0 0118 0z" />
              </svg>
              Play
            </button>
            <button class="download-btn" @click="downloadSegment(segment)">
              <svg class="download-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor">
                <path
                  stroke-linecap="round"
                  stroke-linejoin="round"
                  stroke-width="2"
                  d="M4 16v1a3 3 0 003 3h10a3 3 0 003-3v-1m-4-4l-4 4m0 0l-4-4m4 4V4"
                />
              </svg>
              Download
            </button>
          </div>
        </div>
      </div>
      <button class="download-all-btn" @click="downloadAllSegments">
        Download All Segments
      </button>
    </div>
  </div>
</template>

<script lang="ts">
import { FFmpeg } from '@ffmpeg/ffmpeg'
import type { LogEvent } from '@ffmpeg/ffmpeg/dist/esm/types'
import { fetchFile, toBlobURL } from '@ffmpeg/util'
import { defineComponent, ref, computed } from 'vue'

interface Segment {
  id: number
  name: string
  blob: Blob
  url: string
  size: number
  start?: number
  end?: number
}

interface SegmentPreview {
  start: number
  end: number
}

const baseURL = 'https://cdn.jsdelivr.net/npm/@ffmpeg/core@0.12.10/dist/esm'

export default defineComponent({
  name: 'MP3Splitter',
  setup() {
    const ffmpeg = new FFmpeg()
    const fileInput = ref<HTMLInputElement | null>(null)
    const uploadedFile = ref<File | null>(null)
    const isDragOver = ref(false)
    const isProcessing = ref(false)
    const ffmpegLoaded = ref(false)
    const segments = ref<Segment[]>([])
    const statusMessage = ref('')
    const errorMessage = ref('')
    const progress = ref(0)
    const audioPlayer = ref<HTMLAudioElement | null>(null)
    const audioUrl = ref('')
    const audioDuration = ref(0)
    const splitPoints = ref<number[]>([])
    const currentTime = ref(0)
    const isPlaying = ref(false)
    const isPlayingSegment = ref<number | null>(null)
    const draggingMarkerIndex = ref<number | null>(null)
    const progressBar = ref<HTMLElement | null>(null)

    const sortedSplitPoints = computed(() => {
      return [...splitPoints.value].sort((a, b) => a - b)
    })

    const segmentsPreview = computed(() => {
      const points = sortedSplitPoints.value
      const segments: SegmentPreview[] = []
      
      let start = 0
      for (const point of points) {
        segments.push({ start, end: point })
        start = point
      }
      segments.push({ start, end: audioDuration.value })
      
      return segments
    })

    // File upload handlers
    function triggerFileInput() {
      fileInput.value?.click()
    }

    function handleFileSelect(event: Event) {
      const target = event.target as HTMLInputElement
      const file = target.files?.[0]
      if (file) {
        validateAndSetFile(file)
      }
    }

    function handleDrop(event: DragEvent) {
      isDragOver.value = false
      const file = event.dataTransfer?.files[0]
      if (file) {
        validateAndSetFile(file)
      }
    }

    function validateAndSetFile(file: File) {
      errorMessage.value = ''
      
      if (!file.type.includes('audio/mpeg') && !file.name.toLowerCase().endsWith('.mp3')) {
        errorMessage.value = 'Please upload an MP3 file'
        return
      }

      if (file.size > 50 * 1024 * 1024) {
        errorMessage.value = 'Warning: Large files may cause browser memory issues'
      }

      uploadedFile.value = file
      segments.value = []
      splitPoints.value = []
      
      if (audioUrl.value) {
        URL.revokeObjectURL(audioUrl.value)
      }
      audioUrl.value = URL.createObjectURL(file)
      audioDuration.value = 0
    }

    function removeFile() {
      uploadedFile.value = null
      segments.value = []
      splitPoints.value = []
      errorMessage.value = ''
      if (audioPlayer.value) {
        audioPlayer.value.pause()
      }
      isPlayingSegment.value = null
    }

    // Format functions
    function formatFileSize(bytes: number): string {
      if (bytes === 0) return '0 Bytes'
      const k = 1024
      const sizes = ['Bytes', 'KB', 'MB', 'GB']
      const i = Math.floor(Math.log(bytes) / Math.log(k))
      return Math.round(bytes / Math.pow(k, i) * 100) / 100 + ' ' + sizes[i]
    }

    function formatTime(seconds: number): string {
      if (!seconds || isNaN(seconds)) return '0:00'
      const mins = Math.floor(seconds / 60)
      const secs = Math.floor(seconds % 60)
      return `${mins}:${secs.toString().padStart(2, '0')}`
    }

    function formatDuration(seconds: number): string {
      return formatTime(seconds)
    }

    // Audio player handlers
    function onAudioLoaded() {
      if (audioPlayer.value) {
        audioDuration.value = audioPlayer.value.duration
      }
    }

    function onTimeUpdate() {
      if (audioPlayer.value) {
        currentTime.value = audioPlayer.value.currentTime
        
        // Handle segment preview playback
        if (isPlayingSegment.value !== null) {
          const segment = segmentsPreview.value[isPlayingSegment.value]
          if (segment && currentTime.value >= segment.end) {
            audioPlayer.value.pause()
            isPlayingSegment.value = null
          }
        }
      }
    }

    function onAudioEnded() {
      isPlayingSegment.value = null
    }

    function togglePlayPause() {
      if (!audioPlayer.value) return

      if (isPlaying.value) {
        audioPlayer.value.pause()
        isPlaying.value = false
        isPlayingSegment.value = null
      } else {
        audioPlayer.value.play()
        isPlaying.value = true
        isPlayingSegment.value = null
      }
    }

    function playSegment(index: number) {
      if (!audioPlayer.value) return

      if (isPlayingSegment.value === index) {
        audioPlayer.value.pause()
        isPlayingSegment.value = null
        isPlaying.value = false
        return
      }

      if (isPlayingSegment.value !== null || isPlaying.value) {
        audioPlayer.value.pause()
      }

      const segment = segmentsPreview.value[index]
      if (segment) {
        audioPlayer.value.currentTime = segment.start
        audioPlayer.value.play()
        isPlaying.value = true
        isPlayingSegment.value = index
      }
    }

    function playResultSegment(segment: Segment) {
      if (segment.url) {
        const audio = new Audio(segment.url)
        audio.play()
      }
    }

    // Split point management
    function addSplitPointAtCurrent() {
      if (!audioDuration.value || !audioPlayer.value) return
      
      const newPoint = currentTime.value
      if (newPoint > 0 && newPoint < audioDuration.value) {
        // Check if point already exists
        if (!splitPoints.value.some(p => Math.abs(p - newPoint) < 0.5)) {
          splitPoints.value.push(newPoint)
        }
      }
    }

    function seekToPosition(event: MouseEvent) {
      if (!audioPlayer.value || !progressBar.value) return
      
      const rect = progressBar.value.getBoundingClientRect()
      const percentage = Math.max(0, Math.min(1, (event.clientX - rect.left) / rect.width))
      const newTime = percentage * audioDuration.value
      audioPlayer.value.currentTime = newTime
      currentTime.value = newTime
    }

    function removeSplitPoint(index: number) {
      const points = sortedSplitPoints.value
      const pointToRemove = points[index]
      splitPoints.value = splitPoints.value.filter(p => p !== pointToRemove)
    }


    function startDragMarker(index: number, event: MouseEvent | TouchEvent) {
      event.preventDefault()
      draggingMarkerIndex.value = index
      
      const moveHandler = (e: MouseEvent | TouchEvent) => {
        const clientX = 'touches' in e ? e.touches[0].clientX : e.clientX
        const slider = document.querySelector('.main-slider') as HTMLInputElement
        if (!slider) return
        
        const rect = slider.getBoundingClientRect()
        const percentage = Math.max(0, Math.min(1, (clientX - rect.left) / rect.width))
        const newTime = percentage * audioDuration.value
        
        // Update the split point
        const points = [...splitPoints.value]
        points[index] = newTime
        splitPoints.value = points
      }
      
      const upHandler = () => {
        draggingMarkerIndex.value = null
        document.removeEventListener('mousemove', moveHandler)
        document.removeEventListener('mouseup', upHandler)
        document.removeEventListener('touchmove', moveHandler)
        document.removeEventListener('touchend', upHandler)
      }
      
      document.addEventListener('mousemove', moveHandler)
      document.addEventListener('mouseup', upHandler)
      document.addEventListener('touchmove', moveHandler)
      document.addEventListener('touchend', upHandler)
    }

    // Load FFmpeg
    async function loadFFmpeg() {
      if (ffmpegLoaded.value) return

      statusMessage.value = 'Loading FFmpeg core...'
      progress.value = 10

      ffmpeg.on('log', ({ message: msg }: LogEvent) => {
        console.log('FFmpeg:', msg)
      })

      try {
        await ffmpeg.load({
          workerURL: await toBlobURL(`${baseURL}/ffmpeg-core.worker.js`, 'text/javascript'),
          coreURL: await toBlobURL(`${baseURL}/ffmpeg-core.js`, 'text/javascript'),
          wasmURL: await toBlobURL(`${baseURL}/ffmpeg-core.wasm`, 'application/wasm')
        })
        ffmpegLoaded.value = true
        progress.value = 20
        statusMessage.value = 'FFmpeg loaded successfully'
      } catch (error) {
        console.error('Failed to load FFmpeg:', error)
        errorMessage.value = 'Failed to load FFmpeg. Please try again.'
        throw error
      }
    }

    // Split MP3
    async function splitMP3() {
      if (!uploadedFile.value || splitPoints.value.length === 0) return

      try {
        isProcessing.value = true
        errorMessage.value = ''
        segments.value = []
        progress.value = 0

        await loadFFmpeg()

        statusMessage.value = 'Writing file to virtual filesystem...'
        progress.value = 20
        const inputFileName = 'input.mp3'
        await ffmpeg.writeFile(inputFileName, await fetchFile(uploadedFile.value))

        const points = sortedSplitPoints.value
        const newSegments: Segment[] = []

        for (let i = 0; i <= points.length; i++) {
          const start = i === 0 ? 0 : points[i - 1]
          const end = i === points.length ? audioDuration.value : points[i]
          const duration = end - start

          statusMessage.value = `Creating Part ${i + 1} (${formatTime(start)} - ${formatTime(end)})...`
          progress.value = 20 + (i / (points.length + 1)) * 60

          const outputFileName = `part${i + 1}.mp3`
          
          await ffmpeg.exec([
            '-i',
            inputFileName,
            '-ss',
            start.toString(),
            '-t',
            duration.toString(),
            '-c',
            'copy',
            outputFileName
          ])

          const data = await ffmpeg.readFile(outputFileName)
          const uint8Array = data as Uint8Array
          const blob = new Blob([new Uint8Array(uint8Array)], { type: 'audio/mpeg' })
          const url = URL.createObjectURL(blob)

          newSegments.push({
            id: i + 1,
            name: outputFileName,
            blob,
            url,
            size: blob.size,
            start,
            end
          })

          await ffmpeg.deleteFile(outputFileName)
        }

        segments.value = newSegments

        try {
          await ffmpeg.deleteFile(inputFileName)
        } catch (e) {
          console.warn('Could not delete input file:', e)
        }

        statusMessage.value = `Successfully split MP3 into ${newSegments.length} parts`
        progress.value = 100

      } catch (error) {
        console.error('Error splitting MP3:', error)
        errorMessage.value = 'Failed to split MP3. Please try again.'
        statusMessage.value = ''
      } finally {
        isProcessing.value = false
      }
    }

    // Download functions
    function downloadSegment(segment: Segment) {
      const link = document.createElement('a')
      link.href = segment.url
      link.download = segment.name
      document.body.appendChild(link)
      link.click()
      document.body.removeChild(link)
    }

    function downloadAllSegments() {
      segments.value.forEach((segment, index) => {
        setTimeout(() => {
          downloadSegment(segment)
        }, index * 200)
      })
    }

    return {
      fileInput,
      uploadedFile,
      isDragOver,
      isProcessing,
      segments,
      statusMessage,
      errorMessage,
      progress,
      audioPlayer,
      audioUrl,
      audioDuration,
      splitPoints,
      sortedSplitPoints,
      currentTime,
      isPlaying,
      isPlayingSegment,
      segmentsPreview,
      progressBar,
      triggerFileInput,
      handleFileSelect,
      handleDrop,
      removeFile,
      formatFileSize,
      formatTime,
      formatDuration,
      onAudioLoaded,
      onTimeUpdate,
      onAudioEnded,
      togglePlayPause,
      playSegment,
      playResultSegment,
      addSplitPointAtCurrent,
      removeSplitPoint,
      seekToPosition,
      startDragMarker,
      splitMP3,
      downloadSegment,
      downloadAllSegments
    }
  }
})
</script>

<style scoped>
.mp3-splitter {
  max-width: 900px;
  margin: 0 auto;
  padding: 2rem;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, sans-serif;
}

h1 {
  text-align: center;
  color: #2c3e50;
  margin-bottom: 0.5rem;
}

.subtitle {
  text-align: center;
  color: #666;
  margin-bottom: 2rem;
}

/* Upload Section */
.upload-section {
  margin-bottom: 2rem;
}

.drop-zone {
  border: 3px dashed #cbd5e0;
  border-radius: 12px;
  padding: 3rem 2rem;
  text-align: center;
  cursor: pointer;
  transition: all 0.3s ease;
  background: #f8fafc;
}

.drop-zone:hover {
  border-color: #3b82f6;
  background: #eff6ff;
}

.drop-zone.drag-over {
  border-color: #3b82f6;
  background: #dbeafe;
  transform: scale(1.02);
}

.drop-zone.has-file {
  border-style: solid;
  border-color: #10b981;
  background: #ecfdf5;
}

.drop-content {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 1rem;
}

.upload-icon {
  width: 64px;
  height: 64px;
  color: #94a3b8;
}

.or-text {
  color: #94a3b8;
  font-size: 0.875rem;
}

.file-info {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0.5rem;
}

.music-icon {
  width: 48px;
  height: 48px;
  color: #10b981;
}

.file-name {
  font-weight: 600;
  color: #065f46;
  word-break: break-all;
}

.file-size {
  color: #047857;
  font-size: 0.875rem;
}

.remove-file-btn {
  margin-top: 0.5rem;
  padding: 0.5rem 1rem;
  background: #ef4444;
  color: white;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-weight: 500;
  transition: background 0.2s;
}

.remove-file-btn:hover {
  background: #dc2626;
}

.error-message {
  margin-top: 0.75rem;
  padding: 0.75rem 1rem;
  background: #fee2e2;
  color: #991b1b;
  border-radius: 6px;
  font-size: 0.875rem;
}

/* Split Points Section */
.split-points-section {
  display: flex;
  flex-direction: column;
  gap: 1.5rem;
  padding: 1.5rem;
  background: #f8fafc;
  border-radius: 12px;
  border: 1px solid #e5e7eb;
}

.split-points-section h3 {
  margin: 0 0 1rem 0;
  color: #374151;
  font-size: 1.125rem;
}

/* Audio Player Controls */
.audio-player-controls {
  display: flex;
  align-items: center;
  gap: 1.5rem;
  margin-bottom: 1.5rem;
}

.play-pause-btn {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 56px;
  height: 56px;
  padding: 0;
  background: #3b82f6;
  color: white;
  border: none;
  border-radius: 50%;
  cursor: pointer;
  transition: all 0.2s;
  flex-shrink: 0;
}

.play-pause-btn:hover:not(:disabled) {
  background: #2563eb;
  transform: scale(1.05);
}

.play-pause-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.play-icon, .pause-icon {
  width: 28px;
  height: 28px;
}

.time-display {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  font-size: 1.125rem;
  font-weight: 600;
  color: #374151;
  font-variant-numeric: tabular-nums;
  flex: 1;
}

.separator {
  color: #9ca3af;
}

.add-split-btn {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  padding: 0.75rem 1.25rem;
  background: #10b981;
  color: white;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  font-weight: 600;
  font-size: 0.9rem;
  transition: all 0.2s;
  flex-shrink: 0;
}

.add-split-btn:hover:not(:disabled) {
  background: #059669;
  transform: translateY(-1px);
}

.add-split-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.plus-icon {
  width: 20px;
  height: 20px;
}

/* Progress Bar */
.progress-container {
  position: relative;
}

.progress-bar-track {
  position: relative;
  width: 100%;
  height: 12px;
  background: #e5e7eb;
  border-radius: 6px;
  cursor: pointer;
  transition: background 0.2s;
}

.progress-bar-track:hover {
  background: #d1d5db;
}

.progress-fill {
  height: 100%;
  background: linear-gradient(90deg, #3b82f6, #8b5cf6);
  border-radius: 6px;
  transition: width 0.1s linear;
}

/* Split Markers on Progress Bar */
.split-markers {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  height: 100%;
  pointer-events: none;
}

.split-marker {
  position: absolute;
  top: 50%;
  transform: translate(-50%, -50%);
  pointer-events: auto;
  cursor: grab;
  z-index: 10;
}

.split-marker:active {
  cursor: grabbing;
}

.marker-dot {
  width: 20px;
  height: 20px;
  background: #ef4444;
  border: 3px solid white;
  border-radius: 50%;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.3);
}

.marker-label {
  position: absolute;
  top: 28px;
  left: 50%;
  transform: translateX(-50%);
  background: #1e293b;
  color: white;
  padding: 0.25rem 0.75rem;
  border-radius: 4px;
  font-size: 0.75rem;
  white-space: nowrap;
  font-weight: 500;
}

.remove-marker-btn {
  position: absolute;
  top: -8px;
  right: -8px;
  width: 22px;
  height: 22px;
  padding: 0;
  background: #ef4444;
  color: white;
  border: 2px solid white;
  border-radius: 50%;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.2s;
}

.remove-marker-btn:hover {
  background: #dc2626;
  transform: scale(1.15);
}

.remove-marker-btn svg {
  width: 14px;
  height: 14px;
}

/* Keyboard Shortcuts Hint */
.keyboard-hint {
  display: flex;
  gap: 1.5rem;
  padding: 0.75rem;
  background: #f1f5f9;
  border-radius: 6px;
  font-size: 0.8rem;
  color: #64748b;
  margin-top: 0.5rem;
}

.keyboard-hint span {
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

/* Segments Preview */
.segments-preview {
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
  padding: 1rem;
  background: #e0f2fe;
  border-radius: 8px;
  border-left: 4px solid #3b82f6;
}

.segment-preview-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0.75rem;
  background: white;
  border: 1px solid #bfdbfe;
  border-radius: 6px;
  transition: all 0.2s;
}

.segment-preview-item:hover {
  border-color: #3b82f6;
}

.segment-preview-item.playing {
  background: #dbeafe;
  border-color: #3b82f6;
}

.segment-info {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
}

.segment-name {
  font-weight: 500;
  color: #1e293b;
  font-size: 0.9rem;
}

.segment-duration {
  font-size: 0.8rem;
  color: #64748b;
}

.play-preview-btn {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 36px;
  height: 36px;
  padding: 0;
  background: #3b82f6;
  color: white;
  border: none;
  border-radius: 50%;
  cursor: pointer;
  transition: all 0.2s;
  flex-shrink: 0;
}

.play-preview-btn:hover:not(:disabled) {
  background: #2563eb;
  transform: scale(1.1);
}

.play-preview-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.play-icon, .pause-icon {
  width: 18px;
  height: 18px;
}

/* Split Button */
.split-btn {
  width: 100%;
  padding: 1rem;
  background: #3b82f6;
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 1.125rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s;
}

.split-btn:hover:not(:disabled) {
  background: #2563eb;
}

.split-btn:disabled {
  background: #9ca3af;
  cursor: not-allowed;
}

/* Progress Section */
.progress-section {
  margin: 2rem 0;
  padding: 1rem;
  background: #f8fafc;
  border-radius: 8px;
}

.status-message {
  text-align: center;
  color: #4b5563;
  margin-bottom: 0.5rem;
  font-weight: 500;
}

.progress-bar {
  width: 100%;
  height: 8px;
  background: #e5e7eb;
  border-radius: 4px;
  overflow: hidden;
}

.progress-fill {
  height: 100%;
  background: linear-gradient(90deg, #3b82f6, #8b5cf6);
  transition: width 0.3s ease;
}

/* Segments Section */
.segments-section {
  margin-top: 2rem;
}

.segments-section h2 {
  color: #2c3e50;
  margin-bottom: 1rem;
}

.segments-list {
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
  margin-bottom: 1.5rem;
}

.segment-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 1rem;
  background: #f8fafc;
  border: 1px solid #e5e7eb;
  border-radius: 8px;
  transition: all 0.2s;
}

.segment-item:hover {
  background: #f1f5f9;
  border-color: #cbd5e0;
}

.segment-info {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
}

.segment-name {
  font-weight: 600;
  color: #1e293b;
}

.segment-size {
  font-size: 0.875rem;
  color: #64748b;
}

.segment-actions {
  display: flex;
  gap: 0.5rem;
}

.play-segment-btn {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  padding: 0.5rem 1rem;
  background: #10b981;
  color: white;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-weight: 500;
  transition: background 0.2s;
}

.play-segment-btn:hover {
  background: #059669;
}

.download-btn {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  padding: 0.5rem 1rem;
  background: #3b82f6;
  color: white;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-weight: 500;
  transition: background 0.2s;
}

.download-btn:hover {
  background: #2563eb;
}

.download-icon, .play-icon {
  width: 18px;
  height: 18px;
}

.download-all-btn {
  width: 100%;
  padding: 1rem;
  background: #8b5cf6;
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 1rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s;
}

.download-all-btn:hover {
  background: #7c3aed;
}
</style>

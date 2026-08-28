<template>
    <div class="asrvosk-plugin">
        <div v-if="hasError" class="error-banner">
            {{ errorMessage }}
        </div>
        <div v-if="!hasError" class="mic clickable" :class="[status, continuous ? 'continuous' : 'non-continuous']" @click="$_handleMicClick">
            <img :src="micIcon" alt="">
        </div>
    </div>
</template>

<script>
import BasePluginComponent from '/js/BasePluginComponent.js';

export default {
    name: "asrjs",
    mixins: [BasePluginComponent],
    data() {
        return {
            status: 'loading',
            audio: {},
            continuous: false,
            keyboardShortcut: null,
            vad: null, // Store VAD instance
            vadInitialized: false,
            accumulatedAudioBuffer: null, // Float32Array for audio accumulation on semantic VAD "nok"
            pendingTranscription: false, // Flag to prevent duplicate transcriptions
            audioChunks: [], // Store audio chunks for transcription
            speakerIdAvailable: false, // Cache speakerid availability
            voiceProfilesEnabled: false, // Privacy gate: only send mic audio when the user opted in
            audioContext: null,
            processor: null,
            source: null,
            recordingBuffer: [], // Buffer for native frequency audio
            isRecording: false,
            chunkInterval: null,
            nativeSampleRate: null, // Store the actual native sample rate (typically 48kHz)
            speakerIdBuffer: [], // Buffer for downsampled audio for speakerid
            lastChunkSentTime: 0, // Track when we last sent a chunk to speakerid
            chunkDuration: 3.0, // Fixed chunk duration in seconds for speakerid
            wakewordEnabled: false, // Wakeword detection enabled from settings
            wakewordProcessing: false, // Flag to prevent overlapping wakeword requests
            wakewordDetected: false, // Flag to track if wakeword was detected
            wakewordChannelOpened: false // One-shot gate: once the channel is opened (wake/click) it stays open until the conversation ends
        };
    },
    computed: {
        hasError() {
            return Boolean(this.error);
        },
        errorMessage() {
            if (!this.error) {
                return '';
            }
            return this.error.message || this.t('Microphone access problem. Verify that Windows has access to your microphone, then restart IGOOR.');
        },
        micIcon() {
            // Slashed mic = ASR is unavailable right now (still loading, or muted during TTS)
            if (this.status === 'loading' || this.status === 'paused') {
                return '/img/icons/src/microphone-slash.svg';
            }
            // Plain mic = ASR available (listening / recording / wakeword-armed)
            return '/img/icons/src/microphone.svg';
        }
    },
    created() {
        this.audio = {
            on: new Audio('/plugins/asrvosk/samples/on.wav'),
            off: new Audio('/plugins/asrvosk/samples/off.wav')
        };
        // Wakeword detection sound
        this.wakewordSound = new Audio('/plugins/asrjs/samples/on.wav');
        this.wakewordSound.load();
        Object.values(this.audio).forEach(audio => audio.load());
    },
    async mounted() {
        // Load settings directly via REST API
        try {
            const settings = await this.callPluginRestEndpoint('asrjs', 'settings');
            console.log('ASRJS settings received:', settings);
            this.settings = settings;
            this.continuous = settings.continuous || false;
            this.wakewordEnabled = settings.wakeword_enabled || false;
            if (settings.shortcut) {
                console.log('ASRJS SHORTCUT:', settings.shortcut);
                this.keyboardShortcut = settings.shortcut;
            }
        } catch (error) {
            console.error('Error loading settings via REST:', error);
        }

        window.addEventListener('keydown', this.$_handleKeyPress);

        // If continuous mode, load VAD library for automatic speech detection
        if (this.continuous) {
            await this.$_loadVADLibrary();
        } else {
            // Non-continuous: set status to listening immediately
            this.status = 'listening';
        }

        // Check speakerid availability during initialization
        await this.$_checkSpeakerIdAvailability();

        // Periodically refresh the voice-profiles gate so a toggle in settings takes
        // effect without an app reload (the backend enforces it regardless).
        this.voiceProfilesRefreshInterval = setInterval(() => this.$_refreshVoiceProfilesEnabled(), 20000);

        // Initialize microphone access
        await this.$_initializeMicrophone();
    },
    beforeDestroy() {
        window.removeEventListener('keydown', this.$_handleKeyPress);

        if (this.voiceProfilesRefreshInterval) {
            clearInterval(this.voiceProfilesRefreshInterval);
            this.voiceProfilesRefreshInterval = null;
        }

        // Cleanup VAD
        if (this.vad) {
            this.vad.destroy();
        }

        // Cleanup Web Audio API components
        if (this.processor) {
            this.processor.disconnect();
        }
        if (this.source) {
            this.source.disconnect();
        }
        if (this.audioContext && this.audioContext.state !== 'closed') {
            this.audioContext.close();
        }
        if (this.mediaStream) {
            this.mediaStream.getTracks().forEach(track => track.stop());
        }
        if (this.chunkInterval) {
            clearInterval(this.chunkInterval);
        }
    },
    methods: {
        $_loadVADLibrary() {
            return new Promise(async (resolve, reject) => {
                // Check if already loaded (check both window and self for PyWebView compatibility)
                if (window.vad || self.vad) {
                    if (!window.vad && self.vad) window.vad = self.vad;
                    console.log('VAD library already loaded, skipping');
                    this.$_initializeVAD().then(resolve);
                    return;
                }

                try {
                    // Step 1: Load ONNX Runtime (ort.js) — required dependency for VAD bundle
                    await this.$_loadScript('/plugins/asrjs/static/vad/ort.js', 'ort');
                    console.log('ONNX Runtime loaded, window.ort available:', !!window.ort);

                    // CRITICAL: The VAD bundle UMD reads self.ort at parse-time (before onload).
                    // In PyWebView/WebView2, self !== window, so we must sync BEFORE loading the bundle.
                    if (window.ort && !self.ort) {
                        console.log('Syncing window.ort → self.ort (required before VAD bundle loads)');
                        self.ort = window.ort;
                    }

                    // Step 2: Load VAD bundle (depends on self.ort at parse-time)
                    await this.$_loadScript('/plugins/asrjs/static/vad/bundle.min.js', 'vad');
                    console.log('VAD bundle loaded, window.vad available:', !!window.vad);

                    if (!window.vad) {
                        throw new Error('VAD bundle loaded but window.vad is not defined');
                    }

                    await this.$_initializeVAD();
                    resolve();
                } catch (error) {
                    console.error('Failed to load VAD library:', error);
                    this.status = 'error';
                    reject(error);
                }
            });
        },

        $_loadScript(src, globalName) {
            return new Promise((resolve, reject) => {
                // If the global is already available, skip loading
                if (window[globalName] || self[globalName]) {
                    this.$_syncGlobal(globalName);
                    resolve();
                    return;
                }

                // Remove any stale script tag (from a previous failed load)
                const existingScript = document.querySelector(`script[src="${src}"]`);
                if (existingScript) {
                    console.log(`Removing stale script tag for ${src}`);
                    existingScript.remove();
                }

                const script = document.createElement('script');
                script.src = src;
                script.async = true;

                script.onload = () => {
                    console.log(`Script loaded: ${src}`);
                    // Sync between self and window (PyWebView/WebView2: self !== window)
                    this.$_syncGlobal(globalName);
                    resolve();
                };

                script.onerror = () => {
                    console.error(`Failed to load script: ${src}`);
                    reject(new Error(`Failed to load script: ${src}`));
                };

                document.head.appendChild(script);
            });
        },

        // PyWebView/WebView2: self !== window. Some scripts set window[name] (var-based),
        // some UMD bundles use self[name]. Sync both directions so all code can find them.
        $_syncGlobal(name) {
            if (window[name] && !self[name]) {
                console.log(`Syncing window.${name} → self.${name}`);
                self[name] = window[name];
            } else if (self[name] && !window[name]) {
                console.log(`Syncing self.${name} → window.${name}`);
                window[name] = self[name];
            }
        },

        async $_initializeMicrophoneMinimal() {
            try {
                console.log('Attempting microphone initialization with minimal constraints...');

                // Try with very minimal constraints - let browser choose everything
                const stream = await navigator.mediaDevices.getUserMedia({
                    audio: true  // Just ask for audio, no specific constraints
                });

                // Verify actual sample rate from stream
                const audioTrack = stream.getAudioTracks()[0];
                const settings = audioTrack.getSettings();
                console.log('Microphone initialized with minimal constraints at:', settings.sampleRate);

                this.mediaStream = stream;
                this.nativeSampleRate = settings.sampleRate || 48000;

                // Initialize Web Audio API
                this.audioContext = new (window.AudioContext || window.webkitAudioContext)({
                    sampleRate: this.nativeSampleRate
                });

                this.source = this.audioContext.createMediaStreamSource(stream);

                // Load and start AudioWorklet processor
                await this.audioContext.audioWorklet.addModule('/plugins/asrjs/frontend/audio-processor.js');
                this.processor = new AudioWorkletNode(this.audioContext, 'audio-processor');

                // Handle messages from AudioWorklet
                this.processor.port.onmessage = (event) => {
                    if (event.data.type === 'speakerid-chunk') {
                        // Send chunk to speakerid with chunk_id
                        this.$_sendFixedChunkToSpeakerID(event.data.data, event.data.chunk_id);
                    } else if (event.data.type === 'audio-data') {
                        // Store audio data for final WAV file
                        this.recordingBuffer = this.recordingBuffer.concat(event.data.data);
                    } else if (event.data.type === 'wakeword-chunk') {
                        // Send chunk to wakeword detection
                        this.$_sendWakewordChunk(event.data.data);
                    }
                };

                // Connect the audio nodes
                this.source.connect(this.processor);
                this.processor.connect(this.audioContext.destination);

                // Enable wakeword detection if enabled in settings
                if (this.wakewordEnabled && this.continuous && !this.wakewordChannelOpened) {
                    this.processor.port.postMessage({ type: 'enable-wakeword' });
                    console.log('Wakeword detection enabled in AudioWorklet');
                }

                // Re-apply the voice-profiles gate now that the worklet exists. The initial
                // $_checkSpeakerIdAvailability() runs before this.processor is created, so its
                // enable-speakerid message was dropped — without this the worklet wouldn't
                // capture for speaker ID until the 20s status refresh re-applied the gate.
                this.$_applyVoiceProfilesEnabled(this.voiceProfilesEnabled);

                // Set up audio level monitoring
                const analyser = this.audioContext.createAnalyser();
                analyser.fftSize = 256;
                this.source.connect(analyser);

                const dataArray = new Uint8Array(analyser.frequencyBinCount);

                // Monitor audio levels
                let levelCount = 0;
                const monitorAudio = () => {
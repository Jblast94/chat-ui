# Voice Agent Implementation Guide

## 1. Overview

This guide provides step-by-step instructions for implementing the Chatterbox TTS voice agent integration into the existing SvelteKit chat UI. The implementation will enable real-time voice conversations with natural human-like dialogue patterns.

## 2. Prerequisites

### 2.1 Required Dependencies

Add the following dependencies to your `package.json`:

```json
{
  "dependencies": {
    "@types/web": "^0.0.99",
    "uuid": "^9.0.0"
  },
  "devDependencies": {
    "@types/uuid": "^9.0.0"
  }
}
```

### 2.2 Environment Setup

Add the following environment variables to your `.env` file:

```bash
# Chatterbox TTS Configuration
CHATTERBOX_RUNPOD_ENDPOINT_ID=your_endpoint_id_here
CHATTERBOX_RUNPOD_API_KEY=your_api_key_here
CHATTERBOX_BASE_URL=https://api.runpod.ai/v2

# Voice Feature Configuration
VOICE_FEATURES_ENABLED=true
VOICE_DEFAULT_EXAGGERATION=0.5
VOICE_DEFAULT_CFG_WEIGHT=0.5
VOICE_DEFAULT_SPEED=1.0

# Audio Cache Configuration
AUDIO_CACHE_MAX_SIZE_MB=50
AUDIO_CACHE_TTL_HOURS=168
AUDIO_MAX_DURATION_SECONDS=300
```

## 3. Core Implementation Steps

### 3.1 Step 1: Create Voice Types and Interfaces

Create `src/lib/types/Voice.ts`:

```typescript
import type { Message } from './Message';

export interface VoiceSettings {
  exaggeration: number; // 0.0 - 1.0
  cfgWeight: number; // 0.0 - 1.0
  speed: number; // 0.5 - 2.0
  volume: number; // 0.0 - 1.0
  voiceProfile?: string;
}

export interface VoiceConversationState {
  isListening: boolean;
  isSpeaking: boolean;
  isProcessing: boolean;
  currentAudioUrl?: string;
  audioQueue: AudioQueueItem[];
  voiceSettings: VoiceSettings;
  errorState?: VoiceError;
  isVoiceEnabled: boolean;
}

export interface AudioQueueItem {
  id: string;
  text: string;
  audioUrl: string;
  duration: number;
  messageId: string;
  timestamp: number;
}

export interface VoiceError {
  type: 'network' | 'audio' | 'speech_recognition' | 'tts_service' | 'browser_support';
  message: string;
  recoverable: boolean;
  fallbackAction?: string;
  timestamp: number;
}

export interface TTSRequest {
  text: string;
  voiceSettings: VoiceSettings;
  messageId: string;
}

export interface TTSResponse {
  status: 'COMPLETED' | 'FAILED' | 'IN_PROGRESS';
  output?: {
    audio_url: string;
    duration: number;
  };
  error?: string;
  executionTime?: number;
}

export interface VoiceMessage extends Message {
  voice?: {
    audioUrl?: string;
    duration?: number;
    voiceSettings?: VoiceSettings;
    synthesisTime?: number;
    isPlaying?: boolean;
    isCached?: boolean;
  };
}
```

### 3.2 Step 2: Create TTS Service Client

Create `src/lib/services/ChatterboxTTSService.ts`:

```typescript
import type { TTSRequest, TTSResponse, VoiceSettings } from '$lib/types/Voice';
import { browser } from '$app/environment';
import { PUBLIC_CHATTERBOX_RUNPOD_ENDPOINT_ID } from '$env/static/public';

export class ChatterboxTTSService {
  private baseUrl: string;
  private endpointId: string;
  private apiKey: string;
  private requestTimeout: number = 30000;
  private maxRetries: number = 3;
  private retryDelay: number = 1000;

  constructor() {
    this.baseUrl = 'https://api.runpod.ai/v2';
    this.endpointId = PUBLIC_CHATTERBOX_RUNPOD_ENDPOINT_ID;
    this.apiKey = this.getApiKey();
  }

  private getApiKey(): string {
    // In production, this should be handled server-side
    // For now, we'll use a public environment variable
    return import.meta.env.VITE_CHATTERBOX_RUNPOD_API_KEY || '';
  }

  async synthesizeText(request: TTSRequest): Promise<TTSResponse> {
    if (!browser) {
      throw new Error('TTS service can only be used in browser environment');
    }

    const { text, voiceSettings, messageId } = request;
    
    const payload = {
      input: {
        text: this.preprocessText(text),
        exaggeration: voiceSettings.exaggeration,
        cfg_weight: voiceSettings.cfgWeight,
        speed: voiceSettings.speed,
        ...(voiceSettings.voiceProfile && { audio_prompt_path: voiceSettings.voiceProfile })
      }
    };

    let lastError: Error;
    
    for (let attempt = 0; attempt < this.maxRetries; attempt++) {
      try {
        const response = await this.makeRequest(payload);
        
        if (response.status === 'COMPLETED' && response.output?.audio_url) {
          return response;
        } else if (response.status === 'FAILED') {
          throw new Error(response.error || 'TTS synthesis failed');
        }
        
        // If in progress, wait and retry
        await this.delay(this.retryDelay * Math.pow(2, attempt));
        
      } catch (error) {
        lastError = error as Error;
        
        if (attempt < this.maxRetries - 1) {
          await this.delay(this.retryDelay * Math.pow(2, attempt));
        }
      }
    }
    
    throw lastError!;
  }

  private async makeRequest(payload: any): Promise<TTSResponse> {
    const controller = new AbortController();
    const timeoutId = setTimeout(() => controller.abort(), this.requestTimeout);

    try {
      const response = await fetch(`${this.baseUrl}/${this.endpointId}/runsync`, {
        method: 'POST',
        headers: {
          'Authorization': `Bearer ${this.apiKey}`,
          'Content-Type': 'application/json'
        },
        body: JSON.stringify(payload),
        signal: controller.signal
      });

      clearTimeout(timeoutId);

      if (!response.ok) {
        throw new Error(`HTTP ${response.status}: ${response.statusText}`);
      }

      return await response.json();
    } catch (error) {
      clearTimeout(timeoutId);
      throw error;
    }
  }

  private preprocessText(text: string): string {
    // Clean up text for better TTS synthesis
    return text
      .replace(/\*\*(.*?)\*\*/g, '$1') // Remove bold markdown
      .replace(/\*(.*?)\*/g, '$1') // Remove italic markdown
      .replace(/`(.*?)`/g, '$1') // Remove code markdown
      .replace(/\[([^\]]+)\]\([^)]+\)/g, '$1') // Remove links, keep text
      .replace(/#{1,6}\s/g, '') // Remove headers
      .replace(/\n{2,}/g, '. ') // Replace multiple newlines with periods
      .replace(/\n/g, ' ') // Replace single newlines with spaces
      .trim();
  }

  private delay(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }

  // Test connection to the TTS service
  async testConnection(): Promise<boolean> {
    try {
      const testRequest: TTSRequest = {
        text: 'Test connection',
        voiceSettings: {
          exaggeration: 0.5,
          cfgWeight: 0.5,
          speed: 1.0,
          volume: 1.0
        },
        messageId: 'test'
      };
      
      const response = await this.synthesizeText(testRequest);
      return response.status === 'COMPLETED';
    } catch (error) {
      console.error('TTS connection test failed:', error);
      return false;
    }
  }
}

export const ttsService = new ChatterboxTTSService();
```

### 3.3 Step 3: Create Voice State Management

Create `src/lib/stores/voiceStore.ts`:

```typescript
import { writable, derived, get } from 'svelte/store';
import type { 
  VoiceConversationState, 
  VoiceSettings, 
  AudioQueueItem, 
  VoiceError 
} from '$lib/types/Voice';
import { browser } from '$app/environment';

// Default voice settings
const defaultVoiceSettings: VoiceSettings = {
  exaggeration: 0.5,
  cfgWeight: 0.5,
  speed: 1.0,
  volume: 1.0,
  voiceProfile: undefined
};

// Initial state
const initialState: VoiceConversationState = {
  isListening: false,
  isSpeaking: false,
  isProcessing: false,
  currentAudioUrl: undefined,
  audioQueue: [],
  voiceSettings: defaultVoiceSettings,
  errorState: undefined,
  isVoiceEnabled: false
};

// Create the main voice store
export const voiceStore = writable<VoiceConversationState>(initialState);

// Derived stores for specific states
export const isListening = derived(voiceStore, $voice => $voice.isListening);
export const isSpeaking = derived(voiceStore, $voice => $voice.isSpeaking);
export const isProcessing = derived(voiceStore, $voice => $voice.isProcessing);
export const voiceSettings = derived(voiceStore, $voice => $voice.voiceSettings);
export const audioQueue = derived(voiceStore, $voice => $voice.audioQueue);
export const voiceError = derived(voiceStore, $voice => $voice.errorState);
export const isVoiceEnabled = derived(voiceStore, $voice => $voice.isVoiceEnabled);

// Voice store actions
export const voiceActions = {
  // Initialize voice features
  initialize(): void {
    if (!browser) return;
    
    const isSupported = this.checkBrowserSupport();
    voiceStore.update(state => ({
      ...state,
      isVoiceEnabled: isSupported
    }));
    
    if (isSupported) {
      this.loadSettings();
    }
  },

  // Check browser support for voice features
  checkBrowserSupport(): boolean {
    const hasWebAudio = 'AudioContext' in window || 'webkitAudioContext' in window;
    const hasSpeechRecognition = 'SpeechRecognition' in window || 'webkitSpeechRecognition' in window;
    const hasMediaDevices = navigator.mediaDevices && navigator.mediaDevices.getUserMedia;
    
    return hasWebAudio && hasSpeechRecognition && hasMediaDevices;
  },

  // Load voice settings from localStorage
  loadSettings(): void {
    if (!browser) return;
    
    try {
      const saved = localStorage.getItem('voice-settings');
      if (saved) {
        const settings = JSON.parse(saved);
        voiceStore.update(state => ({
          ...state,
          voiceSettings: { ...defaultVoiceSettings, ...settings }
        }));
      }
    } catch (error) {
      console.error('Failed to load voice settings:', error);
    }
  },

  // Save voice settings to localStorage
  saveSettings(settings: Partial<VoiceSettings>): void {
    if (!browser) return;
    
    voiceStore.update(state => {
      const newSettings = { ...state.voiceSettings, ...settings };
      
      try {
        localStorage.setItem('voice-settings', JSON.stringify(newSettings));
      } catch (error) {
        console.error('Failed to save voice settings:', error);
      }
      
      return {
        ...state,
        voiceSettings: newSettings
      };
    });
  },

  // Set listening state
  setListening(isListening: boolean): void {
    voiceStore.update(state => ({
      ...state,
      isListening,
      errorState: isListening ? undefined : state.errorState
    }));
  },

  // Set speaking state
  setSpeaking(isSpeaking: boolean, audioUrl?: string): void {
    voiceStore.update(state => ({
      ...state,
      isSpeaking,
      currentAudioUrl: isSpeaking ? audioUrl : undefined
    }));
  },

  // Set processing state
  setProcessing(isProcessing: boolean): void {
    voiceStore.update(state => ({
      ...state,
      isProcessing
    }));
  },

  // Add item to audio queue
  addToQueue(item: AudioQueueItem): void {
    voiceStore.update(state => ({
      ...state,
      audioQueue: [...state.audioQueue, item]
    }));
  },

  // Remove item from audio queue
  removeFromQueue(itemId: string): void {
    voiceStore.update(state => ({
      ...state,
      audioQueue: state.audioQueue.filter(item => item.id !== itemId)
    }));
  },

  // Clear audio queue
  clearQueue(): void {
    voiceStore.update(state => ({
      ...state,
      audioQueue: []
    }));
  },

  // Set error state
  setError(error: VoiceError | undefined): void {
    voiceStore.update(state => ({
      ...state,
      errorState: error
    }));
  },

  // Reset all states
  reset(): void {
    voiceStore.update(state => ({
      ...state,
      isListening: false,
      isSpeaking: false,
      isProcessing: false,
      currentAudioUrl: undefined,
      audioQueue: [],
      errorState: undefined
    }));
  },

  // Get current state
  getState(): VoiceConversationState {
    return get(voiceStore);
  }
};

// Initialize on module load
if (browser) {
  voiceActions.initialize();
}
```

### 3.4 Step 4: Create Audio Manager

Create `src/lib/services/AudioManager.ts`:

```typescript
import type { AudioQueueItem, VoiceError } from '$lib/types/Voice';
import { voiceActions } from '$lib/stores/voiceStore';

export class AudioManager {
  private audioContext: AudioContext | null = null;
  private currentAudio: HTMLAudioElement | null = null;
  private audioQueue: AudioQueueItem[] = [];
  private isPlaying: boolean = false;
  private volume: number = 1.0;

  constructor() {
    this.initializeAudioContext();
  }

  private initializeAudioContext(): void {
    try {
      this.audioContext = new (window.AudioContext || (window as any).webkitAudioContext)();
    } catch (error) {
      console.error('Failed to initialize AudioContext:', error);
      this.handleError({
        type: 'audio',
        message: 'Failed to initialize audio system',
        recoverable: false,
        timestamp: Date.now()
      });
    }
  }

  async playAudio(audioUrl: string, onComplete?: () => void): Promise<void> {
    try {
      // Resume audio context if suspended
      if (this.audioContext?.state === 'suspended') {
        await this.audioContext.resume();
      }

      // Stop current audio if playing
      if (this.currentAudio) {
        this.stopAudio();
      }

      // Create new audio element
      this.currentAudio = new Audio(audioUrl);
      this.currentAudio.volume = this.volume;
      this.currentAudio.crossOrigin = 'anonymous';

      // Set up event listeners
      this.currentAudio.addEventListener('loadstart', () => {
        voiceActions.setProcessing(true);
      });

      this.currentAudio.addEventListener('canplaythrough', () => {
        voiceActions.setProcessing(false);
      });

      this.currentAudio.addEventListener('play', () => {
        this.isPlaying = true;
        voiceActions.setSpeaking(true, audioUrl);
      });

      this.currentAudio.addEventListener('ended', () => {
        this.isPlaying = false;
        voiceActions.setSpeaking(false);
        onComplete?.();
        this.playNextInQueue();
      });

      this.currentAudio.addEventListener('error', (event) => {
        this.handleAudioError(event);
      });

      // Start playback
      await this.currentAudio.play();
      
    } catch (error) {
      console.error('Failed to play audio:', error);
      this.handleError({
        type: 'audio',
        message: 'Failed to play audio',
        recoverable: true,
        fallbackAction: 'retry',
        timestamp: Date.now()
      });
    }
  }

  stopAudio(): void {
    if (this.currentAudio) {
      this.currentAudio.pause();
      this.currentAudio.currentTime = 0;
      this.currentAudio = null;
    }
    this.isPlaying = false;
    voiceActions.setSpeaking(false);
  }

  pauseAudio(): void {
    if (this.currentAudio && this.isPlaying) {
      this.currentAudio.pause();
      this.isPlaying = false;
      voiceActions.setSpeaking(false);
    }
  }

  resumeAudio(): void {
    if (this.currentAudio && !this.isPlaying) {
      this.currentAudio.play().then(() => {
        this.isPlaying = true;
        voiceActions.setSpeaking(true, this.currentAudio!.src);
      }).catch(error => {
        this.handleError({
          type: 'audio',
          message: 'Failed to resume audio',
          recoverable: true,
          timestamp: Date.now()
        });
      });
    }
  }

  setVolume(volume: number): void {
    this.volume = Math.max(0, Math.min(1, volume));
    if (this.currentAudio) {
      this.currentAudio.volume = this.volume;
    }
  }

  addToQueue(item: AudioQueueItem): void {
    this.audioQueue.push(item);
    voiceActions.addToQueue(item);
    
    // Start playing if nothing is currently playing
    if (!this.isPlaying) {
      this.playNextInQueue();
    }
  }

  private async playNextInQueue(): Promise<void> {
    if (this.audioQueue.length === 0) {
      return;
    }

    const nextItem = this.audioQueue.shift()!;
    voiceActions.removeFromQueue(nextItem.id);
    
    await this.playAudio(nextItem.audioUrl, () => {
      // Audio completed, play next in queue
      this.playNextInQueue();
    });
  }

  clearQueue(): void {
    this.audioQueue = [];
    voiceActions.clearQueue();
  }

  getCurrentPlaybackTime(): number {
    return this.currentAudio?.currentTime || 0;
  }

  getDuration(): number {
    return this.currentAudio?.duration || 0;
  }

  isCurrentlyPlaying(): boolean {
    return this.isPlaying;
  }

  private handleAudioError(event: Event): void {
    const audio = event.target as HTMLAudioElement;
    let message = 'Unknown audio error';
    
    if (audio.error) {
      switch (audio.error.code) {
        case MediaError.MEDIA_ERR_ABORTED:
          message = 'Audio playback was aborted';
          break;
        case MediaError.MEDIA_ERR_NETWORK:
          message = 'Network error occurred during audio playback';
          break;
        case MediaError.MEDIA_ERR_DECODE:
          message = 'Audio decoding error';
          break;
        case MediaError.MEDIA_ERR_SRC_NOT_SUPPORTED:
          message = 'Audio format not supported';
          break;
      }
    }

    this.handleError({
      type: 'audio',
      message,
      recoverable: true,
      fallbackAction: 'retry',
      timestamp: Date.now()
    });
  }

  private handleError(error: VoiceError): void {
    console.error('Audio Manager Error:', error);
    voiceActions.setError(error);
    
    // Reset states on error
    this.isPlaying = false;
    voiceActions.setSpeaking(false);
    voiceActions.setProcessing(false);
  }

  // Cleanup method
  destroy(): void {
    this.stopAudio();
    this.clearQueue();
    
    if (this.audioContext) {
      this.audioContext.close();
      this.audioContext = null;
    }
  }
}

export const audioManager = new AudioManager();
```

### 3.5 Step 5: Create Speech Recognition Service

Create `src/lib/services/SpeechRecognitionService.ts`:

```typescript
import type { VoiceError } from '$lib/types/Voice';
import { voiceActions } from '$lib/stores/voiceStore';
import { browser } from '$app/environment';

export class SpeechRecognitionService {
  private recognition: SpeechRecognition | null = null;
  private isListening: boolean = false;
  private onTranscriptCallback?: (text: string) => void;
  private onErrorCallback?: (error: VoiceError) => void;
  private silenceTimer: number | null = null;
  private silenceThreshold: number = 3000; // 3 seconds

  constructor() {
    this.initializeRecognition();
  }

  private initializeRecognition(): void {
    if (!browser) return;

    const SpeechRecognition = window.SpeechRecognition || (window as any).webkitSpeechRecognition;
    
    if (!SpeechRecognition) {
      this.handleError({
        type: 'browser_support',
        message: 'Speech recognition not supported in this browser',
        recoverable: false,
        timestamp: Date.now()
      });
      return;
    }

    this.recognition = new SpeechRecognition();
    this.setupRecognitionConfig();
    this.setupEventListeners();
  }

  private setupRecognitionConfig(): void {
    if (!this.recognition) return;

    this.recognition.continuous = true;
    this.recognition.interimResults = true;
    this.recognition.lang = 'en-US';
    this.recognition.maxAlternatives = 1;
  }

  private setupEventListeners(): void {
    if (!this.recognition) return;

    this.recognition.onstart = () => {
      this.isListening = true;
      voiceActions.setListening(true);
      this.resetSilenceTimer();
    };

    this.recognition.onresult = (event: SpeechRecognitionEvent) => {
      this.handleSpeechResult(event);
      this.resetSilenceTimer();
    };

    this.recognition.onerror = (event: SpeechRecognitionErrorEvent) => {
      this.handleSpeechError(event);
    };

    this.recognition.onend = () => {
      this.isListening = false;
      voiceActions.setListening(false);
      this.clearSilenceTimer();
    };

    this.recognition.onspeechstart = () => {
      this.resetSilenceTimer();
    };

    this.recognition.onspeechend = () => {
      this.resetSilenceTimer();
    };
  }

  startListening(onTranscript: (text: string) => void, onError?: (error: VoiceError) => void): void {
    if (!this.recognition) {
      this.handleError({
        type: 'browser_support',
        message: 'Speech recognition not available',
        recoverable: false,
        timestamp: Date.now()
      });
      return;
    }

    if (this.isListening) {
      this.stopListening();
    }

    this.onTranscriptCallback = onTranscript;
    this.onErrorCallback = onError;

    try {
      this.recognition.start();
    } catch (error) {
      this.handleError({
        type: 'speech_recognition',
        message: 'Failed to start speech recognition',
        recoverable: true,
        fallbackAction: 'retry',
        timestamp: Date.now()
      });
    }
  }

  stopListening(): void {
    if (this.recognition && this.isListening) {
      this.recognition.stop();
    }
    this.clearSilenceTimer();
  }

  private handleSpeechResult(event: SpeechRecognitionEvent): void {
    let finalTranscript = '';
    let interimTranscript = '';

    for (let i = event.resultIndex; i < event.results.length; i++) {
      const result = event.results[i];
      const transcript = result[0].transcript;

      if (result.isFinal) {
        finalTranscript += transcript;
      } else {
        interimTranscript += transcript;
      }
    }

    // Send final transcript to callback
    if (finalTranscript.trim() && this.onTranscriptCallback) {
      this.onTranscriptCallback(finalTranscript.trim());
      this.stopListening(); // Stop after getting final result
    }
  }

  private handleSpeechError(event: SpeechRecognitionErrorEvent): void {
    let message = 'Speech recognition error';
    let recoverable = true;
    let fallbackAction: string | undefined;

    switch (event.error) {
      case 'no-speech':
        message = 'No speech detected';
        fallbackAction = 'retry';
        break;
      case 'audio-capture':
        message = 'Audio capture failed';
        recoverable = false;
        break;
      case 'not-allowed':
        message = 'Microphone access denied';
        recoverable = false;
        break;
      case 'network':
        message = 'Network error during speech recognition';
        fallbackAction = 'retry';
        break;
      case 'aborted':
        message = 'Speech recognition aborted';
        fallbackAction = 'retry';
        break;
      case 'bad-grammar':
        message = 'Grammar error in speech recognition';
        break;
      case 'language-not-supported':
        message = 'Language not supported';
        recoverable = false;
        break;
    }

    const error: VoiceError = {
      type: 'speech_recognition',
      message,
      recoverable,
      fallbackAction,
      timestamp: Date.now()
    };

    this.handleError(error);
  }

  private resetSilenceTimer(): void {
    this.clearSilenceTimer();
    
    this.silenceTimer = window.setTimeout(() => {
      if (this.isListening) {
        this.stopListening();
      }
    }, this.silenceThreshold);
  }

  private clearSilenceTimer(): void {
    if (this.silenceTimer) {
      clearTimeout(this.silenceTimer);
      this.silenceTimer = null;
    }
  }

  private handleError(error: VoiceError): void {
    console.error('Speech Recognition Error:', error);
    voiceActions.setError(error);
    
    if (this.onErrorCallback) {
      this.onErrorCallback(error);
    }
    
    // Stop listening on error
    if (this.isListening) {
      this.stopListening();
    }
  }

  isCurrentlyListening(): boolean {
    return this.isListening;
  }

  setSilenceThreshold(ms: number): void {
    this.silenceThreshold = Math.max(1000, ms); // Minimum 1 second
  }

  // Check if speech recognition is supported
  static isSupported(): boolean {
    if (!browser) return false;
    return 'SpeechRecognition' in window || 'webkitSpeechRecognition' in window;
  }

  // Cleanup method
  destroy(): void {
    this.stopListening();
    this.clearSilenceTimer();
    this.recognition = null;
    this.onTranscriptCallback = undefined;
    this.onErrorCallback = undefined;
  }
}

export const speechRecognitionService = new SpeechRecognitionService();
```

## 4. Component Implementation

### 4.1 Voice Input Component

Create `src/lib/components/voice/VoiceInput.svelte`:

```svelte
<script lang="ts">
  import { onMount, onDestroy } from 'svelte';
  import { speechRecognitionService } from '$lib/services/SpeechRecognitionService';
  import { voiceActions, isListening, voiceError } from '$lib/stores/voiceStore';
  import type { VoiceError } from '$lib/types/Voice';
  
  export let onTranscript: (text: string) => void;
  export let disabled: boolean = false;
  export let placeholder: string = 'Click to speak...';
  
  let isSupported = false;
  let permissionGranted = false;
  let currentTranscript = '';
  
  onMount(async () => {
    isSupported = speechRecognitionService.constructor.isSupported();
    
    if (isSupported) {
      await requestMicrophonePermission();
    }
  });
  
  onDestroy(() => {
    if ($isListening) {
      stopListening();
    }
  });
  
  async function requestMicrophonePermission(): Promise<void> {
    try {
      const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
      stream.getTracks().forEach(track => track.stop());
      permissionGranted = true;
    } catch (error) {
      console.error('Microphone permission denied:', error);
      permissionGranted = false;
      
      voiceActions.setError({
        type: 'speech_recognition',
        message: 'Microphone access is required for voice input',
        recoverable: false,
        timestamp: Date.now()
      });
    }
  }
  
  function startListening(): void {
    if (!isSupported || !permissionGranted || disabled) return;
    
    currentTranscript = '';
    
    speechRecognitionService.startListening(
      (transcript: string) => {
        currentTranscript = transcript;
        onTranscript(transcript);
      },
      (error: VoiceError) => {
        console.error('Voice input error:', error);
      }
    );
  }
  
  function stopListening(): void {
    speechRecognitionService.stopListening();
    currentTranscript = '';
  }
  
  function toggleListening(): void {
    if ($isListening) {
      stopListening();
    } else {
      startListening();
    }
  }
  
  $: buttonClass = `
    relative inline-flex items-center justify-center
    w-12 h-12 rounded-full transition-all duration-200
    ${$isListening 
      ? 'bg-red-500 hover:bg-red-600 text-white animate-pulse' 
      : 'bg-blue-500 hover:bg-blue-600 text-white'
    }
    ${disabled || !isSupported || !permissionGranted 
      ? 'opacity-50 cursor-not-allowed' 
      : 'cursor-pointer hover:scale-105'
    }
  `;
</script>

<div class="voice-input-container">
  {#if !isSupported}
    <div class="text-sm text-gray-500 mb-2">
      Voice input not supported in this browser
    </div>
  {:else if !permissionGranted}
    <div class="text-sm text-red-500 mb-2">
      Microphone access required
    </div>
  {/if}
  
  <button
    class={buttonClass}
    on:click={toggleListening}
    disabled={disabled || !isSupported || !permissionGranted}
    title={$isListening ? 'Stop listening' : 'Start voice input'}
  >
    {#if $isListening}
      <!-- Stop icon -->
      <svg class="w-6 h-6" fill="currentColor" viewBox="0 0 24 24">
        <rect x="6" y="6" width="12" height="12" rx="2"/>
      </svg>
    {:else}
      <!-- Microphone icon -->
      <svg class="w-6 h-6" fill="currentColor" viewBox="0 0 24 24">
        <path d="M12 1a3 3 0 0 0-3 3v8a3 3 0 0 0 6 0V4a3 3 0 0 0-3-3z"/>
        <path d="M19 10v2a7 7 0 0 1-14 0v-2"/>
        <line x1="12" y1="19" x2="12" y2="23"/>
        <line x1="8" y1="23" x2="16" y2="23"/>
      </svg>
    {/if}
    
    {#if $isListening}
      <div class="absolute inset-0 rounded-full border-2 border-white animate-ping"></div>
    {/if}
  </button>
  
  {#if $isListening && currentTranscript}
    <div class="mt-2 p-2 bg-gray-100 rounded text-sm text-gray-700">
      {currentTranscript}
    </div>
  {/if}
  
  {#if $voiceError && $voiceError.type === 'speech_recognition'}
    <div class="mt-2 text-sm text-red-500">
      {$voiceError.message}
    </div>
  {/if}
</div>

<style>
  .voice-input-container {
    display: flex;
    flex-direction: column;
    align-items: center;
  }
</style>
```

### 4.2 Voice Output Component

Create `src/lib/components/voice/VoiceOutput.svelte`:

```svelte
<script lang="ts">
  import { onMount, onDestroy } from 'svelte';
  import { ttsService } from '$lib/services/ChatterboxTTSService';
  import { audioManager } from '$lib/services/AudioManager';
  import { voiceActions, isSpeaking, isProcessing, voiceSettings } from '$lib/stores/voiceStore';
  import type { TTSRequest, AudioQueueItem } from '$lib/types/Voice';
  import { v4 as uuidv4 } from 'uuid';
  
  export let text: string;
  export let messageId: string;
  export let autoPlay: boolean = true;
  export let showControls: boolean = true;
  
  let audioUrl: string | null = null;
  let duration: number = 0;
  let isGenerating: boolean = false;
  let error: string | null = null;
  
  onMount(() => {
    if (autoPlay && text.trim()) {
      generateAndPlayAudio();
    }
  });
  
  onDestroy(() => {
    // Cleanup if component is destroyed
    if (audioUrl) {
      audioManager.stopAudio();
    }
  });
  
  async function generateAndPlayAudio(): Promise<void> {
    if (!text.trim() || isGenerating) return;
    
    try {
      isGenerating = true;
      error = null;
      voiceActions.setProcessing(true);
      
      const request: TTSRequest = {
        text: text.trim(),
        voiceSettings: $voiceSettings,
        messageId
      };
      
      const response = await ttsService.synthesizeText(request);
      
      if (response.status === 'COMPLETED' && response.output) {
        audioUrl = response.output.audio_url;
        duration = response.output.duration;
        
        if (autoPlay) {
          await playAudio();
        }
      } else {
        throw new Error('TTS synthesis failed');
      }
      
    } catch (err) {
      console.error('TTS Error:', err);
      error = err instanceof Error ? err.message : 'Failed to generate audio';
      
      voiceActions.setError({
        type: 'tts_service',
        message: error,
        recoverable: true,
        fallbackAction: 'retry',
        timestamp: Date.now()
      });
      
    } finally {
      isGenerating = false;
      voiceActions.setProcessing(false);
    }
  }
  
  async function playAudio(): Promise<void> {
    if (!audioUrl) {
      await generateAndPlayAudio();
      return;
    }
    
    const queueItem: AudioQueueItem = {
      id: uuidv4(),
      text,
      audioUrl,
      duration,
      messageId,
      timestamp: Date.now()
    };
    
    audioManager.addToQueue(queueItem);
  }
  
  function stopAudio(): void {
    audioManager.stopAudio();
  }
  
  function retryGeneration(): void {
    error = null;
    audioUrl = null;
    generateAndPlayAudio();
  }
  
  $: canPlay = audioUrl && !isGenerating;
  $: isCurrentlyPlaying = $isSpeaking && audioManager.isCurrentlyPlaying();
</script>

<div class="voice-output-container">
  {#if showControls}
    <div class="voice-controls flex items-center gap-2">
      {#if isGenerating || $isProcessing}
        <div class="flex items-center gap-2 text-sm text-gray-500">
          <div class="animate-spin w-4 h-4 border-2 border-blue-500 border-t-transparent rounded-full"></div>
          Generating audio...
        </div>
      {:else if error}
        <div class="flex items-center gap-2">
          <span class="text-sm text-red-500">{error}</span>
          <button
            class="px-2 py-1 text-xs bg-red-100 text-red-700 rounded hover:bg-red-200"
            on:click={retryGeneration}
          >
            Retry
          </button>
        </div>
      {:else if canPlay}
        <div class="flex items-center gap-2">
          {#if isCurrentlyPlaying}
            <button
              class="p-2 bg-red-500 text-white rounded-full hover:bg-red-600 transition-colors"
              on:click={stopAudio}
              title="Stop audio"
            >
              <!-- Stop icon -->
              <svg class="w-4 h-4" fill="currentColor" viewBox="0 0 24 24">
                <rect x="6" y="6" width="12" height="12" rx="2"/>
              </svg>
            </button>
          {:else}
            <button
              class="p-2 bg-green-500 text-white rounded-full hover:bg-green-600 transition-colors"
              on:click={playAudio}
              title="Play audio"
            >
              <!-- Play icon -->
              <svg class="w-4 h-4" fill="currentColor" viewBox="0 0 24 24">
                <polygon points="5,3 19,12 5,21"/>
              </svg>
            </button>
          {/if}
          
          {#if duration > 0}
            <span class="text-xs text-gray-500">
              {Math.round(duration)}s
            </span>
          {/if}
        </div>
      {:else}
        <button
          class="p-2 bg-blue-500 text-white rounded-full hover:bg-blue-600 transition-colors"
          on:click={generateAndPlayAudio}
          title="Generate audio"
        >
          <!-- Speaker icon -->
          <svg class="w-4 h-4" fill="currentColor" viewBox="0 0 24 24">
            <polygon points="11,5 6,9 2,9 2,15 6,15 11,19"/>
            <path d="M19.07 4.93a10 10 0 0 1 0 14.14M15.54 8.46a5 5 0 0 1 0 7.07"/>
          </svg>
        </button>
      {/if}
    </div>
  {/if}
  
  {#if isCurrentlyPlaying}
    <div class="mt-2">
      <div class="flex items-center gap-2 text-sm text-green-600">
        <div class="w-2 h-2 bg-green-500 rounded-full animate-pulse"></div>
        Playing audio...
      </div>
    </div>
  {/if}
</div>

<style>
  .voice-output-container {
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
  }
  
  .voice-controls {
    align-items: center;
  }
</style>
```

## 5. Integration with Existing Chat Interface

### 5.1 Update Message Component

Modify the existing message component to include voice output. Add this to your message component:

```svelte
<!-- Add to your existing message component -->
<script lang="ts">
  import VoiceOutput from '$lib/components/voice/VoiceOutput.svelte';
  import { isVoiceEnabled } from '$lib/stores/voiceStore';
  
  // ... existing props
  export let message: Message;
</script>

<!-- Existing message content -->
<div class="message-content">
  <!-- ... existing message rendering ... -->
  
  {#if $isVoiceEnabled && message.from === 'assistant' && message.content.trim()}
    <div class="mt-2">
      <VoiceOutput 
        text={message.content} 
        messageId={message.id}
        autoPlay={false}
        showControls={true}
      />
    </div>
  {/if}
</div>
```

### 5.2 Update Chat Input Component

Modify the chat input to include voice input:

```svelte
<!-- Add to your existing chat input component -->
<script lang="ts">
  import VoiceInput from '$lib/components/voice/VoiceInput.svelte';
  import { isVoiceEnabled } from '$lib/stores/voiceStore';
  
  // ... existing props
  let inputValue = '';
  
  function handleVoiceTranscript(transcript: string): void {
    inputValue = transcript;
    // Optionally auto-submit
    // handleSubmit();
  }
</script>

<!-- Existing input form -->
<form class="chat-input-form" on:submit={handleSubmit}>
  <div class="input-container">
    <input 
      bind:value={inputValue}
      placeholder="Type your message..."
      class="chat-input"
    />
    
    {#if $isVoiceEnabled}
      <div class="voice-input-wrapper">
        <VoiceInput 
          onTranscript={handleVoiceTranscript}
          disabled={loading}
        />
      </div>
    {/if}
    
    <button type="submit" class="send-button">
      Send
    </button>
  </div>
</form>

<style>
  .input-container {
    display: flex;
    align-items: center;
    gap: 0.5rem;
  }
  
  .voice-input-wrapper {
    flex-shrink: 0;
  }
</style>
```

## 6. Testing and Validation

### 6.1 Component Testing

Create test files for each component:

```typescript
// src/lib/components/voice/__tests__/VoiceInput.test.ts
import { render, fireEvent } from '@testing-library/svelte';
import VoiceInput from '../VoiceInput.svelte';

describe('VoiceInput', () => {
  test('renders voice input button', () => {
    const { getByRole } = render(VoiceInput, {
      props: {
        onTranscript: () => {}
      }
    });
    
    expect(getByRole('button')).toBeInTheDocument();
  });
  
  test('calls onTranscript when speech is recognized', async () => {
    const mockOnTranscript = vi.fn();
    
    const { getByRole } = render(VoiceInput, {
      props: {
        onTranscript: mockOnTranscript
      }
    });
    
    // Mock speech recognition
    // ... test implementation
  });
});
```

### 6.2 Service Testing

```typescript
// src/lib/services/__tests__/ChatterboxTTSService.test.ts
import { ChatterboxTTSService } from '../ChatterboxTTSService';

describe('ChatterboxTTSService', () => {
  let service: ChatterboxTTSService;
  
  beforeEach(() => {
    service = new ChatterboxTTSService();
  });
  
  test('synthesizes text successfully', async () => {
    const request = {
      text: 'Hello world',
      voiceSettings: {
        exaggeration: 0.5,
        cfgWeight: 0.5,
        speed: 1.0,
        volume: 1.0
      },
      messageId: 'test-123'
    };
    
    // Mock fetch response
    global.fetch = vi.fn().mockResolvedValue({
      ok: true,
      json: () => Promise.resolve({
        status: 'COMPLETED',
        output: {
          audio_url: 'https://example.com/audio.wav',
          duration: 2.5
        }
      })
    });
    
    const response = await service.synthesizeText(request);
    
    expect(response.status).toBe('COMPLETED');
    expect(response.output?.audio_url).toBeDefined();
  });
});
```

## 7. Deployment and Configuration

### 7.1 Environment Variables

Ensure these environment variables are set in production:

```bash
# Production .env
CHATTERBOX_RUNPOD_ENDPOINT_ID=your_production_endpoint_id
CHATTERBOX_RUNPOD_API_KEY=your_production_api_key
VOICE_FEATURES_ENABLED=true
VOICE_DEFAULT_EXAGGERATION=0.5
VOICE_DEFAULT_CFG_WEIGHT=0.5
```

### 7.2 Build Configuration

Update your `vite.config.ts` to handle voice-related assets:

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import { sveltekit } from '@sveltejs/kit/vite';

export default defineConfig({
  plugins: [sveltekit()],
  define: {
    // Ensure voice features are available in production
    'import.meta.env.VITE_VOICE_FEATURES_ENABLED': JSON.stringify(
      process.env.VOICE_FEATURES_ENABLED || 'true'
    )
  },
  server: {
    // Allow audio streaming in development
    headers: {
      'Cross-Origin-Embedder-Policy': 'credentialless',
      'Cross-Origin-Opener-Policy': 'same-origin'
    }
  }
});
```

## 8. Performance Optimization

### 8.1 Lazy Loading

Implement lazy loading for voice components:

```typescript
// src/lib/components/voice/index.ts
export const VoiceInput = () => import('./VoiceInput.svelte');
export const VoiceOutput = () => import('./VoiceOutput.svelte');
export const VoiceSettings = () => import('./VoiceSettings.svelte');
```

### 8.2 Audio Caching

Implement audio caching in your service worker:

```javascript
// src/service-worker.js
const AUDIO_CACHE = 'voice-audio-v1';

self.addEventListener('fetch', event => {
  if (event.request.url.includes('audio') || event.request.url.includes('.wav')) {
    event.respondWith(
      caches.open(AUDIO_CACHE).then(cache => {
        return cache.match(event.request).then(response => {
          if (response) {
            return response;
          }
          
          return fetch(event.request).then(fetchResponse => {
            cache.put(event.request, fetchResponse.clone());
            return fetchResponse;
          });
        });
      })
    );
  }
});
```

This implementation guide provides a complete foundation for integrating Chatterbox TTS voice agent capabilities into your SvelteKit chat UI. Follow the steps sequentially, test each component thoroughly, and customize the voice settings according to your specific requirements.
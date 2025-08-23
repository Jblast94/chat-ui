# Voice Settings Component Documentation

## 1. Overview

The Voice Settings component provides users with comprehensive controls to customize their voice agent experience, including TTS parameters, audio preferences, and conversation flow settings.

## 2. Component Structure

### 2.1 VoiceSettings.svelte

```svelte
<script lang="ts">
  import { onMount } from 'svelte';
  import { voiceActions, voiceSettings, isVoiceEnabled } from '$lib/stores/voiceStore';
  import { ttsService } from '$lib/services/ChatterboxTTSService';
  import { audioManager } from '$lib/services/AudioManager';
  import type { VoiceSettings as VoiceSettingsType } from '$lib/types/Voice';
  
  export let isOpen: boolean = false;
  export let onClose: () => void = () => {};
  
  let localSettings: VoiceSettingsType;
  let isTestingConnection: boolean = false;
  let connectionStatus: 'unknown' | 'connected' | 'failed' = 'unknown';
  let isPlayingPreview: boolean = false;
  let previewText: string = 'This is a preview of how your voice settings will sound.';
  
  // Voice profile options
  const voiceProfiles = [
    { id: '', name: 'Default Voice', description: 'Standard synthetic voice' },
    { id: 'friendly', name: 'Friendly', description: 'Warm and approachable tone' },
    { id: 'professional', name: 'Professional', description: 'Clear and authoritative' },
    { id: 'casual', name: 'Casual', description: 'Relaxed and conversational' },
    { id: 'energetic', name: 'Energetic', description: 'Upbeat and dynamic' }
  ];
  
  onMount(() => {
    localSettings = { ...$voiceSettings };
    testConnection();
  });
  
  async function testConnection(): Promise<void> {
    isTestingConnection = true;
    connectionStatus = 'unknown';
    
    try {
      const isConnected = await ttsService.testConnection();
      connectionStatus = isConnected ? 'connected' : 'failed';
    } catch (error) {
      console.error('Connection test failed:', error);
      connectionStatus = 'failed';
    } finally {
      isTestingConnection = false;
    }
  }
  
  async function playPreview(): Promise<void> {
    if (isPlayingPreview) {
      audioManager.stopAudio();
      return;
    }
    
    try {
      isPlayingPreview = true;
      
      const response = await ttsService.synthesizeText({
        text: previewText,
        voiceSettings: localSettings,
        messageId: 'preview'
      });
      
      if (response.status === 'COMPLETED' && response.output) {
        await audioManager.playAudio(response.output.audio_url, () => {
          isPlayingPreview = false;
        });
      }
    } catch (error) {
      console.error('Preview failed:', error);
      isPlayingPreview = false;
    }
  }
  
  function saveSettings(): void {
    voiceActions.saveSettings(localSettings);
    onClose();
  }
  
  function resetToDefaults(): void {
    localSettings = {
      exaggeration: 0.5,
      cfgWeight: 0.5,
      speed: 1.0,
      volume: 1.0,
      voiceProfile: undefined
    };
  }
  
  function handleSliderChange(property: keyof VoiceSettingsType, value: number): void {
    localSettings = {
      ...localSettings,
      [property]: value
    };
  }
  
  function handleVoiceProfileChange(profileId: string): void {
    localSettings = {
      ...localSettings,
      voiceProfile: profileId || undefined
    };
  }
  
  $: hasChanges = JSON.stringify(localSettings) !== JSON.stringify($voiceSettings);
</script>

{#if isOpen}
  <div class="voice-settings-overlay" on:click={onClose}>
    <div class="voice-settings-modal" on:click|stopPropagation>
      <div class="modal-header">
        <h2 class="text-xl font-semibold text-gray-800">Voice Settings</h2>
        <button 
          class="close-button"
          on:click={onClose}
          title="Close settings"
        >
          <svg class="w-6 h-6" fill="none" stroke="currentColor" viewBox="0 0 24 24">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12"/>
          </svg>
        </button>
      </div>
      
      <div class="modal-content">
        <!-- Connection Status -->
        <div class="settings-section">
          <h3 class="section-title">Connection Status</h3>
          <div class="connection-status">
            {#if isTestingConnection}
              <div class="flex items-center gap-2">
                <div class="animate-spin w-4 h-4 border-2 border-blue-500 border-t-transparent rounded-full"></div>
                <span class="text-sm text-gray-600">Testing connection...</span>
              </div>
            {:else if connectionStatus === 'connected'}
              <div class="flex items-center gap-2 text-green-600">
                <svg class="w-5 h-5" fill="currentColor" viewBox="0 0 20 20">
                  <path fill-rule="evenodd" d="M10 18a8 8 0 100-16 8 8 0 000 16zm3.707-9.293a1 1 0 00-1.414-1.414L9 10.586 7.707 9.293a1 1 0 00-1.414 1.414l2 2a1 1 0 001.414 0l4-4z" clip-rule="evenodd"/>
                </svg>
                <span class="text-sm font-medium">Connected to Chatterbox TTS</span>
              </div>
            {:else if connectionStatus === 'failed'}
              <div class="flex items-center gap-2 text-red-600">
                <svg class="w-5 h-5" fill="currentColor" viewBox="0 0 20 20">
                  <path fill-rule="evenodd" d="M10 18a8 8 0 100-16 8 8 0 000 16zM8.707 7.293a1 1 0 00-1.414 1.414L8.586 10l-1.293 1.293a1 1 0 101.414 1.414L10 11.414l1.293 1.293a1 1 0 001.414-1.414L11.414 10l1.293-1.293a1 1 0 00-1.414-1.414L10 8.586 8.707 7.293z" clip-rule="evenodd"/>
                </svg>
                <span class="text-sm font-medium">Connection failed</span>
              </div>
            {/if}
            
            <button 
              class="retry-button"
              on:click={testConnection}
              disabled={isTestingConnection}
            >
              Retry
            </button>
          </div>
        </div>
        
        <!-- Voice Profile Selection -->
        <div class="settings-section">
          <h3 class="section-title">Voice Profile</h3>
          <div class="voice-profiles">
            {#each voiceProfiles as profile}
              <label class="profile-option">
                <input 
                  type="radio" 
                  name="voiceProfile"
                  value={profile.id}
                  checked={localSettings.voiceProfile === profile.id}
                  on:change={() => handleVoiceProfileChange(profile.id)}
                />
                <div class="profile-info">
                  <div class="profile-name">{profile.name}</div>
                  <div class="profile-description">{profile.description}</div>
                </div>
              </label>
            {/each}
          </div>
        </div>
        
        <!-- Voice Parameters -->
        <div class="settings-section">
          <h3 class="section-title">Voice Parameters</h3>
          
          <!-- Exaggeration -->
          <div class="parameter-control">
            <label class="parameter-label">
              <span>Exaggeration</span>
              <span class="parameter-value">{localSettings.exaggeration.toFixed(2)}</span>
            </label>
            <input 
              type="range" 
              min="0" 
              max="1" 
              step="0.01"
              bind:value={localSettings.exaggeration}
              class="parameter-slider"
            />
            <div class="parameter-description">
              Controls emotional expressiveness and vocal variation
            </div>
          </div>
          
          <!-- CFG Weight -->
          <div class="parameter-control">
            <label class="parameter-label">
              <span>CFG Weight</span>
              <span class="parameter-value">{localSettings.cfgWeight.toFixed(2)}</span>
            </label>
            <input 
              type="range" 
              min="0" 
              max="1" 
              step="0.01"
              bind:value={localSettings.cfgWeight}
              class="parameter-slider"
            />
            <div class="parameter-description">
              Balances between creativity and adherence to the voice model
            </div>
          </div>
          
          <!-- Speed -->
          <div class="parameter-control">
            <label class="parameter-label">
              <span>Speed</span>
              <span class="parameter-value">{localSettings.speed.toFixed(2)}x</span>
            </label>
            <input 
              type="range" 
              min="0.5" 
              max="2.0" 
              step="0.1"
              bind:value={localSettings.speed}
              class="parameter-slider"
            />
            <div class="parameter-description">
              Controls the speaking rate of the generated voice
            </div>
          </div>
          
          <!-- Volume -->
          <div class="parameter-control">
            <label class="parameter-label">
              <span>Volume</span>
              <span class="parameter-value">{Math.round(localSettings.volume * 100)}%</span>
            </label>
            <input 
              type="range" 
              min="0" 
              max="1" 
              step="0.01"
              bind:value={localSettings.volume}
              class="parameter-slider"
            />
            <div class="parameter-description">
              Controls the playback volume of generated audio
            </div>
          </div>
        </div>
        
        <!-- Preview Section -->
        <div class="settings-section">
          <h3 class="section-title">Preview</h3>
          <div class="preview-section">
            <textarea 
              bind:value={previewText}
              placeholder="Enter text to preview voice settings..."
              class="preview-text"
              rows="3"
            ></textarea>
            
            <button 
              class="preview-button"
              on:click={playPreview}
              disabled={!previewText.trim() || connectionStatus !== 'connected'}
            >
              {#if isPlayingPreview}
                <svg class="w-4 h-4" fill="currentColor" viewBox="0 0 24 24">
                  <rect x="6" y="6" width="12" height="12" rx="2"/>
                </svg>
                Stop Preview
              {:else}
                <svg class="w-4 h-4" fill="currentColor" viewBox="0 0 24 24">
                  <polygon points="5,3 19,12 5,21"/>
                </svg>
                Play Preview
              {/if}
            </button>
          </div>
        </div>
      </div>
      
      <div class="modal-footer">
        <button 
          class="reset-button"
          on:click={resetToDefaults}
        >
          Reset to Defaults
        </button>
        
        <div class="action-buttons">
          <button 
            class="cancel-button"
            on:click={onClose}
          >
            Cancel
          </button>
          
          <button 
            class="save-button"
            on:click={saveSettings}
            disabled={!hasChanges}
          >
            Save Settings
          </button>
        </div>
      </div>
    </div>
  </div>
{/if}

<style>
  .voice-settings-overlay {
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background-color: rgba(0, 0, 0, 0.5);
    display: flex;
    align-items: center;
    justify-content: center;
    z-index: 1000;
    padding: 1rem;
  }
  
  .voice-settings-modal {
    background: white;
    border-radius: 12px;
    box-shadow: 0 20px 25px -5px rgba(0, 0, 0, 0.1), 0 10px 10px -5px rgba(0, 0, 0, 0.04);
    max-width: 600px;
    width: 100%;
    max-height: 90vh;
    overflow: hidden;
    display: flex;
    flex-direction: column;
  }
  
  .modal-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 1.5rem;
    border-bottom: 1px solid #e5e7eb;
  }
  
  .close-button {
    padding: 0.5rem;
    color: #6b7280;
    hover: color: #374151;
    transition: color 0.2s;
  }
  
  .modal-content {
    flex: 1;
    overflow-y: auto;
    padding: 1.5rem;
  }
  
  .settings-section {
    margin-bottom: 2rem;
  }
  
  .section-title {
    font-size: 1.125rem;
    font-weight: 600;
    color: #374151;
    margin-bottom: 1rem;
  }
  
  .connection-status {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 1rem;
    background-color: #f9fafb;
    border-radius: 8px;
  }
  
  .retry-button {
    padding: 0.5rem 1rem;
    background-color: #3b82f6;
    color: white;
    border-radius: 6px;
    font-size: 0.875rem;
    transition: background-color 0.2s;
  }
  
  .retry-button:hover:not(:disabled) {
    background-color: #2563eb;
  }
  
  .retry-button:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }
  
  .voice-profiles {
    display: grid;
    gap: 0.75rem;
  }
  
  .profile-option {
    display: flex;
    align-items: center;
    gap: 0.75rem;
    padding: 1rem;
    border: 2px solid #e5e7eb;
    border-radius: 8px;
    cursor: pointer;
    transition: all 0.2s;
  }
  
  .profile-option:hover {
    border-color: #3b82f6;
    background-color: #f8fafc;
  }
  
  .profile-option:has(input:checked) {
    border-color: #3b82f6;
    background-color: #eff6ff;
  }
  
  .profile-info {
    flex: 1;
  }
  
  .profile-name {
    font-weight: 500;
    color: #374151;
  }
  
  .profile-description {
    font-size: 0.875rem;
    color: #6b7280;
    margin-top: 0.25rem;
  }
  
  .parameter-control {
    margin-bottom: 1.5rem;
  }
  
  .parameter-label {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 0.5rem;
    font-weight: 500;
    color: #374151;
  }
  
  .parameter-value {
    font-family: monospace;
    background-color: #f3f4f6;
    padding: 0.25rem 0.5rem;
    border-radius: 4px;
    font-size: 0.875rem;
  }
  
  .parameter-slider {
    width: 100%;
    height: 6px;
    background: #e5e7eb;
    border-radius: 3px;
    outline: none;
    margin-bottom: 0.5rem;
  }
  
  .parameter-slider::-webkit-slider-thumb {
    appearance: none;
    width: 20px;
    height: 20px;
    background: #3b82f6;
    border-radius: 50%;
    cursor: pointer;
  }
  
  .parameter-slider::-moz-range-thumb {
    width: 20px;
    height: 20px;
    background: #3b82f6;
    border-radius: 50%;
    cursor: pointer;
    border: none;
  }
  
  .parameter-description {
    font-size: 0.875rem;
    color: #6b7280;
    line-height: 1.4;
  }
  
  .preview-section {
    display: flex;
    flex-direction: column;
    gap: 1rem;
  }
  
  .preview-text {
    width: 100%;
    padding: 0.75rem;
    border: 2px solid #e5e7eb;
    border-radius: 8px;
    resize: vertical;
    font-family: inherit;
    font-size: 0.875rem;
  }
  
  .preview-text:focus {
    outline: none;
    border-color: #3b82f6;
  }
  
  .preview-button {
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 0.5rem;
    padding: 0.75rem 1.5rem;
    background-color: #10b981;
    color: white;
    border-radius: 8px;
    font-weight: 500;
    transition: background-color 0.2s;
    align-self: flex-start;
  }
  
  .preview-button:hover:not(:disabled) {
    background-color: #059669;
  }
  
  .preview-button:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }
  
  .modal-footer {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 1.5rem;
    border-top: 1px solid #e5e7eb;
    background-color: #f9fafb;
  }
  
  .reset-button {
    padding: 0.75rem 1rem;
    color: #6b7280;
    border: 1px solid #d1d5db;
    border-radius: 6px;
    font-weight: 500;
    transition: all 0.2s;
  }
  
  .reset-button:hover {
    color: #374151;
    border-color: #9ca3af;
  }
  
  .action-buttons {
    display: flex;
    gap: 0.75rem;
  }
  
  .cancel-button {
    padding: 0.75rem 1.5rem;
    color: #6b7280;
    border: 1px solid #d1d5db;
    border-radius: 6px;
    font-weight: 500;
    transition: all 0.2s;
  }
  
  .cancel-button:hover {
    color: #374151;
    border-color: #9ca3af;
  }
  
  .save-button {
    padding: 0.75rem 1.5rem;
    background-color: #3b82f6;
    color: white;
    border-radius: 6px;
    font-weight: 500;
    transition: background-color 0.2s;
  }
  
  .save-button:hover:not(:disabled) {
    background-color: #2563eb;
  }
  
  .save-button:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }
</style>
```

### 2.2 Voice Settings Integration

To integrate the voice settings component into your chat interface:

```svelte
<!-- In your main chat component -->
<script lang="ts">
  import VoiceSettings from '$lib/components/voice/VoiceSettings.svelte';
  import { isVoiceEnabled } from '$lib/stores/voiceStore';
  
  let showVoiceSettings = false;
  
  function openVoiceSettings(): void {
    showVoiceSettings = true;
  }
  
  function closeVoiceSettings(): void {
    showVoiceSettings = false;
  }
</script>

<!-- Voice settings button in your UI -->
{#if $isVoiceEnabled}
  <button 
    class="voice-settings-trigger"
    on:click={openVoiceSettings}
    title="Voice Settings"
  >
    <svg class="w-5 h-5" fill="currentColor" viewBox="0 0 20 20">
      <path fill-rule="evenodd" d="M11.49 3.17c-.38-1.56-2.6-1.56-2.98 0a1.532 1.532 0 01-2.286.948c-1.372-.836-2.942.734-2.106 2.106.54.886.061 2.042-.947 2.287-1.561.379-1.561 2.6 0 2.978a1.532 1.532 0 01.947 2.287c-.836 1.372.734 2.942 2.106 2.106a1.532 1.532 0 012.287.947c.379 1.561 2.6 1.561 2.978 0a1.533 1.533 0 012.287-.947c1.372.836 2.942-.734 2.106-2.106a1.533 1.533 0 01.947-2.287c1.561-.379 1.561-2.6 0-2.978a1.532 1.532 0 01-.947-2.287c.836-1.372-.734-2.942-2.106-2.106a1.532 1.532 0 01-2.287-.947zM10 13a3 3 0 100-6 3 3 0 000 6z" clip-rule="evenodd"/>
    </svg>
  </button>
{/if}

<!-- Voice settings modal -->
<VoiceSettings 
  isOpen={showVoiceSettings}
  onClose={closeVoiceSettings}
/>
```

## 3. Advanced Features

### 3.1 Voice Profile Management

Create a service for managing custom voice profiles:

```typescript
// src/lib/services/VoiceProfileService.ts
import type { VoiceSettings } from '$lib/types/Voice';

export interface VoiceProfile {
  id: string;
  name: string;
  description: string;
  settings: VoiceSettings;
  isCustom: boolean;
  createdAt: number;
}

export class VoiceProfileService {
  private storageKey = 'voice-profiles';
  
  getProfiles(): VoiceProfile[] {
    try {
      const stored = localStorage.getItem(this.storageKey);
      return stored ? JSON.parse(stored) : this.getDefaultProfiles();
    } catch (error) {
      console.error('Failed to load voice profiles:', error);
      return this.getDefaultProfiles();
    }
  }
  
  saveProfile(profile: Omit<VoiceProfile, 'id' | 'createdAt'>): VoiceProfile {
    const newProfile: VoiceProfile = {
      ...profile,
      id: this.generateId(),
      createdAt: Date.now()
    };
    
    const profiles = this.getProfiles();
    profiles.push(newProfile);
    
    this.saveProfiles(profiles);
    return newProfile;
  }
  
  updateProfile(id: string, updates: Partial<VoiceProfile>): void {
    const profiles = this.getProfiles();
    const index = profiles.findIndex(p => p.id === id);
    
    if (index !== -1) {
      profiles[index] = { ...profiles[index], ...updates };
      this.saveProfiles(profiles);
    }
  }
  
  deleteProfile(id: string): void {
    const profiles = this.getProfiles().filter(p => p.id !== id);
    this.saveProfiles(profiles);
  }
  
  private saveProfiles(profiles: VoiceProfile[]): void {
    try {
      localStorage.setItem(this.storageKey, JSON.stringify(profiles));
    } catch (error) {
      console.error('Failed to save voice profiles:', error);
    }
  }
  
  private getDefaultProfiles(): VoiceProfile[] {
    return [
      {
        id: 'default',
        name: 'Default',
        description: 'Standard voice settings',
        settings: {
          exaggeration: 0.5,
          cfgWeight: 0.5,
          speed: 1.0,
          volume: 1.0
        },
        isCustom: false,
        createdAt: Date.now()
      },
      {
        id: 'expressive',
        name: 'Expressive',
        description: 'High emotion and variation',
        settings: {
          exaggeration: 0.8,
          cfgWeight: 0.3,
          speed: 0.9,
          volume: 1.0
        },
        isCustom: false,
        createdAt: Date.now()
      },
      {
        id: 'calm',
        name: 'Calm',
        description: 'Steady and measured tone',
        settings: {
          exaggeration: 0.2,
          cfgWeight: 0.7,
          speed: 0.8,
          volume: 0.9
        },
        isCustom: false,
        createdAt: Date.now()
      }
    ];
  }
  
  private generateId(): string {
    return `profile_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
  }
}

export const voiceProfileService = new VoiceProfileService();
```

### 3.2 Accessibility Features

Add accessibility enhancements to the voice settings:

```svelte
<!-- Enhanced accessibility features -->
<script lang="ts">
  import { onMount } from 'svelte';
  
  let modalElement: HTMLElement;
  let firstFocusableElement: HTMLElement;
  let lastFocusableElement: HTMLElement;
  
  onMount(() => {
    if (isOpen) {
      setupFocusTrap();
      firstFocusableElement?.focus();
    }
  });
  
  function setupFocusTrap(): void {
    const focusableElements = modalElement.querySelectorAll(
      'button, input, select, textarea, [tabindex]:not([tabindex="-1"])'
    );
    
    firstFocusableElement = focusableElements[0] as HTMLElement;
    lastFocusableElement = focusableElements[focusableElements.length - 1] as HTMLElement;
  }
  
  function handleKeydown(event: KeyboardEvent): void {
    if (event.key === 'Escape') {
      onClose();
      return;
    }
    
    if (event.key === 'Tab') {
      if (event.shiftKey) {
        if (document.activeElement === firstFocusableElement) {
          event.preventDefault();
          lastFocusableElement?.focus();
        }
      } else {
        if (document.activeElement === lastFocusableElement) {
          event.preventDefault();
          firstFocusableElement?.focus();
        }
      }
    }
  }
</script>

<!-- Add to modal element -->
<div 
  class="voice-settings-modal"
  bind:this={modalElement}
  on:keydown={handleKeydown}
  role="dialog"
  aria-labelledby="voice-settings-title"
  aria-describedby="voice-settings-description"
>
  <div class="modal-header">
    <h2 id="voice-settings-title" class="text-xl font-semibold text-gray-800">
      Voice Settings
    </h2>
    <!-- ... rest of header -->
  </div>
  
  <div id="voice-settings-description" class="sr-only">
    Configure voice synthesis parameters and audio preferences for the voice agent
  </div>
  
  <!-- ... rest of modal content -->
</div>
```

## 4. Usage Examples

### 4.1 Basic Integration

```svelte
<!-- Simple integration in chat interface -->
<script lang="ts">
  import VoiceSettings from '$lib/components/voice/VoiceSettings.svelte';
  
  let showSettings = false;
</script>

<div class="chat-header">
  <h1>Chat</h1>
  <button on:click={() => showSettings = true}>
    Voice Settings
  </button>
</div>

<VoiceSettings 
  isOpen={showSettings} 
  onClose={() => showSettings = false}
/>
```

### 4.2 Advanced Integration with Profiles

```svelte
<!-- Advanced integration with profile management -->
<script lang="ts">
  import VoiceSettings from '$lib/components/voice/VoiceSettings.svelte';
  import { voiceProfileService } from '$lib/services/VoiceProfileService';
  import { voiceActions } from '$lib/stores/voiceStore';
  
  let showSettings = false;
  let profiles = voiceProfileService.getProfiles();
  
  function applyProfile(profileId: string): void {
    const profile = profiles.find(p => p.id === profileId);
    if (profile) {
      voiceActions.saveSettings(profile.settings);
    }
  }
</script>

<!-- Quick profile selector -->
<div class="profile-selector">
  <label for="voice-profile">Voice Profile:</label>
  <select id="voice-profile" on:change={(e) => applyProfile(e.target.value)}>
    {#each profiles as profile}
      <option value={profile.id}>{profile.name}</option>
    {/each}
  </select>
  
  <button on:click={() => showSettings = true}>
    Customize
  </button>
</div>

<VoiceSettings 
  isOpen={showSettings} 
  onClose={() => showSettings = false}
/>
```

This voice settings component provides a comprehensive interface for users to customize their voice agent experience, with support for real-time preview, profile management, and accessibility features.
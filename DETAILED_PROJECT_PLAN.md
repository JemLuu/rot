# Live Lecture Transcription with Minecraft Parkour - Detailed Implementation Plan

## Project Structure

```
src/
├── components/
│   ├── ui/
│   │   ├── Button.tsx
│   │   ├── Modal.tsx
│   │   └── LoadingSpinner.tsx
│   ├── video/
│   │   ├── VideoPlayer.tsx
│   │   └── VideoControls.tsx
│   ├── transcription/
│   │   ├── TextOverlay.tsx
│   │   ├── TranscriptHistory.tsx
│   │   └── SessionControls.tsx
│   ├── summary/
│   │   ├── SummaryModal.tsx
│   │   └── SummaryControls.tsx
│   └── layout/
│       ├── Header.tsx
│       └── MainLayout.tsx
├── hooks/
│   ├── useTranscription.ts
│   ├── useVideoPlayer.ts
│   ├── useSession.ts
│   └── useChatGPT.ts
├── store/
│   ├── transcriptionStore.ts
│   ├── sessionStore.ts
│   └── settingsStore.ts
├── services/
│   ├── speechRecognition.ts
│   ├── assemblyAI.ts
│   ├── openAI.ts
│   └── fileExport.ts
├── utils/
│   ├── audio.ts
│   ├── formatters.ts
│   └── constants.ts
├── types/
│   ├── transcription.ts
│   ├── session.ts
│   └── api.ts
├── assets/
│   ├── videos/
│   │   └── minecraft-parkour.mp4
│   └── icons/
└── styles/
    └── globals.css
```

## Component Architecture Details

### Core Components

#### VideoPlayer.tsx
```typescript
interface VideoPlayerProps {
  src: string;
  isPlaying: boolean;
  volume: number;
  onLoadedData: () => void;
  onError: (error: MediaError) => void;
}

// Features:
// - Seamless looping
// - Volume control
// - Fullscreen support
// - Performance optimization for long sessions
```

#### TextOverlay.tsx
```typescript
interface TextOverlayProps {
  currentText: string;
  isListening: boolean;
  position: 'top' | 'bottom' | 'center';
  fontSize: number;
  backgroundColor: string;
  textColor: string;
}

// Features:
// - Real-time text display
// - Smooth fade-in animations
// - Auto-sizing based on text length
// - Configurable positioning
```

#### TranscriptHistory.tsx
```typescript
interface TranscriptEntry {
  id: string;
  text: string;
  timestamp: number;
  confidence: number;
}

interface TranscriptHistoryProps {
  entries: TranscriptEntry[];
  isVisible: boolean;
  onEntryEdit: (id: string, newText: string) => void;
  onEntryDelete: (id: string) => void;
}
```

#### SessionControls.tsx
```typescript
interface SessionControlsProps {
  isRecording: boolean;
  isPaused: boolean;
  sessionDuration: number;
  wordCount: number;
  onStart: () => void;
  onStop: () => void;
  onPause: () => void;
  onExport: () => void;
  onSummarize: () => void;
}
```

### Hooks Architecture

#### useTranscription.ts
```typescript
interface TranscriptionState {
  isListening: boolean;
  currentText: string;
  transcript: TranscriptEntry[];
  error: string | null;
  confidence: number;
}

interface TranscriptionActions {
  startListening: () => Promise<void>;
  stopListening: () => void;
  pauseListening: () => void;
  clearTranscript: () => void;
  addManualEntry: (text: string) => void;
}
```

#### useSession.ts
```typescript
interface SessionData {
  id: string;
  startTime: number;
  endTime: number | null;
  duration: number;
  wordCount: number;
  transcript: TranscriptEntry[];
  summary?: string;
}
```

## Detailed Phase-by-Phase Implementation

### Phase 1: Project Setup & Configuration

#### 1.1 Initialize Project
```bash
# Create Vite React TypeScript project
npm create vite@latest lecture-transcription -- --template react-ts
cd lecture-transcription

# Install core dependencies
npm install

# Install UI libraries
npm install tailwindcss postcss autoprefixer
npm install framer-motion
npm install @headlessui/react @heroicons/react

# Install state management
npm install zustand

# Install API libraries
npm install openai
npm install @types/web-speech-api

# Install development dependencies
npm install -D @types/node
npm install -D eslint-plugin-react-hooks
npm install -D prettier eslint-config-prettier
```

#### 1.2 Configure Tailwind CSS
```bash
npx tailwindcss init -p
```

**tailwind.config.js:**
```javascript
module.exports = {
  content: ['./index.html', './src/**/*.{js,ts,jsx,tsx}'],
  theme: {
    extend: {
      animation: {
        'fade-in': 'fadeIn 0.3s ease-in-out',
        'slide-up': 'slideUp 0.3s ease-out',
      },
      keyframes: {
        fadeIn: {
          '0%': { opacity: '0' },
          '100%': { opacity: '1' },
        },
        slideUp: {
          '0%': { transform: 'translateY(20px)', opacity: '0' },
          '100%': { transform: 'translateY(0)', opacity: '1' },
        },
      },
    },
  },
  plugins: [],
}
```

#### 1.3 TypeScript Configuration
**tsconfig.json additions:**
```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"],
      "@/components/*": ["src/components/*"],
      "@/hooks/*": ["src/hooks/*"],
      "@/store/*": ["src/store/*"],
      "@/services/*": ["src/services/*"],
      "@/utils/*": ["src/utils/*"],
      "@/types/*": ["src/types/*"]
    }
  }
}
```

#### 1.4 Environment Setup
**.env.local:**
```
VITE_OPENAI_API_KEY=your_openai_api_key_here
VITE_ASSEMBLYAI_API_KEY=your_assemblyai_api_key_here
VITE_APP_NAME=Lecture Transcription
```

### Phase 2: Core Video System Implementation

#### 2.1 VideoPlayer Component
```typescript
// src/components/video/VideoPlayer.tsx
import { useRef, useEffect, forwardRef } from 'react';
import { useVideoPlayer } from '@/hooks/useVideoPlayer';

interface VideoPlayerProps {
  src: string;
  className?: string;
  autoPlay?: boolean;
  muted?: boolean;
}

export const VideoPlayer = forwardRef<HTMLVideoElement, VideoPlayerProps>(
  ({ src, className = '', autoPlay = true, muted = true }, ref) => {
    const localRef = useRef<HTMLVideoElement>(null);
    const videoRef = ref || localRef;
    
    const { 
      isPlaying, 
      volume, 
      setVolume, 
      togglePlayPause 
    } = useVideoPlayer(videoRef);

    useEffect(() => {
      const video = videoRef.current;
      if (!video) return;

      const handleEnded = () => {
        video.currentTime = 0;
        video.play();
      };

      video.addEventListener('ended', handleEnded);
      return () => video.removeEventListener('ended', handleEnded);
    }, []);

    return (
      <div className={`relative ${className}`}>
        <video
          ref={videoRef}
          src={src}
          autoPlay={autoPlay}
          muted={muted}
          loop
          playsInline
          className="w-full h-full object-cover"
          onLoadedData={() => console.log('Video loaded')}
          onError={(e) => console.error('Video error:', e)}
        />
        
        {/* Video Controls Overlay */}
        <div className="absolute bottom-4 left-4 right-4 flex items-center justify-between bg-black bg-opacity-50 rounded-lg p-2">
          <button
            onClick={togglePlayPause}
            className="text-white hover:text-blue-400 transition-colors"
          >
            {isPlaying ? '⏸️' : '▶️'}
          </button>
          
          <div className="flex items-center space-x-2">
            <span className="text-white text-sm">🔊</span>
            <input
              type="range"
              min="0"
              max="1"
              step="0.1"
              value={volume}
              onChange={(e) => setVolume(parseFloat(e.target.value))}
              className="w-20"
            />
          </div>
        </div>
      </div>
    );
  }
);
```

#### 2.2 useVideoPlayer Hook
```typescript
// src/hooks/useVideoPlayer.ts
import { useState, useEffect, RefObject } from 'react';

export const useVideoPlayer = (videoRef: RefObject<HTMLVideoElement>) => {
  const [isPlaying, setIsPlaying] = useState(false);
  const [volume, setVolume] = useState(0.5);
  const [duration, setDuration] = useState(0);
  const [currentTime, setCurrentTime] = useState(0);

  useEffect(() => {
    const video = videoRef.current;
    if (!video) return;

    const handlePlay = () => setIsPlaying(true);
    const handlePause = () => setIsPlaying(false);
    const handleTimeUpdate = () => setCurrentTime(video.currentTime);
    const handleLoadedMetadata = () => setDuration(video.duration);

    video.addEventListener('play', handlePlay);
    video.addEventListener('pause', handlePause);
    video.addEventListener('timeupdate', handleTimeUpdate);
    video.addEventListener('loadedmetadata', handleLoadedMetadata);

    return () => {
      video.removeEventListener('play', handlePlay);
      video.removeEventListener('pause', handlePause);
      video.removeEventListener('timeupdate', handleTimeUpdate);
      video.removeEventListener('loadedmetadata', handleLoadedMetadata);
    };
  }, []);

  useEffect(() => {
    const video = videoRef.current;
    if (video) {
      video.volume = volume;
    }
  }, [volume]);

  const togglePlayPause = () => {
    const video = videoRef.current;
    if (video) {
      if (isPlaying) {
        video.pause();
      } else {
        video.play();
      }
    }
  };

  return {
    isPlaying,
    volume,
    duration,
    currentTime,
    setVolume,
    togglePlayPause,
  };
};
```

### Phase 3: Speech Recognition Implementation

#### 3.1 Speech Recognition Service
```typescript
// src/services/speechRecognition.ts
interface SpeechRecognitionConfig {
  continuous: boolean;
  interimResults: boolean;
  language: string;
  maxAlternatives: number;
}

export class SpeechRecognitionService {
  private recognition: SpeechRecognition | null = null;
  private isSupported = false;

  constructor() {
    this.checkSupport();
    this.initializeRecognition();
  }

  private checkSupport() {
    this.isSupported = 'webkitSpeechRecognition' in window || 'SpeechRecognition' in window;
  }

  private initializeRecognition() {
    if (!this.isSupported) return;

    const SpeechRecognition = window.SpeechRecognition || window.webkitSpeechRecognition;
    this.recognition = new SpeechRecognition();
    
    this.recognition.continuous = true;
    this.recognition.interimResults = true;
    this.recognition.lang = 'en-US';
    this.recognition.maxAlternatives = 3;
  }

  public startListening(
    onResult: (transcript: string, isFinal: boolean, confidence: number) => void,
    onError: (error: SpeechRecognitionErrorEvent) => void
  ): Promise<void> {
    return new Promise((resolve, reject) => {
      if (!this.recognition) {
        reject(new Error('Speech recognition not supported'));
        return;
      }

      this.recognition.onstart = () => resolve();
      this.recognition.onerror = onError;
      
      this.recognition.onresult = (event: SpeechRecognitionEvent) => {
        const result = event.results[event.resultIndex];
        const transcript = result[0].transcript;
        const confidence = result[0].confidence;
        const isFinal = result.isFinal;
        
        onResult(transcript, isFinal, confidence);
      };

      this.recognition.start();
    });
  }

  public stopListening() {
    if (this.recognition) {
      this.recognition.stop();
    }
  }

  public isServiceSupported() {
    return this.isSupported;
  }
}
```

#### 3.2 useTranscription Hook
```typescript
// src/hooks/useTranscription.ts
import { useState, useCallback, useRef, useEffect } from 'react';
import { SpeechRecognitionService } from '@/services/speechRecognition';
import { AssemblyAIService } from '@/services/assemblyAI';
import { useTranscriptionStore } from '@/store/transcriptionStore';

export const useTranscription = () => {
  const [isListening, setIsListening] = useState(false);
  const [currentText, setCurrentText] = useState('');
  const [error, setError] = useState<string | null>(null);
  const [confidence, setConfidence] = useState(0);

  const speechService = useRef(new SpeechRecognitionService());
  const assemblyService = useRef(new AssemblyAIService());
  
  const { addEntry, clearTranscript } = useTranscriptionStore();

  const startListening = useCallback(async () => {
    try {
      setError(null);
      setIsListening(true);

      if (speechService.current.isServiceSupported()) {
        await speechService.current.startListening(
          (transcript, isFinal, confidence) => {
            setCurrentText(transcript);
            setConfidence(confidence);
            
            if (isFinal) {
              addEntry({
                id: Date.now().toString(),
                text: transcript,
                timestamp: Date.now(),
                confidence: confidence || 0.8,
              });
              setCurrentText('');
            }
          },
          (error) => {
            console.error('Speech recognition error:', error);
            setError(`Speech recognition error: ${error.error}`);
            fallbackToAssemblyAI();
          }
        );
      } else {
        await fallbackToAssemblyAI();
      }
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Unknown error occurred');
      setIsListening(false);
    }
  }, [addEntry]);

  const fallbackToAssemblyAI = async () => {
    try {
      await assemblyService.current.startTranscription(
        (transcript, isFinal, confidence) => {
          setCurrentText(transcript);
          setConfidence(confidence);
          
          if (isFinal) {
            addEntry({
              id: Date.now().toString(),
              text: transcript,
              timestamp: Date.now(),
              confidence: confidence || 0.8,
            });
            setCurrentText('');
          }
        },
        (error) => {
          setError(`AssemblyAI error: ${error}`);
          setIsListening(false);
        }
      );
    } catch (err) {
      setError('Both speech recognition services failed');
      setIsListening(false);
    }
  };

  const stopListening = useCallback(() => {
    speechService.current.stopListening();
    assemblyService.current.stopTranscription();
    setIsListening(false);
    setCurrentText('');
  }, []);

  const pauseListening = useCallback(() => {
    stopListening();
  }, [stopListening]);

  return {
    isListening,
    currentText,
    error,
    confidence,
    startListening,
    stopListening,
    pauseListening,
  };
};
```

### Phase 4: State Management with Zustand

#### 4.1 Transcription Store
```typescript
// src/store/transcriptionStore.ts
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

export interface TranscriptEntry {
  id: string;
  text: string;
  timestamp: number;
  confidence: number;
}

interface TranscriptionState {
  entries: TranscriptEntry[];
  currentSession: string | null;
  totalWordCount: number;
}

interface TranscriptionActions {
  addEntry: (entry: TranscriptEntry) => void;
  updateEntry: (id: string, text: string) => void;
  deleteEntry: (id: string) => void;
  clearTranscript: () => void;
  startSession: () => void;
  endSession: () => void;
  getSessionTranscript: () => string;
  exportTranscript: () => void;
}

export const useTranscriptionStore = create<TranscriptionState & TranscriptionActions>()(
  persist(
    (set, get) => ({
      entries: [],
      currentSession: null,
      totalWordCount: 0,

      addEntry: (entry) =>
        set((state) => ({
          entries: [...state.entries, entry],
          totalWordCount: state.totalWordCount + entry.text.split(' ').length,
        })),

      updateEntry: (id, text) =>
        set((state) => ({
          entries: state.entries.map((entry) =>
            entry.id === id ? { ...entry, text } : entry
          ),
        })),

      deleteEntry: (id) =>
        set((state) => ({
          entries: state.entries.filter((entry) => entry.id !== id),
        })),

      clearTranscript: () =>
        set(() => ({
          entries: [],
          totalWordCount: 0,
        })),

      startSession: () =>
        set(() => ({
          currentSession: Date.now().toString(),
          entries: [],
          totalWordCount: 0,
        })),

      endSession: () =>
        set(() => ({
          currentSession: null,
        })),

      getSessionTranscript: () => {
        const { entries } = get();
        return entries
          .map((entry) => `[${new Date(entry.timestamp).toLocaleTimeString()}] ${entry.text}`)
          .join('\n');
      },

      exportTranscript: () => {
        const transcript = get().getSessionTranscript();
        const blob = new Blob([transcript], { type: 'text/plain' });
        const url = URL.createObjectURL(blob);
        const a = document.createElement('a');
        a.href = url;
        a.download = `lecture-transcript-${Date.now()}.txt`;
        document.body.appendChild(a);
        a.click();
        document.body.removeChild(a);
        URL.revokeObjectURL(url);
      },
    }),
    {
      name: 'transcription-storage',
    }
  )
);
```

### Phase 5: ChatGPT Integration

#### 5.1 OpenAI Service
```typescript
// src/services/openAI.ts
import OpenAI from 'openai';

const openai = new OpenAI({
  apiKey: import.meta.env.VITE_OPENAI_API_KEY,
  dangerouslyAllowBrowser: true, // Note: In production, use a backend proxy
});

export interface SummaryOptions {
  style: 'bullet-points' | 'paragraph' | 'outline';
  length: 'short' | 'medium' | 'detailed';
  includeKeywords: boolean;
}

export class OpenAIService {
  async summarizeTranscript(
    transcript: string,
    options: SummaryOptions = {
      style: 'bullet-points',
      length: 'medium',
      includeKeywords: true,
    }
  ): Promise<string> {
    const prompt = this.buildSummaryPrompt(transcript, options);

    try {
      const response = await openai.chat.completions.create({
        model: 'gpt-3.5-turbo',
        messages: [
          {
            role: 'system',
            content: 'You are a helpful assistant that summarizes lecture transcripts clearly and concisely.',
          },
          {
            role: 'user',
            content: prompt,
          },
        ],
        max_tokens: this.getMaxTokens(options.length),
        temperature: 0.3,
      });

      return response.choices[0]?.message?.content || 'Unable to generate summary';
    } catch (error) {
      console.error('OpenAI API Error:', error);
      throw new Error('Failed to generate summary. Please check your API key and try again.');
    }
  }

  private buildSummaryPrompt(transcript: string, options: SummaryOptions): string {
    let prompt = `Please summarize the following lecture transcript:\n\n${transcript}\n\n`;

    prompt += `Format: ${options.style === 'bullet-points' ? 'Use bullet points' : 
                        options.style === 'outline' ? 'Use an outline format' : 
                        'Use paragraph format'}\n`;

    prompt += `Length: ${options.length} summary\n`;

    if (options.includeKeywords) {
      prompt += 'Include key terms and concepts at the end.\n';
    }

    return prompt;
  }

  private getMaxTokens(length: string): number {
    switch (length) {
      case 'short': return 200;
      case 'medium': return 500;
      case 'detailed': return 1000;
      default: return 500;
    }
  }
}
```

#### 5.2 useChatGPT Hook
```typescript
// src/hooks/useChatGPT.ts
import { useState } from 'react';
import { OpenAIService, SummaryOptions } from '@/services/openAI';

const openAIService = new OpenAIService();

export const useChatGPT = () => {
  const [isGenerating, setIsGenerating] = useState(false);
  const [summary, setSummary] = useState<string>('');
  const [error, setError] = useState<string | null>(null);

  const generateSummary = async (transcript: string, options?: SummaryOptions) => {
    setIsGenerating(true);
    setError(null);

    try {
      const result = await openAIService.summarizeTranscript(transcript, options);
      setSummary(result);
      return result;
    } catch (err) {
      const errorMessage = err instanceof Error ? err.message : 'Failed to generate summary';
      setError(errorMessage);
      throw new Error(errorMessage);
    } finally {
      setIsGenerating(false);
    }
  };

  const clearSummary = () => {
    setSummary('');
    setError(null);
  };

  return {
    isGenerating,
    summary,
    error,
    generateSummary,
    clearSummary,
  };
};
```

### Phase 6: Main Application Assembly

#### 6.1 Main App Component
```typescript
// src/App.tsx
import { useState } from 'react';
import { MainLayout } from '@/components/layout/MainLayout';
import { VideoPlayer } from '@/components/video/VideoPlayer';
import { TextOverlay } from '@/components/transcription/TextOverlay';
import { SessionControls } from '@/components/transcription/SessionControls';
import { TranscriptHistory } from '@/components/transcription/TranscriptHistory';
import { SummaryModal } from '@/components/summary/SummaryModal';
import { useTranscription } from '@/hooks/useTranscription';
import { useTranscriptionStore } from '@/store/transcriptionStore';
import minecraftVideo from '@/assets/videos/minecraft-parkour.mp4';

export default function App() {
  const [showHistory, setShowHistory] = useState(false);
  const [showSummary, setShowSummary] = useState(false);
  
  const {
    isListening,
    currentText,
    error,
    confidence,
    startListening,
    stopListening,
    pauseListening,
  } = useTranscription();

  const { 
    entries, 
    totalWordCount, 
    currentSession,
    startSession,
    endSession,
    exportTranscript,
    getSessionTranscript,
  } = useTranscriptionStore();

  const handleStartSession = () => {
    startSession();
    startListening();
  };

  const handleStopSession = () => {
    stopListening();
    endSession();
  };

  const handleExport = () => {
    exportTranscript();
  };

  const handleSummarize = () => {
    setShowSummary(true);
  };

  return (
    <MainLayout>
      <div className="relative h-screen overflow-hidden">
        {/* Background Video */}
        <VideoPlayer
          src={minecraftVideo}
          className="absolute inset-0 z-0"
        />

        {/* Text Overlay */}
        <TextOverlay
          currentText={currentText}
          isListening={isListening}
          confidence={confidence}
          className="absolute inset-x-0 top-1/3 z-10"
        />

        {/* Session Controls */}
        <SessionControls
          isRecording={isListening}
          isPaused={false}
          sessionDuration={currentSession ? Date.now() - parseInt(currentSession) : 0}
          wordCount={totalWordCount}
          onStart={handleStartSession}
          onStop={handleStopSession}
          onPause={pauseListening}
          onExport={handleExport}
          onSummarize={handleSummarize}
          onToggleHistory={() => setShowHistory(!showHistory)}
          className="absolute bottom-4 left-4 right-4 z-20"
        />

        {/* Transcript History Sidebar */}
        {showHistory && (
          <TranscriptHistory
            entries={entries}
            isVisible={showHistory}
            onClose={() => setShowHistory(false)}
            className="absolute top-0 right-0 h-full w-96 z-30"
          />
        )}

        {/* Summary Modal */}
        {showSummary && (
          <SummaryModal
            transcript={getSessionTranscript()}
            isOpen={showSummary}
            onClose={() => setShowSummary(false)}
          />
        )}

        {/* Error Display */}
        {error && (
          <div className="absolute top-4 left-4 right-4 z-40 bg-red-500 text-white p-3 rounded-lg">
            <p>Error: {error}</p>
          </div>
        )}
      </div>
    </MainLayout>
  );
}
```

## Environment Setup Instructions

### Prerequisites
1. Node.js 18+ and npm
2. Modern browser (Chrome recommended for best speech recognition)
3. Microphone access
4. OpenAI API key
5. (Optional) AssemblyAI API key for fallback

### Quick Start Commands
```bash
# Clone and setup
git clone <your-repo-url>
cd lecture-transcription
npm install

# Set up environment variables
cp .env.example .env.local
# Edit .env.local with your API keys

# Start development server
npm run dev

# Build for production
npm run build

# Preview production build
npm run preview
```

### Browser Permissions Setup
1. Allow microphone access when prompted
2. For best experience, use Chrome/Chromium browsers
3. Ensure stable internet connection for API calls

### API Keys Configuration
1. **OpenAI API Key**: Get from https://platform.openai.com/api-keys
2. **AssemblyAI API Key**: Get from https://www.assemblyai.com/ (optional fallback)
3. Add keys to `.env.local` file

### Performance Optimization Notes
- Video file should be compressed for web (H.264, <50MB recommended)
- Enable browser hardware acceleration
- Close unnecessary tabs for better performance
- Use wired internet for live transcription reliability

## Testing Checklist

### Functionality Tests
- [ ] Microphone permission handling
- [ ] Video playback and looping
- [ ] Speech recognition accuracy
- [ ] Text overlay positioning
- [ ] Session start/stop/pause
- [ ] Transcript export
- [ ] ChatGPT summarization
- [ ] Error handling and fallbacks

### Browser Compatibility
- [ ] Chrome/Chromium (primary)
- [ ] Firefox (with fallback)
- [ ] Safari (with fallback)
- [ ] Edge (with fallback)

### Performance Tests
- [ ] Long session memory usage
- [ ] Video playback stability
- [ ] API rate limiting
- [ ] Offline behavior

# Deployment Guide

## Deployment Options Overview

### Option 1: Vercel (Recommended - Easiest)
**Best for:** Quick deployment with zero configuration
**Cost:** Free tier available, paid plans from $20/month

#### Steps:
1. **Prepare for deployment:**
```bash
npm run build
```

2. **Install Vercel CLI:**
```bash
npm install -g vercel
```

3. **Deploy:**
```bash
vercel --prod
```

4. **Environment Variables Setup:**
   - Go to Vercel dashboard → Project → Settings → Environment Variables
   - Add: `VITE_OPENAI_API_KEY`
   - Add: `VITE_ASSEMBLYAI_API_KEY` (optional)

5. **Domain Configuration:**
   - Custom domain available on paid plans
   - Free subdomain: `your-app.vercel.app`

#### Vercel Configuration File
**vercel.json:**
```json
{
  "framework": "vite",
  "buildCommand": "npm run build",
  "outputDirectory": "dist",
  "installCommand": "npm install",
  "functions": {
    "app/api/**/*.ts": {
      "runtime": "nodejs18.x"
    }
  },
  "rewrites": [
    {
      "source": "/(.*)",
      "destination": "/index.html"
    }
  ]
}
```

### Option 2: Netlify
**Best for:** Static site hosting with form handling
**Cost:** Free tier available, paid plans from $19/month

#### Steps:
1. **Build the project:**
```bash
npm run build
```

2. **Install Netlify CLI:**
```bash
npm install -g netlify-cli
```

3. **Deploy:**
```bash
netlify deploy --prod --dir=dist
```

4. **Environment Variables:**
   - Netlify dashboard → Site settings → Environment variables
   - Add API keys with `VITE_` prefix

#### Netlify Configuration
**netlify.toml:**
```toml
[build]
  publish = "dist"
  command = "npm run build"

[[redirects]]
  from = "/*"
  to = "/index.html"
  status = 200

[build.environment]
  NODE_VERSION = "18"
```

### Option 3: GitHub Pages + GitHub Actions
**Best for:** Free hosting with GitHub integration
**Cost:** Free

#### Steps:
1. **Create GitHub Actions workflow:**
**.github/workflows/deploy.yml:**
```yaml
name: Deploy to GitHub Pages

on:
  push:
    branches: [ main ]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout
      uses: actions/checkout@v4

    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: '18'
        cache: 'npm'

    - name: Install dependencies
      run: npm ci

    - name: Build
      run: npm run build
      env:
        VITE_OPENAI_API_KEY: ${{ secrets.VITE_OPENAI_API_KEY }}
        VITE_ASSEMBLYAI_API_KEY: ${{ secrets.VITE_ASSEMBLYAI_API_KEY }}

    - name: Setup Pages
      uses: actions/configure-pages@v4

    - name: Upload artifact
      uses: actions/upload-pages-artifact@v3
      with:
        path: './dist'

    - name: Deploy to GitHub Pages
      id: deployment
      uses: actions/deploy-pages@v4
```

2. **Repository Settings:**
   - Go to Settings → Pages
   - Source: GitHub Actions
   - Add secrets in Settings → Secrets and variables → Actions

3. **Base URL Configuration:**
**vite.config.ts:**
```typescript
export default defineConfig({
  plugins: [react()],
  base: '/your-repository-name/', // Add this for GitHub Pages
})
```

### Option 4: Railway
**Best for:** Full-stack applications with database needs
**Cost:** $5/month minimum

#### Steps:
1. **Install Railway CLI:**
```bash
npm install -g @railway/cli
```

2. **Login and deploy:**
```bash
railway login
railway init
railway up
```

3. **Add environment variables:**
```bash
railway variables set VITE_OPENAI_API_KEY=your_key_here
```

### Option 5: DigitalOcean App Platform
**Best for:** Scalable production deployments
**Cost:** $12/month minimum

#### Steps:
1. **Create App Platform app** from DigitalOcean dashboard
2. **Connect GitHub repository**
3. **Configure build settings:**
   - Build Command: `npm run build`
   - Output Directory: `dist`
4. **Add environment variables** in app settings

## Security Considerations for Deployment

### API Key Security
⚠️ **Important:** Never expose API keys in client-side code in production!

#### Recommended Architecture:
```
Frontend (Vite) → Your Backend API → OpenAI/AssemblyAI
```

#### Backend Proxy Setup (Node.js/Express):
```typescript
// backend/src/routes/summary.ts
import express from 'express';
import OpenAI from 'openai';

const router = express.Router();
const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

router.post('/summarize', async (req, res) => {
  try {
    const { transcript } = req.body;
    
    const response = await openai.chat.completions.create({
      model: 'gpt-3.5-turbo',
      messages: [
        { role: 'system', content: 'Summarize this lecture transcript.' },
        { role: 'user', content: transcript }
      ]
    });

    res.json({ summary: response.choices[0].message.content });
  } catch (error) {
    res.status(500).json({ error: 'Failed to generate summary' });
  }
});
```

### Environment-Specific Configurations

#### Development (.env.local):
```bash
VITE_API_BASE_URL=http://localhost:3001
VITE_OPENAI_API_KEY=your_dev_key
```

#### Production (.env.production):
```bash
VITE_API_BASE_URL=https://your-api.herokuapp.com
# Remove direct API keys in production!
```

## Performance Optimization for Production

### Build Optimization
**vite.config.ts:**
```typescript
export default defineConfig({
  plugins: [react()],
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ['react', 'react-dom'],
          transcription: ['zustand', 'framer-motion'],
        },
      },
    },
    chunkSizeWarningLimit: 1000,
  },
  optimizeDeps: {
    include: ['react', 'react-dom'],
  },
})
```

### Asset Optimization
1. **Compress videos:** Use H.264, target <10MB for web
2. **Image optimization:** Use WebP format
3. **Bundle analysis:** Run `npm run build -- --analyze`

### CDN Setup (Optional)
```typescript
// Use CDN for static assets
const videoSrc = process.env.NODE_ENV === 'production' 
  ? 'https://cdn.yoursite.com/minecraft-parkour.mp4'
  : '/src/assets/videos/minecraft-parkour.mp4';
```

## Monitoring & Analytics

### Error Tracking (Sentry)
```bash
npm install @sentry/react @sentry/vite-plugin
```

**src/main.tsx:**
```typescript
import * as Sentry from "@sentry/react";

Sentry.init({
  dsn: "YOUR_SENTRY_DSN",
  environment: process.env.NODE_ENV,
});
```

### Analytics (Optional)
```typescript
// Google Analytics 4
const GA_TRACKING_ID = 'G-XXXXXXXXXX';

gtag('config', GA_TRACKING_ID, {
  page_title: 'Lecture Transcription',
  page_location: window.location.href,
});
```

## Deployment Checklist

### Pre-Deployment
- [ ] Environment variables configured
- [ ] Build succeeds locally (`npm run build`)
- [ ] API keys secured (not in client code for production)
- [ ] Video assets optimized and hosted
- [ ] Error boundaries implemented
- [ ] HTTPS enabled on hosting platform

### Post-Deployment
- [ ] Test microphone permissions on deployed site
- [ ] Verify API integrations work
- [ ] Check video playback on different devices
- [ ] Test export functionality
- [ ] Monitor error logs
- [ ] Set up uptime monitoring

### Domain Setup (Optional)
1. **Purchase domain** (Namecheap, GoDaddy, etc.)
2. **Configure DNS** in hosting platform
3. **Enable HTTPS** (usually automatic)
4. **Update CORS settings** if using separate backend

## Cost Estimates

### Hosting Costs (Monthly):
- **Vercel Free:** $0 (100GB bandwidth)
- **Netlify Free:** $0 (100GB bandwidth)  
- **GitHub Pages:** $0 (1GB storage)
- **Railway:** $5+ (500 hours runtime)
- **DigitalOcean:** $12+ (1GB RAM)

### API Costs (per 1000 requests):
- **OpenAI GPT-3.5:** ~$0.002
- **AssemblyAI:** ~$0.65 per audio hour
- **Total estimated:** $10-50/month depending on usage

## Recommended Production Stack
For a production deployment, I recommend:

1. **Frontend:** Vercel (easy deployment, great performance)
2. **Backend API:** Railway or Vercel Functions (for API key security)
3. **Assets:** Vercel CDN or separate CDN service
4. **Monitoring:** Sentry for error tracking
5. **Domain:** Custom domain for professional appearance

This setup provides excellent performance, security, and scalability while remaining cost-effective.

---

This detailed plan provides everything needed to build and deploy the live lecture transcription app. Each component includes specific implementation details, type definitions, configuration examples, and multiple deployment options to suit different needs and budgets.
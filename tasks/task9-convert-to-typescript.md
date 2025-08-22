# Task 9: Convert to TypeScript

## Priority: Low 📝

## Description
Convert the JavaScript codebase to TypeScript for better type safety, improved developer experience, and enhanced code maintainability.

## Current Issue
The application is written in vanilla JavaScript without type checking, making it prone to runtime type errors and harder to maintain as it grows.

## Solution
Convert to TypeScript with proper type definitions and interfaces:

### Type Definitions
```typescript
// types/index.ts
export interface AppState {
    recognition: SpeechRecognition | null;
    isListening: boolean;
    isSupported: boolean;
    transcript: string;
    interimTranscript: string;
    characterCount: number;
    language: LanguageCode;
    autoScroll: boolean;
    autoSave: boolean;
    isVisible: boolean;
    notifications: Notification[];
    currentError: SpeechRecognitionError | null;
    lastUpdateTime: number | null;
    updateCount: number;
}

export type LanguageCode = 
    | 'en-US' | 'en-GB' | 'es-ES' | 'fr-FR' | 'de-DE' 
    | 'it-IT' | 'pt-BR' | 'ru-RU' | 'ja-JP' | 'ko-KR' 
    | 'zh-CN' | 'ar-SA' | 'hi-IN';

export interface Language {
    code: LanguageCode;
    name: string;
    region?: string;
}

export interface SpeechRecognitionError {
    type: ErrorType;
    message: string;
    code?: string;
    retryable: boolean;
}

export type ErrorType = 
    | 'network' 
    | 'not-allowed' 
    | 'no-speech' 
    | 'aborted' 
    | 'audio-capture' 
    | 'service-not-allowed'
    | 'bad-grammar'
    | 'language-not-supported';

export interface RecognitionConfig {
    continuous: boolean;
    interimResults: boolean;
    maxAlternatives: number;
    lang: LanguageCode;
}

export interface TranscriptResult {
    transcript: string;
    confidence: number;
    isFinal: boolean;
    timestamp: number;
}

export interface StateChangeEvent {
    timestamp: number;
    source: string;
    changes: Record<string, StateChange>;
    prevState: Partial<AppState>;
    newState: Partial<AppState>;
}

export interface StateChange {
    from: any;
    to: any;
}
```

### Main Application Class
```typescript
// src/VoiceToTextApp.ts
import { AppState, LanguageCode, SpeechRecognitionError, RecognitionConfig } from '../types';
import { SpeechRecognitionService } from './services/SpeechRecognitionService';
import { TranscriptManager } from './services/TranscriptManager';
import { UIController } from './ui/UIController';
import { AppStateManager } from './state/AppStateManager';

export class VoiceToTextApp {
    private stateManager: AppStateManager;
    private speechService: SpeechRecognitionService;
    private transcriptManager: TranscriptManager;
    private uiController: UIController;

    constructor() {
        this.stateManager = new AppStateManager();
        this.speechService = new SpeechRecognitionService(this.getRecognitionConfig());
        this.transcriptManager = new TranscriptManager();
        this.uiController = new UIController(this.stateManager);
        
        this.initialize();
    }

    private initialize(): void {
        this.checkBrowserSupport();
        this.setupEventListeners();
        this.initializeUI();
    }

    private getRecognitionConfig(): RecognitionConfig {
        return {
            continuous: true,
            interimResults: true,
            maxAlternatives: 1,
            lang: 'en-US'
        };
    }

    public async startListening(): Promise<void> {
        try {
            this.stateManager.setState({ isListening: true, currentError: null });
            await this.speechService.start();
        } catch (error) {
            this.handleError(error as Error);
        }
    }

    public stopListening(): void {
        this.speechService.stop();
        this.stateManager.setState({ isListening: false });
    }

    public changeLanguage(language: LanguageCode): void {
        this.speechService.setLanguage(language);
        this.stateManager.setState({ language });
    }

    public clearTranscript(): void {
        this.transcriptManager.clear();
        this.stateManager.setState({ 
            transcript: '', 
            interimTranscript: '',
            characterCount: 0
        });
    }

    private handleError(error: Error): void {
        const speechError: SpeechRecognitionError = {
            type: this.mapErrorType(error.name),
            message: error.message,
            retryable: this.isRetryableError(error.name)
        };

        this.stateManager.setState({ 
            currentError: speechError,
            isListening: false
        });
    }

    private mapErrorType(errorName: string): ErrorType {
        const errorMap: Record<string, ErrorType> = {
            'NetworkError': 'network',
            'NotAllowedError': 'not-allowed',
            'NoSpeechError': 'no-speech',
            'AbortError': 'aborted',
            'AudioCaptureError': 'audio-capture',
            'ServiceNotAllowedError': 'service-not-allowed'
        };

        return errorMap[errorName] || 'network';
    }

    private isRetryableError(errorName: string): boolean {
        const retryableErrors = ['NetworkError', 'AbortError', 'AudioCaptureError'];
        return retryableErrors.includes(errorName);
    }
}
```

### Service Classes
```typescript
// src/services/SpeechRecognitionService.ts
export class SpeechRecognitionService {
    private recognition: SpeechRecognition | null = null;
    private config: RecognitionConfig;
    private eventHandlers: Map<string, Function[]> = new Map();

    constructor(config: RecognitionConfig) {
        this.config = config;
        this.initialize();
    }

    private initialize(): void {
        const SpeechRecognition = window.SpeechRecognition || window.webkitSpeechRecognition;
        
        if (!SpeechRecognition) {
            throw new Error('Speech recognition not supported');
        }

        this.recognition = new SpeechRecognition();
        this.setupRecognition();
    }

    private setupRecognition(): void {
        if (!this.recognition) return;

        this.recognition.continuous = this.config.continuous;
        this.recognition.interimResults = this.config.interimResults;
        this.recognition.maxAlternatives = this.config.maxAlternatives;
        this.recognition.lang = this.config.lang;

        this.recognition.onresult = this.handleResult.bind(this);
        this.recognition.onerror = this.handleError.bind(this);
        this.recognition.onend = this.handleEnd.bind(this);
    }

    public async start(): Promise<void> {
        if (!this.recognition) {
            throw new Error('Speech recognition not initialized');
        }

        return new Promise((resolve, reject) => {
            this.recognition!.onstart = () => resolve();
            this.recognition!.onerror = (event) => reject(new Error(event.error));
            this.recognition!.start();
        });
    }

    public stop(): void {
        this.recognition?.stop();
    }

    public setLanguage(language: LanguageCode): void {
        this.config.lang = language;
        if (this.recognition) {
            this.recognition.lang = language;
        }
    }

    public addEventListener(event: string, handler: Function): void {
        if (!this.eventHandlers.has(event)) {
            this.eventHandlers.set(event, []);
        }
        this.eventHandlers.get(event)!.push(handler);
    }

    private handleResult(event: SpeechRecognitionEvent): void {
        const results: TranscriptResult[] = [];
        
        for (let i = event.resultIndex; i < event.results.length; i++) {
            const result = event.results[i];
            results.push({
                transcript: result[0].transcript,
                confidence: result[0].confidence,
                isFinal: result.isFinal,
                timestamp: Date.now()
            });
        }

        this.emit('result', results);
    }

    private handleError(event: SpeechRecognitionErrorEvent): void {
        this.emit('error', new Error(event.error));
    }

    private handleEnd(): void {
        this.emit('end');
    }

    private emit(event: string, data?: any): void {
        const handlers = this.eventHandlers.get(event);
        if (handlers) {
            handlers.forEach(handler => handler(data));
        }
    }
}
```

### State Manager
```typescript
// src/state/AppStateManager.ts
export class AppStateManager {
    private state: AppState;
    private listeners: Map<number, StateListener> = new Map();
    private history: StateChangeEvent[] = [];
    private readonly maxHistorySize = 50;

    constructor(initialState?: Partial<AppState>) {
        this.state = this.createInitialState(initialState);
    }

    private createInitialState(partial?: Partial<AppState>): AppState {
        return {
            recognition: null,
            isListening: false,
            isSupported: false,
            transcript: '',
            interimTranscript: '',
            characterCount: 0,
            language: 'en-US',
            autoScroll: true,
            autoSave: false,
            isVisible: true,
            notifications: [],
            currentError: null,
            lastUpdateTime: null,
            updateCount: 0,
            ...partial
        };
    }

    public getState(): Readonly<AppState> {
        return Object.freeze({ ...this.state });
    }

    public setState(updates: Partial<AppState>, source = 'unknown'): void {
        const previousState = { ...this.state };
        const newState = { ...this.state, ...updates };

        if (this.validateStateChange(previousState, newState)) {
            this.state = newState;
            this.addToHistory(previousState, newState, source);
            this.notifyListeners(previousState, newState, updates);
        } else {
            console.warn('Invalid state change rejected:', updates);
        }
    }

    public subscribe(
        callback: StateChangeCallback, 
        filter?: string[] | string
    ): UnsubscribeFunction {
        const id = Date.now() + Math.random();
        this.listeners.set(id, { callback, filter });
        return () => this.listeners.delete(id);
    }

    private validateStateChange(
        prevState: AppState, 
        newState: AppState
    ): boolean {
        // Type checking and validation logic
        if (newState.transcript.length > 50000) return false;
        if (!/^[a-z]{2}-[A-Z]{2}$/.test(newState.language)) return false;
        
        return true;
    }

    private notifyListeners(
        prevState: AppState, 
        newState: AppState, 
        updates: Partial<AppState>
    ): void {
        this.listeners.forEach(({ callback, filter }) => {
            if (!filter || this.matchesFilter(updates, filter)) {
                try {
                    callback(newState, prevState, updates);
                } catch (error) {
                    console.error('State listener error:', error);
                }
            }
        });
    }
}

interface StateListener {
    callback: StateChangeCallback;
    filter?: string[] | string;
}

type StateChangeCallback = (
    newState: AppState, 
    prevState: AppState, 
    updates: Partial<AppState>
) => void;

type UnsubscribeFunction = () => void;
```

## Build Configuration
```json
// tsconfig.json
{
    "compilerOptions": {
        "target": "ES2020",
        "module": "ESNext",
        "moduleResolution": "node",
        "lib": ["ES2020", "DOM", "DOM.Iterable"],
        "outDir": "./dist",
        "rootDir": "./src",
        "strict": true,
        "esModuleInterop": true,
        "skipLibCheck": true,
        "forceConsistentCasingInFileNames": true,
        "declaration": true,
        "declarationMap": true,
        "sourceMap": true,
        "removeComments": false,
        "noImplicitAny": true,
        "noImplicitReturns": true,
        "noImplicitThis": true,
        "noUnusedLocals": true,
        "noUnusedParameters": true
    },
    "include": ["src/**/*", "types/**/*"],
    "exclude": ["node_modules", "dist"]
}
```

## Acceptance Criteria
- [ ] All JavaScript converted to TypeScript
- [ ] Comprehensive type definitions
- [ ] No TypeScript compilation errors
- [ ] Maintains all existing functionality
- [ ] Improved IDE support with autocomplete
- [ ] Type safety for all function parameters
- [ ] Interface definitions for all data structures
- [ ] Build process generates declaration files

## Files to Create/Modify
- Convert `index.html` JavaScript to separate TypeScript files
- Create type definition files
- Add `tsconfig.json` configuration
- Update build process to compile TypeScript
- Add type definitions for Web Speech API if needed

## Benefits
- Better IDE support and autocomplete
- Compile-time error catching
- Improved code documentation
- Enhanced refactoring capabilities
- Better team collaboration with explicit contracts

## Testing
1. Verify TypeScript compilation succeeds
2. Test all functionality works identically
3. Check IDE autocomplete and error detection
4. Validate type safety prevents runtime errors
5. Test build process and output

## Estimated Time
4-5 hours
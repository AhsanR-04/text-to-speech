# Task 8: Add Accessibility Improvements

## Priority: Medium 📋

## Description
Enhance the application's accessibility by adding ARIA labels, keyboard navigation, screen reader support, and improved usability for users with disabilities.

## Current Issues
- Missing ARIA labels and roles
- No keyboard navigation support
- Limited screen reader compatibility
- No high contrast mode
- Missing focus indicators

## Solution
Implement comprehensive accessibility features:

### ARIA Labels and Roles
```html
<!-- Enhanced HTML structure -->
<main role="main" aria-label="Voice to Text Application">
    <header>
        <h1 id="app-title">Voice to Text Converter</h1>
        <p id="app-description">Convert your speech to text using your browser's speech recognition</p>
    </header>
    
    <section aria-labelledby="controls-heading">
        <h2 id="controls-heading" class="sr-only">Speech Recognition Controls</h2>
        
        <div class="language-section">
            <label for="language-select" id="language-label">
                Select Language:
            </label>
            <select 
                id="language-select" 
                aria-labelledby="language-label"
                aria-describedby="language-help"
            >
                <option value="en-US">English (US)</option>
                <!-- ... other options -->
            </select>
            <div id="language-help" class="sr-only">
                Choose the language you will be speaking in
            </div>
        </div>
        
        <div class="control-buttons" role="group" aria-labelledby="button-group-label">
            <span id="button-group-label" class="sr-only">Recording Controls</span>
            <button 
                id="start-btn"
                aria-describedby="start-help"
                aria-pressed="false"
            >
                <span class="button-text">Start Listening</span>
                <span class="sr-only" id="start-help">Begin speech recognition</span>
            </button>
            
            <button 
                id="clear-btn"
                aria-describedby="clear-help"
            >
                <span class="button-text">Clear</span>
                <span class="sr-only" id="clear-help">Clear all transcribed text</span>
            </button>
        </div>
    </section>
    
    <section aria-labelledby="transcript-heading">
        <h2 id="transcript-heading" class="sr-only">Transcript</h2>
        <label for="transcript" class="sr-only">
            Your transcribed text will appear here
        </label>
        <textarea 
            id="transcript"
            aria-labelledby="transcript-heading"
            aria-describedby="char-count transcript-help"
            aria-live="polite"
            aria-atomic="false"
        ></textarea>
        
        <div id="transcript-help" class="sr-only">
            Text area containing your speech-to-text results. You can edit this text.
        </div>
        
        <div class="transcript-info">
            <span id="char-count" aria-live="polite">0 characters</span>
            <span id="status" aria-live="assertive" aria-atomic="true"></span>
        </div>
    </section>
</main>
```

### Keyboard Navigation
```javascript
class AccessibilityManager {
    constructor(app) {
        this.app = app;
        this.focusableElements = [];
        this.currentFocusIndex = -1;
        this.setupKeyboardNavigation();
        this.setupScreenReader();
    }
    
    setupKeyboardNavigation() {
        // Define keyboard shortcuts
        document.addEventListener('keydown', (event) => {
            if (this.handleGlobalShortcuts(event)) {
                event.preventDefault();
                return;
            }
            
            if (this.handleModalShortcuts(event)) {
                event.preventDefault();
                return;
            }
        });
        
        // Setup focus management
        this.updateFocusableElements();
        this.setupFocusTrap();
    }
    
    handleGlobalShortcuts(event) {
        const { key, ctrlKey, shiftKey, altKey } = event;
        
        // Ctrl+Space: Toggle listening
        if (ctrlKey && key === ' ') {
            this.app.toggleListening();
            this.announceToScreenReader(
                this.app.stateManager.getState().isListening ? 
                'Started listening' : 'Stopped listening'
            );
            return true;
        }
        
        // Ctrl+Shift+C: Clear transcript
        if (ctrlKey && shiftKey && key === 'C') {
            this.app.clearTranscript();
            this.announceToScreenReader('Transcript cleared');
            return true;
        }
        
        // Ctrl+Shift+D: Download transcript
        if (ctrlKey && shiftKey && key === 'D') {
            this.app.downloadText();
            this.announceToScreenReader('Transcript downloaded');
            return true;
        }
        
        // Alt+L: Focus language selector
        if (altKey && key === 'l') {
            document.getElementById('language-select').focus();
            return true;
        }
        
        // Alt+T: Focus transcript
        if (altKey && key === 't') {
            document.getElementById('transcript').focus();
            return true;
        }
        
        return false;
    }
    
    setupScreenReader() {
        // Create live region for announcements
        this.liveRegion = document.createElement('div');
        this.liveRegion.setAttribute('aria-live', 'polite');
        this.liveRegion.setAttribute('aria-atomic', 'true');
        this.liveRegion.className = 'sr-only';
        this.liveRegion.id = 'announcements';
        document.body.appendChild(this.liveRegion);
        
        // Setup state change announcements
        this.app.stateManager.subscribe((state, prevState) => {
            this.handleStateAnnouncements(state, prevState);
        });
    }
    
    announceToScreenReader(message, priority = 'polite') {
        this.liveRegion.setAttribute('aria-live', priority);
        this.liveRegion.textContent = message;
        
        // Clear after announcement
        setTimeout(() => {
            this.liveRegion.textContent = '';
        }, 1000);
    }
    
    handleStateAnnouncements(state, prevState) {
        if (state.isListening !== prevState.isListening) {
            const message = state.isListening ? 
                'Speech recognition started. Begin speaking.' : 
                'Speech recognition stopped.';
            this.announceToScreenReader(message);
        }
        
        if (state.currentError !== prevState.currentError && state.currentError) {
            this.announceToScreenReader(`Error: ${state.currentError}`, 'assertive');
        }
        
        // Announce transcript updates (debounced)
        if (state.transcript !== prevState.transcript) {
            this.debouncedTranscriptAnnouncement();
        }
    }
    
    debouncedTranscriptAnnouncement = this.debounce(() => {
        const transcript = this.app.stateManager.getState().transcript;
        const wordCount = transcript.split(/\s+/).filter(word => word.length > 0).length;
        this.announceToScreenReader(`Transcript updated. ${wordCount} words captured.`);
    }, 2000);
    
    // Enhanced focus management
    setupFocusTrap() {
        const modal = document.querySelector('[role="dialog"]');
        if (modal) {
            const focusableElements = modal.querySelectorAll(
                'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
            );
            
            if (focusableElements.length) {
                focusableElements[0].focus();
                
                modal.addEventListener('keydown', (event) => {
                    if (event.key === 'Tab') {
                        this.handleTabTrapping(event, focusableElements);
                    }
                });
            }
        }
    }
    
    handleTabTrapping(event, focusableElements) {
        const firstElement = focusableElements[0];
        const lastElement = focusableElements[focusableElements.length - 1];
        
        if (event.shiftKey) {
            if (document.activeElement === firstElement) {
                lastElement.focus();
                event.preventDefault();
            }
        } else {
            if (document.activeElement === lastElement) {
                firstElement.focus();
                event.preventDefault();
            }
        }
    }
}
```

### CSS Accessibility Enhancements
```css
/* Screen reader only content */
.sr-only {
    position: absolute;
    width: 1px;
    height: 1px;
    padding: 0;
    margin: -1px;
    overflow: hidden;
    clip: rect(0, 0, 0, 0);
    white-space: nowrap;
    border: 0;
}

/* High contrast mode support */
@media (prefers-contrast: high) {
    .btn {
        border: 2px solid currentColor;
    }
    
    .transcript {
        border: 2px solid var(--primary-color);
    }
}

/* Reduced motion support */
@media (prefers-reduced-motion: reduce) {
    * {
        animation-duration: 0.01ms !important;
        animation-iteration-count: 1 !important;
        transition-duration: 0.01ms !important;
    }
}

/* Focus indicators */
button:focus,
select:focus,
textarea:focus {
    outline: 3px solid var(--focus-color);
    outline-offset: 2px;
}

/* Color contrast improvements */
:root {
    --focus-color: #0066cc;
    --error-color: #d73027;
    --success-color: #1a9641;
    --text-contrast-ratio: 4.5; /* WCAG AA compliance */
}
```

## Acceptance Criteria
- [ ] All interactive elements have ARIA labels
- [ ] Keyboard navigation works for all functions
- [ ] Screen reader announces state changes
- [ ] Focus management works correctly
- [ ] High contrast mode supported
- [ ] Reduced motion preferences respected
- [ ] WCAG 2.1 AA compliance achieved
- [ ] Keyboard shortcuts documented

## Files to Modify
- `index.html` (add ARIA attributes, semantic structure)
- Add AccessibilityManager class
- Update CSS for accessibility features
- Add keyboard shortcut documentation

## Testing
1. Test with screen readers (NVDA, JAWS, VoiceOver)
2. Navigate entire app using only keyboard
3. Test with high contrast mode
4. Verify focus indicators are visible
5. Test color contrast ratios
6. Validate ARIA implementation
7. Test reduced motion preferences

## Estimated Time
2-3 hours
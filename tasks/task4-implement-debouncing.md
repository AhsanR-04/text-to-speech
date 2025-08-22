# Task 4: Implement Debouncing

## Priority: High ⚠️

## Description
Implement debouncing for transcript updates to reduce excessive DOM updates and improve performance, especially during continuous speech recognition.

## Current Issue
```javascript
// Current - Updates on every result
updateTranscript() {
    const fullText = this.transcript + this.interimTranscript;
    this.transcriptElement.value = fullText;
}
```

The current implementation updates the DOM on every interim result, causing performance issues.

## Solution
Implement debounced transcript updates:

```javascript
// Add debounce utility function
debounce(func, wait) {
    let timeout;
    return function executedFunction(...args) {
        const later = () => {
            clearTimeout(timeout);
            func(...args);
        };
        clearTimeout(timeout);
        timeout = setTimeout(later, wait);
    };
}

// Initialize debounced functions in constructor
constructor() {
    // ... existing code
    this.debouncedUpdateTranscript = this.debounce(this.updateTranscriptDOM.bind(this), 100);
    this.debouncedScrollToBottom = this.debounce(this.scrollToBottom.bind(this), 150);
}

// Separate DOM update logic
updateTranscriptDOM() {
    const fullText = this.transcript + this.interimTranscript;
    if (this.transcriptElement.value !== fullText) {
        this.transcriptElement.value = fullText;
        this.debouncedScrollToBottom();
    }
}

// Update the main update method
updateTranscript() {
    this.debouncedUpdateTranscript();
}

scrollToBottom() {
    this.transcriptElement.scrollTop = this.transcriptElement.scrollHeight;
}
```

## Additional Optimizations
```javascript
// Optimize character counting
updateCharacterCount = this.debounce(() => {
    const count = (this.transcript + this.interimTranscript).length;
    if (this.characterCountElement) {
        this.characterCountElement.textContent = `${count} characters`;
    }
}, 200);

// Batch multiple UI updates
batchUIUpdates() {
    // Use requestAnimationFrame for smooth updates
    if (!this.uiUpdateScheduled) {
        this.uiUpdateScheduled = true;
        requestAnimationFrame(() => {
            this.updateTranscriptDOM();
            this.updateCharacterCount();
            this.uiUpdateScheduled = false;
        });
    }
}
```

## Acceptance Criteria
- [ ] DOM updates are debounced with appropriate delays
- [ ] Performance improves during continuous speech
- [ ] User experience remains smooth and responsive
- [ ] No visual lag or stuttering
- [ ] Character counting is optimized
- [ ] Scroll behavior is smooth

## Files to Modify
- `index.html` (add debounce functionality to VoiceToTextApp class)

## Performance Targets
- Reduce DOM updates by 70-80% during continuous speech
- Maintain sub-200ms perceived responsiveness
- Smooth scrolling without stuttering

## Testing
1. Continuous speech for 2+ minutes
2. Monitor DOM update frequency in DevTools
3. Test on slower devices/browsers
4. Verify no visual delays or stuttering
5. Test character counting performance
6. Measure before/after performance metrics

## Estimated Time
1 hour
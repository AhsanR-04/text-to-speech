# Task 3: Input Validation and Sanitization

## Priority: High ⚠️

## Description
Add input validation and sanitization for transcript content to prevent potential security issues and ensure data integrity.

## Current Issue
```javascript
// Current implementation trusts all speech input
this.transcript += finalTranscript;
```

The application currently accepts all speech recognition results without any validation or sanitization.

## Solution
Implement input validation and sanitization:

```javascript
sanitizeTranscript(text) {
    // Remove potentially harmful characters
    const sanitized = text
        .replace(/<script\b[^<]*(?:(?!<\/script>)<[^<]*)*<\/script>/gi, '')
        .replace(/<[^>]*>/g, '')
        .replace(/javascript:/gi, '')
        .replace(/on\w+\s*=/gi, '');
    
    return sanitized.trim();
}

validateTranscript(text) {
    // Check for reasonable length limits
    const MAX_TRANSCRIPT_LENGTH = 50000; // 50KB limit
    const MAX_SINGLE_ADDITION = 5000;    // 5KB per addition
    
    if (this.transcript.length + text.length > MAX_TRANSCRIPT_LENGTH) {
        this.showNotification('Transcript is too long. Please download and clear to continue.', 'warning');
        return false;
    }
    
    if (text.length > MAX_SINGLE_ADDITION) {
        this.showNotification('Single speech input too long. Please speak in shorter segments.', 'warning');
        return false;
    }
    
    return true;
}

updateTranscriptSafely(finalTranscript) {
    const sanitizedText = this.sanitizeTranscript(finalTranscript);
    
    if (this.validateTranscript(sanitizedText)) {
        this.transcript += sanitizedText;
        this.updateTranscript();
    }
}
```

## Acceptance Criteria
- [ ] HTML tags are stripped from transcript content
- [ ] JavaScript injection attempts are prevented
- [ ] Transcript length limits implemented
- [ ] User warnings for validation failures
- [ ] Sanitization doesn't affect normal speech content
- [ ] Performance impact is minimal

## Files to Modify
- `index.html` (add sanitization methods to VoiceToTextApp class)

## Security Considerations
- Prevent XSS through transcript display
- Limit memory usage through length validation
- Maintain speech recognition accuracy

## Testing
1. Test with normal speech content
2. Try edge cases:
   - Very long speeches
   - Speaking HTML-like content
   - Special characters and symbols
3. Verify sanitization doesn't break legitimate content
4. Performance test with long transcripts

## Estimated Time
45 minutes
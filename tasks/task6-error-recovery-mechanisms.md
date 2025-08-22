# Task 6: Add Comprehensive Error Recovery

## Priority: Medium 📋

## Description
Implement comprehensive error recovery mechanisms that automatically handle common speech recognition failures and provide users with recovery options.

## Current Issue
When errors occur, the application simply stops and shows an error message without attempting recovery or providing actionable solutions.

## Solution
Implement intelligent error recovery with automatic retry and user guidance:

```javascript
class ErrorRecoveryManager {
    constructor(app) {
        this.app = app;
        this.retryAttempts = new Map();
        this.maxRetries = {
            'network': 3,
            'audio-capture': 2,
            'no-speech': 1,
            'aborted': 2
        };
    }
    
    async handleError(errorType, originalError) {
        const attempts = this.retryAttempts.get(errorType) || 0;
        const maxRetries = this.maxRetries[errorType] || 1;
        
        if (attempts < maxRetries) {
            return await this.attemptRecovery(errorType, attempts + 1);
        } else {
            return this.showRecoveryOptions(errorType, originalError);
        }
    }
    
    async attemptRecovery(errorType, attempt) {
        this.retryAttempts.set(errorType, attempt);
        
        switch (errorType) {
            case 'network':
                return this.handleNetworkRecovery(attempt);
            case 'audio-capture':
                return this.handleAudioRecovery(attempt);
            case 'not-allowed':
                return this.handlePermissionRecovery();
            case 'no-speech':
                return this.handleNoSpeechRecovery();
            case 'aborted':
                return this.handleAbortedRecovery(attempt);
            default:
                return false;
        }
    }
    
    async handleNetworkRecovery(attempt) {
        this.app.showNotification(`Network error. Retrying... (${attempt}/3)`, 'warning');
        
        // Wait before retry with exponential backoff
        await this.sleep(1000 * Math.pow(2, attempt - 1));
        
        // Test network connectivity
        if (await this.testConnectivity()) {
            return this.app.startListening();
        }
        return false;
    }
    
    async handleAudioRecovery(attempt) {
        this.app.showNotification(`Microphone issue. Attempting to reconnect... (${attempt}/2)`, 'warning');
        
        // Re-initialize audio
        try {
            await navigator.mediaDevices.getUserMedia({ audio: true });
            return this.app.startListening();
        } catch (error) {
            return false;
        }
    }
    
    showRecoveryOptions(errorType, originalError) {
        const recoveryUI = this.createRecoveryUI(errorType);
        this.app.showRecoveryDialog(recoveryUI);
    }
}
```

### Recovery UI Components
```javascript
createRecoveryUI(errorType) {
    const recoveryOptions = {
        'network': [
            { text: 'Check Connection', action: () => this.testConnectivity() },
            { text: 'Retry Now', action: () => this.app.startListening() },
            { text: 'Work Offline', action: () => this.enableOfflineMode() }
        ],
        'not-allowed': [
            { text: 'Grant Permission', action: () => this.requestPermission() },
            { text: 'Setup Guide', action: () => this.showPermissionGuide() }
        ],
        'audio-capture': [
            { text: 'Test Microphone', action: () => this.testMicrophone() },
            { text: 'Select Different Mic', action: () => this.showDeviceSelector() },
            { text: 'Troubleshoot', action: () => this.showAudioTroubleshooting() }
        ]
    };
    
    return recoveryOptions[errorType] || [];
}
```

## Acceptance Criteria
- [ ] Automatic retry for recoverable errors
- [ ] Exponential backoff for network errors
- [ ] User-friendly recovery options
- [ ] Audio device re-initialization
- [ ] Permission re-request handling
- [ ] Network connectivity testing
- [ ] Recovery success/failure tracking
- [ ] User guidance for manual fixes

## Files to Modify
- `index.html` (add ErrorRecoveryManager class)
- Update existing error handling to use recovery manager
- Add recovery UI components

## Recovery Scenarios
1. **Network Errors**: Auto-retry with backoff, connectivity test
2. **Permission Denied**: Guide user to grant permissions
3. **Audio Capture**: Re-initialize audio, device selection
4. **No Speech**: Prompt user, adjust sensitivity
5. **Service Unavailable**: Fallback options, offline mode

## Testing
1. Simulate various error conditions
2. Test automatic recovery success rates
3. Verify user guidance effectiveness
4. Test recovery UI interactions
5. Measure time to recovery
6. Test multiple sequential errors

## Estimated Time
2-3 hours